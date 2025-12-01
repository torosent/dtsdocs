---
layout: default
title: WebJobs Integration
parent: .NET SDK
grand_parent: SDKs Overview
nav_order: 3
permalink: /docs/developer-guide/sdks/dotnet/webjobs/
---

# Running Durable Functions as WebJobs
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Guide for running Durable Functions in Azure WebJobs.
{: .fs-6 .fw-300 }

---

## Overview

Durable Functions can run outside of Azure Functions using the WebJobs SDK. This approach is useful when you need:

- More control over the hosting environment
- Integration with existing WebJob applications
- Different scaling or deployment requirements

Behind the scenes, Durable Functions is built on the Durable Task Framework, which can run in WebJobs.

---

## Prerequisites

- Visual Studio 2019 or later with **Azure development** workload
- .NET Framework 4.6.1 or later
- Familiarity with the WebJobs SDK
- Azure Storage account or Azurite emulator

---

## Create Console Application

1. Create a new Console App project:

```
File → New Project → Console App (.NET Framework)
```

2. Target .NET Framework 4.6.1 or later

---

## Install NuGet Packages

Install the required packages:

```powershell
# Core WebJobs SDK packages
Install-Package Microsoft.Azure.WebJobs.Extensions -version 2.2.0
Install-Package Microsoft.Extensions.Logging -version 2.0.1

# Durable Task extension
Install-Package Microsoft.Azure.WebJobs.Extensions.DurableTask -version 1.8.7

# Logging providers
Install-Package Microsoft.Azure.WebJobs.Logging.ApplicationInsights -version 2.2.0
Install-Package System.Configuration.ConfigurationManager -version 4.4.1
Install-Package Microsoft.Extensions.Logging.Console -version 2.0.1
```

> **Note:** Version numbers shown are examples. Use the latest stable versions.

---

## Configure JobHost

Configure the `JobHost` in your `Main` method:

```csharp
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using Microsoft.Extensions.Logging;
using System;

class Program
{
    static void Main(string[] args)
    {
        // Create configuration
        var config = new JobHostConfiguration();
        
        // Configure Durable Task extension
        config.UseDurableTask(new DurableTaskExtension
        {
            HubName = "MyTaskHub",
            // Additional configuration options
            MaxConcurrentActivityFunctions = 10,
            MaxConcurrentOrchestratorFunctions = 5
        });
        
        // Configure logging
        config.LoggerFactory = ConfigureLogging();
        
        // Enable development settings
        if (config.IsDevelopment)
        {
            config.UseDevelopmentSettings();
        }
        
        // Create and run the host
        using (var host = new JobHost(config))
        {
            host.RunAndBlock();
        }
    }
    
    private static ILoggerFactory ConfigureLogging()
    {
        var loggerFactory = new LoggerFactory();
        
        // Add console logging
        loggerFactory.AddConsole(LogLevel.Information);
        
        // Add Application Insights (optional)
        var instrumentationKey = Environment.GetEnvironmentVariable("APPINSIGHTS_INSTRUMENTATIONKEY");
        if (!string.IsNullOrEmpty(instrumentationKey))
        {
            loggerFactory.AddApplicationInsights(instrumentationKey, null);
        }
        
        return loggerFactory;
    }
}
```

---

## Define Functions

Define your orchestrator and activity functions:

### Orchestrator Function

```csharp
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using System.Collections.Generic;
using System.Threading.Tasks;

public static class HelloSequence
{
    [FunctionName("HelloSequence")]
    public static async Task<List<string>> Run(
        [OrchestrationTrigger] IDurableOrchestrationContext context)
    {
        var outputs = new List<string>();
        
        outputs.Add(await context.CallActivityAsync<string>("SayHello", "Tokyo"));
        outputs.Add(await context.CallActivityAsync<string>("SayHello", "Seattle"));
        outputs.Add(await context.CallActivityAsync<string>("SayHello", "London"));
        
        return outputs;
    }
}
```

### Activity Function

```csharp
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using Microsoft.Extensions.Logging;

public static class SayHello
{
    [FunctionName("SayHello")]
    public static string Run(
        [ActivityTrigger] string name,
        ILogger log)
    {
        log.LogInformation($"Saying hello to {name}.");
        return $"Hello {name}!";
    }
}
```

---

## Start Orchestrations

### HTTP Trigger

Add an HTTP trigger to start orchestrations:

```csharp
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using Microsoft.Azure.WebJobs.Extensions.Http;
using System.Net.Http;
using System.Threading.Tasks;

public static class HttpStart
{
    [FunctionName("HttpStart")]
    public static async Task<HttpResponseMessage> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "orchestrators/{functionName}")] 
        HttpRequestMessage req,
        [DurableClient] IDurableOrchestrationClient starter,
        string functionName)
    {
        // Get input from request body
        var input = await req.Content.ReadAsAsync<object>();
        
        // Start the orchestration
        string instanceId = await starter.StartNewAsync(functionName, input);
        
        // Return status endpoints
        return starter.CreateCheckStatusResponse(req, instanceId);
    }
}
```

### Queue Trigger

Start orchestrations from queue messages:

```csharp
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using System.Threading.Tasks;

public static class QueueStart
{
    [FunctionName("QueueStart")]
    public static async Task Run(
        [QueueTrigger("orchestration-queue")] OrchestrationRequest request,
        [DurableClient] IDurableOrchestrationClient starter)
    {
        await starter.StartNewAsync(request.FunctionName, request.Input);
    }
}

public class OrchestrationRequest
{
    public string FunctionName { get; set; }
    public object Input { get; set; }
}
```

### Programmatic Start

Start orchestrations programmatically:

```csharp
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using System.Threading.Tasks;

public static class OrchestrationStarter
{
    public static async Task StartOrchestration(
        IDurableOrchestrationClient client,
        string functionName,
        object input)
    {
        string instanceId = await client.StartNewAsync(functionName, input);
        Console.WriteLine($"Started orchestration: {instanceId}");
    }
}
```

---

## Configuration

### App.config

Configure connection strings in `App.config`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <connectionStrings>
    <add name="AzureWebJobsStorage" 
         connectionString="UseDevelopmentStorage=true" />
    <add name="AzureWebJobsDashboard" 
         connectionString="UseDevelopmentStorage=true" />
  </connectionStrings>
  <appSettings>
    <add key="APPINSIGHTS_INSTRUMENTATIONKEY" value="your-key-here" />
  </appSettings>
</configuration>
```

### DurableTaskExtension Options

Configure extension options:

```csharp
config.UseDurableTask(new DurableTaskExtension
{
    // Task hub name (storage container prefix)
    HubName = "MyTaskHub",
    
    // Concurrency settings
    MaxConcurrentActivityFunctions = 10,
    MaxConcurrentOrchestratorFunctions = 5,
    
    // Extended sessions for performance
    ExtendedSessionsEnabled = true,
    ExtendedSessionIdleTimeoutInSeconds = 30,
    
    // Control queue settings
    ControlQueueBatchSize = 32,
    PartitionCount = 4,
    
    // Event Grid notifications
    EventGridTopicEndpoint = "https://your-topic.region.eventgrid.azure.net/api/events",
    EventGridKeySettingName = "EventGridKey"
});
```

---

## Storage Providers

### Azure Storage (Default)

The default storage provider uses Azure Storage:

```csharp
config.UseDurableTask(new DurableTaskExtension
{
    HubName = "MyTaskHub",
    StorageProvider = new AzureStorageOptions
    {
        ConnectionStringName = "AzureWebJobsStorage",
        TrackingStoreConnectionStringName = "AzureWebJobsStorage"
    }
});
```

### Netherite Storage Provider

For high-performance scenarios:

```powershell
Install-Package Microsoft.Azure.DurableTask.Netherite
```

```csharp
config.UseDurableTask(new DurableTaskExtension
{
    HubName = "MyTaskHub",
    StorageProvider = new NetheriteStorageOptions
    {
        EventHubsConnectionStringName = "EventHubsConnection",
        StorageConnectionStringName = "AzureWebJobsStorage"
    }
});
```

---

## WebJobs SDK 3.x

For WebJobs SDK 3.x (equivalent to Azure Functions 2.x+):

### Package Updates

```powershell
Install-Package Microsoft.Azure.WebJobs -version 3.0.0
Install-Package Microsoft.Azure.WebJobs.Extensions.DurableTask -version 2.0.0
```

### Host Builder Configuration

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using System.Threading.Tasks;

class Program
{
    static async Task Main(string[] args)
    {
        var host = new HostBuilder()
            .ConfigureWebJobs(builder =>
            {
                builder.AddAzureStorageCoreServices();
                builder.AddDurableTask(options =>
                {
                    options.HubName = "MyTaskHub";
                });
            })
            .ConfigureLogging((context, builder) =>
            {
                builder.AddConsole();
            })
            .Build();
        
        await host.RunAsync();
    }
}
```

---

## Deployment

### Deploy to Azure App Service

1. Create a WebJob-enabled App Service
2. Publish your console app as a WebJob
3. Configure connection strings in Application Settings

### Configure Always On

Enable "Always On" in App Service to keep the WebJob running:

1. Navigate to App Service → Configuration → General settings
2. Enable "Always On"

### Scaling

WebJobs scale differently than Azure Functions:

- **Single Instance**: One WebJob runs per App Service instance
- **Scale Out**: Scale the App Service plan to add instances
- **Manual Control**: Configure instance count directly

---

## Monitoring

### Application Insights

Enable Application Insights for monitoring:

```csharp
var loggerFactory = new LoggerFactory();
loggerFactory.AddApplicationInsights(instrumentationKey, null);

config.LoggerFactory = loggerFactory;
```

### Durable Functions Monitor

The Durable Functions Monitor extension can connect to WebJobs-hosted task hubs:

1. Install the extension in VS Code
2. Configure connection string
3. Connect to your task hub

---

## Limitations

When running Durable Functions in WebJobs:

| Feature | Supported | Notes |
|---------|-----------|-------|
| Orchestrations | ✅ | Full support |
| Activities | ✅ | Full support |
| Entities | ✅ | Full support |
| Timers | ✅ | Full support |
| External Events | ✅ | Full support |
| HTTP Management APIs | ❌ | Implement manually |
| Auto-scaling | ❌ | Manual scaling required |
| Consumption billing | ❌ | App Service pricing |

---

## Sample Project

Complete sample project structure:

```
MyDurableWebJob/
├── Program.cs
├── App.config
├── Functions/
│   ├── Orchestrators/
│   │   └── HelloSequence.cs
│   ├── Activities/
│   │   └── SayHello.cs
│   └── Triggers/
│       └── HttpStart.cs
└── packages.config
```

### Program.cs

```csharp
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.DurableTask;
using Microsoft.Extensions.Logging;
using System;

namespace MyDurableWebJob
{
    class Program
    {
        static void Main(string[] args)
        {
            var config = new JobHostConfiguration
            {
                JobActivator = new MyActivator()
            };
            
            config.UseDurableTask(new DurableTaskExtension
            {
                HubName = Environment.GetEnvironmentVariable("TASK_HUB") ?? "DefaultTaskHub"
            });
            
            var loggerFactory = new LoggerFactory();
            loggerFactory.AddConsole(LogLevel.Information);
            config.LoggerFactory = loggerFactory;
            
            using (var host = new JobHost(config))
            {
                Console.WriteLine("Starting WebJob host...");
                host.RunAndBlock();
            }
        }
    }
}
```
