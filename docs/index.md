# Azure Durable Documentation

Welcome to the unified documentation for **Azure Durable** — Microsoft's comprehensive platform for building reliable, stateful orchestrations in the cloud.

Build **fault-tolerant workflows** that automatically handle failures, retries, and state persistence. Whether you're building serverless applications with Azure Functions or containerized workloads on Kubernetes, Azure Durable provides the tools and infrastructure you need to succeed.

---

## Why Azure Durable?

| Challenge | Azure Durable Solution |
|-----------|------------------------|
| **Workflows fail mid-execution** | Automatic state persistence and recovery |
| **Manual retry logic is error-prone** | Built-in retry policies with exponential backoff |
| **Long-running processes timeout** | Durable timers that survive restarts |
| **Scaling stateful workloads is hard** | Managed infrastructure that scales automatically |
| **Debugging distributed workflows** | Built-in monitoring dashboard with execution history |

---

## What is Azure Durable?

Azure Durable is an umbrella term for Microsoft's durable execution platform, which includes:

| Component | Description | Best For |
|-----------|-------------|----------|
| **[Azure Durable Functions](./durable-functions/overview.md)** | Serverless stateful workflows on Azure Functions | Event-driven, pay-per-execution workloads |
| **[Durable Task Scheduler](./durable-task-scheduler/overview.md)** | Fully managed orchestration backend | Production workloads requiring high performance |
| **[Durable Task SDKs](./sdks/overview.md)** | Portable libraries for any compute platform | Container-based and on-premises deployments |

---

## Quick Navigation

| I want to... | Go to... |
|--------------|----------|
| **Understand the concepts** | [📘 Core Concepts](./concepts/index.md) |
| **Build serverless workflows** | [⚡ Durable Functions](./durable-functions/overview.md) |
| **Run on containers/Kubernetes** | [🔧 Durable Task SDKs](./sdks/overview.md) |
| **Learn orchestration patterns** | [🔄 Patterns](./patterns/index.md) |
| **Choose the right approach** | [⚖️ When to Use What](./comparison/when-to-use.md) |
| **Set up the managed backend** | [☁️ Durable Task Scheduler](./durable-task-scheduler/overview.md) |

---

### 📘 [Concepts](./concepts/index.md)
Understand the core concepts behind durable orchestrations, including orchestrators, activities, entities, and state management.

### 🔧 [SDKs](./sdks/overview.md)
Explore the Durable Task SDKs for .NET, Python, and Java — portable libraries for running orchestrations anywhere.

### 🔄 [Patterns](./patterns/index.md)
Learn common orchestration patterns like function chaining, fan-out/fan-in, human interaction, and more.

### 📂 [Samples](./sdks/samples.md)
Get started with practical, language-specific code examples for .NET, Python, and Java.

### 🏗️ [Architecture Guides](./architecture/index.md)
Understand how to architect your applications using Durable Functions and Durable Task SDKs with the Durable Task Scheduler.

### ⚖️ [When to Use What](./comparison/when-to-use.md)
Guidance on choosing between Durable Functions and Durable Task SDKs based on your requirements.

### ✅ [Advantages of Durable Task Scheduler](./comparison/advantages.md)
Learn why the Durable Task Scheduler is the recommended backend for production workloads.

---

## Getting Started

Choose your path based on your deployment needs:

### Option 1: Durable Functions (Serverless)

**Best for:** Event-driven workloads, pay-per-execution, Azure-native development

```bash
# Create a new Durable Functions project
func init MyDurableFunctionsApp --worker-runtime dotnet-isolated
cd MyDurableFunctionsApp
dotnet add package Microsoft.Azure.Functions.Worker.Extensions.DurableTask
```

[📖 Durable Functions Quickstart →](./durable-functions/quickstart.md)

### Option 2: Durable Task SDKs (Portable)

**Best for:** Container-based deployments, Kubernetes, on-premises, multi-cloud

```bash
# .NET
dotnet add package Microsoft.DurableTask.Worker.AzureManaged
dotnet add package Microsoft.DurableTask.Client.AzureManaged

# Python
pip install durabletask-azure

# Java - Add to pom.xml
```

[📖 Durable Task SDKs Quickstart →](./sdks/quickstart.md)

---

## How It Works

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

| Benefit | Description |
|---------|-------------|
| ✅ **Automatic State Management** | Your orchestration state is automatically persisted and recovered |
| ✅ **Fault Tolerance** | Automatic retries and replay on failures |
| ✅ **Scalability** | Handle thousands of concurrent orchestrations |
| ✅ **Observability** | Built-in dashboard for monitoring and debugging |
| ✅ **Flexibility** | Run on Azure Functions, Container Apps, Kubernetes, or VMs |

---

## Documentation Structure

```
docs/
├── index.md                    # This file
├── concepts/                   # Core concepts (shared across all platforms)
│   ├── orchestrators.md
│   ├── activities.md
│   ├── entities.md
│   └── state-management.md
├── durable-functions/          # Azure Durable Functions (serverless)
│   ├── overview.md
│   ├── quickstart.md
│   └── programming-model.md
├── durable-task-scheduler/     # Managed backend service
│   ├── overview.md
│   ├── setup.md
│   ├── dashboard.md
│   └── identity.md
├── sdks/                       # Durable Task SDKs (portable)
│   ├── overview.md
│   ├── quickstart.md
│   ├── dotnet.md
│   ├── python.md
│   ├── java.md
│   └── samples.md
├── patterns/                   # Orchestration patterns
│   ├── function-chaining.md
│   ├── fan-out-fan-in.md
│   ├── human-interaction.md
│   └── external-events.md
├── architecture/               # Deployment guides
│   ├── durable-functions-dts.md
│   ├── aca-dts.md
│   └── aks-dts.md
└── comparison/                 # Decision guides
    ├── when-to-use.md
    └── advantages.md
```

---

## Related Resources

- [Azure Durable Functions Documentation (Microsoft Learn)](https://learn.microsoft.com/azure/azure-functions/durable/)
- [Durable Task Scheduler Documentation (Microsoft Learn)](https://learn.microsoft.com/azure/azure-functions/durable/durable-task-scheduler/)
- [Durable Task SDK for .NET (GitHub)](https://github.com/microsoft/durabletask-dotnet)
- [Durable Task Samples (GitHub)](https://github.com/Azure-Samples/Durable-Task-Scheduler)
- [Durable Task Framework (GitHub)](https://github.com/Azure/durabletask)

---

## Feedback

Found an issue or have a suggestion? [Open an issue on GitHub](https://github.com/Azure/durabletask/issues).

---

*Last updated: December 2025*
