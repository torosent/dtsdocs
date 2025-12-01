---
layout: default
title: Migration Guide
parent: Azure Functions (Durable Functions)
grand_parent: Hosting Options
nav_order: 6
permalink: /docs/hosting-options/azure-functions/migration-guide/
---

# Migration Guide: In-Process to Isolated Worker Model
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

This guide walks you through migrating your Durable Functions application from the in-process model to the isolated worker model.
{: .fs-6 .fw-300 }

{: .warning }
> **Important:** Support for the in-process model ends on **November 10, 2026**. We highly recommend migrating to the isolated worker model for continued support and access to new features.

---

## Why Migrate?

### In-Process Model End of Support

Microsoft has announced that the in-process model for .NET Azure Functions will reach end of support on November 10, 2026. After this date:

- No security updates will be provided
- No bug fixes will be released
- New features will only be available in the isolated worker model

### Benefits of the Isolated Worker Model

Migrating to the isolated worker model provides:

| Benefit | Description |
|---------|-------------|
| **No Assembly Conflicts** | Your code runs in a separate process, eliminating version conflicts |
| **Full Process Control** | Control startup, configuration, and middleware via `Program.cs` |
| **Standard DI Patterns** | Use familiar .NET dependency injection |
| **.NET Version Flexibility** | Support for LTS, STS, and .NET Framework |
| **Middleware Support** | Full ASP.NET Core middleware pipeline |
| **Better Performance** | ASP.NET Core integration for HTTP triggers |
| **Platform Support** | Access to Flex Consumption plan and .NET Aspire |

---

## Prerequisites

Before starting the migration:

- **Azure Functions Core Tools v4.x** or later
- **.NET 8.0 SDK** (or your target .NET version)
- **Visual Studio 2022** or **VS Code with Azure Functions extension**
- Familiarity with [Durable Functions concepts](../../concepts/orchestrators.md)

---

## Migration Overview

The migration process involves these main steps:

1. [Identify apps to migrate](#step-1-identify-apps-to-migrate)
2. [Update the project file](#step-2-update-the-project-file)
3. [Add Program.cs](#step-3-add-programcs)
4. [Update package references](#step-4-update-package-references)
5. [Update function code](#step-5-update-function-code)
6. [Update local.settings.json](#step-6-update-localsettingsjson)
7. [Test locally](#step-7-test-locally)
8. [Deploy to Azure](#step-8-deploy-to-azure)

---

## Step 1: Identify Apps to Migrate

Use this Azure PowerShell script to find function apps in your subscription using the in-process model:

```powershell
$FunctionApps = Get-AzFunctionApp

$AppInfo = @{}

foreach ($App in $FunctionApps)
{
     if ($App.Runtime -eq 'dotnet')
     {
          $AppInfo.Add($App.Name, $App.Runtime)
     }
}

$AppInfo
```

Apps showing `dotnet` as the runtime are using the in-process model. Apps using `dotnet-isolated` are already on the isolated worker model.

---

## Step 2: Update the Project File

### Before (In-Process)

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <AzureFunctionsVersion>v4</AzureFunctionsVersion>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Sdk.Functions" Version="4.1.1" />
    <PackageReference Include="Microsoft.Azure.WebJobs.Extensions.DurableTask" Version="2.13.0" />
  </ItemGroup>
</Project>
```

### After (Isolated Worker)

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <AzureFunctionsVersion>v4</AzureFunctionsVersion>
    <OutputType>Exe</OutputType>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker" Version="1.21.0" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="1.17.2" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http.AspNetCore" Version="1.2.1" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.DurableTask" Version="1.1.0" />
    <PackageReference Include="Microsoft.ApplicationInsights.WorkerService" Version="2.22.0" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.ApplicationInsights" Version="1.2.0" />
  </ItemGroup>
  <ItemGroup>
    <Using Include="System.Threading.ExecutionContext" Alias="ExecutionContext"/>
  </ItemGroup>
</Project>
```

### Key Changes

1. Add `<OutputType>Exe</OutputType>` - The isolated worker is an executable
2. Add `<FrameworkReference Include="Microsoft.AspNetCore.App" />` - For ASP.NET Core integration
3. Replace `Microsoft.NET.Sdk.Functions` with `Microsoft.Azure.Functions.Worker.*` packages
4. Replace `Microsoft.Azure.WebJobs.Extensions.DurableTask` with `Microsoft.Azure.Functions.Worker.Extensions.DurableTask`

---

## Step 3: Add Program.cs

Create a new `Program.cs` file in your project root:

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var host = new HostBuilder()
    .ConfigureFunctionsWebApplication()
    .ConfigureServices(services => {
        services.AddApplicationInsightsTelemetryWorkerService();
        services.ConfigureFunctionsApplicationInsights();
    })
    .Build();

host.Run();
```

### With Custom Services

If you had a `FunctionsStartup` class, move that configuration to `Program.cs`:

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var host = new HostBuilder()
    .ConfigureFunctionsWebApplication()
    .ConfigureServices(services => {
        // Application Insights
        services.AddApplicationInsightsTelemetryWorkerService();
        services.ConfigureFunctionsApplicationInsights();
        
        // Your custom services (previously in FunctionsStartup)
        services.AddSingleton<IMyService, MyService>();
        services.AddHttpClient<IApiClient, ApiClient>();
    })
    .Build();

host.Run();
```

### Delete FunctionsStartup

If you have a `Startup.cs` with `[assembly: FunctionsStartup(...)]`, delete it after moving the configuration to `Program.cs`.

---

## Step 4: Update Package References

### Durable Functions Package Changes

| In-Process Package | Isolated Worker Package |
|--------------------|------------------------|
| `Microsoft.Azure.WebJobs.Extensions.DurableTask` | `Microsoft.Azure.Functions.Worker.Extensions.DurableTask` |
| `Microsoft.DurableTask.SqlServer.AzureFunctions` | `Microsoft.Azure.Functions.Worker.Extensions.DurableTask.SqlServer` |
| `Microsoft.Azure.DurableTask.Netherite.AzureFunctions` | `Microsoft.Azure.Functions.Worker.Extensions.DurableTask.Netherite` |

### Common Extension Package Changes

| In-Process | Isolated Worker |
|------------|-----------------|
| `Microsoft.Azure.WebJobs.Extensions.Storage` | `Microsoft.Azure.Functions.Worker.Extensions.Storage.Blobs`, `.Queues`, `.Tables` |
| `Microsoft.Azure.WebJobs.Extensions.CosmosDB` | `Microsoft.Azure.Functions.Worker.Extensions.CosmosDB` |
| `Microsoft.Azure.WebJobs.Extensions.ServiceBus` | `Microsoft.Azure.Functions.Worker.Extensions.ServiceBus` |
| `Microsoft.Azure.WebJobs.Extensions.EventHubs` | `Microsoft.Azure.Functions.Worker.Extensions.EventHubs` |
| `Microsoft.Azure.WebJobs.Extensions.EventGrid` | `Microsoft.Azure.Functions.Worker.Extensions.EventGrid` |

{: .important }
> Remove any references to `Microsoft.Azure.WebJobs.*` namespaces and `Microsoft.Azure.Functions.Extensions` from your project.

---

## Step 5: Update Function Code

### Namespace Changes

```csharp
// Before (In-Process)
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using Microsoft.Azure.WebJobs.Extensions.Http;

// After (Isolated Worker)
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.DurableTask;
using Microsoft.DurableTask.Client;
```

### Function Attribute Changes

```csharp
// Before (In-Process)
[FunctionName("MyOrchestrator")]

// After (Isolated Worker)
[Function(nameof(MyOrchestrator))]
```

### Orchestrator Function Changes

**Before (In-Process):**

```csharp
[FunctionName("OrderOrchestrator")]
public static async Task<OrderResult> RunOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context,
    ILogger log)
{
    var order = context.GetInput<Order>();
    
    await context.CallActivityAsync("ValidateOrder", order);
    await context.CallActivityAsync("ProcessPayment", order.Payment);
    await context.CallActivityAsync("ShipOrder", order);
    
    return new OrderResult { Success = true };
}
```

**After (Isolated Worker):**

```csharp
[Function(nameof(OrderOrchestrator))]
public static async Task<OrderResult> OrderOrchestrator(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    ILogger logger = context.CreateReplaySafeLogger(nameof(OrderOrchestrator));
    var order = context.GetInput<Order>();
    
    await context.CallActivityAsync("ValidateOrder", order);
    await context.CallActivityAsync("ProcessPayment", order.Payment);
    await context.CallActivityAsync("ShipOrder", order);
    
    return new OrderResult { Success = true };
}
```

### Key Differences

| Aspect | In-Process | Isolated Worker |
|--------|------------|-----------------|
| Context type | `IDurableOrchestrationContext` | `TaskOrchestrationContext` |
| Logger | `ILogger` parameter | `context.CreateReplaySafeLogger()` |
| Attribute | `[FunctionName]` | `[Function]` |

### Activity Function Changes

**Before (In-Process):**

```csharp
[FunctionName("ValidateOrder")]
public static bool ValidateOrder(
    [ActivityTrigger] Order order,
    ILogger log)
{
    log.LogInformation("Validating order {OrderId}", order.Id);
    return order.Items.Any() && order.TotalAmount > 0;
}
```

**After (Isolated Worker):**

```csharp
[Function(nameof(ValidateOrder))]
public static bool ValidateOrder(
    [ActivityTrigger] Order order,
    FunctionContext executionContext)
{
    ILogger logger = executionContext.GetLogger(nameof(ValidateOrder));
    logger.LogInformation("Validating order {OrderId}", order.Id);
    return order.Items.Any() && order.TotalAmount > 0;
}
```

### Client Function Changes

**Before (In-Process):**

```csharp
[FunctionName("StartOrder")]
public static async Task<IActionResult> StartOrder(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
    [DurableClient] IDurableOrchestrationClient client,
    ILogger log)
{
    var order = await req.ReadFromJsonAsync<Order>();
    string instanceId = await client.StartNewAsync("OrderOrchestrator", order);
    
    return client.CreateCheckStatusResponse(req, instanceId);
}
```

**After (Isolated Worker):**

```csharp
[Function("StartOrder")]
public static async Task<HttpResponseData> StartOrder(
    [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req,
    [DurableClient] DurableTaskClient client,
    FunctionContext executionContext)
{
    ILogger logger = executionContext.GetLogger("StartOrder");
    var order = await req.ReadFromJsonAsync<Order>();
    string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
        nameof(OrderOrchestrator), 
        order
    );
    
    return await client.CreateCheckStatusResponseAsync(req, instanceId);
}
```

### Client Type Changes

| In-Process | Isolated Worker |
|------------|-----------------|
| `IDurableOrchestrationClient` | `DurableTaskClient` |
| `StartNewAsync()` | `ScheduleNewOrchestrationInstanceAsync()` |
| `CreateCheckStatusResponse()` | `CreateCheckStatusResponseAsync()` |
| `HttpRequest` / `IActionResult` | `HttpRequestData` / `HttpResponseData` |

### Entity Function Changes

**Before (In-Process):**

```csharp
[FunctionName(nameof(Counter))]
public static void Counter([EntityTrigger] IDurableEntityContext ctx)
{
    switch (ctx.OperationName.ToLowerInvariant())
    {
        case "add":
            ctx.SetState(ctx.GetState<int>() + ctx.GetInput<int>());
            break;
        case "get":
            ctx.Return(ctx.GetState<int>());
            break;
    }
}
```

**After (Isolated Worker):**

```csharp
[Function(nameof(Counter))]
public static Task Counter([EntityTrigger] TaskEntityDispatcher dispatcher)
{
    return dispatcher.DispatchAsync<CounterEntity>();
}

public class CounterEntity
{
    public int Value { get; set; }
    
    public void Add(int amount) => Value += amount;
    public int Get() => Value;
}
```

---

## Step 6: Update local.settings.json

```json
{
    "IsEncrypted": false,
    "Values": {
        "AzureWebJobsStorage": "UseDevelopmentStorage=true",
        "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
        "DURABLE_TASK_SCHEDULER_CONNECTION_STRING": "Endpoint=http://localhost:8080;Authentication=None"
    }
}
```

The key change is `FUNCTIONS_WORKER_RUNTIME` from `dotnet` to `dotnet-isolated`.

---

## Step 7: Test Locally

### Start the Emulator

```bash
docker run -d -p 8080:8080 -p 8082:8082 mcr.microsoft.com/dts/dts-emulator:latest
```

### Run the Function App

```bash
func start
```

### Verify Functionality

Test all your orchestrations, activities, and entities to ensure they work correctly:

1. Start an orchestration via HTTP trigger
2. Monitor the orchestration status
3. Verify activity execution order
4. Test entity operations if applicable
5. Check Application Insights telemetry

---

## Step 8: Deploy to Azure

### Recommended: Use Deployment Slots

We recommend using deployment slots to minimize downtime:

1. **Create a staging slot** for your function app
2. **Update staging slot configuration:**
   - Set `FUNCTIONS_WORKER_RUNTIME` to `dotnet-isolated`
   - Update .NET stack version if needed
3. **Deploy migrated code** to the staging slot
4. **Test thoroughly** in the staging slot
5. **Perform slot swap** to move changes to production

### Update Application Settings

In the Azure Portal or via CLI:

```bash
az functionapp config appsettings set \
    --name <FUNCTION_APP_NAME> \
    --resource-group <RESOURCE_GROUP> \
    --settings FUNCTIONS_WORKER_RUNTIME=dotnet-isolated
```

### Update Stack Configuration

If targeting a different .NET version:

```bash
az functionapp config set \
    --name <FUNCTION_APP_NAME> \
    --resource-group <RESOURCE_GROUP> \
    --net-framework-version v8.0
```

---

## Common Migration Issues

### Issue: Assembly Load Errors

**Symptom:** `Could not load file or assembly` errors

**Solution:** Ensure you've removed all `Microsoft.Azure.WebJobs.*` package references and replaced with isolated worker equivalents.

### Issue: Binding Attribute Not Found

**Symptom:** `The type or namespace 'QueueTrigger' could not be found`

**Solution:** Add the appropriate extension package and update using statements:

```csharp
// Add using statement
using Microsoft.Azure.Functions.Worker;

// Install package
// dotnet add package Microsoft.Azure.Functions.Worker.Extensions.Storage.Queues
```

### Issue: IDurableOrchestrationContext Not Found

**Symptom:** `The type or namespace 'IDurableOrchestrationContext' could not be found`

**Solution:** Replace with `TaskOrchestrationContext`:

```csharp
using Microsoft.DurableTask;

[Function(nameof(MyOrchestrator))]
public static async Task MyOrchestrator([OrchestrationTrigger] TaskOrchestrationContext context)
{
    // ...
}
```

### Issue: JSON Serialization Differences

**Symptom:** Serialization errors or unexpected data formats

**Solution:** The isolated model uses `System.Text.Json` by default. Configure serialization in `Program.cs`:

```csharp
var host = new HostBuilder()
    .ConfigureFunctionsWebApplication()
    .ConfigureServices(services => {
        services.Configure<JsonSerializerOptions>(options => {
            options.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        });
    })
    .Build();
```

To use Newtonsoft.Json instead:

```csharp
services.Configure<WorkerOptions>(options => {
    options.Serializer = new NewtonsoftJsonObjectSerializer();
});
```

---

## Checklist

Use this checklist to ensure a complete migration:

- [ ] Updated project file with `<OutputType>Exe</OutputType>`
- [ ] Replaced `Microsoft.NET.Sdk.Functions` with worker packages
- [ ] Replaced `Microsoft.Azure.WebJobs.Extensions.DurableTask` with isolated package
- [ ] Created `Program.cs` with host configuration
- [ ] Removed `FunctionsStartup` class (if present)
- [ ] Updated all `[FunctionName]` to `[Function]`
- [ ] Replaced `IDurableOrchestrationContext` with `TaskOrchestrationContext`
- [ ] Replaced `IDurableOrchestrationClient` with `DurableTaskClient`
- [ ] Updated logging to use DI or `FunctionContext`
- [ ] Updated `local.settings.json` with `dotnet-isolated` runtime
- [ ] Removed all `Microsoft.Azure.WebJobs.*` using statements
- [ ] Added `Microsoft.Azure.Functions.Worker` using statements
- [ ] Tested all functions locally
- [ ] Deployed to staging slot and verified
- [ ] Swapped to production

---

## Next Steps

- [Learn about the Isolated Worker Model →](https://learn.microsoft.com/azure/azure-functions/dotnet-isolated-process-guide)
- [Explore Durable Functions Patterns →](../../patterns/index.md)
- [Configure Durable Task Scheduler →](../../durable-task-scheduler/setup.md)
- [View Code Samples →](./samples.md)

---

## Additional Resources

- [Official Microsoft Migration Guide](https://learn.microsoft.com/azure/azure-functions/migrate-dotnet-to-isolated-model)
- [Isolated Worker Model Differences](https://learn.microsoft.com/azure/azure-functions/dotnet-isolated-in-process-differences)
- [Durable Functions for .NET Isolated](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-dotnet-isolated-overview)
