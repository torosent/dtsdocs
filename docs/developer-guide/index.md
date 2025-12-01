---
layout: default
title: Developer Reference
nav_order: 4
has_children: true
permalink: /docs/developer-guide/
---

# Developer Reference
{: .no_toc }

Comprehensive reference documentation for building durable workflows with Azure Durable Functions and the Durable Task SDKs.
{: .fs-6 .fw-300 }

---

## Overview

This developer reference covers two main areas:

1. **Azure Functions** - Configuration and APIs specific to Durable Functions running in Azure Functions
2. **SDKs** - Programming interfaces for building portable, durable workflows across different hosting platforms

---

## Azure Functions

Configuration and management APIs for Durable Functions in Azure Functions.

| Topic | Description |
|-------|-------------|
| [**host.json Settings →**](./azure-functions/host-json.md) | Configure Durable Functions behavior via host.json |
| [**Bindings →**](./azure-functions/bindings.md) | Orchestration, activity, entity triggers and client bindings |
| [**HTTP API →**](./azure-functions/http-api.md) | Built-in HTTP APIs for orchestration management |

---

## SDKs

The Durable Task SDKs provide programming interfaces for building durable workflows. Choose between Azure Functions extensions or portable SDKs for different hosting scenarios.

### SDK Overview

| SDK | Description | Link |
|-----|-------------|------|
| **Overview** | Understanding the different SDK flavors and when to use each | [**SDK Overview →**](./sdks/index.md) |

### .NET

| Topic | Description |
|-------|-------------|
| [**.NET Overview →**](./sdks/dotnet/index.md) | Complete .NET SDK reference |
| [**Unit Testing →**](./sdks/dotnet/unit-testing.md) | Testing Durable Functions with mocking |
| [**Durable Entities →**](./sdks/dotnet/entities.md) | Entity functions developer guide |
| [**WebJobs SDK →**](./sdks/dotnet/webjobs.md) | Using Durable Task as WebJobs |
| [**API Reference: Microsoft.DurableTask →**](./sdks/dotnet/api-reference-portable.md) | Portable SDK API reference |
| [**API Reference: WebJobs.Extensions.DurableTask →**](./sdks/dotnet/api-reference-legacy.md) | In-process Azure Functions API |
| [**API Reference: Worker.Extensions.DurableTask →**](./sdks/dotnet/api-reference-isolated.md) | .NET Isolated worker API |

### Python

| Topic | Description |
|-------|-------------|
| [**Python Overview →**](./sdks/python/index.md) | Complete Python SDK reference |
| [**Durable Functions →**](./sdks/python/durable-functions.md) | Azure Functions Python SDK |
| [**durabletask SDK →**](./sdks/python/durabletask-sdk.md) | Portable Python SDK |

### Node.js

| Topic | Description |
|-------|-------------|
| [**Node.js Overview →**](./sdks/nodejs/index.md) | Complete Node.js SDK reference |
| [**Durable Functions →**](./sdks/nodejs/durable-functions.md) | Azure Functions JavaScript/TypeScript SDK |
| [**durabletask SDK →**](./sdks/nodejs/durabletask-sdk.md) | Portable Node.js SDK |

### Java

| Topic | Description |
|-------|-------------|
| [**Java Overview →**](./sdks/java/index.md) | Complete Java SDK reference |

### PowerShell

| Topic | Description |
|-------|-------------|
| [**PowerShell Overview →**](./sdks/powershell/index.md) | Complete PowerShell SDK reference |

---

## Quick Links

<div class="code-example" markdown="1">

### Getting Started

- [**Quick Start →**](./quickstart.md) - Build your first orchestration
- [**Samples →**](./samples.md) - Example implementations

### By Hosting Platform

- [**Azure Functions →**](../hosting-options/azure-functions/index.md) - Serverless hosting
- [**Container Apps →**](../hosting-options/container-apps/index.md) - Containerized hosting
- [**Kubernetes →**](../hosting-options/kubernetes/index.md) - AKS and self-managed K8s

</div>

---

## SDK Comparison

| Feature | .NET | Python | Java | Node.js | PowerShell |
|---------|------|--------|------|---------|------------|
| **Azure Functions Support** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Portable SDK** | ✅ | ✅ | ✅ | 🚧 | ❌ |
| **Orchestrations** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Activities** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Durable Timers** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Sub-Orchestrations** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **External Events** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Durable Entities** | ✅ | ✅ | ❌ | ✅ | ❌ |
| **Retry Policies** | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## Next Steps

1. [**Start with the Quick Start →**](./quickstart.md)
2. [**Understand the SDKs →**](./sdks/index.md)
3. [**Explore patterns →**](../patterns/index.md)
4. [**Review hosting options →**](../hosting-options/index.md)
