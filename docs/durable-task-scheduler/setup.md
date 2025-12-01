---
layout: default
title: Setup Guide
parent: Durable Task Scheduler
nav_order: 2
permalink: /docs/durable-task-scheduler/setup/
---

# Setting Up Durable Task Scheduler
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

This guide walks you through creating and configuring a Durable Task Scheduler resource in Azure.

---

## Prerequisites

Before you begin, ensure you have:

- An Azure subscription
- Azure CLI installed (version 2.50.0 or later)
- Docker installed (for local development)

---

## Local Development Setup

Start with the local emulator before deploying to Azure.

### 1. Run the Emulator

```bash
# Pull the emulator image
docker pull mcr.microsoft.com/dts/dts-emulator:latest

# Run with default settings (single task hub named "default")
docker run -itP mcr.microsoft.com/dts/dts-emulator:latest

# Or run with multiple task hubs
docker run -itP \
  -e DTS_TASK_HUB_NAMES=dev,staging,prod \
  mcr.microsoft.com/dts/dts-emulator:latest
```

### 2. Note the Ports

The emulator exposes:
- **gRPC endpoint**: Random port mapped to container port 8080
- **Dashboard**: Random port mapped to container port 8082

Find the mapped ports:
```bash
docker ps
```

### 3. Connection String

For local development, use:
```
Endpoint=http://localhost:<grpc-port>;Authentication=None
```

---

## Azure Portal Setup

### 1. Create a Scheduler

1. Sign in to the [Azure portal](https://portal.azure.com)
2. Search for **Durable Task Scheduler** and select it
3. Click **Create**
4. Fill in the required fields:

   | Field | Description |
   |-------|-------------|
   | Subscription | Your Azure subscription |
   | Resource Group | Create new or select existing |
   | Name | Unique name for your scheduler |
   | Region | Azure region for deployment |

5. Click **Review + Create**, then **Create**

> **Note**: Deployment takes approximately 15-20 minutes.

### 2. Create a Task Hub

1. Navigate to your scheduler resource
2. In the left menu, select **Task Hubs**
3. Click **+ Create**
4. Enter a name for your task hub
5. Click **Create**

### 3. Get the Endpoint

1. Go to your scheduler's **Overview** page
2. Find the **Endpoint** in the Essentials section
3. Copy the endpoint URL (format: `{name}.{region}.durabletask.io`)

---

## Azure CLI Setup

### 1. Create a Resource Group

```bash
az group create \
  --name myDurableResourceGroup \
  --location westus2
```

### 2. Create a Scheduler

```bash
az durabletask scheduler create \
  --resource-group myDurableResourceGroup \
  --name myDurableScheduler \
  --location westus2
```

### 3. Create a Task Hub

```bash
az durabletask taskhub create \
  --resource-group myDurableResourceGroup \
  --scheduler-name myDurableScheduler \
  --name myTaskHub
```

### 4. Get the Endpoint

```bash
az durabletask scheduler show \
  --resource-group myDurableResourceGroup \
  --name myDurableScheduler \
  --query 'properties.endpoint' \
  --output tsv
```

---

## Configure Your Application

### Durable Functions Configuration

#### 1. Update host.json

```json
{
  "version": "2.0",
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "type": "durabletaskscheduler",
        "connectionStringName": "DURABLE_TASK_SCHEDULER_CONNECTION_STRING",
        "taskHubName": "%TASKHUB_NAME%"
      }
    }
  }
}
```

#### 2. Set Environment Variables

**Local (local.settings.json):**
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "DURABLE_TASK_SCHEDULER_CONNECTION_STRING": "Endpoint=http://localhost:8080;Authentication=None",
    "TASKHUB_NAME": "default"
  }
}
```

**Azure (App Settings):**
```bash
az functionapp config appsettings set \
  --resource-group myResourceGroup \
  --name myFunctionApp \
  --settings \
    "DURABLE_TASK_SCHEDULER_CONNECTION_STRING=Endpoint=myscheduler.westus2.durabletask.io;Authentication=ManagedIdentity;ClientID=<client-id>" \
    "TASKHUB_NAME=myTaskHub"
```

### Durable Task SDK Configuration

#### .NET

```csharp
// Program.cs
using Microsoft.DurableTask.Worker;
using Microsoft.DurableTask.Client;

var builder = Host.CreateApplicationBuilder(args);

// Get connection info from environment
var connectionString = Environment.GetEnvironmentVariable("DURABLE_TASK_SCHEDULER_CONNECTION_STRING");
var taskHubName = Environment.GetEnvironmentVariable("TASKHUB_NAME") ?? "default";

// Add the Durable Task worker
builder.Services.AddDurableTaskWorker(options =>
{
    options.AddOrchestrator<MyOrchestrator>();
    options.AddActivity<MyActivity>();
})
.UseDurableTaskScheduler(connectionString, taskHubName);

// Add the Durable Task client
builder.Services.AddDurableTaskClient()
    .UseDurableTaskScheduler(connectionString, taskHubName);

var host = builder.Build();
await host.RunAsync();
```

#### Python

```python
# worker.py
import os
from durabletask.worker import DurableTaskWorker

connection_string = os.environ.get("DURABLE_TASK_SCHEDULER_CONNECTION_STRING")
task_hub = os.environ.get("TASKHUB_NAME", "default")

worker = DurableTaskWorker(connection_string, task_hub)
worker.add_orchestrator(my_orchestrator)
worker.add_activity(my_activity)

worker.start()
```

#### Java

```java
// App.java
String connectionString = System.getenv("DURABLE_TASK_SCHEDULER_CONNECTION_STRING");
String taskHub = System.getenv("TASKHUB_NAME");

DurableTaskWorker worker = DurableTaskSchedulerWorkerBuilder
    .forConnectionString(connectionString)
    .taskHub(taskHub)
    .addOrchestrator(new MyOrchestrator())
    .addActivity(new MyActivity())
    .build();

worker.start();
```

---

## Connection String Formats

### Local Development (Emulator)

```
Endpoint=http://localhost:<port>;Authentication=None
```

### Azure with User-Assigned Managed Identity

```
Endpoint=<scheduler-endpoint>;Authentication=ManagedIdentity;ClientID=<client-id>
```

### Azure with System-Assigned Managed Identity

```
Endpoint=<scheduler-endpoint>;Authentication=ManagedIdentity
```

---

## Verify Your Setup

### Check Emulator Dashboard

1. Open `http://localhost:<dashboard-port>` in your browser
2. Click on your task hub
3. Start an orchestration and verify it appears

### Check Azure Dashboard

1. Navigate to your scheduler in the Azure portal
2. Click on your task hub
3. Click the **Dashboard URL** in the Essentials section
4. Verify you can see orchestration instances

---

## Common Issues

### "Failed to connect to Durable Task Scheduler"

- **Local**: Ensure the emulator is running and ports are correct
- **Azure**: Verify the endpoint URL and managed identity configuration

### "Access denied" or "Unauthorized"

- Ensure managed identity has the **Durable Task Data Contributor** role
- Check that the identity is assigned to your app

### "Task hub not found"

- Verify the task hub name matches exactly
- Ensure the task hub exists in the scheduler

---

## Next Steps

- [Configure Managed Identity →](./identity.md)
- [Explore the Dashboard →](./dashboard.md)
- [View Architecture Guides →](../architecture/index.md)
