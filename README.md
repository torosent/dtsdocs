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
*   **Content**: Orchestrators, Activities, Entities, State Management, Error Handling, Versioning.
*   **Reasoning**: Users often struggle with the "magic" of replay history or the constraints of deterministic code. Defining these once creates a shared vocabulary for both DF and SDK users.

### 2. Azure Durable Functions (`docs/durable-functions/`)
**The "Serverless Path"**
*   **Content**: Overview, Storage Providers (Azure Storage, MSSQL, Netherite), Triggers & Bindings.
*   **Reasoning**: This section focuses on the Azure Functions extension. It includes storage provider configuration because DF offers the unique flexibility of "Bring Your Own Backend" (Azure Storage, MSSQL, etc.), unlike the SDKs which primarily target DTS.

### 3. Durable Task SDKs (`docs/sdks/`)
**The "Portable Path"**
*   **Content**: .NET, Python, Java client libraries.
*   **Reasoning**: For users running on Azure Container Apps (ACA), Kubernetes (AKS), or on-premises. This section focuses on the libraries that allow durable execution *without* the Azure Functions host.

### 4. Durable Task Scheduler (`docs/durable-task-scheduler/`)
**The "Managed Backend"**
*   **Content**: Setup, Dashboard, Identity, billing.
*   **Reasoning**: DTS is a standalone Azure service. It requires its own guide for provisioning, monitoring, and security, distinct from the compute platform that connects to it.

### 5. Patterns (`docs/patterns/`)
**The "How-To"**
*   **Content**: Function Chaining, Fan-out/Fan-in, Async HTTP APIs, Human Interaction.
*   **Reasoning**: These patterns are the "standard library" of durable execution. Centralizing them prevents "dialect" fragmentation between languages and platforms.

### 6. Architecture (`docs/architecture/`)
**The "Production Guide"**
*   **Content**: DF + DTS, ACA + SDKs, AKS + SDKs.
*   **Reasoning**: Addresses "Day 2" operations. How do I deploy this? How does networking work? This bridges the gap between code and cloud infrastructure.

### 7. Comparison (`docs/comparison/`)
**The "Decision Guide"**
*   **Content**: When to use DF vs. SDKs? Advantages of DTS.
*   **Reasoning**: Honest, transparent guidance to help customers choose the right tool for their specific constraints (e.g., "I need VNet injection" or "I need to run on-prem").

## Sections

| Section | Notes |
|:--------|:------|
| **Concepts** | Covers all major concepts including Task Hubs, Versioning, and Framework vs SDKs. |
| **Durable Functions** | Includes Storage Providers, Quickstart, and Programming Model. |
| **SDKs** | Includes Language guides and Quickstart. |
| **DTS** | Setup and Dashboard guides are in place. |
| **Patterns** | Major patterns documented. |
| **Architecture** | Guides for ACA, AKS, and DF. |

## Documentation Map

The following tree illustrates the file structure and the purpose of each directory:

```text
docs/
├── concepts/                  # Shared mental model (applies to both DF and SDKs)
│   ├── index.md               # Concept landing page
│   ├── orchestrators.md       # Defining workflows
│   ├── activities.md          # Performing work
│   ├── entities.md            # Managing stateful objects
│   ├── framework-vs-sdk.md    # Architectural comparison
│   └── ...
├── durable-functions/         # Azure Functions extension (Serverless)
│   ├── overview.md            # DF specific introduction
│   ├── quickstart.md          # Create your first Durable Function
│   ├── programming-model.md   # Triggers, bindings, and constraints
│   └── storage-providers.md   # Configuring Azure Storage, MSSQL, Netherite
├── sdks/                      # Durable Task Client Libraries (Portable)
│   ├── index.md               # SDK landing page
│   ├── quickstart.md          # Create your first portable worker
│   ├── dotnet.md              # .NET SDK guide
│   ├── java.md                # Java SDK guide
│   └── python.md              # Python SDK guide
├── durable-task-scheduler/    # The managed backend service
│   └── overview.md            # Service capabilities and setup
├── patterns/                  # Common architectural patterns
│   └── index.md               # Chaining, Fan-out/in, etc.
└── architecture/              # Production deployment guides
    └── index.md               # Topology diagrams and networking
```

## Local Development

To preview this documentation site locally:

```bash
# Install dependencies
bundle install

# Serve locally
bundle exec jekyll serve
```

Visit `http://localhost:4000` to view the site.
