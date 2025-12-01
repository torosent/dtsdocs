---
layout: default
title: PowerShell SDK
nav_order: 5
parent: SDKs
grand_parent: Developer Guide
has_children: false
permalink: /developer-guide/sdks/powershell/
---

# PowerShell SDK for Durable Functions

Build durable workflows in PowerShell using the Azure Functions Durable PowerShell SDK.

## SDK Options

### AzureFunctions.PowerShell.Durable.SDK (Recommended)

The standalone **AzureFunctions.PowerShell.Durable.SDK** module is the recommended SDK for building Durable Functions in PowerShell.

| Property | Details |
|----------|---------|
| Module | [AzureFunctions.PowerShell.Durable.SDK](https://www.powershellgallery.com/packages/AzureFunctions.PowerShell.Durable.SDK) |
| PowerShell Version | 7.x |
| Functions Runtime | 3.0+ |
| Extension Bundle | 2.x+ |

{: .important }
> The standalone SDK is recommended over the legacy built-in SDK. The legacy SDK may not receive new features or bug fixes and may eventually be removed.

### Portable SDK

There is no portable SDK for PowerShell. Durable Functions PowerShell is only supported in Azure Functions.

---

## Getting Started

### Prerequisites

- PowerShell 7.0 or later
- Azure Functions Core Tools 4.x
- An Azure Storage account
- VS Code with Azure Functions extension (recommended)

### Create a New Project

1. Create a new Azure Functions project:
   ```bash
   func init MyDurableFunctionApp --worker-runtime powershell
   cd MyDurableFunctionApp
   ```

2. Configure local settings (`local.settings.json`):
   ```json
   {
     "IsEncrypted": false,
     "Values": {
       "AzureWebJobsStorage": "UseDevelopmentStorage=true",
       "FUNCTIONS_WORKER_RUNTIME": "powershell",
       "FUNCTIONS_WORKER_RUNTIME_VERSION": "7.4",
       "ExternalDurablePowerShellSDK": "true"
     }
   }
   ```

3. Add the SDK to `requirements.psd1`:
   ```powershell
   @{
       'AzureFunctions.PowerShell.Durable.SDK' = '2.*'
   }
   ```

4. Import the SDK in `profile.ps1`:
   ```powershell
   Import-Module AzureFunctions.PowerShell.Durable.SDK -ErrorAction Stop
   ```

---

## SDK Cmdlet Reference

### Get Available Cmdlets

```powershell
Get-Help *-Durable*
```

### Get Detailed Help

```powershell
Get-Help Invoke-DurableOrchestration -Full
```

---

## Orchestrator Functions

### Creating an Orchestrator

Create an orchestrator function with `orchestrationTrigger` binding:

**HelloOrchestrator/function.json**:
```json
{
  "bindings": [
    {
      "name": "Context",
      "type": "orchestrationTrigger",
      "direction": "in"
    }
  ]
}
```

**HelloOrchestrator/run.ps1**:
```powershell
param($Context)

$outputs = @()

$outputs += Invoke-DurableActivity -FunctionName 'SayHello' -Input 'Tokyo'
$outputs += Invoke-DurableActivity -FunctionName 'SayHello' -Input 'Seattle'
$outputs += Invoke-DurableActivity -FunctionName 'SayHello' -Input 'London'

$outputs
```

### Invoke-DurableActivity

Calls an activity function and waits for the result.

```powershell
Invoke-DurableActivity -FunctionName <String> [-Input <Object>] [-NoWait]
```

**Parameters:**
| Parameter | Description |
|-----------|-------------|
| `-FunctionName` | Name of the activity function to invoke |
| `-Input` | Input data to pass to the activity |
| `-NoWait` | Returns immediately without waiting (for parallel execution) |

**Example:**
```powershell
# Sequential execution
$result = Invoke-DurableActivity -FunctionName 'ProcessData' -Input @{ id = "123" }

# Parallel execution (fan-out)
$task1 = Invoke-DurableActivity -FunctionName 'ProcessItem' -Input 'Item1' -NoWait
$task2 = Invoke-DurableActivity -FunctionName 'ProcessItem' -Input 'Item2' -NoWait
$task3 = Invoke-DurableActivity -FunctionName 'ProcessItem' -Input 'Item3' -NoWait
```

### Invoke-DurableSubOrchestrator

Calls a sub-orchestrator function.

```powershell
Invoke-DurableSubOrchestrator -FunctionName <String> [-Input <Object>] [-InstanceId <String>]
```

**Example:**
```powershell
$result = Invoke-DurableSubOrchestrator -FunctionName 'ChildOrchestrator' -Input $inputData
```

### Wait-DurableTask

Waits for one or more durable tasks to complete (fan-in).

```powershell
Wait-DurableTask -Task <DurableTask[]> [-Any]
```

**Parameters:**
| Parameter | Description |
|-----------|-------------|
| `-Task` | Array of tasks to wait for |
| `-Any` | Return when any task completes (instead of all) |

**Example:**
```powershell
# Wait for all tasks
$tasks = @(
    Invoke-DurableActivity -FunctionName 'Task1' -Input 'A' -NoWait
    Invoke-DurableActivity -FunctionName 'Task2' -Input 'B' -NoWait
    Invoke-DurableActivity -FunctionName 'Task3' -Input 'C' -NoWait
)
$results = Wait-DurableTask -Task $tasks

# Wait for any task (race)
$winner = Wait-DurableTask -Task $tasks -Any
```

### Start-DurableTimer

Creates a durable timer that waits until a specified time.

```powershell
Start-DurableTimer -Duration <TimeSpan>
Start-DurableTimer -ExpirationTime <DateTime>
```

**Example:**
```powershell
# Wait for 1 hour
Start-DurableTimer -Duration (New-TimeSpan -Hours 1)

# Wait until specific time
Start-DurableTimer -ExpirationTime (Get-Date).AddDays(1)
```

### Start-DurableExternalEventListener

Waits for an external event to be raised.

```powershell
Start-DurableExternalEventListener -EventName <String> [-NoWait]
```

**Example:**
```powershell
# Wait for approval event
$approval = Start-DurableExternalEventListener -EventName 'ApprovalEvent'

# Non-blocking external event listener
$eventTask = Start-DurableExternalEventListener -EventName 'ApprovalEvent' -NoWait
```

### Set-DurableCustomStatus

Sets a custom status for the orchestration.

```powershell
Set-DurableCustomStatus -CustomStatus <Object>
```

**Example:**
```powershell
Set-DurableCustomStatus -CustomStatus @{ stage = "processing"; progress = 50 }
```

---

## Activity Functions

### Creating an Activity

**SayHello/function.json**:
```json
{
  "bindings": [
    {
      "name": "name",
      "type": "activityTrigger",
      "direction": "in"
    }
  ]
}
```

**SayHello/run.ps1**:
```powershell
param($name)

"Hello, $name!"
```

### Activity Best Practices

1. Activities should be idempotent when possible
2. Keep activities focused on a single task
3. Handle errors gracefully and return meaningful error messages
4. Use appropriate timeouts for external operations

---

## Client Functions

### Starting an Orchestration

**HttpStart/function.json**:
```json
{
  "bindings": [
    {
      "authLevel": "anonymous",
      "name": "Request",
      "type": "httpTrigger",
      "direction": "in",
      "methods": ["post", "get"],
      "route": "orchestrators/{functionName}"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "Response"
    },
    {
      "name": "starter",
      "type": "durableClient",
      "direction": "in"
    }
  ]
}
```

**HttpStart/run.ps1**:
```powershell
param($Request, $TriggerMetadata)

$FunctionName = $Request.Params.functionName
$InstanceId = Start-DurableOrchestration -FunctionName $FunctionName -Input $Request.Body

Write-Host "Started orchestration with ID = '$InstanceId'"

$Response = New-DurableOrchestrationCheckStatusResponse -Request $Request -InstanceId $InstanceId
Push-OutputBinding -Name Response -Value $Response
```

### Start-DurableOrchestration

Starts a new orchestration instance.

```powershell
Start-DurableOrchestration -FunctionName <String> [-Input <Object>] [-InstanceId <String>]
```

**Parameters:**
| Parameter | Description |
|-----------|-------------|
| `-FunctionName` | Name of the orchestrator function |
| `-Input` | Input data for the orchestration |
| `-InstanceId` | Optional custom instance ID |

### Get-DurableStatus

Gets the status of an orchestration instance.

```powershell
Get-DurableStatus -InstanceId <String> [-ShowHistory] [-ShowHistoryOutput] [-ShowInput]
```

**Example:**
```powershell
$status = Get-DurableStatus -InstanceId $instanceId -ShowHistory
Write-Host "Status: $($status.RuntimeStatus)"
Write-Host "Output: $($status.Output)"
```

### Stop-DurableOrchestration

Terminates a running orchestration.

```powershell
Stop-DurableOrchestration -InstanceId <String> [-Reason <String>]
```

### Suspend-DurableOrchestration

Suspends a running orchestration.

```powershell
Suspend-DurableOrchestration -InstanceId <String> [-Reason <String>]
```

### Resume-DurableOrchestration

Resumes a suspended orchestration.

```powershell
Resume-DurableOrchestration -InstanceId <String> [-Reason <String>]
```

### Send-DurableExternalEvent

Raises an event to a waiting orchestration.

```powershell
Send-DurableExternalEvent -InstanceId <String> -EventName <String> [-EventData <Object>]
```

**Example:**
```powershell
Send-DurableExternalEvent -InstanceId $instanceId -EventName 'ApprovalEvent' -EventData @{
    approved = $true
    approvedBy = "user@example.com"
}
```

### New-DurableRetryPolicy

Creates a retry policy for activities.

```powershell
New-DurableRetryPolicy -FirstRetryInterval <TimeSpan> -MaxNumberOfAttempts <Int32> 
    [-BackoffCoefficient <Double>] [-MaxRetryInterval <TimeSpan>] [-RetryTimeout <TimeSpan>]
```

**Example:**
```powershell
$retryPolicy = New-DurableRetryPolicy -FirstRetryInterval (New-TimeSpan -Seconds 5) `
    -MaxNumberOfAttempts 3 `
    -BackoffCoefficient 2.0

Invoke-DurableActivity -FunctionName 'UnreliableActivity' -Input $data -RetryPolicy $retryPolicy
```

---

## Common Patterns

### Function Chaining

```powershell
param($Context)

$x = Invoke-DurableActivity -FunctionName 'F1' -Input $null
$y = Invoke-DurableActivity -FunctionName 'F2' -Input $x
$z = Invoke-DurableActivity -FunctionName 'F3' -Input $y
$result = Invoke-DurableActivity -FunctionName 'F4' -Input $z

$result
```

### Fan-Out/Fan-In

```powershell
param($Context)

# Get work items
$workItems = Invoke-DurableActivity -FunctionName 'GetWorkItems'

# Fan-out: schedule parallel tasks
$tasks = @()
foreach ($item in $workItems) {
    $tasks += Invoke-DurableActivity -FunctionName 'ProcessItem' -Input $item -NoWait
}

# Fan-in: wait for all tasks to complete
$results = Wait-DurableTask -Task $tasks

# Aggregate results
$aggregated = Invoke-DurableActivity -FunctionName 'AggregateResults' -Input $results

$aggregated
```

### Async HTTP APIs

The HTTP starter function automatically returns status URLs:

```powershell
# Response includes:
# - statusQueryGetUri
# - sendEventPostUri  
# - terminatePostUri
# - purgeHistoryDeleteUri
# - suspendPostUri
# - resumePostUri
```

### Human Interaction with Timeout

```powershell
param($Context)

# Send approval request
Invoke-DurableActivity -FunctionName 'SendApprovalRequest' -Input $Context.Input

# Create timeout (3 days)
$timeout = New-TimeSpan -Days 3
$timerTask = Start-DurableTimer -Duration $timeout -NoWait

# Wait for approval event
$approvalTask = Start-DurableExternalEventListener -EventName 'ApprovalEvent' -NoWait

# Race: wait for either approval or timeout
$winner = Wait-DurableTask -Task @($timerTask, $approvalTask) -Any

if ($winner -eq $approvalTask -and $approvalTask.Result.approved) {
    Invoke-DurableActivity -FunctionName 'ProcessApproval' -Input $approvalTask.Result
    return @{ status = "Approved" }
} else {
    Invoke-DurableActivity -FunctionName 'Escalate' -Input $Context.Input
    return @{ status = "Escalated" }
}
```

### Monitor Pattern (Eternal Orchestration)

```powershell
param($Context)

$input = $Context.Input
$jobId = $input.jobId
$pollingInterval = New-TimeSpan -Seconds 60
$expiryTime = (Get-Date).AddHours(6)

while ((Get-Date) -lt $expiryTime) {
    $status = Invoke-DurableActivity -FunctionName 'CheckJobStatus' -Input $jobId
    
    if ($status -eq 'Completed') {
        Invoke-DurableActivity -FunctionName 'SendNotification' -Input $jobId
        return @{ status = "Completed" }
    }
    
    # Wait before next check
    Start-DurableTimer -Duration $pollingInterval
}

Invoke-DurableActivity -FunctionName 'SendTimeoutNotification' -Input $jobId
return @{ status = "Timeout" }
```

### Retry with Custom Policy

```powershell
param($Context)

$retryPolicy = New-DurableRetryPolicy `
    -FirstRetryInterval (New-TimeSpan -Seconds 5) `
    -MaxNumberOfAttempts 5 `
    -BackoffCoefficient 2.0 `
    -MaxRetryInterval (New-TimeSpan -Minutes 5)

try {
    $result = Invoke-DurableActivity -FunctionName 'UnreliableExternalCall' `
        -Input $Context.Input `
        -RetryPolicy $retryPolicy
    return @{ success = $true; result = $result }
} catch {
    return @{ success = $false; error = $_.Exception.Message }
}
```

---

## Limitations

{: .note }
> Some features available in other languages are not supported in PowerShell.

| Feature | Status |
|---------|--------|
| Durable Entities | Not supported |
| Call HTTP | Not supported |
| Custom retry handlers | Limited support |

---

## Migration from Legacy SDK

If you're using the built-in legacy SDK, migrate to the standalone SDK:

### 1. Update local.settings.json

Add the external SDK flag:
```json
{
  "Values": {
    "ExternalDurablePowerShellSDK": "true"
  }
}
```

### 2. Update requirements.psd1

```powershell
@{
    'AzureFunctions.PowerShell.Durable.SDK' = '2.*'
}
```

### 3. Update profile.ps1

```powershell
Import-Module AzureFunctions.PowerShell.Durable.SDK -ErrorAction Stop
```

### 4. Update Cmdlet Names (if needed)

| Legacy Name | New Name |
|-------------|----------|
| `New-DurableRetryOptions` | `New-DurableRetryPolicy` |

### 5. Behavioral Changes

1. **Exception Handling**: Exceptions from activities in `Wait-DurableTask` are now propagated instead of silently ignored
2. **Null Values**: Null values are no longer dropped from `Wait-DurableTask` results

---

## Troubleshooting

### Module Import Errors

If you see module import errors:

1. Ensure `ExternalDurablePowerShellSDK` is set to `"true"` in local.settings.json
2. Verify the module is listed in requirements.psd1
3. Check that profile.ps1 imports the module
4. Delete `.python_packages` and `modules` folders and restart

### Connection Errors

If orchestrations fail to start:

1. Verify `AzureWebJobsStorage` connection string is valid
2. For local development, ensure Azurite or Azure Storage Emulator is running
3. Check firewall settings for Azure Storage access

### Replay Issues

If orchestrations behave unexpectedly:

1. Ensure all I/O operations are in activity functions
2. Use `$Context.CurrentUtcDateTime` instead of `Get-Date` in orchestrators
3. Avoid non-deterministic operations in orchestrators

---

## Resources

- [PowerShell Gallery - AzureFunctions.PowerShell.Durable.SDK](https://www.powershellgallery.com/packages/AzureFunctions.PowerShell.Durable.SDK)
- [Migration Guide](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-powershell-v2-sdk-migration-guide)
- [PowerShell Quickstart](https://learn.microsoft.com/azure/azure-functions/durable/quickstart-powershell-vscode)
- [GitHub - PowerShell SDK](https://github.com/Azure/azure-functions-durable-powershell)
