---
layout: default
title: Dashboard
parent: Durable Task Scheduler
nav_order: 3
permalink: /docs/durable-task-scheduler/dashboard/
---

# Durable Task Scheduler Dashboard
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

The Durable Task Scheduler includes a built-in monitoring dashboard that provides real-time visibility into your orchestrations. This guide covers how to access and use the dashboard effectively.

---

## Overview

The dashboard allows you to:

- View all orchestration instances
- Monitor orchestration status and progress
- Inspect execution history and timelines
- Debug failed orchestrations
- Perform management operations (terminate, suspend, resume)

```
┌─────────────────────────────────────────────────────────────────┐
│                    DASHBOARD                                    │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Task Hub: myTaskHub                                      │ │
│  ├───────────────────────────────────────────────────────────┤ │
│  │  Orchestrations    │ Status │ Created    │ Duration       │ │
│  ├────────────────────┼────────┼────────────┼────────────────┤ │
│  │  order-12345       │ ✅ Done│ 2 min ago  │ 1.2s           │ │
│  │  order-12346       │ 🔄 Run │ 1 min ago  │ -              │ │
│  │  order-12347       │ ❌ Fail│ 5 min ago  │ 0.8s           │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## Accessing the Dashboard

### Local Development (Emulator)

When using the emulator, the dashboard is available without authentication:

```
http://localhost:<dashboard-port>
```

Find the dashboard port:
```bash
docker ps
# Look for the port mapped to container port 8082
```

### Azure Portal

1. Navigate to your Durable Task Scheduler resource
2. Click on your task hub
3. Find the **Dashboard URL** in the Essentials section
4. Click the URL to open the dashboard

### Direct URL

Access the dashboard directly at:
```
https://dashboard.durabletask.io/?endpoint={scheduler-endpoint}&taskhub={taskhub-name}
```

---

## Authentication

### Required Role

To access the dashboard in Azure, you need one of these roles:

| Role | Permissions |
|------|-------------|
| **Durable Task Data Contributor** | Full access (read, write, manage) |
| **Durable Task Data Reader** | Read-only access |

### Grant Access via Portal

1. Navigate to your scheduler or task hub
2. Click **Access control (IAM)**
3. Click **Add role assignment**
4. Select **Durable Task Data Contributor**
5. Choose **User, group, or service principal**
6. Search for and select your user
7. Click **Review + assign**

### Grant Access via CLI

```bash
# Get your user's object ID
assignee=$(az ad user show --id "your.email@example.com" --query "id" --output tsv)

# Define the scope (scheduler or task hub level)
scope="/subscriptions/{subscription}/resourceGroups/{rg}/providers/Microsoft.DurableTask/schedulers/{scheduler}"

# Or for task hub level
scope="/subscriptions/{subscription}/resourceGroups/{rg}/providers/Microsoft.DurableTask/schedulers/{scheduler}/taskHubs/{taskhub}"

# Assign the role
az role assignment create \
  --assignee "$assignee" \
  --role "Durable Task Data Contributor" \
  --scope "$scope"
```

---

## Dashboard Features

### Orchestration List

The main view shows all orchestration instances:

| Column | Description |
|--------|-------------|
| **Instance ID** | Unique identifier for the orchestration |
| **Name** | Orchestrator function name |
| **Status** | Current state (Running, Completed, Failed, etc.) |
| **Created** | When the orchestration started |
| **Last Updated** | Most recent activity |
| **Duration** | Total execution time |

### Filtering and Search

Filter orchestrations by:
- **Status**: Running, Completed, Failed, Suspended, Terminated
- **Time range**: Created or updated within a time window
- **Instance ID**: Search for specific instances

### Orchestration Details

Click an orchestration to see:

#### Overview Tab
- Input and output data
- Start and end times
- Current status and any error messages

#### Timeline Tab
Visual representation of execution:

```
Start ──▶ Activity1 ──▶ Activity2 ──▶ Activity3 ──▶ Complete
  │          │            │            │              │
  0s        0.5s         1.2s         2.1s           3.0s
```

#### History Tab
Complete event history:

| Event | Time | Details |
|-------|------|---------|
| OrchestratorStarted | 0ms | - |
| TaskScheduled | 5ms | Activity: ValidateOrder |
| TaskCompleted | 120ms | Result: true |
| TaskScheduled | 125ms | Activity: ProcessPayment |
| ... | ... | ... |

---

## Management Operations

### Terminate an Orchestration

Stop a running orchestration immediately:

1. Select the orchestration
2. Click **Terminate**
3. Optionally provide a reason
4. Confirm

### Suspend an Orchestration

Pause a running orchestration:

1. Select the orchestration
2. Click **Suspend**

### Resume an Orchestration

Resume a suspended orchestration:

1. Select the orchestration
2. Click **Resume**

### Rewind an Orchestration

Replay from a failed state:

1. Select the failed orchestration
2. Click **Rewind**
3. Choose the rewind point

---

## Debugging with the Dashboard

### Investigate Failed Orchestrations

1. Filter by **Status: Failed**
2. Click the failed orchestration
3. Check the **Overview** tab for error messages
4. Review the **Timeline** to see where it failed
5. Examine the **History** for detailed event information

### Identify Performance Issues

1. Look for orchestrations with long durations
2. Check the **Timeline** for slow activities
3. Identify bottlenecks in parallel operations

### Monitor Long-Running Orchestrations

1. Filter by **Status: Running**
2. Check the **Last Updated** time
3. Investigate if expected activities are completing

---

## Dashboard Views by Role

### Developer View

Focus on debugging and understanding orchestration behavior:
- Detailed history inspection
- Input/output examination
- Timeline analysis

### Operations View

Focus on monitoring and health:
- Status distribution overview
- Long-running orchestration detection
- Failed orchestration alerts

---

## Tips and Best Practices

### ✅ Do

- Use meaningful instance IDs for easy searching
- Check the dashboard after deployments
- Set up alerts for failed orchestrations
- Regularly review long-running instances

### ❌ Don't

- Leave failed orchestrations uninvestigated
- Ignore growing numbers of suspended instances
- Overlook unusual patterns in orchestration timing

---

## Troubleshooting

### "Unable to load dashboard"

- Verify you have the required role assignment
- Check that the scheduler endpoint is correct
- Ensure you're signed in with the right account

### "No orchestrations found"

- Verify the task hub name is correct
- Check that orchestrations have been started
- Confirm the time filter isn't excluding results

### "Access denied"

- Verify role assignment (may take a few minutes to propagate)
- Check scope (scheduler vs. task hub level)
- Ensure the correct identity is being used

---

## Next Steps

- [Configure Identity Access →](./identity.md)
- [Learn about Patterns →](../patterns/index.md)
- [View Code Samples →](../samples/index.md)
