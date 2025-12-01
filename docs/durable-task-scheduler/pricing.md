---
layout: default
title: Pricing & SKUs
parent: Durable Task Scheduler
nav_order: 6
permalink: /docs/durable-task-scheduler/pricing/
---

# Pricing and SKU Options
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

The Durable Task Scheduler offers two pricing models to accommodate different workload requirements, usage patterns, and billing preferences.
{: .fs-6 .fw-300 }

---

## What is an Action?

An **action** is a message dispatched by the Durable Task Scheduler to your application, triggering the execution of orchestrator, activity, or entity functions.

### Actions Include

| Action Type | Description |
|-------------|-------------|
| Starting an orchestration | Initial execution of an orchestrator |
| Starting a suborchestration | Child orchestration execution |
| Starting an activity | Activity function execution |
| Completing a timer | Timer expiration event |
| Triggering an external event | External event received |
| Executing an entity operation | Entity function execution |
| Suspending/resuming/terminating | Management operations |
| Processing results | Activity, entity, or suborchestration completion |

### Calculating Actions

For each orchestration, count the number of actions:

```
Orchestration Start                    = 1 action
┌─────────────────────────────────────────────────────┐
│                                                     │
│  Activity 1 (schedule + result)      = 2 actions   │
│  Activity 2 (schedule + result)      = 2 actions   │
│  Activity 3 (schedule + result)      = 2 actions   │
│                                                     │
└─────────────────────────────────────────────────────┘
Total                                  = 7 actions
```

**Example: Hello Cities Orchestration**

```csharp
[Function(nameof(HelloOrchestrator))]
public static async Task<string> HelloOrchestrator(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var result = "";
    result += await context.CallActivityAsync<string>(nameof(SayHello), "Tokyo");   // 2 actions
    result += await context.CallActivityAsync<string>(nameof(SayHello), "Seattle"); // 2 actions
    result += await context.CallActivityAsync<string>(nameof(SayHello), "London");  // 2 actions
    return result;
}
// Orchestrator start = 1 action
// Total = 7 actions per orchestration
```

---

## Available SKUs

### Quick Comparison

| Feature | Dedicated | Consumption (Preview) |
|---------|-----------|----------------------|
| **Billing Model** | Fixed monthly per CU | Pay-per-action |
| **Best For** | Production, high-volume | Development, variable workloads |
| **Performance** | Up to 2,000 actions/sec per CU | Up to 500 actions/sec |
| **Data Retention** | Up to 90 days | Up to 30 days |
| **High Availability** | Yes (3+ CUs) | No |
| **Production Ready** | ✅ Yes | ⚠️ Preview |

---

## Dedicated SKU

The Dedicated SKU provides consistent performance through preallocated **Capacity Units (CUs)**.

### Key Features

| Feature | Details |
|---------|---------|
| **Base Cost** | Fixed monthly cost per CU (regional pricing) |
| **Performance** | Up to 2,000 actions/second per CU |
| **Storage** | 50 GB of orchestration data per CU |
| **Retention** | Up to 90 days |
| **Scaling** | 1-3 CUs per deployment |
| **High Availability** | Requires minimum 3 CUs |

### Capacity Planning

Use these calculations to determine the number of CUs you need:

#### Formula

```
Monthly Actions = Orchestrations per Month × Actions per Orchestration
Actions per Second = Monthly Actions ÷ 2,628,000 (seconds/month)
Required CUs = Actions per Second ÷ 2,000
```

### Example Calculations

#### Example 1: Standard Workload

**Scenario:** 20 million orchestrations/month, averaging 12 actions each

| Metric | Calculation | Result |
|--------|-------------|--------|
| Monthly Actions | 20,000,000 × 12 | 240,000,000 actions |
| Actions/Second | 240,000,000 ÷ 2,628,000 | ~91 actions/sec |
| Required CUs | 91 ÷ 2,000 | **1 CU sufficient** |

#### Example 2: Enterprise Workload

**Scenario:** 500 million orchestrations/month, averaging 13 actions each

| Metric | Calculation | Result |
|--------|-------------|--------|
| Monthly Actions | 500,000,000 × 13 | 6.5 billion actions |
| Actions/Second | 6,500,000,000 ÷ 2,628,000 | ~2,473 actions/sec |
| Required CUs | 2,473 ÷ 2,000 | **2 CUs sufficient** |

#### Example 3: SaaS Platform

**Scenario:** 800 million orchestrations/month, averaging 15 actions each

| Metric | Calculation | Result |
|--------|-------------|--------|
| Monthly Actions | 800,000,000 × 15 | 12 billion actions |
| Actions/Second | 12,000,000,000 ÷ 2,628,000 | ~4,571 actions/sec |
| Required CUs | 4,571 ÷ 2,000 | **3 CUs sufficient** |

### High Availability Configuration

For production workloads requiring high availability:

```
┌─────────────────────────────────────────────────────────────┐
│              HIGH AVAILABILITY DEPLOYMENT                    │
│                                                             │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐               │
│   │  CU 1   │    │  CU 2   │    │  CU 3   │               │
│   │ Primary │    │ Replica │    │ Replica │               │
│   └─────────┘    └─────────┘    └─────────┘               │
│                                                             │
│   Minimum 3 CUs required for high availability              │
└─────────────────────────────────────────────────────────────┘
```

---

## Consumption SKU (Preview)

{: .warning }
> The Consumption SKU is currently in preview and recommended for development and testing scenarios.

The Consumption SKU offers a pay-as-you-use model, ideal for variable workloads and development.

### Key Features

| Feature | Details |
|---------|---------|
| **Billing** | Pay only for actions dispatched |
| **Base Cost** | No upfront costs or minimum commitments |
| **Performance** | Up to 500 actions/second |
| **Retention** | Up to 30 days |
| **Best For** | Development, testing, variable workloads |

### Pricing Calculation

```
Monthly Cost = Total Actions × $0.003 per action
```

### Example Calculations

#### Example 1: Development/Testing

**Scenario:** 10,000 orchestrations/month, 3 actions each (Hello Cities pattern)

| Metric | Calculation | Result |
|--------|-------------|--------|
| Monthly Actions | 10,000 × 3 | 30,000 actions |
| Monthly Cost | 30,000 × $0.003 | **$90/month** |

#### Example 2: E-commerce Application

**Scenario:** 20,000 orchestrations/month during promotional events, 7 actions each

| Metric | Calculation | Result |
|--------|-------------|--------|
| Monthly Actions | 20,000 × 7 | 140,000 actions |
| Monthly Cost | 140,000 × $0.003 | **$420/month** |

---

## Choosing the Right SKU

### Use Dedicated When

- ✅ Running production workloads
- ✅ Need predictable pricing
- ✅ Require high availability
- ✅ Have consistent, high-volume traffic
- ✅ Need longer data retention (up to 90 days)
- ✅ Require performance above 500 actions/second

### Use Consumption When

- ✅ Development and testing environments
- ✅ Variable or unpredictable workloads
- ✅ Low-volume applications
- ✅ Want to minimize costs for light usage
- ✅ Prototyping new orchestrations

### Decision Matrix

| Scenario | Recommended SKU | Reason |
|----------|-----------------|--------|
| Production API orchestrations | Dedicated | Reliability, performance |
| Development environment | Consumption | Cost-effective |
| Proof of concept | Consumption | No commitment |
| High-throughput data processing | Dedicated (2+ CUs) | Performance requirements |
| Occasional batch jobs | Consumption | Pay only when used |
| Mission-critical workflows | Dedicated (3 CUs) | High availability |

---

## Cost Optimization Tips

### For Dedicated SKU

1. **Right-size your CUs** — Start with 1 CU and scale up based on actual usage
2. **Monitor utilization** — Use the dashboard to track actions/second
3. **Enable auto-purge** — Reduce storage usage with [retention policies](./auto-purge.md)

### For Consumption SKU

1. **Optimize activity calls** — Batch operations to reduce action count
2. **Use sub-orchestrations wisely** — Each sub-orchestration adds overhead
3. **Monitor action counts** — Track spending via Azure Cost Management

### General Tips

| Tip | Impact |
|-----|--------|
| Use fan-out/fan-in patterns efficiently | Reduces total actions |
| Avoid unnecessary timer operations | Each timer = 1 action |
| Batch entity operations | Fewer round trips |
| Monitor and tune retry policies | Retries add actions |

---

## Provisioning

### Create Dedicated Scheduler

```bash
az durabletask scheduler create \
  --resource-group myResourceGroup \
  --name myScheduler \
  --location westus2 \
  --sku Dedicated \
  --capacity-units 1
```

### Create Consumption Scheduler

```bash
az durabletask scheduler create \
  --resource-group myResourceGroup \
  --name myScheduler \
  --location westus2 \
  --sku Consumption
```

---

## Next Steps

- [Setup Guide →](./setup.md)
- [Auto-Purge Configuration →](./auto-purge.md)
- [Dashboard →](./dashboard.md)
