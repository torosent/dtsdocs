---
layout: home
title: Home
nav_order: 1
permalink: /
---

# Azure Durable Documentation

Build workflows that never fail. Azure Durable automatically handles retries, state persistence, and crash recovery—so you can focus on your business logic.
{: .fs-6 .fw-300 }

[Get Started]({{ site.baseurl }}/docs/){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View Concepts]({{ site.baseurl }}/docs/concepts/){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## What is Azure Durable?

Azure Durable is Microsoft's durable execution platform for building fault-tolerant workflows that automatically handle failures, retries, and state persistence.

| Component | Description |
|:----------|:------------|
| **[Azure Durable Functions]({{ site.baseurl }}/docs/durable-functions/overview/)** | Serverless stateful workflows on Azure Functions |
| **[Durable Task Scheduler]({{ site.baseurl }}/docs/durable-task-scheduler/overview/)** | Fully managed, high-performance orchestration backend |
| **[Durable Task SDKs]({{ site.baseurl }}/docs/sdks/overview/)** | Portable libraries for containers, Kubernetes, and more |

---

## Choose Your Path

### ⚡ Serverless with Azure Functions

Event-driven, pay-per-execution workloads:

```bash
func init MyApp --worker-runtime dotnet-isolated
dotnet add package Microsoft.Azure.Functions.Worker.Extensions.DurableTask
```

[Durable Functions Quickstart →]({{ site.baseurl }}/docs/durable-functions/quickstart/){: .btn .btn-outline }

### 🐳 Containers & Kubernetes

Full control over infrastructure and scaling:

```bash
dotnet add package Microsoft.DurableTask.Worker.AzureManaged
```

[Durable Task SDK Quickstart →]({{ site.baseurl }}/docs/sdks/quickstart/){: .btn .btn-outline }

---

## Key Benefits

- ✅ **Automatic State Persistence** — Orchestration state survives crashes and restarts
- ✅ **Built-in Fault Tolerance** — Automatic retries with configurable policies
- ✅ **Horizontal Scalability** — Handle thousands of concurrent orchestrations
- ✅ **Monitoring Dashboard** — Visual debugging and execution history
- ✅ **Multi-Platform** — Run on Functions, Container Apps, AKS, or VMs

[Explore the Documentation →]({{ site.baseurl }}/docs/){: .btn .btn-primary }
