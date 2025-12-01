---
layout: default
title: Node.js SDK
nav_order: 3
parent: SDKs
grand_parent: Developer Guide
has_children: true
permalink: /developer-guide/sdks/nodejs/
---

# Node.js SDK for Durable Task Framework

Build durable workflows in JavaScript and TypeScript using the Durable Functions npm packages.

## SDK Options

### Azure Functions - durable-functions

The **durable-functions** npm package provides Durable Functions support for Node.js Azure Functions applications with full integration with Azure Functions triggers and bindings.

| Property | Details |
|----------|---------|
| Package | [durable-functions](https://www.npmjs.com/package/durable-functions) |
| Programming Models | v3 (function.json), v4 (code-first) |
| Languages | JavaScript, TypeScript |
| Azure Integration | Full Azure Functions hosting |
| Entities Support | Yes |

### Portable SDK - durabletask (Coming Soon)

A portable SDK for Node.js is planned but not yet available. The portable SDK will enable running durable workflows outside of Azure Functions with any backend provider including the Durable Task Scheduler.

## Quick Comparison

| Feature | durable-functions (v4) | durable-functions (v3) |
|---------|----------------------|----------------------|
| Registration | Code-based (`df.app.*`) | function.json |
| Type Safety | Strong TypeScript support | Basic types |
| Node.js Version | 18.x+ | 8+ |
| Runtime Version | Functions 4.25+ | Functions 2.0+ |
| Bundle Version | 3.15+ | 2.x |

## Getting Started

### Prerequisites

- Node.js 18.x or later (for v4 programming model)
- Azure Functions Core Tools 4.0.5382 or later
- npm or yarn package manager

### Installation

```bash
# Install the durable-functions package
npm install durable-functions

# For TypeScript projects
npm install --save-dev typescript @types/node
```

### Project Setup (v4 Programming Model)

**package.json**:
```json
{
  "name": "durable-functions-app",
  "version": "1.0.0",
  "main": "dist/src/functions/*.js",
  "scripts": {
    "build": "tsc",
    "watch": "tsc -w",
    "start": "npm run build && func start",
    "prestart": "npm run build"
  },
  "dependencies": {
    "@azure/functions": "^4.0.0",
    "durable-functions": "^3.0.0"
  },
  "devDependencies": {
    "@types/node": "^18.0.0",
    "typescript": "^5.0.0"
  }
}
```

**host.json**:
```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    }
  },
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  }
}
```

### Basic Example (TypeScript v4 Model)

```typescript
import * as df from "durable-functions";
import { app, HttpRequest, HttpResponseInit, InvocationContext } from "@azure/functions";

// Activity function
const hello: df.ActivityHandler = (input: string): string => {
    return `Hello, ${input}!`;
};

df.app.activity("hello", { handler: hello });

// Orchestrator function
const helloOrchestrator: df.OrchestrationHandler = function* (context: df.OrchestrationContext) {
    const outputs: string[] = [];
    outputs.push(yield context.df.callActivity("hello", "Tokyo"));
    outputs.push(yield context.df.callActivity("hello", "Seattle"));
    outputs.push(yield context.df.callActivity("hello", "London"));
    return outputs;
};

df.app.orchestration("helloOrchestrator", helloOrchestrator);

// HTTP starter function
app.http("httpStart", {
    route: "orchestrators/{orchestratorName}",
    extraInputs: [df.input.durableClient()],
    handler: async (req: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> => {
        const client = df.getClient(context);
        const body = await req.json();
        const instanceId = await client.startNew(req.params.orchestratorName, { input: body });

        context.log(`Started orchestration with ID = '${instanceId}'.`);

        return client.createCheckStatusResponse(req, instanceId);
    },
});
```

### Basic Example (JavaScript v4 Model)

```javascript
const df = require("durable-functions");
const { app } = require("@azure/functions");

// Activity function
const hello = (input) => {
    return `Hello, ${input}!`;
};

df.app.activity("hello", { handler: hello });

// Orchestrator function
const helloOrchestrator = function* (context) {
    const outputs = [];
    outputs.push(yield context.df.callActivity("hello", "Tokyo"));
    outputs.push(yield context.df.callActivity("hello", "Seattle"));
    outputs.push(yield context.df.callActivity("hello", "London"));
    return outputs;
};

df.app.orchestration("helloOrchestrator", helloOrchestrator);

// HTTP starter function
app.http("httpStart", {
    route: "orchestrators/{orchestratorName}",
    extraInputs: [df.input.durableClient()],
    handler: async (req, context) => {
        const client = df.getClient(context);
        const body = await req.json();
        const instanceId = await client.startNew(req.params.orchestratorName, { input: body });

        context.log(`Started orchestration with ID = '${instanceId}'.`);

        return client.createCheckStatusResponse(req, instanceId);
    },
});
```

## Version Information

### Package Version vs Programming Model

{: .important }
> Don't confuse the npm package version with the programming model version:
> - **durable-functions v3.x** is required for the **v4 programming model**
> - **durable-functions v2.x** is required for the **v3 programming model**

| Package Version | Programming Model | Node.js | Functions Runtime |
|-----------------|-------------------|---------|-------------------|
| 3.x | v4 (recommended) | 18.x+ | 4.25+ |
| 2.x | v3 | 8+ | 2.0+ |

## Next Steps

- [Azure Functions SDK Reference](durable-functions) - Complete API reference for the durable-functions package
- [Portable SDK](durabletask-sdk) - Information about the upcoming portable SDK
- [Migration Guide](#migration-from-v3-to-v4) - Upgrade from v3 to v4 programming model

## Migration from v3 to v4

### Key Changes

1. **No more function.json files** - Register functions in code using `df.app.*` methods
2. **Updated package version** - Use durable-functions v3.x for v4 programming model
3. **New type exports** - `OrchestrationHandler`, `ActivityHandler`, `EntityHandler`, etc.
4. **API simplifications** - Options objects instead of multiple parameters

### Registration Changes

**v3 Model (with function.json)**:
```typescript
// function.json required for each function
import { AzureFunction, Context } from "@azure/functions";

const helloActivity: AzureFunction = async function (context: Context): Promise<string> {
    return `Hello, ${context.bindings.name}!`;
};

export default helloActivity;
```

**v4 Model (code-based)**:
```typescript
import * as df from "durable-functions";
import { ActivityHandler } from "durable-functions";

const helloActivity: ActivityHandler = (input: string): string => {
    return `Hello, ${input}!`;
};

df.app.activity("hello", { handler: helloActivity });
```

### Client API Changes

**v3 Model**:
```typescript
const status = await client.getStatus("instanceId", false, false, true);
```

**v4 Model**:
```typescript
const status = await client.getStatus("instanceId", {
    showHistory: false,
    showHistoryOutput: false,
    showInput: true
});
```

See [durable-functions API Reference](durable-functions) for complete migration details.
