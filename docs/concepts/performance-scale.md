---
layout: default
title: Performance & Scale
parent: Core Concepts
nav_order: 20
permalink: /docs/concepts/performance-scale/
---

# Performance and Scale
{: .no_toc }

> **Note**: This article has been superseded by [Distributed Execution](./distributed-execution.md) which provides backend-agnostic guidance on scaling, partitioning, and high availability.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Durable Functions and the Durable Task Framework are designed to scale massively, but understanding the underlying performance characteristics is important for optimizing your application.

---

## Scaling Behavior

The scaling behavior depends on the compute platform (Azure Functions, Kubernetes, etc.) and the storage provider.

### Azure Functions (Consumption/Elastic Premium)

In Azure Functions, the **Scale Controller** monitors the work item queues in your task hub.
- If queues are growing, it adds more worker instances (VMs).
- If queues are empty, it scales down.

### Partitioning

To enable scaling, the framework partitions the orchestration work items.
- **Orchestrator Partitions**: Orchestration instances are distributed across a fixed number of partitions (default is 4 for Azure Storage). Each partition is processed by a single worker at a time.
- **Activity Work Items**: Activity messages are stored in a separate queue and can be processed by any available worker concurrently.

> **Implication**: The maximum throughput for *orchestrator* execution is limited by the number of partitions. If you have 4 partitions, you can have at most 4 workers processing orchestrator steps concurrently. Activity execution scales out much further.

---

## High Availability

The Durable Task Framework is designed for high availability.

- **State Persistence**: All state is persisted to durable storage (Azure Storage, MSSQL, etc.). If a worker crashes, the state is safe.
- **Automatic Recovery**: If a worker processing an orchestration crashes, the lease on that partition expires, and another worker picks it up. The orchestration replays from the last checkpoint and resumes.
- **At-Least-Once Delivery**: Messages are guaranteed to be processed at least once. Your code should be idempotent to handle potential duplicate executions.

---

## Performance Best Practices

1. **Minimize Orchestrator History**: Large histories slow down replay. Use `ContinueAsNew` for infinite loops.
2. **Avoid Large Inputs/Outputs**: Inputs and outputs are serialized and stored. Large payloads increase latency and storage costs. Pass references (e.g., Blob URLs) instead of raw data.
3. **Increase Partition Count**: For high-throughput scenarios using Azure Storage, consider increasing the partition count (up to 16) *before* creating the task hub.
4. **Use Fan-Out**: Parallelize work using the Fan-Out/Fan-In pattern to maximize throughput.
5. **Choose the Right Storage Provider**:
    - **Azure Storage**: General purpose, easiest to set up.
    - **Netherite**: High throughput, optimized for Event Hubs.
    - **MSSQL**: Enterprise environments, disconnected scenarios.

---

## Throttling

You can configure concurrency limits to prevent your application from overwhelming downstream systems.

**host.json (Azure Functions):**

```json
{
  "extensions": {
    "durableTask": {
      "maxConcurrentActivityFunctions": 10,
      "maxConcurrentOrchestratorFunctions": 10
    }
  }
}
```

These settings control how many functions run concurrently *per worker instance*.
