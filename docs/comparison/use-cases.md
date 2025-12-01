---
layout: default
title: Use Cases
parent: Comparison
nav_order: 4
permalink: /docs/comparison/use-cases/
---

# Real-World Use Cases
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Discover how organizations use Azure Durable to solve complex business problems.
{: .fs-6 .fw-300 }

---

## E-Commerce: Order Processing

### The Challenge
Online retailers need to process orders reliably, coordinating inventory checks, payments, shipping, and notifications—all while handling failures gracefully.

### The Solution

```csharp
[Function(nameof(ProcessOrder))]
public async Task<OrderResult> ProcessOrder(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var order = context.GetInput<Order>();
    
    try
    {
        // Step 1: Reserve inventory
        var inventoryReserved = await context.CallActivityAsync<bool>(
            nameof(ReserveInventory), order);
        
        if (!inventoryReserved)
        {
            await context.CallActivityAsync(nameof(NotifyOutOfStock), order);
            return new OrderResult { Status = "OutOfStock" };
        }
        
        // Step 2: Process payment (with retry)
        var paymentResult = await context.CallActivityAsync<PaymentResult>(
            nameof(ProcessPayment), 
            order,
            new TaskOptions { Retry = new RetryPolicy(3, TimeSpan.FromSeconds(5)) });
        
        if (!paymentResult.Success)
        {
            await context.CallActivityAsync(nameof(ReleaseInventory), order);
            return new OrderResult { Status = "PaymentFailed" };
        }
        
        // Step 3: Initiate shipping
        var trackingNumber = await context.CallActivityAsync<string>(
            nameof(CreateShipment), order);
        
        // Step 4: Send confirmation
        await context.CallActivityAsync(nameof(SendConfirmation), new {
            order.Email,
            order.OrderId,
            trackingNumber
        });
        
        return new OrderResult { 
            Status = "Completed", 
            TrackingNumber = trackingNumber 
        };
    }
    catch (TaskFailedException)
    {
        // Compensate on failure
        await context.CallActivityAsync(nameof(ReleaseInventory), order);
        await context.CallActivityAsync(nameof(RefundPayment), order);
        throw;
    }
}
```

### Benefits
- **Reliability**: Orders complete even if services temporarily fail
- **Consistency**: Automatic compensation prevents orphaned reservations
- **Visibility**: Dashboard shows order status and processing history
- **Scalability**: Handles Black Friday traffic spikes automatically

---

## Healthcare: Patient Onboarding

### The Challenge
Healthcare providers must onboard patients through a multi-step process involving identity verification, insurance validation, consent collection, and appointment scheduling—often requiring human approval steps.

### The Solution

```csharp
[Function(nameof(PatientOnboarding))]
public async Task<OnboardingResult> PatientOnboarding(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var patient = context.GetInput<PatientInfo>();
    
    // Step 1: Verify identity
    var identityVerified = await context.CallActivityAsync<bool>(
        nameof(VerifyIdentity), patient);
    
    if (!identityVerified)
    {
        // Request manual review
        await context.CallActivityAsync(nameof(RequestManualReview), patient);
        
        // Wait for human approval (up to 48 hours)
        var approval = await context.WaitForExternalEvent<ApprovalResult>(
            "ManualApproval",
            TimeSpan.FromHours(48));
        
        if (!approval.Approved)
        {
            return new OnboardingResult { Status = "Rejected" };
        }
    }
    
    // Step 2: Validate insurance (fan-out to multiple providers)
    var insuranceTask = context.CallActivityAsync<InsuranceResult>(
        nameof(ValidateInsurance), patient);
    var eligibilityTask = context.CallActivityAsync<EligibilityResult>(
        nameof(CheckEligibility), patient);
    
    await Task.WhenAll(insuranceTask, eligibilityTask);
    
    // Step 3: Collect consent (wait for patient action)
    await context.CallActivityAsync(nameof(SendConsentRequest), patient);
    
    var consent = await context.WaitForExternalEvent<ConsentResult>(
        "ConsentReceived",
        TimeSpan.FromDays(7));
    
    if (!consent.Granted)
    {
        return new OnboardingResult { Status = "ConsentNotProvided" };
    }
    
    // Step 4: Schedule appointment
    var appointment = await context.CallActivityAsync<Appointment>(
        nameof(ScheduleAppointment), patient);
    
    return new OnboardingResult { 
        Status = "Completed",
        AppointmentId = appointment.Id
    };
}
```

### Benefits
- **Compliance**: Full audit trail of each onboarding step
- **Human-in-the-loop**: Natural support for approval workflows
- **Long-running**: Workflow can span days or weeks
- **Resilience**: Patient progress is never lost

---

## Financial Services: Loan Processing

### The Challenge
Banks need to process loan applications through credit checks, risk assessment, document verification, and compliance validation—with strict SLAs and audit requirements.

### The Solution

```csharp
[Function(nameof(ProcessLoanApplication))]
public async Task<LoanDecision> ProcessLoanApplication(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var application = context.GetInput<LoanApplication>();
    var logger = context.CreateReplaySafeLogger<ProcessLoanApplication>();
    
    logger.LogInformation("Processing loan {LoanId}", application.Id);
    
    // Parallel initial checks
    var creditTask = context.CallActivityAsync<CreditScore>(
        nameof(CheckCredit), application);
    var fraudTask = context.CallActivityAsync<FraudResult>(
        nameof(FraudDetection), application);
    var employmentTask = context.CallActivityAsync<EmploymentResult>(
        nameof(VerifyEmployment), application);
    
    await Task.WhenAll(creditTask, fraudTask, employmentTask);
    
    // Evaluate results
    if (fraudTask.Result.RiskLevel == "High")
    {
        return new LoanDecision { 
            Approved = false, 
            Reason = "Failed fraud check" 
        };
    }
    
    // Risk-based routing
    var riskScore = await context.CallActivityAsync<int>(
        nameof(CalculateRiskScore), new {
            Credit = creditTask.Result,
            Fraud = fraudTask.Result,
            Employment = employmentTask.Result
        });
    
    if (riskScore > 70)
    {
        // Auto-approve low-risk applications
        return await AutoApprove(context, application);
    }
    else if (riskScore > 40)
    {
        // Medium risk - require additional docs
        return await RequestAdditionalDocuments(context, application);
    }
    else
    {
        // High risk - manual underwriting
        return await ManualUnderwriting(context, application);
    }
}
```

### Benefits
- **Speed**: Parallel checks reduce processing time
- **Accuracy**: Consistent rule application
- **Audit**: Complete history for regulatory compliance
- **Flexibility**: Easy to modify decision rules

---

## Manufacturing: IoT Device Provisioning

### The Challenge
Manufacturing companies need to provision thousands of IoT devices with certificates, firmware, and configuration—coordinating across multiple systems.

### The Solution

```csharp
[Function(nameof(ProvisionDevice))]
public async Task<ProvisioningResult> ProvisionDevice(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var device = context.GetInput<DeviceInfo>();
    
    // Step 1: Generate certificates
    var cert = await context.CallActivityAsync<Certificate>(
        nameof(GenerateCertificate), device);
    
    // Step 2: Register in IoT Hub
    var registration = await context.CallActivityAsync<DeviceRegistration>(
        nameof(RegisterInIoTHub), new { device, cert });
    
    // Step 3: Push firmware (with retry for network issues)
    await context.CallActivityAsync(
        nameof(DeployFirmware), 
        new { device.Id, registration.ConnectionString },
        new TaskOptions { 
            Retry = new RetryPolicy(5, TimeSpan.FromMinutes(1)) 
        });
    
    // Step 4: Configure device settings
    await context.CallActivityAsync(nameof(ApplyConfiguration), device);
    
    // Step 5: Validate connectivity
    var connected = await context.CallActivityAsync<bool>(
        nameof(ValidateConnectivity), device.Id);
    
    if (!connected)
    {
        // Create support ticket for manual intervention
        await context.CallActivityAsync(nameof(CreateSupportTicket), device);
        
        // Wait for resolution (up to 24 hours)
        var resolved = await context.WaitForExternalEvent<bool>(
            "ConnectivityResolved",
            TimeSpan.FromHours(24));
        
        if (!resolved)
        {
            return new ProvisioningResult { 
                Status = "Failed", 
                Error = "Connectivity timeout" 
            };
        }
    }
    
    return new ProvisioningResult { 
        Status = "Completed",
        DeviceId = device.Id,
        ConnectionString = registration.ConnectionString
    };
}

// Bulk provisioning with fan-out
[Function(nameof(BulkProvision))]
public async Task<BulkResult> BulkProvision(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var devices = context.GetInput<List<DeviceInfo>>();
    
    // Fan-out: Provision all devices in parallel
    var tasks = devices.Select(device => 
        context.CallSubOrchestratorAsync<ProvisioningResult>(
            nameof(ProvisionDevice), device));
    
    var results = await Task.WhenAll(tasks);
    
    return new BulkResult {
        Total = devices.Count,
        Succeeded = results.Count(r => r.Status == "Completed"),
        Failed = results.Count(r => r.Status == "Failed")
    };
}
```

### Benefits
- **Scale**: Provision thousands of devices in parallel
- **Reliability**: Retries handle transient network failures
- **Tracking**: Know exactly which devices succeeded or failed
- **Automation**: Minimal manual intervention required

---

## Media: Video Processing Pipeline

### The Challenge
Media companies need to process uploaded videos through transcoding, thumbnail generation, content moderation, and metadata extraction—handling large files and long processing times.

### The Solution

```csharp
[Function(nameof(ProcessVideo))]
public async Task<VideoResult> ProcessVideo(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var video = context.GetInput<VideoUpload>();
    
    // Step 1: Validate video file
    var validation = await context.CallActivityAsync<ValidationResult>(
        nameof(ValidateVideo), video);
    
    if (!validation.IsValid)
    {
        return new VideoResult { Status = "Invalid", Errors = validation.Errors };
    }
    
    // Step 2: Start parallel processing
    var transcodeTask = context.CallActivityAsync<TranscodeResult>(
        nameof(TranscodeVideo), new { video, Formats = new[] { "1080p", "720p", "480p" } });
    
    var thumbnailTask = context.CallActivityAsync<ThumbnailResult>(
        nameof(GenerateThumbnails), video);
    
    var moderationTask = context.CallActivityAsync<ModerationResult>(
        nameof(ContentModeration), video);
    
    var metadataTask = context.CallActivityAsync<VideoMetadata>(
        nameof(ExtractMetadata), video);
    
    await Task.WhenAll(transcodeTask, thumbnailTask, moderationTask, metadataTask);
    
    // Step 3: Check moderation results
    if (moderationTask.Result.Flagged)
    {
        await context.CallActivityAsync(nameof(NotifyModerationTeam), new {
            video.Id,
            moderationTask.Result.Reasons
        });
        
        // Wait for human review
        var review = await context.WaitForExternalEvent<ReviewResult>(
            "ModerationReview",
            TimeSpan.FromHours(24));
        
        if (!review.Approved)
        {
            await context.CallActivityAsync(nameof(DeleteVideo), video.Id);
            return new VideoResult { Status = "Rejected" };
        }
    }
    
    // Step 4: Publish to CDN
    var cdnUrls = await context.CallActivityAsync<Dictionary<string, string>>(
        nameof(PublishToCdn), new {
            video.Id,
            transcodeTask.Result.OutputFiles,
            thumbnailTask.Result.Thumbnails
        });
    
    // Step 5: Update database
    await context.CallActivityAsync(nameof(UpdateVideoRecord), new {
        video.Id,
        cdnUrls,
        metadataTask.Result
    });
    
    return new VideoResult {
        Status = "Published",
        StreamingUrls = cdnUrls,
        Metadata = metadataTask.Result
    };
}
```

### Benefits
- **Parallel processing**: Transcoding, thumbnails, and moderation run simultaneously
- **Content safety**: Built-in moderation workflow
- **Cost efficiency**: Only pay for actual processing time
- **Resumability**: Large file processing survives interruptions

---

## Common Patterns Across Use Cases

| Pattern | Description | Example Use Cases |
|---------|-------------|-------------------|
| **Sequential Steps** | Execute activities in order | Order processing, loan approval |
| **Parallel Fan-out** | Run multiple activities simultaneously | Device provisioning, video processing |
| **Human Approval** | Wait for external events | Patient onboarding, content moderation |
| **Saga/Compensation** | Undo steps on failure | E-commerce orders, financial transactions |
| **Sub-orchestrations** | Compose complex workflows | Bulk operations, multi-tenant processing |
| **Timers** | Schedule future work | Reminder systems, SLA monitoring |

---

## Getting Started with Your Use Case

1. **Identify the workflow steps** - What activities need to happen?
2. **Determine parallelism** - Which steps can run simultaneously?
3. **Plan for failures** - What compensation logic is needed?
4. **Consider human interaction** - Are there approval steps?
5. **Choose deployment model** - Serverless (Durable Functions) or containers (SDK)?

---

## Next Steps

- [Orchestration Patterns →](../patterns/index.md)
- [Durable Functions Quickstart →](../durable-functions/quickstart.md)
- [Durable Task SDK Quickstart →](../sdks/quickstart.md)
- [Architecture Guides →](../architecture/index.md)
