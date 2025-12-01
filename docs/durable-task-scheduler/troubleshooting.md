---
layout: default
title: Troubleshooting
parent: Durable Task Scheduler
nav_order: 7
permalink: /docs/durable-task-scheduler/troubleshooting/
---

# Troubleshooting Guide
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

This guide helps you diagnose and resolve common issues with the Durable Task Scheduler.
{: .fs-6 .fw-300 }

{: .note }
> If you're unable to diagnose your problem after going through this guide, you can file a support ticket via **Help** > **Support + troubleshooting** in your Durable Task Scheduler resource on the Azure portal.

---

## Connection Issues

### Check Connection String and Access

When your app isn't running as expected, first verify:
- The correct connection string format
- Authentication is set up correctly

#### Local Development

1. **Verify connection string format:**
   ```
   Endpoint=http://localhost:<port>;Authentication=None
   ```
   
   Ensure the port number is mapped to `8080` on the container running the emulator.

2. **Check emulator is running:**
   ```bash
   docker ps
   # Look for mcr.microsoft.com/dts/dts-emulator
   ```

3. **Verify Azurite is started:**
   
   The Azure Storage emulator (Azurite) is required for Azure Functions components:
   ```bash
   azurite --silent --location ./azurite --debug ./azurite/debug.log
   ```

#### Running on Azure

1. **Check environment variables:**
   - `DURABLE_TASK_SCHEDULER_CONNECTION_STRING`
   - `TASKHUB_NAME`

2. **Verify connection string format:**

   **User-assigned managed identity:**
   ```
   Endpoint={scheduler endpoint};Authentication=ManagedIdentity;ClientID={client id}
   ```

   **System-assigned managed identity:**
   ```
   Endpoint={scheduler endpoint};Authentication=ManagedIdentity
   ```

3. **Verify RBAC permissions:**
   
   Ensure the identity has the required role assigned on the task hub or scheduler:
   ```bash
   az role assignment list \
     --assignee {identity-principal-id} \
     --scope "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.DurableTask/schedulers/{scheduler}"
   ```

4. **For user-assigned managed identity:**
   
   Ensure the identity is assigned to your app:
   ```bash
   az functionapp identity show --name {app-name} --resource-group {rg}
   ```

---

## Deployment Errors

### "Encountered an error (ServiceUnavailable) from host runtime"

**Cause:** Missing or incorrect environment variables.

**Solution:**
1. Verify required environment variables are set:
   - `DURABLE_TASK_SCHEDULER_CONNECTION_STRING`
   - `TASKHUB_NAME`

2. Redeploy your app:
   ```bash
   func azure functionapp publish {app-name}
   ```

3. If you see errors loading functions, click **Refresh** in the portal.

### "Cannot delete resource while nested resources exist"

**Cause:** Task hubs still exist in the scheduler.

**Error message:**
```json
{
  "error": {
    "code": "CannotDeleteResource",
    "message": "Cannot delete resource while nested resources exist. Some existing nested resource IDs include: 'Microsoft.DurableTask/schedulers/YOUR_SCHEDULER/taskhubs/YOUR_TASKHUB'."
  }
}
```

**Solution:** Delete all task hubs before deleting the scheduler:

```bash
# List task hubs
az durabletask taskhub list \
  --scheduler-name {scheduler} \
  --resource-group {rg}

# Delete each task hub
az durabletask taskhub delete \
  --scheduler-name {scheduler} \
  --resource-group {rg} \
  --name {taskhub}

# Then delete the scheduler
az durabletask scheduler delete \
  --name {scheduler} \
  --resource-group {rg}
```

---

## Dashboard Issues

### "Unknown error retrieving details of this task hub"

**Possible causes:**

1. **Missing permissions:**
   
   Your identity doesn't have the required role. Grant access:
   ```bash
   az role assignment create \
     --assignee {your-email} \
     --role "Durable Task Data Contributor" \
     --scope "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.DurableTask/schedulers/{scheduler}/taskHubs/{taskhub}"
   ```

2. **Task hub was deleted:**
   
   Verify the task hub exists:
   ```bash
   az durabletask taskhub show \
     --scheduler-name {scheduler} \
     --resource-group {rg} \
     --name {taskhub}
   ```

### "Unable to load dashboard"

**Solutions:**
- Verify you have the required role assignment
- Check that the scheduler endpoint is correct
- Ensure you're signed in with the correct Azure account
- Clear browser cache and try again

### "Access denied"

**Solutions:**
- Role assignments may take a few minutes to propagate—wait and retry
- Check scope (scheduler vs. task hub level)
- Verify the correct identity is being used

---

## Build and Runtime Errors

### "Can't determine Project to build"

**Error:** `Expected 1 .csproj or .fsproj but found 2`

**Solution:**
1. Delete the `bin` and `obj` directories:
   ```bash
   rm -rf bin obj
   ```

2. Run again:
   ```bash
   func start
   ```

### "Can't find native binaries for ARM"

**Cause:** gRPC library issues on ARM devices (e.g., M1/M2/M3 Macs).

**Solution:** Add the following workaround to your `.csproj` or `extensions.csproj` file:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <!-- Existing content... -->

  <!-- Add these groups for ARM workaround -->
  <ItemGroup>
    <PackageReference Include="Contrib.Grpc.Core.M1" Version="2.41.0" />
  </ItemGroup>
  
  <Target Name="CopyGrpcNativeAssetsToOutDir" AfterTargets="Build">
    <ItemGroup>
       <NativeAssetToCopy Condition="$([MSBuild]::IsOSPlatform('OSX'))" 
                          Include="$(OutDir)runtimes/osx-arm64/native/*"/>
    </ItemGroup>
    <Copy SourceFiles="@(NativeAssetToCopy)" 
          DestinationFolder="$(OutDir).azurefunctions/runtimes/osx-arm64/native"/>
  </Target>
</Project>
```

### gRPC Runtime Issues on Mac (ARM64)

For M1/M2/M3 Mac users experiencing gRPC runtime issues:

1. Add the `Contrib.Grpc.Core.M1` package reference (version 2.41.0)
2. Add a custom after-build target:

```xml
<ItemGroup>
  <PackageReference Include="Contrib.Grpc.Core.M1" Version="2.41.0" />
</ItemGroup>

<Target Name="CopyGrpcNativeAssetsToOutDir" AfterTargets="Build">
  <ItemGroup>
     <NativeAssetToCopy Condition="$([MSBuild]::IsOSPlatform('OSX'))" 
                        Include="$(OutDir)runtimes/osx-arm64/native/*"/>
  </ItemGroup>
  <Copy SourceFiles="@(NativeAssetToCopy)" 
        DestinationFolder="$(OutDir).azurefunctions/runtimes/osx-arm64/native"/>
</Target>
```

---

## Orchestration Issues

### Orchestration Stuck in "Pending"

**Possible causes:**
1. Worker not processing—check app logs
2. Connection issues to scheduler
3. Emulator not running (local development)

**Solutions:**
1. Check emulator logs:
   ```bash
   docker logs $(docker ps -q --filter ancestor=mcr.microsoft.com/dts/dts-emulator)
   ```

2. Verify connection string is correct

3. Check the dashboard for errors

### Activity Timeout

**Cause:** Long-running activity exceeding timeout.

**Solutions:**
1. Increase timeout in retry policy
2. Optimize activity code
3. Consider breaking into smaller activities

### Orchestration Not Completing

**Possible causes:**
1. Activity failures without proper error handling
2. Waiting for external events that never arrive
3. Timers that haven't expired

**Debugging steps:**
1. Open the dashboard
2. Find the orchestration by instance ID
3. Check the **Timeline** view for the last completed step
4. Check the **History** view for error details

---

## Diagnostic Tools

### Enable Verbose Logging

Add to `host.json`:
```json
{
  "version": "2.0",
  "logging": {
    "logLevel": {
      "default": "Debug",
      "DurableTask": "Debug"
    }
  }
}
```

### Check Emulator Logs

```bash
# Find container ID
docker ps

# View logs
docker logs {container-id}

# Follow logs in real-time
docker logs -f {container-id}
```

### Use the Dashboard

The dashboard provides detailed execution information:

1. **Timeline View** — Visual execution flow
2. **History View** — Detailed event sequence with timestamps
3. **Sequence View** — Event ordering

### Application Insights

In production, use Application Insights for:
- End-to-end transaction tracing
- Performance monitoring
- Error tracking

---

## Common Error Messages

| Error Message | Cause | Solution |
|---------------|-------|----------|
| `Connection refused on port 8080` | Emulator not running | Start the emulator container |
| `Function app won't start` | Missing packages | Run `dotnet restore` |
| `Orchestration stuck in Pending` | Worker not processing | Check emulator and app logs |
| `Activity timeout` | Long-running activity | Optimize or increase timeout |
| `401 Unauthorized` | Missing authentication | Configure managed identity |
| `403 Forbidden` | Missing RBAC role | Assign appropriate role |

---

## Getting Support

If you can't resolve your issue:

1. **Gather diagnostic information:**
   - Error messages and stack traces
   - Connection string (redact secrets)
   - `host.json` configuration
   - Emulator or Function App logs

2. **Check Azure Status:**
   [Azure Status Page](https://status.azure.com/)

3. **File a support ticket:**
   - Navigate to your Durable Task Scheduler resource
   - Go to **Help** > **Support + troubleshooting**
   - Create a new support request

4. **Community resources:**
   - [Azure Functions GitHub Issues](https://github.com/Azure/azure-functions-durable-extension/issues)
   - [Stack Overflow](https://stackoverflow.com/questions/tagged/azure-durable-functions)

---

## Next Steps

- [Setup Guide →](./setup.md)
- [Identity Configuration →](./identity.md)
- [Dashboard →](./dashboard.md)
