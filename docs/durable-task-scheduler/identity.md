---
layout: default
title: Identity Configuration
parent: Durable Task Scheduler
nav_order: 4
permalink: /docs/durable-task-scheduler/identity/
---

# Identity Configuration for Durable Task Scheduler
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

The Durable Task Scheduler uses **managed identity** for authentication. This guide explains how to configure identity-based access for your applications.

---

## Overview

Durable Task Scheduler supports two types of managed identity:

| Type | Description | Recommended For |
|------|-------------|-----------------|
| **User-Assigned** | Independent identity resource | Production (can be reused) |
| **System-Assigned** | Tied to the app lifecycle | Development, simple scenarios |

> **Recommendation**: Use **user-assigned managed identity** for production workloads because it's independent of your app's lifecycle and can be reused.

---

## Available Roles

| Role | Description | Use Case |
|------|-------------|----------|
| **Durable Task Data Contributor** | Full data access | Most apps (read, write, manage) |
| **Durable Task Worker** | Worker operations only | Apps that only process work items |
| **Durable Task Data Reader** | Read-only access | Monitoring tools, dashboards |

> **Note**: Most applications require the **Durable Task Data Contributor** role.

---

## Azure Portal Setup

### Step 1: Create a User-Assigned Managed Identity

1. In the Azure portal, search for **Managed Identities**
2. Click **+ Create**
3. Select your subscription and resource group
4. Enter a name (e.g., `durable-task-identity`)
5. Choose a region
6. Click **Review + Create**, then **Create**

### Step 2: Assign Role to the Identity

1. Navigate to your **Durable Task Scheduler** resource
2. Click on your **Task Hub**
3. In the left menu, select **Access control (IAM)**
4. Click **+ Add** → **Add role assignment**
5. Select **Durable Task Data Contributor**
6. Click **Next**
7. For **Assign access to**, select **Managed identity**
8. Click **+ Select members**
9. Choose **User-assigned managed identity**
10. Select your identity and click **Select**
11. Click **Review + assign**

### Step 3: Assign Identity to Your App

#### For Azure Functions

1. Navigate to your Function App
2. Go to **Settings** → **Identity**
3. Click the **User assigned** tab
4. Click **+ Add**
5. Select your managed identity
6. Click **Add**

#### For Azure Container Apps

1. Navigate to your Container App
2. Go to **Settings** → **Identity**
3. Click the **User assigned** tab
4. Click **+ Add**
5. Select your managed identity
6. Click **Add**

### Step 4: Configure Connection String

Add environment variables to your app:

| Variable | Value |
|----------|-------|
| `DURABLE_TASK_SCHEDULER_CONNECTION_STRING` | `Endpoint={endpoint};Authentication=ManagedIdentity;ClientID={client-id}` |
| `TASKHUB_NAME` | Your task hub name |

Where:
- `{endpoint}` = Your scheduler endpoint (e.g., `myscheduler.westus2.durabletask.io`)
- `{client-id}` = Your managed identity's Client ID

---

## Azure CLI Setup

### Step 1: Create User-Assigned Managed Identity

```bash
# Set variables
RESOURCE_GROUP="myResourceGroup"
IDENTITY_NAME="durable-task-identity"
LOCATION="westus2"

# Create the identity
az identity create \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --location $LOCATION

# Get the identity's principal ID and client ID
IDENTITY_PRINCIPAL_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query 'principalId' \
  --output tsv)

IDENTITY_CLIENT_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query 'clientId' \
  --output tsv)

IDENTITY_RESOURCE_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query 'id' \
  --output tsv)
```

### Step 2: Assign Role to the Identity

```bash
# Set variables
SCHEDULER_NAME="myScheduler"
TASKHUB_NAME="myTaskHub"
SUBSCRIPTION_ID=$(az account show --query 'id' --output tsv)

# Scope to task hub level
SCOPE="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.DurableTask/schedulers/$SCHEDULER_NAME/taskHubs/$TASKHUB_NAME"

# Or scope to scheduler level (all task hubs)
# SCOPE="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.DurableTask/schedulers/$SCHEDULER_NAME"

# Assign the role
az role assignment create \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --role "Durable Task Data Contributor" \
  --scope $SCOPE
```

### Step 3: Assign Identity to Your App

#### For Azure Functions

```bash
FUNCTION_APP_NAME="myFunctionApp"

az functionapp identity assign \
  --resource-group $RESOURCE_GROUP \
  --name $FUNCTION_APP_NAME \
  --identities $IDENTITY_RESOURCE_ID
```

#### For Azure Container Apps

```bash
CONTAINER_APP_NAME="myContainerApp"

az containerapp identity assign \
  --resource-group $RESOURCE_GROUP \
  --name $CONTAINER_APP_NAME \
  --identities $IDENTITY_RESOURCE_ID
```

### Step 4: Configure Connection String

```bash
# Get the scheduler endpoint
SCHEDULER_ENDPOINT=$(az durabletask scheduler show \
  --resource-group $RESOURCE_GROUP \
  --name $SCHEDULER_NAME \
  --query 'properties.endpoint' \
  --output tsv)

# Build the connection string
CONNECTION_STRING="Endpoint=$SCHEDULER_ENDPOINT;Authentication=ManagedIdentity;ClientID=$IDENTITY_CLIENT_ID"
```

#### For Azure Functions

```bash
az functionapp config appsettings set \
  --resource-group $RESOURCE_GROUP \
  --name $FUNCTION_APP_NAME \
  --settings \
    "DURABLE_TASK_SCHEDULER_CONNECTION_STRING=$CONNECTION_STRING" \
    "TASKHUB_NAME=$TASKHUB_NAME"
```

#### For Azure Container Apps

```bash
az containerapp update \
  --resource-group $RESOURCE_GROUP \
  --name $CONTAINER_APP_NAME \
  --set-env-vars \
    "DURABLE_TASK_SCHEDULER_CONNECTION_STRING=$CONNECTION_STRING" \
    "TASKHUB_NAME=$TASKHUB_NAME"
```

---

## System-Assigned Identity Setup

If you prefer system-assigned identity:

### Enable System-Assigned Identity

```bash
# For Functions
az functionapp identity assign \
  --resource-group $RESOURCE_GROUP \
  --name $FUNCTION_APP_NAME

# For Container Apps
az containerapp identity assign \
  --resource-group $RESOURCE_GROUP \
  --name $CONTAINER_APP_NAME \
  --system-assigned
```

### Get the Principal ID

```bash
# For Functions
PRINCIPAL_ID=$(az functionapp identity show \
  --resource-group $RESOURCE_GROUP \
  --name $FUNCTION_APP_NAME \
  --query 'principalId' \
  --output tsv)

# For Container Apps
PRINCIPAL_ID=$(az containerapp identity show \
  --resource-group $RESOURCE_GROUP \
  --name $CONTAINER_APP_NAME \
  --query 'principalId' \
  --output tsv)
```

### Assign Role

```bash
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Durable Task Data Contributor" \
  --scope $SCOPE
```

### Connection String (No ClientID)

For system-assigned identity, omit the ClientID:

```
Endpoint={endpoint};Authentication=ManagedIdentity
```

---

## Connection String Reference

### Format

```
Endpoint={endpoint};Authentication={auth-type}[;ClientID={client-id}]
```

### Examples

| Scenario | Connection String |
|----------|-------------------|
| **Local Emulator** | `Endpoint=http://localhost:8080;Authentication=None` |
| **User-Assigned Identity** | `Endpoint=myscheduler.westus2.durabletask.io;Authentication=ManagedIdentity;ClientID=abc123` |
| **System-Assigned Identity** | `Endpoint=myscheduler.westus2.durabletask.io;Authentication=ManagedIdentity` |

---

## Granting Dashboard Access

To access the Durable Task Scheduler dashboard, your personal identity (email) also needs a role assignment:

```bash
# Get your user's object ID
USER_ID=$(az ad user show --id "your.email@example.com" --query "id" --output tsv)

# Assign the role
az role assignment create \
  --assignee $USER_ID \
  --role "Durable Task Data Contributor" \
  --scope $SCOPE
```

---

## Troubleshooting

### "Access denied" or "Unauthorized"

1. Verify the role assignment exists:
   ```bash
   az role assignment list --scope $SCOPE --assignee $IDENTITY_PRINCIPAL_ID
   ```

2. Check if the identity is assigned to your app:
   ```bash
   az functionapp identity show --resource-group $RESOURCE_GROUP --name $FUNCTION_APP_NAME
   ```

3. Wait a few minutes — role assignments can take up to 5 minutes to propagate

### "Invalid connection string"

1. Verify the endpoint format (should not include `https://`)
2. Check that ClientID is correct for user-assigned identity
3. Ensure Authentication type is `ManagedIdentity`

### "Failed to acquire token"

1. Verify the managed identity has the correct role
2. Check the scope of the role assignment
3. Ensure the identity is assigned to your app

---

## Next Steps

- [Set up the Dashboard →](./dashboard.md)
- [Deploy to Azure Container Apps →](../architecture/aca-dts.md)
- [View Code Samples →](../samples/index.md)
