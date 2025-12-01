---
layout: default
title: Human Interaction
parent: Orchestration Patterns
nav_order: 4
permalink: /docs/patterns/human-interaction/
---

# Human Interaction Pattern
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Pause orchestration execution to wait for human input, approval, or decision-making.

---

## Overview

The human interaction pattern enables workflows to incorporate human decision-making, approvals, or data input. The orchestration suspends until an external event is received.

```
┌──────────────────────────────────────────────────────────────────┐
│                     HUMAN INTERACTION PATTERN                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│     ┌──────────────┐         ┌──────────────┐                    │
│     │  Start       │         │   Human      │                    │
│     │  Request     │         │   (Email,    │                    │
│     └──────┬───────┘         │    UI, etc.) │                    │
│            │                 └──────┬───────┘                    │
│            ▼                        │                            │
│     ┌──────────────┐                │                            │
│     │  Send        │                │                            │
│     │  Notification│───────────────►│                            │
│     └──────┬───────┘                │                            │
│            │                        │                            │
│            ▼                        │                            │
│     ┌──────────────┐                │                            │
│     │  Wait for    │◄───────────────┤                            │
│     │  Response    │   (Approve/    │                            │
│     │  (Timer)     │    Reject)     │                            │
│     └──────┬───────┘                │                            │
│            │                        │                            │
│            ├── Approved ────────────┘                            │
│            │                                                      │
│            ├── Rejected ──► Notify Rejection                     │
│            │                                                      │
│            └── Timeout ───► Escalate or Cancel                   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Use Cases

- **Expense Approvals**: Manager approves expense reports
- **Document Review**: Legal review of contracts
- **Order Confirmation**: Customer confirms high-value orders
- **Exception Handling**: Human review of flagged transactions
- **Data Validation**: Manual verification of suspicious data

---

## Implementation

### C# (.NET)

```csharp
using Microsoft.DurableTask;

[DurableTask(nameof(ApprovalOrchestrator))]
public class ApprovalOrchestrator : TaskOrchestrator<ApprovalRequest, ApprovalResult>
{
    public override async Task<ApprovalResult> RunAsync(
        TaskOrchestrationContext context, 
        ApprovalRequest request)
    {
        // Step 1: Send notification to approver
        await context.CallActivityAsync(
            nameof(SendApprovalNotification),
            new NotificationPayload
            {
                RequestId = context.InstanceId,
                ApproverEmail = request.ApproverEmail,
                Details = request.Description,
                Amount = request.Amount,
                ApprovalUrl = $"https://app.example.com/approve/{context.InstanceId}"
            }
        );
        
        // Step 2: Wait for approval with timeout
        using var cts = new CancellationTokenSource();
        var approvalTask = context.WaitForExternalEvent<ApprovalResponse>("ApprovalReceived");
        var timeoutTask = context.CreateTimer(
            context.CurrentUtcDateTime.AddHours(72), 
            cts.Token
        );
        
        var winner = await Task.WhenAny(approvalTask, timeoutTask);
        
        if (winner == approvalTask)
        {
            // Cancel the timer
            cts.Cancel();
            
            var response = await approvalTask;
            
            if (response.Approved)
            {
                // Step 3a: Process approved request
                await context.CallActivityAsync(
                    nameof(ProcessApprovedRequest),
                    request
                );
                
                return new ApprovalResult
                {
                    Status = "Approved",
                    ApprovedBy = response.ApproverName,
                    ApprovedAt = context.CurrentUtcDateTime
                };
            }
            else
            {
                // Step 3b: Handle rejection
                await context.CallActivityAsync(
                    nameof(NotifyRejection),
                    new RejectionNotification
                    {
                        RequestId = context.InstanceId,
                        Reason = response.Reason
                    }
                );
                
                return new ApprovalResult
                {
                    Status = "Rejected",
                    RejectedBy = response.ApproverName,
                    Reason = response.Reason
                };
            }
        }
        else
        {
            // Step 3c: Handle timeout - escalate
            await context.CallActivityAsync(
                nameof(EscalateRequest),
                request
            );
            
            return new ApprovalResult
            {
                Status = "Escalated",
                Reason = "Approval timeout exceeded"
            };
        }
    }
}
```

### Sending the Approval Event

```csharp
// From your API controller
[HttpPost("approve/{instanceId}")]
public async Task<IActionResult> Approve(
    string instanceId, 
    [FromBody] ApprovalInput input,
    [FromServices] DurableTaskClient client)
{
    await client.RaiseEventAsync(
        instanceId,
        "ApprovalReceived",
        new ApprovalResponse
        {
            Approved = input.Approved,
            ApproverName = User.Identity.Name,
            Reason = input.Reason
        }
    );
    
    return Ok(new { message = "Approval recorded" });
}
```

### Python

```python
from datetime import timedelta
from durabletask import task

@task.orchestrator
def approval_orchestrator(ctx: task.OrchestrationContext, request: dict):
    """Wait for human approval with timeout."""
    
    # Step 1: Send notification
    yield ctx.call_activity(
        "send_approval_notification",
        input={
            "request_id": ctx.instance_id,
            "approver_email": request["approver_email"],
            "details": request["description"],
            "amount": request["amount"]
        }
    )
    
    # Step 2: Wait for approval or timeout
    approval_event = ctx.wait_for_external_event("ApprovalReceived")
    timeout_event = ctx.create_timer(timedelta(hours=72))
    
    winner = yield task.when_any([approval_event, timeout_event])
    
    if winner == approval_event:
        response = yield approval_event
        
        if response["approved"]:
            # Process approved
            yield ctx.call_activity("process_approved", input=request)
            return {
                "status": "Approved",
                "approved_by": response["approver_name"]
            }
        else:
            # Handle rejection
            yield ctx.call_activity(
                "notify_rejection",
                input={
                    "request_id": ctx.instance_id,
                    "reason": response["reason"]
                }
            )
            return {
                "status": "Rejected",
                "reason": response["reason"]
            }
    else:
        # Timeout - escalate
        yield ctx.call_activity("escalate_request", input=request)
        return {"status": "Escalated", "reason": "Timeout"}
```

### Java

```java
public class ApprovalOrchestrator implements TaskOrchestrator {
    @Override
    public ApprovalResult run(TaskOrchestrationContext ctx) {
        ApprovalRequest request = ctx.getInput(ApprovalRequest.class);
        
        // Send notification
        ctx.callActivity(
            "SendApprovalNotification",
            new NotificationPayload(ctx.getInstanceId(), request),
            Void.class
        ).await();
        
        // Wait for approval or timeout
        Task<ApprovalResponse> approvalTask = ctx.waitForExternalEvent(
            "ApprovalReceived", 
            ApprovalResponse.class
        );
        Task<Void> timeoutTask = ctx.createTimer(Duration.ofHours(72));
        
        Task<?> winner = ctx.anyOf(approvalTask, timeoutTask).await();
        
        if (winner == approvalTask) {
            ApprovalResponse response = approvalTask.await();
            
            if (response.isApproved()) {
                ctx.callActivity("ProcessApproved", request, Void.class).await();
                return new ApprovalResult("Approved", response.getApproverName());
            } else {
                ctx.callActivity("NotifyRejection", 
                    new RejectionNotification(ctx.getInstanceId(), response.getReason()),
                    Void.class
                ).await();
                return new ApprovalResult("Rejected", response.getReason());
            }
        } else {
            ctx.callActivity("EscalateRequest", request, Void.class).await();
            return new ApprovalResult("Escalated", "Timeout");
        }
    }
}
```

---

## Multi-Level Approvals

```csharp
public override async Task<ApprovalResult> RunAsync(
    TaskOrchestrationContext context, 
    MultiLevelApprovalRequest request)
{
    var approvers = request.ApprovalChain; // e.g., [manager, director, vp]
    
    foreach (var approver in approvers)
    {
        // Send notification to current approver
        await context.CallActivityAsync(
            nameof(SendApprovalNotification),
            new NotificationPayload { ApproverEmail = approver.Email }
        );
        
        // Wait for this level of approval
        var approvalTask = context.WaitForExternalEvent<ApprovalResponse>(
            $"Approval_{approver.Level}"
        );
        var timeoutTask = context.CreateTimer(
            context.CurrentUtcDateTime.AddHours(24), 
            CancellationToken.None
        );
        
        var winner = await Task.WhenAny(approvalTask, timeoutTask);
        
        if (winner == approvalTask)
        {
            var response = await approvalTask;
            if (!response.Approved)
            {
                return new ApprovalResult { Status = "Rejected", Level = approver.Level };
            }
            // Continue to next level
        }
        else
        {
            return new ApprovalResult { Status = "Timeout", Level = approver.Level };
        }
    }
    
    // All levels approved
    return new ApprovalResult { Status = "FullyApproved" };
}
```

---

## Parallel Approvals

```csharp
public override async Task<ApprovalResult> RunAsync(
    TaskOrchestrationContext context, 
    ParallelApprovalRequest request)
{
    // Send notifications to all approvers
    foreach (var approver in request.Approvers)
    {
        await context.CallActivityAsync(
            nameof(SendApprovalNotification),
            new NotificationPayload { ApproverEmail = approver.Email }
        );
    }
    
    // Wait for all approvals
    var approvalTasks = request.Approvers.Select(a =>
        context.WaitForExternalEvent<ApprovalResponse>($"Approval_{a.Id}")
    ).ToList();
    
    var timeoutTask = context.CreateTimer(
        context.CurrentUtcDateTime.AddHours(24), 
        CancellationToken.None
    );
    
    var allApprovalsTask = Task.WhenAll(approvalTasks);
    var winner = await Task.WhenAny(allApprovalsTask, timeoutTask);
    
    if (winner == allApprovalsTask)
    {
        var responses = await allApprovalsTask;
        var allApproved = responses.All(r => r.Approved);
        
        return new ApprovalResult
        {
            Status = allApproved ? "Approved" : "Rejected",
            Responses = responses.ToList()
        };
    }
    else
    {
        return new ApprovalResult { Status = "Timeout" };
    }
}
```

---

## UI Integration

### Approval Dashboard API

```csharp
[ApiController]
[Route("api/approvals")]
public class ApprovalsController : ControllerBase
{
    private readonly DurableTaskClient _client;
    
    [HttpGet("pending")]
    public async Task<IActionResult> GetPendingApprovals()
    {
        var query = new OrchestrationQuery
        {
            RuntimeStatus = new[] { OrchestrationRuntimeStatus.Running },
            InstanceIdPrefix = "approval-"
        };
        
        var instances = await _client.QueryInstancesAsync(query);
        
        return Ok(instances.Select(i => new
        {
            i.InstanceId,
            i.CreatedAt,
            Request = i.ReadInputAs<ApprovalRequest>()
        }));
    }
    
    [HttpPost("{instanceId}/approve")]
    public async Task<IActionResult> Approve(string instanceId, [FromBody] ApprovalInput input)
    {
        await _client.RaiseEventAsync(instanceId, "ApprovalReceived", new ApprovalResponse
        {
            Approved = true,
            ApproverName = User.Identity.Name
        });
        
        return Ok();
    }
    
    [HttpPost("{instanceId}/reject")]
    public async Task<IActionResult> Reject(string instanceId, [FromBody] RejectionInput input)
    {
        await _client.RaiseEventAsync(instanceId, "ApprovalReceived", new ApprovalResponse
        {
            Approved = false,
            ApproverName = User.Identity.Name,
            Reason = input.Reason
        });
        
        return Ok();
    }
}
```

---

## Best Practices

1. **Always include timeouts** - Prevent indefinitely stuck workflows
2. **Send reminders** - For long approval windows, schedule reminder activities
3. **Provide context** - Include all information needed for decision-making
4. **Enable escalation** - Define what happens on timeout
5. **Log decisions** - Track who approved/rejected and when
6. **Support delegation** - Allow approvers to delegate to others

---

## Common Timeout Strategies

| Strategy | Description | Implementation |
|----------|-------------|----------------|
| Escalate | Forward to next level | Call escalation activity |
| Auto-Approve | Approve if no response | Return approved result |
| Auto-Reject | Reject if no response | Return rejected result |
| Remind | Send reminder, extend timeout | Nested when_any with reminder |
| Cancel | Cancel the request | Terminate with cancellation |

---

## Related Patterns

- [External Events](external-events.md) - General event handling
- [Function Chaining](function-chaining.md) - Sequential processing after approval
- [Async HTTP APIs](async-http.md) - API patterns for approval endpoints
