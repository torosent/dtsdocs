# Proposal: Unified Azure Durable Documentation Strategy

## Executive Summary

This repository represents a strategic proposal for unifying the documentation of the "Azure Durable" ecosystem. Currently, documentation may be fragmented between Azure Durable Functions, the underlying Durable Task Framework, and the new Durable Task Scheduler.

This proposal aims to consolidate these under a single **Azure Durable** brand, providing a cohesive user journey from core concepts to production architecture, regardless of the user's chosen compute platform (Serverless vs. Containers).

## Core Philosophy

1.  **Concept-First Approach**: Durable execution concepts (Orchestrators, Activities, Replay, Event Sourcing) are universal. They are centralized in a shared `concepts/` directory to avoid duplication and reinforce that the mental model is consistent across all products.
2.  **Product Clarity**: Distinct, top-level sections for **Durable Functions** (Serverless) and **Durable Task SDKs** (Portable/Containerized) allow users to self-select their path immediately while sharing the same conceptual foundation.
3.  **Backend as a Product**: The **Durable Task Scheduler (DTS)** is elevated to a top-level product section. This highlights its role not just as an implementation detail, but as a premier, managed service for orchestration state management.
4.  **Pattern Reuse**: Architectural patterns (Fan-out/Fan-in, Chaining) are identical across platforms. Shared pattern documentation reduces maintenance and ensures consistency.

## Documentation Structure

The documentation is organized into the following logical sections:

### 1. Core Concepts (`docs/concepts/`)
**The "What"**
*   **Content**: Orchestrators, Activities, Entities, State Management, Error Handling, Versioning, Task Hubs, Code Constraints, and more.
*   **Reasoning**: Users often struggle with the "magic" of replay history or the constraints of deterministic code. Defining these once creates a shared vocabulary for both DF and SDK users.

### 2. Developer Guide (`docs/developer-guide/`)
**The "How-To Code"**
*   **Content**: Language-specific guides, SDK references, and programming models.
*   **Subsections**:
    *   `azure-functions/`: Specifics for the Azure Functions extension.
    *   `sdks/`: Guides for the portable Durable Task Client libraries (.NET, Java, Python, Node.js).
*   **Reasoning**: Focuses on the developer experience—how to write orchestrations and activities in your preferred language.

### 3. Hosting Options (`docs/hosting-options/`)
**The "Where-To Run"**
*   **Content**: Deployment guides for various compute platforms.
*   **Subsections**:
    *   `azure-functions/`: Running on the Azure Functions service (Serverless/Dedicated).
    *   `container-apps/`: Running on Azure Container Apps (ACA).
    *   `kubernetes/`: Running on Azure Kubernetes Service (AKS) or generic K8s.
*   **Reasoning**: Separates the "code" from the "infrastructure". You can run the same code in different places.

### 4. Durable Task Scheduler (`docs/durable-task-scheduler/`)
**The "Managed Backend"**
*   **Content**: Overview, Setup, Dashboard, Identity management, Auto-Purge retention policies, Pricing & SKUs, Troubleshooting.
*   **Reasoning**: DTS is a standalone Azure service. It requires its own guide for provisioning, monitoring, security, pricing, and troubleshooting distinct from the compute platform that connects to it.

### 5. Patterns (`docs/patterns/`)
**The "Standard Library"**
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
├── developer-guide/           # How to write code
│   ├── index.md               # Guide landing page
│   ├── quickstart.md          # General quickstart
│   ├── dotnet.md              # .NET guide
│   ├── python.md              # Python guide
│   ├── java.md                # Java guide
│   ├── samples.md             # Code samples
│   ├── azure-functions/       # DF specific docs
│   │   ├── index.md
│   │   ├── bindings.md
│   │   ├── host-json.md
│   │   └── http-api.md
│   └── sdks/                  # Portable SDK docs
│       ├── index.md
│       ├── dotnet/
│       ├── java/
│       ├── nodejs/
│       └── python/
│
├── hosting-options/           # Where to run code
│   ├── index.md
│   ├── azure-functions/       # Hosting on Azure Functions
│   │   ├── index.md
│   │   ├── deployment.md
│   │   ├── scaling.md
│   │   └── storage-providers.md
│   ├── container-apps/        # Hosting on ACA
│   │   ├── index.md
│   │   ├── deployment.md
│   │   └── scaling.md
│   └── kubernetes/            # Hosting on K8s
│       ├── index.md
│       ├── deployment.md
│       └── scaling.md
│
├── durable-task-scheduler/    # The managed backend service
│   ├── index.md               # DTS landing page
│   ├── overview.md            # Service capabilities
│   ├── setup.md               # Provisioning guide
│   ├── dashboard.md           # Monitoring and debugging
│   ├── identity.md            # Authentication and authorization
│   ├── auto-purge.md          # Retention policies configuration
│   ├── pricing.md             # Pricing and SKU options
│   └── troubleshooting.md     # Diagnose and resolve issues
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
| **Developer Guide** | Language-specific guides, SDK references, and programming models for both DF and portable SDKs. |
| **Hosting Options** | Deployment guides for Azure Functions, Container Apps, and Kubernetes. |
| **DTS** | Overview, Setup, Dashboard, Identity management, Auto-Purge, Pricing & SKUs, and Troubleshooting guides. |
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
