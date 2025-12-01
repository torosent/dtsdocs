---
layout: default
title: Portable SDK
nav_order: 2
parent: Node.js SDK
grand_parent: SDKs
permalink: /developer-guide/sdks/nodejs/durabletask-sdk/
---

# Portable durabletask SDK for Node.js

{: .important }
> The portable durabletask SDK for Node.js is **coming soon**. This page provides information about the planned SDK and its expected capabilities.

## Overview

The portable durabletask SDK for Node.js will enable developers to build durable workflows in JavaScript and TypeScript that can run outside of Azure Functions, with support for multiple backend providers including the Durable Task Scheduler.

## Current Status

| Status | Details |
|--------|---------|
| Availability | Coming Soon |
| Repository | TBD |
| npm Package | TBD |

## Planned Features

The portable Node.js SDK is expected to include:

- **Backend Agnostic** - Run with Durable Task Scheduler, Azure Storage, or other providers
- **Hosting Flexibility** - Deploy on Kubernetes, Container Apps, VMs, or any Node.js environment
- **TypeScript Support** - Full TypeScript type definitions
- **Familiar API** - Similar programming model to the Azure Functions durable-functions package
- **Orchestrations & Activities** - Support for workflows, activities, and timers

## Expected Architecture

```
┌─────────────────────────────────────────┐
│           Your Application              │
├─────────────────────────────────────────┤
│     durabletask SDK for Node.js         │
├─────────────────────────────────────────┤
│         Backend Provider                │
│  (Durable Task Scheduler / Storage)     │
└─────────────────────────────────────────┘
```

## Current Alternatives

While the portable Node.js SDK is in development, you can use:

### Azure Functions with durable-functions

The recommended approach for Node.js durable workflows is currently the **durable-functions** npm package with Azure Functions:

```typescript
import * as df from "durable-functions";

const orchestrator: df.OrchestrationHandler = function* (context) {
    const result1 = yield context.df.callActivity("Activity1", "input1");
    const result2 = yield context.df.callActivity("Activity2", result1);
    return result2;
};

df.app.orchestration("myOrchestrator", orchestrator);
```

See [Azure Functions SDK](durable-functions) for complete documentation.

### Other Language SDKs

If you need a portable SDK today, consider:

| Language | Portable SDK | Status |
|----------|--------------|--------|
| .NET | Microsoft.DurableTask | Available |
| Python | durabletask | Available |
| Java | durabletask-java | Available |

## Expected API Preview

{: .note }
> The following API is speculative and subject to change when the SDK is released.

Based on other portable SDKs in the Durable Task family, the Node.js portable SDK may look similar to:

```typescript
// Hypothetical API - subject to change
import { TaskOrchestrationContext, task } from "@durabletask/durabletask";

// Define an orchestration
@task.orchestration
async function* myOrchestration(context: TaskOrchestrationContext) {
    const result = yield context.callActivity("processData", { input: "value" });
    return result;
}

// Define an activity
@task.activity
async function processData(input: any): Promise<any> {
    // Process the input
    return { processed: input };
}

// Create and start a worker
const worker = new DurableTaskWorker({
    connectionString: "your-durable-task-scheduler-connection",
    orchestrations: [myOrchestration],
    activities: [processData]
});

await worker.start();
```

## Migration Path

When the portable SDK becomes available, migration from durable-functions will involve:

1. **Package Change** - Switch from `durable-functions` to the portable package
2. **Worker Setup** - Add explicit worker configuration
3. **Backend Configuration** - Configure connection to Durable Task Scheduler
4. **API Adaptation** - Adjust to any API differences (expected to be minimal)

## Stay Updated

For updates on the portable Node.js SDK:

- Watch the [Durable Task Framework GitHub](https://github.com/Azure/durabletask)
- Check [Azure Functions Durable JS](https://github.com/Azure/azure-functions-durable-js) for announcements
- Follow [Azure updates](https://azure.microsoft.com/updates/)

## Request the SDK

If you have a specific need for the portable Node.js SDK, consider:

1. Opening an issue on the Durable Task Framework GitHub repository
2. Providing your use case and requirements
3. Engaging with the community discussion

## Related Resources

- [Azure Functions SDK Reference](durable-functions) - Current Node.js SDK for Azure Functions
- [Python Portable SDK](../python/durabletask-sdk) - Available portable SDK for Python
- [.NET Portable SDK](../dotnet/api-reference-portable) - Available portable SDK for .NET
