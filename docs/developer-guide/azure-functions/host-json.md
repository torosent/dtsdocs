---
layout: default
title: host.json Settings
parent: Azure Functions
grand_parent: Developer Reference
nav_order: 1
permalink: /docs/developer-guide/azure-functions/host-json/
---

# host.json Settings for Durable Functions
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Configure Durable Functions behavior through the `host.json` file. These settings control performance, storage, and extension behavior.

---

## Configuration Structure

The Durable Functions settings are located in the `extensions.durableTask` section of your `host.json` file.

### Durable Functions 2.x

```json
{
  "version": "2.0",
  "extensions": {
    "durableTask": {
      "hubName": "MyTaskHub",
      "storageProvider": {
        "connectionStringName": "AzureWebJobsStorage",
        "controlQueueBatchSize": 32,
        "controlQueueBufferThreshold": 256,
        "controlQueueVisibilityTimeout": "00:05:00",
        "maxQueuePollingInterval": "00:00:30",
        "partitionCount": 4,
        "trackingStoreConnectionStringName": "TrackingStorage",
        "trackingStoreNamePrefix": "DurableTask",
        "useLegacyPartitionManagement": false,
        "useTablePartitionManagement": true,
        "workItemQueueVisibilityTimeout": "00:05:00"
      },
      "tracing": {
        "traceInputsAndOutputs": false,
        "traceReplayEvents": false
      },
      "notifications": {
        "eventGrid": {
          "topicEndpoint": "https://topic_name.westus2-1.eventgrid.azure.net/api/events",
          "keySettingName": "EventGridKey",
          "publishRetryCount": 3,
          "publishRetryInterval": "00:00:30",
          "publishEventTypes": ["Started", "Completed", "Failed", "Terminated"]
        }
      },
      "maxConcurrentActivityFunctions": 10,
      "maxConcurrentOrchestratorFunctions": 10,
      "maxConcurrentEntityFunctions": 10,
      "extendedSessionsEnabled": false,
      "extendedSessionIdleTimeoutInSeconds": 30,
      "useAppLease": true,
      "useGracefulShutdown": false,
      "maxEntityOperationBatchSize": 50,
      "maxOrchestrationActions": 100000,
      "storeInputsInOrchestrationHistory": false
    }
  }
}
```

### Durable Functions 1.x

```json
{
  "extensions": {
    "durableTask": {
      "hubName": "MyTaskHub",
      "controlQueueBatchSize": 32,
      "partitionCount": 4,
      "controlQueueVisibilityTimeout": "00:05:00",
      "workItemQueueVisibilityTimeout": "00:05:00",
      "maxConcurrentActivityFunctions": 10,
      "maxConcurrentOrchestratorFunctions": 10,
      "maxQueuePollingInterval": "00:00:30",
      "azureStorageConnectionStringName": "AzureWebJobsStorage",
      "trackingStoreConnectionStringName": "TrackingStorage",
      "trackingStoreNamePrefix": "DurableTask",
      "traceInputsAndOutputs": false,
      "logReplayEvents": false,
      "eventGridTopicEndpoint": "https://topic_name.westus2-1.eventgrid.azure.net/api/events",
      "eventGridKeySettingName": "EventGridKey",
      "eventGridPublishRetryCount": 3,
      "eventGridPublishRetryInterval": "00:00:30",
      "eventGridPublishEventTypes": ["Started", "Completed", "Failed", "Terminated"]
    }
  }
}
```

---

## Core Settings

### hubName

| Default | Description |
|---------|-------------|
| `DurableFunctionsHub` (v1.x), `TestHubName` (v2.x) | The name of the task hub. Must start with a letter and consist of only letters and numbers. |

Task hub names isolate multiple Durable Functions applications from each other, even when using the same storage backend.

```json
{
  "extensions": {
    "durableTask": {
      "hubName": "ProductionTaskHub"
    }
  }
}
```

### maxConcurrentActivityFunctions

| Default | Description |
|---------|-------------|
| Consumption: 10, Dedicated/Premium: 10× processor count | Maximum concurrent activity function executions per host instance |

```json
{
  "extensions": {
    "durableTask": {
      "maxConcurrentActivityFunctions": 20
    }
  }
}
```

### maxConcurrentOrchestratorFunctions

| Default | Description |
|---------|-------------|
| Consumption: 5, Dedicated/Premium: 10× processor count | Maximum concurrent orchestrator function executions per host instance |

```json
{
  "extensions": {
    "durableTask": {
      "maxConcurrentOrchestratorFunctions": 10
    }
  }
}
```

### maxConcurrentEntityFunctions

| Default | Description |
|---------|-------------|
| Consumption: 5, Dedicated/Premium: 10× processor count | Maximum concurrent entity function executions per host instance |

> **Note:** This setting applies when using the Durable Task Scheduler. Otherwise, entity concurrency is controlled by `maxConcurrentOrchestratorFunctions`.

---

## Storage Provider Settings

### connectionStringName / azureStorageConnectionStringName

| Default | Description |
|---------|-------------|
| `AzureWebJobsStorage` | Name of app setting containing the storage connection string |

```json
{
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "connectionStringName": "DurableTaskStorage"
      }
    }
  }
}
```

### partitionCount

| Default | Description |
|---------|-------------|
| 4 | Number of control queue partitions (1-16) |

> **Warning:** Changing this value requires a new task hub. Cannot be modified for an existing task hub.

### controlQueueBatchSize

| Default | Description |
|---------|-------------|
| 32 | Number of messages to pull from control queue at a time |

### controlQueueBufferThreshold

| Default | Description |
|---------|-------------|
| Consumption (Python): 32, Consumption (others): 128, Dedicated/Premium: 256 | Maximum control queue messages buffered in memory |

Reducing this value can lower memory consumption in high-throughput scenarios.

### controlQueueVisibilityTimeout

| Default | Description |
|---------|-------------|
| `00:05:00` | Visibility timeout for dequeued control queue messages (hh:mm:ss) |

### workItemQueueVisibilityTimeout

| Default | Description |
|---------|-------------|
| `00:05:00` | Visibility timeout for dequeued work item queue messages (hh:mm:ss) |

### maxQueuePollingInterval

| Default | Description |
|---------|-------------|
| `00:00:30` | Maximum polling interval for control and work-item queues (hh:mm:ss) |

Lower values reduce latency but increase storage costs.

### trackingStoreConnectionStringName

| Default | Description |
|---------|-------------|
| Uses main connection | App setting name for History and Instances table storage |

Use this to store history in a separate storage account for isolation.

### trackingStoreNamePrefix

| Default | Description |
|---------|-------------|
| `DurableTask` | Prefix for History and Instances tables |

### useLegacyPartitionManagement

| Default | Description |
|---------|-------------|
| `false` | Use legacy partition management algorithm |

> **Warning:** Setting to `true` is not recommended. May cause duplicate function executions during scale-out.

### useTablePartitionManagement

| Default | Description |
|---------|-------------|
| v3.x: `true`, v2.x: `false` | Use table-based partition management for reduced costs |

---

## Performance Settings

### extendedSessionsEnabled

| Default | Description |
|---------|-------------|
| `false` | Cache orchestrator and entity sessions in memory |

Enable for improved throughput when processing many small orchestrations.

```json
{
  "extensions": {
    "durableTask": {
      "extendedSessionsEnabled": true,
      "extendedSessionIdleTimeoutInSeconds": 30
    }
  }
}
```

### extendedSessionIdleTimeoutInSeconds

| Default | Description |
|---------|-------------|
| 30 | Seconds an idle session remains in memory before unloading |

Only used when `extendedSessionsEnabled` is `true`.

### maxEntityOperationBatchSize

| Default | Description |
|---------|-------------|
| Consumption: 50, Dedicated/Premium: 5000 | Maximum entity operations processed as a batch |

Set to 1 to disable batching.

### maxOrchestrationActions

| Default | Description |
|---------|-------------|
| 100,000 | Maximum actions an orchestrator can perform per execution cycle |

---

## Tracing Settings

### traceInputsAndOutputs

| Default | Description |
|---------|-------------|
| `false` | Log full function inputs and outputs |

> **Warning:** May expose sensitive data. Use with caution.

```json
{
  "extensions": {
    "durableTask": {
      "tracing": {
        "traceInputsAndOutputs": true,
        "traceReplayEvents": false
      }
    }
  }
}
```

### traceReplayEvents / logReplayEvents

| Default | Description |
|---------|-------------|
| `false` | Write orchestration replay events to Application Insights |

---

## Event Grid Notifications

Configure Event Grid to receive orchestration lifecycle events.

```json
{
  "extensions": {
    "durableTask": {
      "notifications": {
        "eventGrid": {
          "topicEndpoint": "https://your-topic.westus2-1.eventgrid.azure.net/api/events",
          "keySettingName": "EventGridKey",
          "publishRetryCount": 3,
          "publishRetryInterval": "00:00:30",
          "publishEventTypes": ["Started", "Completed", "Failed", "Terminated"]
        }
      }
    }
  }
}
```

| Property | Description |
|----------|-------------|
| `topicEndpoint` | Event Grid custom topic URL |
| `keySettingName` | App setting containing the Event Grid key |
| `publishRetryCount` | Number of retry attempts on failure |
| `publishRetryInterval` | Retry interval (hh:mm:ss) |
| `publishEventTypes` | Event types to publish: `Started`, `Completed`, `Failed`, `Terminated` |

---

## Advanced Settings

### useAppLease

| Default | Description |
|---------|-------------|
| `true` | Require app-level blob lease before processing messages |

Prevents multiple app instances from processing the same task hub simultaneously.

### useGracefulShutdown

| Default | Description |
|---------|-------------|
| `false` | (Preview) Enable graceful shutdown to prevent in-process failures |

### storeInputsInOrchestrationHistory

| Default | Description |
|---------|-------------|
| `false` | Store activity inputs in History table |

Enable to see activity inputs in history queries.

### FetchLargeMessagesAutomatically

| Default | Description |
|---------|-------------|
| `true` | Auto-retrieve large messages exceeding queue size limits |

### QueueClientMessageEncoding

| Default | Description |
|---------|-------------|
| `UTF8` | Queue message encoding: `UTF8` or `Base64` |

Requires DurableTask extension 3.4.0+ or Worker.Extensions.DurableTask 1.7.0+.

---

## Versioning Settings

### defaultVersion

| Default | Description |
|---------|-------------|
| (none) | Default version for new orchestration instances |

```json
{
  "extensions": {
    "durableTask": {
      "defaultVersion": "2.0",
      "versionMatchStrategy": "CurrentOrOlder",
      "versionFailureStrategy": "Reject"
    }
  }
}
```

### versionMatchStrategy

| Default | Options |
|---------|---------|
| `CurrentOrOlder` | `None`, `Strict`, `CurrentOrOlder` |

### versionFailureStrategy

| Default | Options |
|---------|---------|
| `Reject` | `Reject`, `Fail` |

See [Orchestration Versioning](../../concepts/versioning.md) for details.

---

## Example: High-Throughput Configuration

```json
{
  "version": "2.0",
  "extensions": {
    "durableTask": {
      "hubName": "HighThroughputHub",
      "storageProvider": {
        "controlQueueBatchSize": 64,
        "controlQueueBufferThreshold": 512,
        "maxQueuePollingInterval": "00:00:05",
        "partitionCount": 8
      },
      "maxConcurrentActivityFunctions": 50,
      "maxConcurrentOrchestratorFunctions": 25,
      "extendedSessionsEnabled": true,
      "extendedSessionIdleTimeoutInSeconds": 60
    }
  }
}
```

---

## Example: Low-Latency Configuration

```json
{
  "version": "2.0",
  "extensions": {
    "durableTask": {
      "hubName": "LowLatencyHub",
      "storageProvider": {
        "maxQueuePollingInterval": "00:00:01",
        "controlQueueVisibilityTimeout": "00:00:30"
      },
      "maxConcurrentActivityFunctions": 100,
      "maxConcurrentOrchestratorFunctions": 50
    }
  }
}
```

---

## Next Steps

- [Bindings Reference →](./bindings.md)
- [HTTP API Reference →](./http-api.md)
- [Performance and Scale →](../../concepts/performance-scale.md)
