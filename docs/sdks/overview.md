---
layout: default
title: Overview
parent: Durable Task SDKs
nav_order: 1
permalink: /docs/sdks/overview/
---

# Durable Task SDKs
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

The **Durable Task SDKs** are portable client libraries for building durable orchestrations that can run on any compute platform. Unlike Durable Functions, they don't require Azure Functions as a hosting environment.

---

## Overview

The Durable Task SDKs connect directly to the Durable Task Scheduler, allowing you to run orchestrations on:

- Azure Container Apps (ACA)
- Azure Kubernetes Service (AKS)
- Azure App Service
- Virtual Machines
- On-premises servers

```
┌──────────────────────────────────────────────────────────────────┐
│                    YOUR APPLICATION                              │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Durable Task SDK                                          │ │
│  │  ├── Worker (processes orchestrations/activities)          │ │
│  │  └── Client (schedules/manages orchestrations)             │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              │                                   │
│                         gRPC │                                   │
│                              ▼                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Durable Task Scheduler                                    │ │
│  │  ├── Orchestration Engine                                  │ │
│  │  ├── State Storage                                         │ │
│  │  └── Monitoring Dashboard                                  │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

---

## Available SDKs

| Language | Package | Status | Documentation |
|----------|---------|--------|---------------|
| **.NET** | `Microsoft.DurableTask.*` | ✅ GA | [.NET SDK Guide](./dotnet.md) |
| **Python** | `durabletask-azure` | ✅ GA | [Python SDK Guide](./python.md) |
| **Java** | `com.microsoft.durabletask` | ⚠️ Preview | [Java SDK Guide](./java.md) |
| **JavaScript** | Coming soon | 🔜 Planned | - |

---

## SDK vs Durable Functions

| Aspect | Durable Task SDKs | Durable Functions |
|--------|-------------------|-------------------|
| **Hosting** | Any compute platform | Azure Functions |
| **Programming Model** | Standalone library | Functions extension |
| **Triggers** | Custom (HTTP, timers, etc.) | Functions bindings |
| **Scaling** | Manual or platform-specific | Functions auto-scaling |
| **Language Support** | .NET, Python, Java | C#, JS, Python, Java, PowerShell |
| **Entity Functions** | Limited | Full support |

---

## When to Use the SDKs

### ✅ Great For

- **Container-based workloads** — Running on ACA, AKS, or Docker
- **Custom hosting requirements** — VMs, on-premises, multi-cloud
- **Microservice architectures** — Integrating with existing services
- **Non-Functions projects** — Web APIs, background services

### ⚠️ Consider Durable Functions Instead

- **Serverless scenarios** — Pay-per-execution pricing
- **Azure integrations** — Leverage Functions bindings
- **Multi-language teams** — PowerShell support
- **Full entity support** — Complex entity scenarios

---

## Architecture

### Worker Pattern

The SDKs follow a worker-client pattern:

```
┌─────────────────────────────────────────────────────────────────┐
│                        WORKER PROCESS                           │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  DurableTaskWorker                                        │ │
│  │  ├── Receives work items from scheduler                   │ │
│  │  ├── Executes orchestrators                               │ │
│  │  ├── Executes activities                                  │ │
│  │  └── Reports results back to scheduler                    │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT PROCESS                           │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  DurableTaskClient                                        │ │
│  │  ├── Schedules new orchestrations                         │ │
│  │  ├── Queries orchestration status                         │ │
│  │  ├── Waits for completion                                 │ │
│  │  └── Manages orchestration lifecycle                      │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Deployment Topology

Typical deployment patterns:

#### Single Process (Worker + Client)

```
┌─────────────────────────────────────┐
│           APPLICATION               │
│  ┌─────────────────────────────┐   │
│  │  Worker + Client            │   │
│  │  ├── HTTP API (client)      │   │
│  │  └── Background (worker)    │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
                │
                ▼
      ┌─────────────────┐
      │    Scheduler    │
      └─────────────────┘
```

#### Separate Processes

```
┌─────────────────┐    ┌─────────────────┐
│   API Service   │    │  Worker Service │
│   (client only) │    │  (worker only)  │
└────────┬────────┘    └────────┬────────┘
         │                      │
         └──────────┬───────────┘
                    ▼
          ┌─────────────────┐
          │    Scheduler    │
          └─────────────────┘
```

---

## Quick Start

### .NET

```bash
# Create a new project
dotnet new console -n MyOrchestrationApp
cd MyOrchestrationApp

# Add the SDK packages
dotnet add package Microsoft.DurableTask.Worker.AzureManaged
dotnet add package Microsoft.DurableTask.Client.AzureManaged
```

```csharp
// Program.cs
using Microsoft.DurableTask;
using Microsoft.DurableTask.Worker;
using Microsoft.DurableTask.Client;

var builder = Host.CreateApplicationBuilder(args);

// Configure the worker
builder.Services.AddDurableTaskWorker(options =>
{
    options.AddOrchestrator<HelloCitiesOrchestrator>();
    options.AddActivity<SayHelloActivity>();
})
.UseDurableTaskScheduler(
    Environment.GetEnvironmentVariable("DTS_CONNECTION_STRING"),
    Environment.GetEnvironmentVariable("TASKHUB_NAME") ?? "default"
);

// Configure the client
builder.Services.AddDurableTaskClient()
    .UseDurableTaskScheduler(
        Environment.GetEnvironmentVariable("DTS_CONNECTION_STRING"),
        Environment.GetEnvironmentVariable("TASKHUB_NAME") ?? "default"
    );

var host = builder.Build();
await host.RunAsync();
```

### Python

```bash
# Install the SDK
pip install durabletask-azure
```

```python
# worker.py
import os
from durabletask.worker import DurableTaskSchedulerWorker
from durabletask.task import task

connection_string = os.environ.get("DTS_CONNECTION_STRING")
task_hub = os.environ.get("TASKHUB_NAME", "default")

@task.orchestrator()
def hello_cities(ctx):
    result = []
    result.append(yield ctx.call_activity("say_hello", "Tokyo"))
    result.append(yield ctx.call_activity("say_hello", "London"))
    result.append(yield ctx.call_activity("say_hello", "Seattle"))
    return " ".join(result)

@task.activity()
def say_hello(ctx, city: str) -> str:
    return f"Hello, {city}!"

# Create and start the worker
worker = DurableTaskSchedulerWorker(connection_string, task_hub)
worker.add_orchestrator(hello_cities)
worker.add_activity(say_hello)
worker.start()
```

### Java

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.microsoft</groupId>
    <artifactId>durabletask-azure-functions</artifactId>
    <version>1.6.1</version>
</dependency>
```

```java
// App.java
import com.microsoft.durabletask.*;
import com.microsoft.durabletask.azuremanaged.*;

public class App {
    public static void main(String[] args) {
        String connectionString = System.getenv("DTS_CONNECTION_STRING");
        String taskHub = System.getenv("TASKHUB_NAME");
        
        DurableTaskWorker worker = DurableTaskSchedulerWorkerBuilder
            .forConnectionString(connectionString)
            .taskHub(taskHub)
            .addOrchestrator("HelloCities", ctx -> {
                StringBuilder result = new StringBuilder();
                result.append(ctx.callActivity("SayHello", "Tokyo", String.class).await());
                result.append(" ");
                result.append(ctx.callActivity("SayHello", "London", String.class).await());
                result.append(" ");
                result.append(ctx.callActivity("SayHello", "Seattle", String.class).await());
                return result.toString();
            })
            .addActivity("SayHello", ctx -> {
                String city = ctx.getInput(String.class);
                return "Hello, " + city + "!";
            })
            .build();
        
        worker.start();
    }
}
```

---

## NuGet Packages (.NET)

| Package | Description |
|---------|-------------|
| `Microsoft.DurableTask.Abstractions` | Core abstractions (shared between worker/client) |
| `Microsoft.DurableTask.Worker` | Base worker functionality |
| `Microsoft.DurableTask.Worker.AzureManaged` | Durable Task Scheduler integration |
| `Microsoft.DurableTask.Client` | Base client functionality |
| `Microsoft.DurableTask.Client.AzureManaged` | Durable Task Scheduler client |
| `Microsoft.DurableTask.Generators` | Source generators for type-safe APIs |

---

## Local Development

### Run the Emulator

```bash
docker run -itP mcr.microsoft.com/dts/dts-emulator:latest
```

### Connection String

```
Endpoint=http://localhost:<port>;Authentication=None
```

### Dashboard

Open `http://localhost:<dashboard-port>` to view orchestrations.

---

## Deployment Guides

- [Deploy to Azure Container Apps →](../architecture/aca-dts.md)
- [Deploy to Azure Kubernetes Service →](../architecture/aks-dts.md)

---

## Next Steps

- [.NET SDK Details →](./dotnet.md)
- [Python SDK Details →](./python.md)
- [Java SDK Details →](./java.md)
- [View Code Samples →](./samples.md)
