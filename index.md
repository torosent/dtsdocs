---
layout: home
title: Home
nav_order: 1
permalink: /
---

# Azure Durable Documentation

Welcome to the unified documentation for **Azure Durable** — Microsoft's comprehensive platform for building reliable, stateful orchestrations in the cloud.
{: .fs-6 .fw-300 }

Azure Durable enables developers to build fault-tolerant workflows that automatically handle failures, retries, and state persistence. Whether you're building serverless applications with Azure Functions or containerized workloads on Kubernetes, Azure Durable provides the tools and infrastructure you need.

[Get Started]({{ site.baseurl }}/docs/concepts/){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View on GitHub](https://github.com/torosent/dtsdocs){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## What is Azure Durable?

Azure Durable is an umbrella term for Microsoft's durable execution platform, which includes:

| Component | Description |
|:----------|:------------|
| **[Azure Durable Functions]({{ site.baseurl }}/docs/durable-functions/overview/)** | An extension of Azure Functions for writing stateful workflows in a serverless environment |
| **[Durable Task Scheduler]({{ site.baseurl }}/docs/durable-task-scheduler/overview/)** | A fully managed, high-performance backend for orchestration state management |
| **[Durable Task SDKs]({{ site.baseurl }}/docs/sdks/overview/)** | Portable client libraries for building orchestrations on any compute platform |

---

## Quick Navigation

<div class="grid-container">
  <div class="grid-item">
    <h3>📘 <a href="{{ site.baseurl }}/docs/concepts/">Concepts</a></h3>
    <p>Understand the core concepts behind durable orchestrations, including orchestrators, activities, entities, and state management.</p>
  </div>
  <div class="grid-item">
    <h3>🔧 <a href="{{ site.baseurl }}/docs/sdks/overview/">SDKs</a></h3>
    <p>Explore the Durable Task SDKs for .NET, Python, and Java — portable libraries for running orchestrations anywhere.</p>
  </div>
  <div class="grid-item">
    <h3>🔄 <a href="{{ site.baseurl }}/docs/patterns/">Patterns</a></h3>
    <p>Learn common orchestration patterns like function chaining, fan-out/fan-in, human interaction, and more.</p>
  </div>
  <div class="grid-item">
    <h3>📂 <a href="{{ site.baseurl }}/docs/samples/">Samples</a></h3>
    <p>Get started with practical, language-specific code examples for C#, Python, and Java.</p>
  </div>
  <div class="grid-item">
    <h3>🏗️ <a href="{{ site.baseurl }}/docs/architecture/">Architecture Guides</a></h3>
    <p>Understand how to architect your applications using Durable Functions and Durable Task SDKs.</p>
  </div>
  <div class="grid-item">
    <h3>⚖️ <a href="{{ site.baseurl }}/docs/comparison/when-to-use/">When to Use What</a></h3>
    <p>Guidance on choosing between Durable Functions and Durable Task SDKs.</p>
  </div>
</div>

---

## Getting Started

### Option 1: Durable Functions (Serverless)

If you're building on Azure Functions and want a serverless experience:

```bash
# Create a new Durable Functions project
func init MyDurableFunctionsApp --worker-runtime dotnet-isolated
cd MyDurableFunctionsApp
dotnet add package Microsoft.Azure.Functions.Worker.Extensions.DurableTask
```

[Durable Functions Overview →]({{ site.baseurl }}/docs/durable-functions/overview/){: .btn .btn-outline }

### Option 2: Durable Task SDKs (Portable)

If you're running on Azure Container Apps, Kubernetes, or other compute platforms:

```bash
# .NET
dotnet add package Microsoft.DurableTask.Worker.AzureManaged

# Python
pip install durabletask-azure

# Java (add to build.gradle or pom.xml)
```

[Durable Task SDKs Overview →]({{ site.baseurl }}/docs/sdks/overview/){: .btn .btn-outline }

---

## Why Azure Durable?

```
┌─────────────────────────────────────────────────────────────────┐
│                     YOUR APPLICATION                            │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐           │
│  │ Orchestrator│──▶│  Activity   │──▶│  Activity   │           │
│  └─────────────┘   └─────────────┘   └─────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│               DURABLE TASK SCHEDULER                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  • Automatic state persistence                            │  │
│  │  • Fault tolerance & replay                               │  │
│  │  • High throughput                                        │  │
│  │  • Built-in monitoring dashboard                          │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Benefits
{: .text-delta }

- ✅ **Automatic State Management** — Your orchestration state is automatically persisted and recovered
- ✅ **Fault Tolerance** — Automatic retries and replay on failures
- ✅ **Scalability** — Handle thousands of concurrent orchestrations
- ✅ **Observability** — Built-in dashboard for monitoring and debugging
- ✅ **Flexibility** — Run on Azure Functions, Container Apps, Kubernetes, or VMs

---

## Related Resources

- [Azure Durable Functions Documentation](https://learn.microsoft.com/azure/azure-functions/durable/)
- [Durable Task Scheduler Documentation](https://learn.microsoft.com/azure/azure-functions/durable/durable-task-scheduler/)
- [Durable Task SDK (.NET) on GitHub](https://github.com/microsoft/durabletask-dotnet)
- [Durable Task Samples on GitHub](https://github.com/Azure-Samples/Durable-Task-Scheduler)
