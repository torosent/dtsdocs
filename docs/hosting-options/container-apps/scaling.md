---
layout: default
title: Scaling
parent: Azure Container Apps
grand_parent: Hosting Options
nav_order: 2
permalink: /docs/hosting-options/container-apps/scaling/
---

# Azure Container Apps Autoscaling
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Configure autoscaling for Durable Task SDK apps hosted in Azure Container Apps.
{: .fs-6 .fw-300 }

---

## Overview

Azure Container Apps provides KEDA-based autoscaling that can automatically scale your worker containers based on the Durable Task Scheduler's workload. This ensures your solution handles spikes in workload and prevents resource exhaustion.

The custom scaler:

- **Monitors** the number of pending orchestrations in the task hub
- **Scales up** worker replicas with increased workload
- **Scales down** when the load decreases
- **Optimizes** resource utilization by matching capacity to demand

{: .note }
> Autoscaling is supported for apps built using the Durable Task SDKs and hosted in Azure Container Apps.

---

## Configure the Autoscaler

You can set the autoscaler configuration via the Azure portal, Bicep templates, or the Azure CLI.

### Azure Portal

1. Navigate to your Container App in the Azure portal
2. Go to **Application** → **Scale**
3. Configure the min and max replicas
4. Add a custom scale rule for Durable Task Scheduler

### Scaler Configuration Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| **Min replicas** | Minimum number of replicas at any given time | `1` |
| **Max replicas** | Maximum number of replicas at any given time | `10` |
| **endpoint** | The Durable Task Scheduler endpoint | `https://dts-ID.centralus.durabletask.io` |
| **maxConcurrentWorkItemsCount** | Max concurrent work items dispatched to your compute | `1` |
| **taskhubName** | Name of the task hub connected to the scheduler | `taskhub-ID` |
| **workItemType** | Work item type being dispatched (`Orchestration`, `Activity`, or `Entity`) | `Orchestration` |
| **Managed identity** | User-assigned or system-assigned managed identity | `someone@example.com` |

---

### Bicep Template

```bicep
resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: 'durable-worker'
  location: location
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${managedIdentity.id}': {}
    }
  }
  properties: {
    environmentId: containerAppEnvironment.id
    configuration: {
      registries: [
        {
          server: '${containerRegistry.name}.azurecr.io'
          identity: managedIdentity.id
        }
      ]
    }
    template: {
      containers: [
        {
          name: 'worker'
          image: '${containerRegistry.name}.azurecr.io/durable-worker:latest'
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
          env: [
            {
              name: 'DTS_ENDPOINT'
              value: scheduler.properties.endpoint
            }
            {
              name: 'TASKHUB_NAME'
              value: taskHub.name
            }
            {
              name: 'AZURE_CLIENT_ID'
              value: managedIdentity.properties.clientId
            }
          ]
        }
      ]
      scale: {
        minReplicas: 1
        maxReplicas: 10
        rules: [
          {
            name: 'durable-task-scaler'
            custom: {
              type: 'durabletask'
              metadata: {
                endpoint: scheduler.properties.endpoint
                taskhubName: taskHub.name
                workItemType: 'Orchestration'
                maxConcurrentWorkItemsCount: '1'
              }
              identity: managedIdentity.id
            }
          }
        ]
      }
    }
  }
}
```

---

### Azure CLI

```bash
# Create container app with scaling rules
az containerapp create \
  --resource-group $RESOURCE_GROUP \
  --name durable-worker \
  --environment $ENVIRONMENT_NAME \
  --image $REGISTRY_NAME.azurecr.io/durable-worker:latest \
  --min-replicas 1 \
  --max-replicas 10 \
  --scale-rule-name durable-scaler \
  --scale-rule-type durabletask \
  --scale-rule-metadata \
    endpoint=https://$SCHEDULER_NAME.$LOCATION.durabletask.io \
    taskhubName=default \
    workItemType=Orchestration \
    maxConcurrentWorkItemsCount=1 \
  --scale-rule-identity $IDENTITY_ID
```

---

## Experiment with the Sample

The [Autoscaling in Azure Container Apps sample](https://github.com/Azure-Samples/Durable-Task-Scheduler/tree/main/samples/scenarios/AutoscalingInACA) demonstrates autoscaling using the Azure Developer CLI.

{: .note }
> Although this sample uses the Durable Task .NET SDK, autoscaling is language-agnostic.

### Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0) or later
- [Docker](https://www.docker.com/products/docker-desktop/) (for building the image)
- [Azure Developer CLI](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)

### Set Up Your Environment

1. Clone the repository:

```bash
git clone https://github.com/Azure-Samples/Durable-Task-Scheduler.git
```

2. Authenticate with Azure:

```bash
azd auth login
```

### Deploy the Solution

1. Navigate to the sample directory:

```bash
cd /path/to/Durable-Task-Scheduler/samples/scenarios/AutoscalingInACA
```

2. Provision resources and deploy:

```bash
azd up
```

3. When prompted, provide:
   - **Environment Name**: Prefix for the resource group
   - **Azure Location**: The Azure region for your resources
   - **Azure Subscription**: Your Azure subscription

The deployment creates:
- Durable Task Scheduler and Task Hub
- Container Apps Environment
- Worker and Client Container Apps
- Managed Identity with appropriate role assignments

### Confirm Successful Deployment

1. Copy the resource group name from the terminal output
2. Sign in to the [Azure portal](https://portal.azure.com/)
3. Navigate to the resource group
4. Select the **worker** container app
5. Go to **Monitoring** → **Log stream**
6. Confirm the worker is processing orchestrations

### Verify Autoscaling

1. In the Azure portal, navigate to your worker app
2. Go to **Application** → **Revisions and replicas**
3. Click the **Replicas** tab to verify scaling
4. Go to **Application** → **Scale**
5. Click the scale name to view scaler settings

---

## Understanding the Custom Scaler

The custom scaler:

1. **Monitors** the Durable Task Scheduler for pending work items
2. **Calculates** the desired replica count based on queue depth
3. **Scales** the container app replicas within configured bounds
4. **Provides** efficient resource utilization

### Work Item Types

| Type | Description | Use Case |
|------|-------------|----------|
| `Orchestration` | Pending orchestrator work items | Scale based on orchestration queue |
| `Activity` | Pending activity work items | Scale based on activity queue |
| `Entity` | Pending entity work items | Scale based on entity queue |

### Scaling Behavior

```
Queue Depth = 0 → Min Replicas (can be 0)
Queue Depth > 0 → Scale up based on maxConcurrentWorkItemsCount
Queue Depth decreasing → Scale down after cooldown
```

---

## Best Practices

### 1. Set Appropriate Concurrency

Match `maxConcurrentWorkItemsCount` to your workload:

```bash
# For I/O-bound activities
--scale-rule-metadata maxConcurrentWorkItemsCount=10

# For CPU-bound activities
--scale-rule-metadata maxConcurrentWorkItemsCount=2
```

### 2. Configure Min Replicas

For production workloads, keep at least one replica running:

```bash
--min-replicas 1
```

### 3. Set Reasonable Max Replicas

Prevent runaway scaling:

```bash
--max-replicas 20
```

### 4. Consider Multiple Scalers

For complex workloads, use multiple scale rules:

```bicep
scale: {
  minReplicas: 1
  maxReplicas: 20
  rules: [
    {
      name: 'orchestration-scaler'
      custom: {
        type: 'durabletask'
        metadata: {
          workItemType: 'Orchestration'
          maxConcurrentWorkItemsCount: '5'
          // ... other settings
        }
      }
    }
    {
      name: 'activity-scaler'
      custom: {
        type: 'durabletask'
        metadata: {
          workItemType: 'Activity'
          maxConcurrentWorkItemsCount: '10'
          // ... other settings
        }
      }
    }
  ]
}
```

---

## Monitoring Scaling

### View Replica Count

```bash
az containerapp replica list \
  --name durable-worker \
  --resource-group $RESOURCE_GROUP \
  --output table
```

### View Scaling Events

```bash
az containerapp logs show \
  --name durable-worker \
  --resource-group $RESOURCE_GROUP \
  --type system \
  --follow
```

### Metrics to Monitor

| Metric | Description |
|--------|-------------|
| `Replica Count` | Current number of running replicas |
| `CPU Usage` | CPU utilization per replica |
| `Memory Usage` | Memory utilization per replica |
| `Request Count` | Incoming requests (for API apps) |

---

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| Not scaling up | Identity missing permissions | Verify managed identity has "Durable Task Data Contributor" role |
| Slow scale-up | Cooldown period | Adjust scale rule cooldown settings |
| Over-scaling | Low `maxConcurrentWorkItemsCount` | Increase the value to process more work per replica |
| Not scaling to zero | Min replicas > 0 | Set `--min-replicas 0` if scale-to-zero is desired |

---

## Current Limitations

{: .warning }
> Currently, autoscaling container apps using **Durable Functions** with Durable Task Scheduler isn't available. The custom scaler works with Durable Task SDKs only.

For Durable Functions autoscaling, consider:
- Using the [MSSQL backend with Container Apps](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-mssql-container-apps-hosting)
- Using Azure Functions hosting plans (Consumption, Premium, Flex Consumption)

---

## Next Steps

- [Deployment Guide →](./deployment.md)
- [Configure Identity →](../../durable-task-scheduler/identity.md)
- [Use the Dashboard →](../../durable-task-scheduler/dashboard.md)
- [Implement Patterns →](../../patterns/index.md)
