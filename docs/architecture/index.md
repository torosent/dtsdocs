---
layout: default
title: Architecture Guides
nav_order: 7
has_children: true
permalink: /docs/architecture/
---

# Architecture Guides
{: .no_toc }

Learn how to architect and deploy durable orchestrations across different Azure compute platforms.
{: .fs-6 .fw-300 }

---

## Overview

Azure Durable technologies can be deployed on various compute platforms, each with different characteristics:

| Platform | Best For | Durable Functions | Durable Task SDK |
|----------|----------|-------------------|------------------|
| Azure Functions | Serverless, event-driven | ✅ Native | ❌ |
| Azure Container Apps | Containerized microservices | ✅ via Extension | ✅ Recommended |
| Azure Kubernetes Service | Full orchestration control | ✅ via Container | ✅ Full Support |
| Virtual Machines | Legacy, special requirements | ❌ | ✅ Full Support |

---

## Architecture Patterns

```
┌──────────────────────────────────────────────────────────────────┐
│                    DEPLOYMENT ARCHITECTURES                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   ┌─────────────────────────────────────────────────────────────┐│
│   │  SERVERLESS (Azure Functions)                               ││
│   │                                                              ││
│   │    Triggers ──► Functions App ──► Durable Task Scheduler    ││
│   │    (HTTP, Queue,    (Scale to zero)                         ││
│   │     Event Grid)                                             ││
│   └─────────────────────────────────────────────────────────────┘│
│                                                                   │
│   ┌─────────────────────────────────────────────────────────────┐│
│   │  CONTAINERIZED (Azure Container Apps)                       ││
│   │                                                              ││
│   │    ┌────────────┐   ┌────────────┐   ┌──────────────────┐   ││
│   │    │  API App   │──►│  Worker    │──►│ Durable Task     │   ││
│   │    │            │   │  App       │   │ Scheduler        │   ││
│   │    └────────────┘   └────────────┘   └──────────────────┘   ││
│   │                                                              ││
│   └─────────────────────────────────────────────────────────────┘│
│                                                                   │
│   ┌─────────────────────────────────────────────────────────────┐│
│   │  KUBERNETES (AKS)                                           ││
│   │                                                              ││
│   │    ┌────────────┐   ┌────────────┐   ┌──────────────────┐   ││
│   │    │  Ingress   │──►│  Worker    │──►│ Durable Task     │   ││
│   │    │  + API     │   │  Pods      │   │ Scheduler        │   ││
│   │    │  Pods      │   │  (HPA)     │   │                  │   ││
│   │    └────────────┘   └────────────┘   └──────────────────┘   ││
│   │                                                              ││
│   └─────────────────────────────────────────────────────────────┘│
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Detailed Guides

### Compute Platforms

- [Durable Functions with Durable Task Scheduler](durable-functions-dts.md)  
  Use Azure Functions with the managed Durable Task Scheduler backend

- [Azure Container Apps with Durable Task SDK](aca-dts.md)  
  Deploy containerized workers on Azure Container Apps

- [Azure Kubernetes Service with Durable Task SDK](aks-dts.md)  
  Full Kubernetes deployment with scaling and observability

---

## Choosing Your Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  ARCHITECTURE DECISION TREE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  "Do you need scale-to-zero?"                                   │
│      │                                                           │
│      ├── Yes ──► Azure Functions                                │
│      │           (with Durable Task Scheduler)                  │
│      │                                                           │
│      └── No ───► "Do you need container flexibility?"           │
│                       │                                          │
│                       ├── Yes, but simple ──► Azure Container   │
│                       │                        Apps             │
│                       │                                          │
│                       └── Yes, full control ──► AKS             │
│                                                                  │
│  "Do you have existing infrastructure?"                         │
│      │                                                           │
│      ├── Kubernetes ──► AKS with Durable Task SDK              │
│      │                                                           │
│      ├── VMs ────────► VM with Durable Task SDK                │
│      │                                                           │
│      └── None ───────► Azure Functions (fastest start)         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Architecture Components

### All Architectures Share

1. **Durable Task Scheduler** - Managed backend service
2. **Task Hub** - Logical grouping of orchestrations
3. **Managed Identity** - Secure authentication
4. **Dashboard** - Monitoring and management UI

### Per-Architecture Specifics

| Component | Functions | Container Apps | AKS |
|-----------|-----------|----------------|-----|
| Compute | Consumption/Premium | Container instances | Pods |
| Scaling | Automatic | KEDA/Rules | HPA/KEDA |
| Networking | VNet Integration | VNet | Full control |
| Secrets | Key Vault ref | Secrets | Secrets/CSI |
| Logging | App Insights | Log Analytics | Choice |

---

## Next Steps

Choose your deployment architecture:

- [Durable Functions + DTS →](durable-functions-dts.md)
- [Container Apps + DTS →](aca-dts.md)  
- [AKS + DTS →](aks-dts.md)
