# Azure Durable Documentation

Welcome to the unified documentation for **Azure Durable** — Microsoft's comprehensive platform for building reliable, stateful orchestrations in the cloud.

Azure Durable enables developers to build fault-tolerant workflows that automatically handle failures, retries, and state persistence. Whether you're building serverless applications with Azure Functions or containerized workloads on Kubernetes, Azure Durable provides the tools and infrastructure you need.

---

## What is Azure Durable?

Azure Durable is an umbrella term for Microsoft's durable execution platform, which includes:

| Component | Description |
|-----------|-------------|
| **[Azure Durable Functions](./durable-functions/overview.md)** | An extension of Azure Functions for writing stateful workflows in a serverless environment |
| **[Durable Task Scheduler](./durable-task-scheduler/overview.md)** | A fully managed, high-performance backend for orchestration state management |
| **[Durable Task SDKs](./sdks/overview.md)** | Portable client libraries for building orchestrations on any compute platform |

---

## Quick Navigation

### 📘 [Concepts](./concepts/index.md)
Understand the core concepts behind durable orchestrations, including orchestrators, activities, entities, and state management.

### 🔧 [SDKs](./sdks/overview.md)
Explore the Durable Task SDKs for .NET, Python, and Java — portable libraries for running orchestrations anywhere.

### 🔄 [Patterns](./patterns/index.md)
Learn common orchestration patterns like function chaining, fan-out/fan-in, human interaction, and more.

### 📂 [Samples](./samples/index.md)
Get started with practical, language-specific code examples for C#, Python, and Java.

### 🏗️ [Architecture Guides](./architecture/index.md)
Understand how to architect your applications using Durable Functions and Durable Task SDKs with the Durable Task Scheduler.

### ⚖️ [When to Use What](./comparison/when-to-use.md)
Guidance on choosing between Durable Functions and Durable Task SDKs based on your requirements.

### ✅ [Advantages of Durable Task Scheduler](./comparison/advantages.md)
Learn why the Durable Task Scheduler is the recommended backend for production workloads.

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

📖 [Durable Functions Quickstart →](./durable-functions/quickstart.md)

### Option 2: Durable Task SDKs (Portable)

If you're running on Azure Container Apps, Kubernetes, or other compute platforms:

```bash
# .NET
dotnet add package Microsoft.DurableTask.Worker.AzureManaged

# Python
pip install durabletask-azure

# Java
# Add to build.gradle or pom.xml
```

📖 [Durable Task SDKs Quickstart →](./sdks/quickstart.md)

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

**Key Benefits:**

- ✅ **Automatic State Management** — Your orchestration state is automatically persisted and recovered
- ✅ **Fault Tolerance** — Automatic retries and replay on failures
- ✅ **Scalability** — Handle thousands of concurrent orchestrations
- ✅ **Observability** — Built-in dashboard for monitoring and debugging
- ✅ **Flexibility** — Run on Azure Functions, Container Apps, Kubernetes, or VMs

---

## Documentation Structure

```
docs/
├── index.md                    # This file
├── concepts/                   # Core concepts
│   ├── index.md
│   ├── orchestrators.md
│   ├── activities.md
│   ├── entities.md
│   └── state-management.md
├── durable-functions/          # Azure Durable Functions
│   ├── overview.md
│   ├── quickstart.md
│   └── programming-model.md
├── durable-task-scheduler/     # Durable Task Scheduler
│   ├── overview.md
│   ├── setup.md
│   ├── dashboard.md
│   └── identity.md
├── sdks/                       # Durable Task SDKs
│   ├── overview.md
│   ├── quickstart.md
│   ├── dotnet.md
│   ├── python.md
│   └── java.md
├── patterns/                   # Orchestration patterns
│   ├── index.md
│   ├── function-chaining.md
│   ├── fan-out-fan-in.md
│   ├── human-interaction.md
│   ├── aggregator.md
│   └── external-events.md
├── architecture/               # Architecture guides
│   ├── index.md
│   ├── durable-functions-dts.md
│   ├── aca-dts.md
│   └── aks-dts.md
├── samples/                    # Code samples
│   ├── index.md
│   ├── csharp/
│   ├── python/
│   └── java/
└── comparison/                 # Comparison guides
    ├── when-to-use.md
    └── advantages.md
```

---

## Related Resources

- [Azure Durable Functions Documentation](https://learn.microsoft.com/azure/azure-functions/durable/)
- [Durable Task Scheduler Documentation](https://learn.microsoft.com/azure/azure-functions/durable/durable-task-scheduler/)
- [Durable Task SDK (.NET) on GitHub](https://github.com/microsoft/durabletask-dotnet)
- [Durable Task Samples on GitHub](https://github.com/Azure-Samples/Durable-Task-Scheduler)
- [Durable Task Framework on GitHub](https://github.com/Azure/durabletask)

---

*Last updated: November 2025*
