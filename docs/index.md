# Azure Durable Documentation

Build workflows that never fail. Azure Durable automatically handles retries, state persistence, and crash recovery—so you can focus on your business logic.
{: .fs-6 .fw-300 }

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

Azure Durable is Microsoft's durable execution platform:

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
| **Build serverless workflows** | [⚡ Durable Functions Quickstart](./durable-functions/quickstart.md) |
| **Run on containers/Kubernetes** | [🔧 SDK Quickstart](./sdks/quickstart.md) |
| **Learn orchestration patterns** | [🔄 Patterns](./patterns/index.md) |
| **Choose the right approach** | [⚖️ When to Use What](./comparison/when-to-use.md) |
| **Set up the managed backend** | [☁️ Durable Task Scheduler](./durable-task-scheduler/overview.md) |
| **Look up terminology** | [📖 Glossary](./glossary.md) |

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

## Related Resources

- [Azure Durable Functions Documentation (Microsoft Learn)](https://learn.microsoft.com/azure/azure-functions/durable/)
- [Durable Task Scheduler Documentation (Microsoft Learn)](https://learn.microsoft.com/azure/azure-functions/durable/durable-task-scheduler/)
- [Durable Task SDK for .NET (GitHub)](https://github.com/microsoft/durabletask-dotnet)
- [Durable Task Samples (GitHub)](https://github.com/Azure-Samples/Durable-Task-Scheduler)

---

## Feedback

Found an issue or have a suggestion? [Open an issue on GitHub](https://github.com/Azure/durabletask/issues).
