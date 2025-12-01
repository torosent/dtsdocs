---
layout: default
title: AKS + DTS
parent: Architecture Guides
nav_order: 3
permalink: /docs/architecture/aks-dts/
---

# Azure Kubernetes Service with Durable Task SDK
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Deploy durable orchestrations to AKS for full control over scaling, networking, and operations.

---

## Overview

Azure Kubernetes Service (AKS) provides the most flexibility for deploying Durable Task SDK workers. Use AKS when you need fine-grained control over infrastructure, custom scaling, or integration with existing Kubernetes workloads.

```
┌──────────────────────────────────────────────────────────────────┐
│                  AKS + DURABLE TASK SDK                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │                   AKS Cluster                             │   │
│   │                                                           │   │
│   │   ┌───────────────────────────────────────────────────┐   │   │
│   │   │  Namespace: durable-system                        │   │   │
│   │   │                                                    │   │   │
│   │   │   ┌─────────────┐      ┌─────────────────────┐    │   │   │
│   │   │   │  API        │      │  Worker Deployment  │    │   │   │
│   │   │   │  Deployment │      │  (HPA enabled)      │    │   │   │
│   │   │   │             │      │                     │    │   │   │
│   │   │   │  Pods: 2    │      │  Pods: 3-20         │    │   │   │
│   │   │   └──────┬──────┘      └──────────┬──────────┘    │   │   │
│   │   │          │                        │               │   │   │
│   │   │          │     ┌──────────────────┘               │   │   │
│   │   │          │     │                                  │   │   │
│   │   │   ┌──────▼─────▼─────┐                           │   │   │
│   │   │   │  Ingress         │                           │   │   │
│   │   │   │  (NGINX/Gateway) │                           │   │   │
│   │   │   └──────────────────┘                           │   │   │
│   │   │                                                    │   │   │
│   │   └────────────────────────────────────────────────────┘   │
│   │                              │                               │
│   └──────────────────────────────┼───────────────────────────┘   │
│                                  │ gRPC                          │
│                                  ▼                               │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │                 Durable Task Scheduler                    │   │
│   └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

1. Azure subscription
2. Azure CLI with aks-preview extension
3. kubectl configured
4. Helm (optional, for ingress)
5. Docker

---

## Step 1: Create Infrastructure

```bash
# Variables
RESOURCE_GROUP="rg-durable-aks"
LOCATION="centralus"
CLUSTER_NAME="durable-aks"
SCHEDULER_NAME="my-scheduler"
REGISTRY_NAME="myacrregistry"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create AKS cluster with workload identity
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --node-count 3 \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --generate-ssh-keys

# Get credentials
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME

# Create Container Registry
az acr create \
  --resource-group $RESOURCE_GROUP \
  --name $REGISTRY_NAME \
  --sku Basic

# Attach ACR to AKS
az aks update \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --attach-acr $REGISTRY_NAME

# Create Durable Task Scheduler
az durabletask scheduler create \
  --resource-group $RESOURCE_GROUP \
  --name $SCHEDULER_NAME \
  --location $LOCATION \
  --sku dedicated

# Create Task Hub
az durabletask taskhub create \
  --resource-group $RESOURCE_GROUP \
  --scheduler-name $SCHEDULER_NAME \
  --name default
```

---

## Step 2: Configure Workload Identity

```bash
# Create managed identity
az identity create \
  --resource-group $RESOURCE_GROUP \
  --name durable-workload-identity

IDENTITY_CLIENT_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name durable-workload-identity \
  --query clientId -o tsv)

IDENTITY_PRINCIPAL_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name durable-workload-identity \
  --query principalId -o tsv)

# Assign Durable Task role
SCHEDULER_ID=$(az durabletask scheduler show \
  --name $SCHEDULER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

az role assignment create \
  --role "Durable Task Data Contributor" \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --scope $SCHEDULER_ID

# Get AKS OIDC issuer
AKS_OIDC_ISSUER=$(az aks show \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --query oidcIssuerProfile.issuerUrl -o tsv)

# Create federated credential
az identity federated-credential create \
  --name durable-federated \
  --identity-name durable-workload-identity \
  --resource-group $RESOURCE_GROUP \
  --issuer $AKS_OIDC_ISSUER \
  --subject system:serviceaccount:durable-system:durable-worker-sa
```

---

## Step 3: Build and Push Images

```bash
# Build and push worker image
cd durable-worker
az acr build --registry $REGISTRY_NAME --image durable-worker:v1 .

# Build and push API image
cd ../durable-api
az acr build --registry $REGISTRY_NAME --image durable-api:v1 .
```

---

## Step 4: Create Kubernetes Manifests

### namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: durable-system
```

### serviceaccount.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: durable-worker-sa
  namespace: durable-system
  annotations:
    azure.workload.identity/client-id: "${IDENTITY_CLIENT_ID}"
```

### configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: durable-config
  namespace: durable-system
data:
  DTS_ENDPOINT: "https://${SCHEDULER_NAME}.${LOCATION}.durabletask.io"
  TASKHUB_NAME: "default"
```

### worker-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: durable-worker
  namespace: durable-system
spec:
  replicas: 3
  selector:
    matchLabels:
      app: durable-worker
  template:
    metadata:
      labels:
        app: durable-worker
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: durable-worker-sa
      containers:
      - name: worker
        image: ${REGISTRY_NAME}.azurecr.io/durable-worker:v1
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
        envFrom:
        - configMapRef:
            name: durable-config
        env:
        - name: AZURE_CLIENT_ID
          value: "${IDENTITY_CLIENT_ID}"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
```

### api-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: durable-api
  namespace: durable-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: durable-api
  template:
    metadata:
      labels:
        app: durable-api
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: durable-worker-sa
      containers:
      - name: api
        image: ${REGISTRY_NAME}.azurecr.io/durable-api:v1
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        envFrom:
        - configMapRef:
            name: durable-config
        env:
        - name: AZURE_CLIENT_ID
          value: "${IDENTITY_CLIENT_ID}"
---
apiVersion: v1
kind: Service
metadata:
  name: durable-api-service
  namespace: durable-system
spec:
  selector:
    app: durable-api
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

### hpa.yaml

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: durable-worker-hpa
  namespace: durable-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: durable-worker
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: durable-api-ingress
  namespace: durable-system
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - durable-api.example.com
    secretName: durable-api-tls
  rules:
  - host: durable-api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: durable-api-service
            port:
              number: 80
```

---

## Step 5: Deploy to AKS

```bash
# Create namespace
kubectl apply -f namespace.yaml

# Substitute variables and apply
envsubst < serviceaccount.yaml | kubectl apply -f -
envsubst < configmap.yaml | kubectl apply -f -
envsubst < worker-deployment.yaml | kubectl apply -f -
envsubst < api-deployment.yaml | kubectl apply -f -
kubectl apply -f hpa.yaml

# Optional: Deploy ingress
kubectl apply -f ingress.yaml

# Verify deployments
kubectl get pods -n durable-system
kubectl get hpa -n durable-system
```

---

## Step 6: Add Health Endpoints to Worker

### Program.cs with Health Checks

```csharp
var builder = Host.CreateApplicationBuilder(args);

// Add health check endpoints
builder.Services.AddHealthChecks();
builder.WebHost.ConfigureKestrel(options => options.ListenAnyIP(8080));

builder.Services.AddDurableTaskWorker(options =>
{
    options.AddOrchestrator<OrderProcessingOrchestrator>();
    options.AddActivity<ValidateOrderActivity>();
})
.UseDurableTaskScheduler(endpoint, taskHub);

var host = builder.Build();

// Map health endpoints
var app = builder.Build();
app.MapHealthChecks("/health");
app.MapHealthChecks("/ready");

await app.RunAsync();
```

---

## KEDA Scaling (Advanced)

For queue-based scaling with KEDA:

### keda-scaledobject.yaml

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: durable-worker-scaler
  namespace: durable-system
spec:
  scaleTargetRef:
    name: durable-worker
  pollingInterval: 15
  cooldownPeriod: 300
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
  - type: azure-queue
    metadata:
      queueName: control-queue
      accountName: storageaccount
    authenticationRef:
      name: azure-queue-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-queue-auth
  namespace: durable-system
spec:
  podIdentity:
    provider: azure-workload
    identityId: ${IDENTITY_CLIENT_ID}
```

---

## Monitoring with Prometheus

### servicemonitor.yaml

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: durable-worker-monitor
  namespace: durable-system
spec:
  selector:
    matchLabels:
      app: durable-worker
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

---

## Deployment Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                    AKS DEPLOYMENT CHECKLIST                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Infrastructure:                                                 │
│  ☐ AKS cluster with OIDC and workload identity                 │
│  ☐ Container Registry attached to AKS                          │
│  ☐ Durable Task Scheduler and Task Hub                         │
│                                                                  │
│  Identity:                                                       │
│  ☐ Managed identity created                                     │
│  ☐ Federated credential configured                              │
│  ☐ Durable Task Data Contributor role assigned                  │
│                                                                  │
│  Application:                                                    │
│  ☐ Container images built and pushed                            │
│  ☐ Kubernetes manifests applied                                 │
│  ☐ HPA configured for worker scaling                            │
│  ☐ Ingress configured for API                                   │
│                                                                  │
│  Optional:                                                       │
│  ☐ KEDA for queue-based scaling                                 │
│  ☐ Prometheus/Grafana for monitoring                            │
│  ☐ cert-manager for TLS                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Testing

```bash
# Port-forward for local testing
kubectl port-forward svc/durable-api-service 8080:80 -n durable-system

# Start orchestration
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"id": "order-123", "items": ["item1"], "total": 99.99}'

# Check status
curl http://localhost:8080/orders/<instanceId>

# Check scaling
kubectl get hpa -n durable-system -w
```

---

## Next Steps

- [Configure Identity →](../durable-task-scheduler/identity.md)
- [Use the Dashboard →](../durable-task-scheduler/dashboard.md)
- [Implement Patterns →](../patterns/index.md)

---

## Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| `CreateContainerConfigError` | Missing environment variables | Check ConfigMap applied; verify `envsubst` processed variables |
| `403 Forbidden` to Scheduler | Workload identity misconfigured | Verify federated credential matches service account namespace/name |
| Pods stuck in `Pending` | Insufficient cluster resources | Scale node pool or reduce resource requests |
| `CrashLoopBackOff` | Application error or missing config | Check pod logs: `kubectl logs -n durable-system <pod-name>` |
| HPA not scaling | Metrics server issue | Verify metrics-server is running: `kubectl get pods -n kube-system` |
| Ingress 502 errors | Service/pod not ready | Check readiness probe; verify service selector matches pod labels |

**Debugging Commands:**

```bash
# View pod logs
kubectl logs -n durable-system -l app=durable-worker --tail=100

# Describe pod for events
kubectl describe pod -n durable-system -l app=durable-worker

# Check workload identity token
kubectl exec -it -n durable-system <pod-name> -- \
  cat /var/run/secrets/azure/tokens/azure-identity-token

# Verify service account annotations
kubectl get sa durable-worker-sa -n durable-system -o yaml

# Test DNS resolution from pod
kubectl exec -it -n durable-system <pod-name> -- \
  nslookup $SCHEDULER_NAME.$LOCATION.durabletask.io

# Check HPA status
kubectl describe hpa durable-worker-hpa -n durable-system

# View KEDA scaler logs (if using KEDA)
kubectl logs -n keda -l app=keda-operator --tail=50
```

**Common Configuration Mistakes:**

1. **Federated credential subject**: Must exactly match `system:serviceaccount:<namespace>:<service-account-name>`
2. **Workload identity label**: Pods must have `azure.workload.identity/use: "true"` label
3. **Service account annotation**: Must include `azure.workload.identity/client-id` annotation
4. **ConfigMap not applied**: Run `kubectl get configmap -n durable-system` to verify
5. **Variable substitution**: Use `envsubst` or Helm for variable replacement before applying manifests
6. **OIDC issuer mismatch**: Federated credential issuer must match AKS cluster's OIDC issuer URL
