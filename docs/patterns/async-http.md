---
layout: default
title: Async HTTP APIs
parent: Orchestration Patterns
nav_order: 3
permalink: /docs/patterns/async-http/
---

# Async HTTP APIs Pattern
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Expose long-running orchestrations as HTTP APIs with polling or webhook-based completion notification.

---

## Overview

The async HTTP API pattern provides a standard way to expose durable orchestrations through HTTP endpoints. Clients start an operation and either poll for completion or receive a webhook callback.

```
┌──────────────────────────────────────────────────────────────────┐
│                     ASYNC HTTP API PATTERN                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   Client                     API                 Orchestration    │
│   ──────                     ───                 ────────────     │
│                                                                   │
│   POST /api/process ─────────►                                    │
│                              │                                    │
│   ◄──── 202 Accepted ────────┤                                    │
│         Location: /status/123│                                    │
│                              │                                    │
│         ┌────────────────────┼─────────────────────────────┐     │
│         │ Option A: Polling  │                             │     │
│         │                    │                             │     │
│   GET /status/123 ──────────►│                             │     │
│   ◄──── 202 Running ─────────┤                             │     │
│         ...                  │      Long-running          │     │
│   GET /status/123 ──────────►│      processing...         │     │
│   ◄──── 200 Completed ───────┤ ◄───────────────────────────│     │
│         {result}             │                             │     │
│         └────────────────────┼─────────────────────────────┘     │
│                              │                                    │
│         ┌────────────────────┼─────────────────────────────┐     │
│         │ Option B: Webhook  │                             │     │
│         │                    │                             │     │
│   ◄───────────────────────────────── POST /callback ───────│     │
│         {result}             │                             │     │
│         └────────────────────┴─────────────────────────────┘     │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Use Cases

- **Report Generation**: Generate large reports asynchronously
- **Data Processing**: Process uploaded files
- **Integration APIs**: Long-running third-party operations
- **Batch Operations**: Submit batch jobs via API
- **Export Operations**: Export data to various formats

---

## Implementation

### API Controller

```csharp
[ApiController]
[Route("api/reports")]
public class ReportsController : ControllerBase
{
    private readonly DurableTaskClient _client;
    
    public ReportsController(DurableTaskClient client)
    {
        _client = client;
    }
    
    /// <summary>
    /// Start a new report generation
    /// </summary>
    [HttpPost]
    public async Task<IActionResult> GenerateReport([FromBody] ReportRequest request)
    {
        // Start orchestration
        var instanceId = await _client.ScheduleNewOrchestrationInstanceAsync(
            nameof(ReportGenerationOrchestrator),
            request
        );
        
        // Return 202 Accepted with status location
        return AcceptedAtAction(
            nameof(GetStatus),
            new { instanceId },
            new
            {
                instanceId,
                statusUrl = Url.Action(nameof(GetStatus), new { instanceId }),
                resultUrl = Url.Action(nameof(GetResult), new { instanceId })
            }
        );
    }
    
    /// <summary>
    /// Check operation status
    /// </summary>
    [HttpGet("{instanceId}/status")]
    public async Task<IActionResult> GetStatus(string instanceId)
    {
        var metadata = await _client.GetInstanceAsync(instanceId);
        
        if (metadata == null)
        {
            return NotFound();
        }
        
        return metadata.RuntimeStatus switch
        {
            OrchestrationRuntimeStatus.Pending or 
            OrchestrationRuntimeStatus.Running => Accepted(new
            {
                instanceId,
                status = metadata.RuntimeStatus.ToString(),
                createdAt = metadata.CreatedAt,
                lastUpdatedAt = metadata.LastUpdatedAt,
                customStatus = metadata.SerializedCustomStatus
            }),
            
            OrchestrationRuntimeStatus.Completed => Ok(new
            {
                instanceId,
                status = "Completed",
                resultUrl = Url.Action(nameof(GetResult), new { instanceId })
            }),
            
            OrchestrationRuntimeStatus.Failed => Ok(new
            {
                instanceId,
                status = "Failed",
                error = metadata.FailureDetails?.ErrorMessage
            }),
            
            _ => Ok(new
            {
                instanceId,
                status = metadata.RuntimeStatus.ToString()
            })
        };
    }
    
    /// <summary>
    /// Get operation result
    /// </summary>
    [HttpGet("{instanceId}/result")]
    public async Task<IActionResult> GetResult(string instanceId)
    {
        var metadata = await _client.GetInstanceAsync(instanceId, getInputsAndOutputs: true);
        
        if (metadata == null)
        {
            return NotFound();
        }
        
        if (metadata.RuntimeStatus != OrchestrationRuntimeStatus.Completed)
        {
            return Conflict(new { message = "Operation not yet completed" });
        }
        
        var result = metadata.ReadOutputAs<ReportResult>();
        return Ok(result);
    }
    
    /// <summary>
    /// Cancel operation
    /// </summary>
    [HttpDelete("{instanceId}")]
    public async Task<IActionResult> Cancel(string instanceId)
    {
        var metadata = await _client.GetInstanceAsync(instanceId);
        
        if (metadata == null)
        {
            return NotFound();
        }
        
        if (metadata.RuntimeStatus == OrchestrationRuntimeStatus.Completed ||
            metadata.RuntimeStatus == OrchestrationRuntimeStatus.Failed)
        {
            return Conflict(new { message = "Cannot cancel completed operation" });
        }
        
        await _client.TerminateInstanceAsync(instanceId, "Cancelled by user");
        
        return NoContent();
    }
}
```

### Orchestrator

```csharp
[DurableTask(nameof(ReportGenerationOrchestrator))]
public class ReportGenerationOrchestrator : TaskOrchestrator<ReportRequest, ReportResult>
{
    public override async Task<ReportResult> RunAsync(
        TaskOrchestrationContext context, 
        ReportRequest request)
    {
        // Update status
        context.SetCustomStatus(new { stage = "Fetching data" });
        
        // Fetch data
        var data = await context.CallActivityAsync<ReportData>(
            nameof(FetchReportData),
            request
        );
        
        // Update status
        context.SetCustomStatus(new { stage = "Processing", recordCount = data.Records.Count });
        
        // Process data
        var processed = await context.CallActivityAsync<ProcessedData>(
            nameof(ProcessReportData),
            data
        );
        
        // Update status
        context.SetCustomStatus(new { stage = "Generating output" });
        
        // Generate report
        var result = await context.CallActivityAsync<ReportResult>(
            nameof(GenerateReportOutput),
            processed
        );
        
        // If webhook URL was provided, send notification
        if (!string.IsNullOrEmpty(request.WebhookUrl))
        {
            await context.CallActivityAsync(
                nameof(SendWebhookNotification),
                new WebhookPayload
                {
                    Url = request.WebhookUrl,
                    InstanceId = context.InstanceId,
                    Result = result
                }
            );
        }
        
        return result;
    }
}
```

---

## Webhook Notification

### Activity for Sending Webhooks

```csharp
[DurableTask(nameof(SendWebhookNotification))]
public class SendWebhookNotification : TaskActivity<WebhookPayload, bool>
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<SendWebhookNotification> _logger;
    
    public SendWebhookNotification(
        HttpClient httpClient,
        ILogger<SendWebhookNotification> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }
    
    public override async Task<bool> RunAsync(
        TaskActivityContext context, 
        WebhookPayload payload)
    {
        try
        {
            var response = await _httpClient.PostAsJsonAsync(
                payload.Url,
                new
                {
                    instanceId = payload.InstanceId,
                    status = "Completed",
                    result = payload.Result,
                    completedAt = DateTime.UtcNow
                }
            );
            
            response.EnsureSuccessStatusCode();
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to send webhook to {Url}", payload.Url);
            return false;
        }
    }
}
```

### Request with Webhook

```json
POST /api/reports
{
    "reportType": "sales",
    "startDate": "2024-01-01",
    "endDate": "2024-12-31",
    "webhookUrl": "https://myapp.example.com/callbacks/reports"
}
```

---

## Polling Client Implementation

### JavaScript/TypeScript Client

```typescript
async function generateReportWithPolling(request: ReportRequest): Promise<ReportResult> {
    // Start the operation
    const startResponse = await fetch('/api/reports', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(request)
    });
    
    if (!startResponse.ok) {
        throw new Error('Failed to start report generation');
    }
    
    const { instanceId, statusUrl } = await startResponse.json();
    
    // Poll for completion
    while (true) {
        const statusResponse = await fetch(statusUrl);
        const status = await statusResponse.json();
        
        if (statusResponse.status === 200) {
            // Completed - fetch result
            if (status.resultUrl) {
                const resultResponse = await fetch(status.resultUrl);
                return await resultResponse.json();
            }
            throw new Error(status.error || 'Operation failed');
        }
        
        // Still running - wait and retry
        console.log(`Status: ${status.customStatus?.stage || 'Processing'}`);
        await delay(2000); // Wait 2 seconds
    }
}

function delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
}
```

### Python Client

```python
import requests
import time

def generate_report_with_polling(request: dict, timeout: int = 300) -> dict:
    # Start the operation
    response = requests.post(
        'https://api.example.com/api/reports',
        json=request
    )
    response.raise_for_status()
    
    data = response.json()
    status_url = data['statusUrl']
    
    # Poll for completion
    start_time = time.time()
    
    while time.time() - start_time < timeout:
        status_response = requests.get(status_url)
        status = status_response.json()
        
        if status_response.status_code == 200:
            # Completed
            if 'resultUrl' in status:
                result_response = requests.get(status['resultUrl'])
                return result_response.json()
            raise Exception(status.get('error', 'Operation failed'))
        
        # Still running
        print(f"Status: {status.get('customStatus', {}).get('stage', 'Processing')}")
        time.sleep(2)
    
    raise TimeoutError('Operation timed out')
```

---

## Response Format Standards

### HTTP Status Codes

| Code | Meaning | When to Use |
|------|---------|-------------|
| 202 | Accepted | Operation started, in progress |
| 200 | OK | Operation completed successfully |
| 404 | Not Found | Instance ID not found |
| 409 | Conflict | Cannot perform action (e.g., cancel completed) |
| 500 | Server Error | Unexpected error |

### Standard Response Structure

```json
// Start Response (202)
{
    "instanceId": "abc123",
    "statusUrl": "/api/reports/abc123/status",
    "resultUrl": "/api/reports/abc123/result",
    "cancelUrl": "/api/reports/abc123"
}

// Status Response - Running (202)
{
    "instanceId": "abc123",
    "status": "Running",
    "createdAt": "2024-01-15T10:00:00Z",
    "lastUpdatedAt": "2024-01-15T10:05:00Z",
    "customStatus": {
        "stage": "Processing",
        "progress": 45
    }
}

// Status Response - Completed (200)
{
    "instanceId": "abc123",
    "status": "Completed",
    "resultUrl": "/api/reports/abc123/result"
}

// Result Response (200)
{
    "reportId": "report-456",
    "downloadUrl": "https://storage.example.com/reports/report-456.pdf",
    "summary": { ... }
}
```

---

## Long-Polling Variant

```csharp
[HttpGet("{instanceId}/status")]
public async Task<IActionResult> GetStatusLongPoll(
    string instanceId,
    [FromQuery] int timeoutSeconds = 30)
{
    var deadline = DateTime.UtcNow.AddSeconds(Math.Min(timeoutSeconds, 60));
    
    while (DateTime.UtcNow < deadline)
    {
        var metadata = await _client.GetInstanceAsync(instanceId);
        
        if (metadata == null)
        {
            return NotFound();
        }
        
        if (metadata.RuntimeStatus != OrchestrationRuntimeStatus.Running &&
            metadata.RuntimeStatus != OrchestrationRuntimeStatus.Pending)
        {
            // Operation finished - return immediately
            return Ok(CreateStatusResponse(metadata));
        }
        
        // Still running - wait a bit
        await Task.Delay(1000);
    }
    
    // Timeout - return current status
    var finalMetadata = await _client.GetInstanceAsync(instanceId);
    return Accepted(CreateStatusResponse(finalMetadata));
}
```

---

## Best Practices

1. **Return 202 immediately** - Don't make clients wait for processing
2. **Include status URLs** - Make it easy for clients to poll
3. **Provide progress information** - Use custom status for progress
4. **Support cancellation** - Allow clients to cancel long operations
5. **Set reasonable timeouts** - Both client-side and server-side
6. **Consider rate limiting** - Prevent polling abuse

---

## Related Patterns

- [Human Interaction](human-interaction.md) - For human-in-the-loop async
- [External Events](external-events.md) - For webhook-based notifications
- [Function Chaining](function-chaining.md) - For the actual processing
