---
layout: default
title: Durable Task SDKs
nav_order: 5
has_children: true
permalink: /docs/sdks/
---

# Durable Task SDKs
{: .no_toc }

Portable client libraries for building durable orchestrations on any compute platform.
{: .fs-6 .fw-300 }

[Get Started]({{ site.baseurl }}/docs/sdks/quickstart/){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View Samples]({{ site.baseurl }}/docs/sdks/samples/){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## In This Section

| Guide | Description |
|:------|:------------|
| [Overview](overview/) | Introduction to the Durable Task SDKs |
| [Quickstart](quickstart/) | Create your first portable worker |
| [.NET SDK](dotnet/) | Full-featured SDK with source generators |
| [Python SDK](python/) | Async/await based with asyncio support |
| [Java SDK](java/) | JVM-based applications (Preview) |
| [Code Samples](samples/) | Practical examples for all platforms |

---

## Available SDKs

| SDK | Package | Status | 
|:----|:--------|:-------|
| **.NET** | `Microsoft.DurableTask.Worker.AzureManaged` | ✅ GA |
| **Python** | `durabletask-azure` | ✅ GA |
| **Java** | `com.microsoft.durabletask` | ⚠️ Preview |
| **JavaScript** | Coming soon | 🔜 Planned |

---

## Choose Your SDK

All SDKs connect to the same Durable Task Scheduler backend and provide equivalent orchestration capabilities. Choose based on your application's runtime and language preferences.

**Key considerations:**
- **Language**: Choose the SDK for your application's programming language
- **Platform**: All SDKs run on Azure Container Apps, Kubernetes, VMs, or on-premises
- **Features**: .NET SDK has the most complete feature set; Python and Java are production-ready
