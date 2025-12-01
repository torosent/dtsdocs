# Proposal: Unified Azure Durable Documentation Strategy

## Executive Summary

This repository represents a strategic proposal for unifying the documentation of the "Azure Durable" ecosystem. Currently, documentation may be fragmented between Azure Durable Functions, the underlying Durable Task Framework, and the new Durable Task Scheduler.

This proposal aims to consolidate these under a single **Azure Durable** brand, providing a cohesive user journey from core concepts to production architecture, regardless of the user's chosen compute platform (Serverless vs. Containers).

## Core Philosophy

1.  **Concept-First Approach**: Durable execution concepts (Orchestrators, Activities, Replay, Event Sourcing) are universal. They are centralized in a shared `concepts/` directory to avoid duplication and reinforce that the mental model is consistent across all products.
2.  **Product Clarity**: Distinct, top-level sections for **Durable Functions** (Serverless) and **Durable Task SDKs** (Portable/Containerized) allow users to self-select their path immediately while sharing the same conceptual foundation.
3.  **Backend as a Product**: The **Durable Task Scheduler (DTS)** is elevated to a top-level product section. This highlights its role not just as an implementation detail, but as a premier, managed service for orchestration state management.
4.  **Pattern Reuse**: Architectural patterns (Fan-out/Fan-in, Chaining) are identical across platforms. Shared pattern documentation reduces maintenance and ensures consistency.

## Proposed Structure & Reasoning

The documentation is organized into the following logical sections:

### 1. Core Concepts (`docs/concepts/`)
**The "What"**
*   **Content**: Orchestrators, Activities, Entities, State Management, Error Handling, Versioning, Task Hubs, Code Constraints, and more.
*   **Reasoning**: Users often struggle with the "magic" of replay history or the constraints of deterministic code. Defining these once creates a shared vocabulary for both DF and SDK users.

### 2. Azure Durable Functions (`docs/durable-functions/`)
**The "Serverless Path"**
*   **Content**: Overview, Quickstart, Programming Model, Storage Providers, Samples.
*   **Reasoning**: This section focuses on the Azure Functions extension. It includes storage provider configuration because DF offers the unique flexibility of "Bring Your Own Backend" (Azure Storage, MSSQL, Netherite, DTS), unlike the SDKs which primarily target DTS.

### 3. Durable Task SDKs (`docs/sdks/`)
**The "Portable Path"**
*   **Content**: Overview, Quickstart, .NET, Python, Java client libraries, Samples.
*   **Reasoning**: For users running on Azure Container Apps (ACA), Kubernetes (AKS), or on-premises. This section focuses on the libraries that allow durable execution *without* the Azure Functions host.

### 4. Durable Task Scheduler (`docs/durable-task-scheduler/`)
**The "Managed Backend"**
*   **Content**: Overview, Setup, Dashboard, Identity management.
*   **Reasoning**: DTS is a standalone Azure service. It requires its own guide for provisioning, monitoring, and security, distinct from the compute platform that connects to it.

### 5. Patterns (`docs/patterns/`)
**The "How-To"**
*   **Content**: Function Chaining, Fan-out/Fan-in, Async HTTP APIs, Human Interaction, External Events, Aggregator.
*   **Reasoning**: These patterns are the "standard library" of durable execution. Centralizing them prevents "dialect" fragmentation between languages and platforms.

### 6. Architecture (`docs/architecture/`)
**The "Production Guide"**
*   **Content**: Durable Functions + DTS, Azure Container Apps + SDKs, AKS + SDKs.
*   **Reasoning**: Addresses "Day 2" operations. How do I deploy this? How does networking work? This bridges the gap between code and cloud infrastructure.

### 7. Comparison (`docs/comparison/`)
**The "Decision Guide"**
*   **Content**: When to Use What, Advantages of DTS, Comparison with Alternatives (Logic Apps, Service Bus, Orleans), Real-World Use Cases.
*   **Reasoning**: Honest, transparent guidance to help customers choose the right tool for their specific constraints (e.g., "I need VNet injection" or "I need to run on-prem").

### 8. Glossary (`docs/glossary.md`)
**The "Reference"**
*   **Content**: Definitions of key terms like Orchestrator, Activity, Checkpoint, Replay, Task Hub, etc.
*   **Reasoning**: Quick reference for users to look up unfamiliar terminology without leaving the documentation.

## Documentation Map

The following tree illustrates the complete file structure:

```text
docs/
├── index.md                   # Documentation landing page
├── glossary.md                # Terminology reference
│
├── concepts/                  # Shared mental model (applies to both DF and SDKs)
│   ├── index.md               # Concept landing page
│   ├── orchestrators.md       # Defining workflows
│   ├── activities.md          # Performing work
│   ├── entities.md            # Managing stateful objects
│   ├── state-management.md    # How state is persisted
│   ├── error-handling.md      # Retry policies and compensation
│   ├── versioning.md          # Handling workflow changes
│   ├── task-hubs.md           # Logical partitioning
│   ├── code-constraints.md    # Determinism requirements
│   ├── framework-vs-sdk.md    # Architectural comparison
│   ├── instance-management.md # Managing orchestration instances
│   ├── data-persistence.md    # Data storage strategies
│   ├── distributed-execution.md # How work is distributed
│   ├── performance-scale.md   # Scaling considerations
│   ├── schedules.md           # Timers and scheduling
│   ├── app-hosting.md         # Hosting options
│   └── backend-providers.md   # Backend storage options
│
├── durable-functions/         # Azure Functions extension (Serverless)
│   ├── overview.md            # DF specific introduction
│   ├── quickstart.md          # Create your first Durable Function
│   ├── programming-model.md   # Triggers, bindings, and constraints
│   ├── storage-providers.md   # Configuring Azure Storage, MSSQL, Netherite, DTS
│   └── samples.md             # Code examples
│
├── sdks/                      # Durable Task Client Libraries (Portable)
│   ├── index.md               # SDK landing page
│   ├── overview.md            # SDK introduction
│   ├── quickstart.md          # Create your first portable worker
│   ├── dotnet.md              # .NET SDK guide
│   ├── python.md              # Python SDK guide
│   ├── java.md                # Java SDK guide
│   └── samples.md             # Code examples
│
├── durable-task-scheduler/    # The managed backend service
│   ├── index.md               # DTS landing page
│   ├── overview.md            # Service capabilities
│   ├── setup.md               # Provisioning guide
│   ├── dashboard.md           # Monitoring and debugging
│   └── identity.md            # Authentication and authorization
│
├── patterns/                  # Common architectural patterns
│   ├── index.md               # Pattern overview
│   ├── function-chaining.md   # Sequential workflows
│   ├── fan-out-fan-in.md      # Parallel processing
│   ├── async-http.md          # Long-running HTTP APIs
│   ├── human-interaction.md   # Approval workflows
│   ├── external-events.md     # Event-driven orchestration
│   └── aggregator.md          # Stateful aggregation
│
├── architecture/              # Production deployment guides
│   ├── index.md               # Architecture overview
│   ├── durable-functions-dts.md # DF + DTS deployment
│   ├── aca-dts.md             # Container Apps + SDKs
│   └── aks-dts.md             # Kubernetes + SDKs
│
└── comparison/                # Decision guides
    ├── index.md               # Comparison overview
    ├── when-to-use.md         # DF vs SDKs decision guide
    ├── advantages.md          # Benefits of DTS
    ├── alternatives.md        # vs Logic Apps, Service Bus, Orleans
    └── use-cases.md           # Real-world examples
```

## Sections Summary

| Section | Content |
|:--------|:--------|
| **Concepts** | 17 concept documents covering orchestrators, activities, entities, state management, error handling, versioning, task hubs, and more. |
| **Durable Functions** | Overview, Quickstart, Programming Model, Storage Providers, and Samples. |
| **SDKs** | Overview, Quickstart, and language-specific guides for .NET, Python, and Java. |
| **DTS** | Overview, Setup, Dashboard, and Identity management guides. |
| **Patterns** | 6 architectural patterns including chaining, fan-out/fan-in, and human interaction. |
| **Architecture** | Deployment guides for Durable Functions, Container Apps, and AKS. |
| **Comparison** | Decision guides, alternatives comparison, and real-world use cases. |
| **Glossary** | Terminology reference for quick lookups. |

## Local Development

To preview this documentation site locally:

```bash
# Install dependencies
bundle install

# Serve locally
bundle exec jekyll serve
```

Visit `http://localhost:4000` to view the site.
