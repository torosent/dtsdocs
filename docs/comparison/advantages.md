---
layout: default
title: Advantages of DTS
parent: Comparison
nav_order: 2
permalink: /docs/comparison/advantages/
---

# Advantages of Durable Task Scheduler
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Why use the Durable Task Scheduler over other storage backends.

---

## Overview

The Durable Task Scheduler is a fully managed Azure service purpose-built for durable orchestration workloads. It replaces self-managed storage backends with a turnkey solution.

---

## Key Advantages

### 1. Fully Managed Service

No infrastructure to provision or maintain.

```
┌─────────────────────────────────────────────────────────────────┐
│                    OPERATIONAL COMPARISON                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Azure Storage Backend         Durable Task Scheduler            │
│  ─────────────────────         ──────────────────────            │
│                                                                  │
│  ☐ Create Storage Account      ☑ Create Scheduler (1 resource)  │
│  ☐ Configure Tables            ☑ Everything included            │
│  ☐ Configure Queues                                              │
│  ☐ Configure Blobs                                               │
│  ☐ Set up monitoring                                             │
│  ☐ Configure networking                                          │
│  ☐ Manage performance                                            │
│                                                                  │
│  You manage: Everything        You manage: Just your code        │
│  Microsoft manages: Nothing    Microsoft manages: Infrastructure │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Benefits:**
- No storage account management
- No queue/table configuration
- Automatic maintenance and updates
- Reduced operational burden

---

### 2. Built-in Dashboard

Visual management and monitoring out of the box.

```
┌─────────────────────────────────────────────────────────────────┐
│                    DASHBOARD FEATURES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  📊 Orchestration List                                    │  │
│  │  ├─ Filter by status (Running, Completed, Failed)        │  │
│  │  ├─ Search by instance ID                                │  │
│  │  └─ Time range filtering                                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  🔍 Instance Details                                      │  │
│  │  ├─ Execution history                                    │  │
│  │  ├─ Input/Output inspection                              │  │
│  │  ├─ Timer and event tracking                             │  │
│  │  └─ Sub-orchestration visualization                      │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  ⚡ Actions                                               │  │
│  │  ├─ Terminate orchestrations                             │  │
│  │  ├─ Purge history                                        │  │
│  │  ├─ Rewind failed instances                              │  │
│  │  └─ Send external events                                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Without Durable Task Scheduler:**
- Build your own monitoring UI
- Query storage directly for status
- Create custom tooling for operations

**With Durable Task Scheduler:**
- Built-in web dashboard
- Real-time orchestration visibility
- One-click operational actions

---

### 3. Better Performance

Purpose-built for orchestration workloads.

| Metric | Azure Storage | Durable Task Scheduler |
|--------|---------------|------------------------|
| Latency | Variable | Optimized |
| Throughput | Limited by storage | Higher capacity |
| Connection | HTTP(S) | gRPC (persistent) |
| Scaling | Manual tuning | Automatic |

**Why gRPC matters:**
- Persistent connections (no connection overhead)
- Binary protocol (smaller payloads)
- Streaming support
- Lower latency for frequent operations

---

### 4. Portable Across Compute Platforms

Same backend, any host.

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMPUTE PORTABILITY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│           ┌──────────────────────────────────────┐              │
│           │     Durable Task Scheduler           │              │
│           │        (Single Backend)              │              │
│           └──────────────────┬───────────────────┘              │
│                              │                                   │
│       ┌──────────────────────┼──────────────────────┐           │
│       │           │          │          │           │           │
│       ▼           ▼          ▼          ▼           ▼           │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │ Azure   │ │Container│ │   AKS   │ │   VM    │ │  Local  │   │
│  │Functions│ │  Apps   │ │         │ │         │ │   Dev   │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
│                                                                  │
│  Same orchestration code runs everywhere!                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Benefits:**
- Migrate between platforms without changing backend
- Dev/test locally, deploy to any platform
- Multi-platform architectures with shared state

---

### 5. Simplified Authentication

Managed identity support built-in.

```csharp
// Old way - Connection strings and secrets
services.AddDurableTaskWorker()
    .UseDurableTaskScheduler(new DurableTaskSchedulerOptions
    {
        ConnectionString = "DefaultEndpointsProtocol=https;AccountName=..."
    });

// New way - Managed identity
services.AddDurableTaskWorker()
    .UseDurableTaskScheduler(endpoint, taskHub); // Uses DefaultAzureCredential
```

**Authentication options:**
- System-assigned managed identity
- User-assigned managed identity  
- DefaultAzureCredential (development)
- No secrets to rotate or manage

---

### 6. Cost Efficiency

Pay for what you use with predictable pricing.

```
┌─────────────────────────────────────────────────────────────────┐
│                    COST COMPARISON                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Azure Storage Approach:                                         │
│  ├─ Storage transactions (reads/writes)                         │
│  ├─ Storage capacity                                            │
│  ├─ Queue message costs                                         │
│  └─ Unpredictable with scale                                    │
│                                                                  │
│  Durable Task Scheduler:                                         │
│  ├─ Simple SKU-based pricing                                    │
│  ├─ Included operations                                         │
│  ├─ Predictable monthly cost                                    │
│  └─ No surprise charges                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 7. Multi-Tenant Isolation

Task hubs provide logical separation.

```
┌─────────────────────────────────────────────────────────────────┐
│                    MULTI-TENANCY                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│               Durable Task Scheduler                             │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │                                                           │ │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │ │
│   │  │  Task Hub   │  │  Task Hub   │  │  Task Hub   │       │ │
│   │  │  tenant-a   │  │  tenant-b   │  │  tenant-c   │       │ │
│   │  │             │  │             │  │             │       │ │
│   │  │ Isolated    │  │ Isolated    │  │ Isolated    │       │ │
│   │  │ state       │  │ state       │  │ state       │       │ │
│   │  └─────────────┘  └─────────────┘  └─────────────┘       │ │
│   │                                                           │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│   Benefits:                                                      │
│   • Complete data isolation                                      │
│   • Separate access control per hub                             │
│   • Independent scaling                                          │
│   • Separate monitoring                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 8. Enterprise-Ready Features

Built for production workloads.

| Feature | Description |
|---------|-------------|
| **High Availability** | Multi-zone redundancy |
| **Disaster Recovery** | Built-in backup and restore |
| **Compliance** | Azure compliance certifications |
| **SLA** | Enterprise SLA backing |
| **Encryption** | Data encrypted at rest and in transit |
| **Audit Logging** | Integration with Azure Monitor |

---

## Comparison Summary

| Aspect | Azure Storage | MSSQL | Durable Task Scheduler |
|--------|---------------|-------|------------------------|
| Setup Complexity | Medium | High | Low |
| Operational Burden | High | High | Minimal |
| Dashboard | Build yourself | Build yourself | Included |
| Performance | Good | Good | Optimized |
| Multi-Language | All | All | .NET, Python, Java |
| Cost Model | Per-transaction | Per-compute | Per-SKU |
| Portability | Limited | Limited | Full |

---

## Migration to Durable Task Scheduler

### From Azure Storage Backend

1. Create Durable Task Scheduler resource
2. Update connection configuration
3. Test with new instances
4. Migrate or complete existing orchestrations
5. Decommission old storage

### Connection Change

```json
// Before (host.json with Azure Storage)
{
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "connectionStringName": "AzureWebJobsStorage"
      }
    }
  }
}

// After (host.json with Durable Task Scheduler)
{
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "type": "azureManaged",
        "connectionStringName": "DurableTaskSchedulerConnection"
      }
    }
  }
}
```

---

## Next Steps

- [Set Up Durable Task Scheduler →](../durable-task-scheduler/setup.md)
- [Configure Identity →](../durable-task-scheduler/identity.md)
- [Explore the Dashboard →](../durable-task-scheduler/dashboard.md)
