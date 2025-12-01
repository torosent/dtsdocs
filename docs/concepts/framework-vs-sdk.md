---
layout: default
title: Framework vs SDKs
parent: Core Concepts
nav_order: 15
permalink: /docs/concepts/framework-vs-sdk/
---

# Durable Task Framework vs. Durable Task SDKs

When building durable workflows on Azure, you may encounter two similar-sounding terms: **Durable Task Framework (DTFx)** and **Durable Task SDKs**. While they share a common lineage and purpose, they serve different roles in the ecosystem.

## At a Glance

| Feature | Durable Task Framework (DTFx) | Durable Task SDKs |
| :--- | :--- | :--- |
| **Primary Use Case** | Powering Azure Functions (In-Process), Low-level .NET extensions | Azure Functions (Isolated), Portable apps (Console, Containers) |
| **Languages** | .NET (C#) only | .NET, Java, Python, Go, JavaScript |
| **Architecture** | Library-based (runs in-process) | Sidecar/Protobuf-based (runs out-of-process) |
| **Complexity** | High (requires deep knowledge of backend providers) | Low (abstracts backend complexity) |

## Durable Task Framework (DTFx)

The **Durable Task Framework** (often referred to as **DTFx**) is the original open-source .NET library that powers Durable Functions.

*   **Repository**: [Azure/durabletask](https://github.com/Azure/durabletask)
*   **Role**: It is the "engine" that manages task orchestration, history replay, and storage provider integration.
*   **Usage**: It is primarily used internally by the Azure Functions host for the **In-Process** model. Developers rarely use DTFx directly unless they are building a custom orchestration engine or extending the framework itself.

## Durable Task SDKs

The **Durable Task SDKs** (Microsoft.DurableTask) are the next generation of client libraries designed to be language-agnostic and portable.

*   **Repository**: [microsoft/durabletask-dotnet](https://github.com/microsoft/durabletask-dotnet), [microsoft/durabletask-java](https://github.com/microsoft/durabletask-java), etc.
*   **Role**: These SDKs provide a clean, idiomatic API for defining orchestrations and activities in various languages. They communicate with a backend (like the Durable Task Scheduler sidecar or the Azure Functions host in Isolated mode) via a gRPC-based protocol (Protobuf).
*   **Usage**: You use these SDKs when:
    *   Building Durable Functions in the **Isolated Worker** model (.NET Isolated, Python, Java, etc.).
    *   Running workflows outside of Azure Functions (e.g., in a Kubernetes container using the Durable Task sidecar).

## Which one should I use?

*   **If you are using Azure Functions (In-Process Model):** You are implicitly using **DTFx**. You interact with it via the `Microsoft.Azure.WebJobs.Extensions.DurableTask` package.
*   **If you are using Azure Functions (Isolated Worker Model):** You are using the **Durable Task SDKs** (e.g., `Microsoft.Azure.Functions.Worker.Extensions.DurableTask`).
*   **If you are building a standalone app (Console, Container):** You should use the **Durable Task SDKs** (e.g., `Microsoft.DurableTask.Worker`).

## Summary

Think of **DTFx** as the low-level engine that powers the platform, and **Durable Task SDKs** as the modern, portable steering wheel that lets you drive it from any language or environment.
