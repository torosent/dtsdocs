---
layout: default
title: Storage Providers
parent: Durable Functions
nav_order: 10
permalink: /docs/durable-functions/storage-providers/
---

# Durable Functions Storage Providers
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Durable Functions is a set of Azure Functions triggers and bindings that are internally powered by the [Durable Task Framework](https://github.com/Azure/durabletask) (DTFx). DTFx supports various backend storage providers, including the Azure Storage provider used by Durable Functions. As of Durable Functions v2.5.0, users can configure their function apps to use DTFx storage providers other than the Azure Storage provider.

> **Note**: The default Azure Storage provider for Durable Functions is the easiest to use since it requires no extra configuration. However, there are cost, scalability, and data management tradeoffs that may favor the use of an alternate backend provider.

Durable Functions supports two types of backend providers: "Bring your own (BYO)" and Azure managed. The BYO options include Azure Storage, Netherite, and Microsoft SQL Server (MSSQL). The Azure managed option is the new durable task scheduler currently in preview.

> **Note**: It's not currently possible to migrate data from one storage backend provider to another. If you want to use a new provider, you should create a new app configured with the new provider.

---

## Durable Task Scheduler (Preview)

The durable task scheduler is a fully managed, high performance backend provider for Durable Functions. It was designed and built from scratch with help from Microsoft Research. This new provider aims to provide the best user experience in aspects such as management, observability, performance, and security.

The key benefits of the durable task scheduler include:

*   **Lower management and operation overhead** compared to BYO backend providers.
*   **First-class observability and management dashboard** provided out-of-the-box.
*   **Highest throughput** of all backends today.
*   **Managed Identity support** for authentication.

Existing Durable Functions users can leverage the scheduler with no code changes.

[Learn more about the Durable Task Scheduler](../durable-task-scheduler/overview.md)

---

## Azure Storage

Azure Storage is the default storage provider for Durable Functions. It uses queues, tables, and blobs to persist orchestration and entity state. It also uses blobs and blob leases to manage partitions.

The key benefits of the Azure Storage provider include:

*   **No setup required**: You can use the storage account created for you by the function app setup experience.
*   **Lowest-cost serverless billing model**: Consumption-based pricing model based entirely on usage.
*   **Best tooling support**: Offers cross-platform local emulation and integrates with Visual Studio, VS Code, and Azure Functions Core Tools.
*   **Most mature**: The original and most battle-tested storage backend.
*   **Identity support**: Can use managed identity instead of secrets.

> **Note**: Standard general purpose Azure Storage accounts are required. We highly recommend using legacy v1 general purpose storage accounts because the newer v2 storage accounts can be more expensive for Durable Functions workloads.

### Configuration

The Azure Storage provider is the default and doesn't require explicit configuration. However, you can customize it in `host.json`.

```json
{
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "type": "AzureStorage",
        "connectionName": "AzureWebJobsStorage"
      }
    }
  }
}
```

---

## Netherite

> **Note**: Support for using the Netherite storage backend with Durable Functions will end 31 March 2028. It is recommended that you start evaluating the Durable Task Scheduler for workloads that you're currently using Netherite for.

The Netherite storage backend was designed and developed by Microsoft Research. It uses Azure Event Hubs and the FASTER database technology on top of Azure Page Blobs. The design of Netherite enables higher-throughput processing of orchestrations and entities compared to other providers.

The key benefits of the Netherite storage provider include:

*   **Higher throughput** at lower cost compared to other storage providers.
*   **Price-performance optimization**, allowing you to scale-up performance as-needed.
*   **Supports up to 32 data partitions** with Event Hubs Basic and Standard SKUs.
*   **More cost-effective** than other providers for high-throughput workloads.

### Configuration

```json
{
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "type": "Netherite",
        "partitionCount": 12
      }
    }
  }
}
```

---

## Microsoft SQL Server (MSSQL)

The Microsoft SQL Server (MSSQL) storage provider persists all state into a Microsoft SQL Server database. It's compatible with both on-premises and cloud-hosted deployments of SQL Server, including Azure SQL Database.

The key benefits of the MSSQL storage provider include:

*   **Supports disconnected environments**: No Azure connectivity is required when using a SQL Server installation.
*   **Portable**: Works across multiple environments and clouds, including Azure-hosted and on-premises.
*   **Strong data consistency**: Enables backup/restore and failover without data loss.
*   **Native support for custom data encryption**.
*   **Integrates with existing database applications** via built-in stored procedures.

### Configuration

```json
{
  "extensions": {
    "durableTask": {
      "storageProvider": {
        "type": "mssql",
        "connectionStringName": "SQLDB_Connection"
      }
    }
  }
}
```

---

## Comparing Storage Providers

| Feature | Azure Storage | Netherite | MSSQL | Durable Task Scheduler |
| :--- | :--- | :--- | :--- | :--- |
| **Official support status** | ✅ GA | ✅ GA | ✅ GA | Public Preview |
| **External dependencies** | Azure Storage (GPv1) | Azure Event Hubs + Azure Storage | SQL Server 2019 or Azure SQL DB | N/A |
| **Local development** | Azurite v3.12+ | In-memory emulation | SQL Server Developer Edition | DTS Emulator |
| **Task hub configuration** | Explicit | Explicit | Implicit by default | Explicit |
| **Maximum throughput** | Moderate | Very high | Moderate | Very high |
| **Max scale-out (nodes)** | 16 | 32 | N/A | N/A |
| **Durable Entities** | ✅ Fully supported | ✅ Fully supported | ⚠️ Supported (except .NET Isolated) | ✅ Fully supported |
| **Disconnected support** | ❌ Azure required | ❌ Azure required | ✅ Fully supported | ❌ Azure required |
| **Identity-based connections** | ✅ Fully supported | ❌ Not supported | ⚠️ Requires runtime-driven scaling | ✅ Fully supported |
| **Flex Consumption plan** | ✅ Fully supported | ❌ Not supported | ✅ Fully supported | ❌ Not supported |

---

## Next Steps

*   [Learn about Durable Functions performance and scale](../concepts/performance-scale.md)
*   [Durable Task Scheduler Overview](../durable-task-scheduler/overview.md)
