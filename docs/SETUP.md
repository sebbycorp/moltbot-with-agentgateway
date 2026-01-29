# 🔧 Detailed Setup Guide

This guide provides comprehensive instructions for setting up Moltbot with AgentGateway on Kubernetes.

## Prerequisites

- **Kubernetes cluster** (v1.28+)
  - Tested on: Kind, K3s, Talos, EKS, GKE, AKS
- **kubectl** configured with cluster access
- **Helm** v3.x installed
- **API Keys** for at least one LLM provider:
  - Anthropic (Claude)
  - OpenAI (GPT)
  - xAI (Grok)

## Quick Test Environment

For local testing, use [Kind](https://kind.sigs.k8s.io/):

```bash
# Install Kind (if needed)
# macOS: brew install kind
# Linux: curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64 && chmod +x ./kind && sudo mv ./kind /usr/local/bin/

# Create cluster
kind create cluster --name agentgateway-demo

# Verify
kubectl cluster-info
```

## Step 1: Install Gateway API CRDs

AgentGateway uses the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/). Install the standard CRDs:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

Verify installation:

```bash
kubectl get crd | grep gateway
# Expected:
# gatewayclasses.gateway.networking.k8s.io
# gateways.gateway.networking.k8s.io
# httproutes.gateway.networking.k8s.io
# ...
```

## Step 2: Install AgentGateway

AgentGateway is distributed via OCI Helm charts from the [kgateway](https://github.com/kgateway-dev/kgateway) project.

### Install CRDs

```bash
helm upgrade -i --create-namespace \
  --namespace agentgateway-system \
  --version v2.2.0-main agentgateway-crds oci://ghcr.io/kgateway-dev/charts/agentgateway-crds
```

### Install Control Plane

```bash
helm upgrade -i -n agentgateway-system agentgateway oci://ghcr.io/kgateway-dev/charts/agentgateway \
  --version v2.2.0-main \
  --set controller.image.pullPolicy=Always \
  --set controller.extraEnv.KGW_ENABLE_GATEWAY_API_EXPERIMENTAL_FEATURES=true
```

### Verify Installation

```bash
# Check control plane pod
kubectl get pods -n agentgateway-system

# Check GatewayClass was created
kubectl get gatewayclass agentgateway
```

## Step 3: Configure API Key Secrets

Create Kubernetes secrets for your LLM provider API keys:

```bash
# Option A: All keys in one secret
kubectl create secret generic llm-api-keys \
  --namespace agentgateway-system \
  --from-literal=anthropic-key=$ANTHROPIC_API_KEY \
  --from-literal=openai-key=$OPENAI_API_KEY \
  --from-literal=xai-key=$XAI_API_KEY

# Option B: Separate secrets per provider
kubectl create secret generic anthropic-api-key \
  --namespace agentgateway-system \
  --from-literal=api-key=$ANTHROPIC_API_KEY

kubectl create secret generic openai-api-key \
  --namespace agentgateway-system \
  --from-literal=api-key=$OPENAI_API_KEY

kubectl create secret generic xai-api-key \
  --namespace agentgateway-system \
  --from-literal=api-key=$XAI_API_KEY
```

## Step 4: Deploy Backend Configurations

Apply the LLM backend configurations from this repo:

```bash
kubectl apply -f manifests/backends/
```

This creates `AgentGatewayBackend` resources for:
- **anthropic-backend** - Routes to `api.anthropic.com`
- **openai-backend** - Routes to `api.openai.com`
- **xai-backend** - Routes to `api.x.ai`

Verify:

```bash
kubectl get agentgatewaybackend -n agentgateway-system
```

## Step 5: Deploy Gateway and Routes

Create the Gateway resource to spawn an agentgateway proxy:

```bash
kubectl apply -f manifests/gateway/
```

### Wait for Gateway to be Ready

```bash
# Check gateway status (may take 1-2 minutes)
kubectl get gateway -n agentgateway-system -w

# Once ADDRESS is assigned:
# NAME           CLASS          ADDRESS        PROGRAMMED   AGE
# agentgateway   agentgateway   10.96.x.x      True         2m
```

### Get External IP

```bash
# For LoadBalancer service
export GATEWAY_IP=$(kubectl get gateway agentgateway -n agentgateway-system \
  -o jsonpath='{.status.addresses[0].value}')

# For NodePort (Kind/local)
export GATEWAY_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
export GATEWAY_PORT=$(kubectl get svc -n agentgateway-system agentgateway-proxy \
  -o jsonpath='{.spec.ports[0].nodePort}')

echo "Gateway: http://$GATEWAY_IP:${GATEWAY_PORT:-8080}"
```

### Test Connectivity

```bash
# Simple health check
curl -s "http://$GATEWAY_IP:8080/healthz" && echo "Gateway is healthy!"
```

## Step 6: Apply Security Policies

Apply all security policies:

```bash
kubectl apply -f manifests/policies/
```

### Verify Policies

```bash
kubectl get agentgatewaypolicy -n agentgateway-system

# Expected output:
# NAME                         AGE
# pii-protection               1m
# jailbreak-prevention         1m
# credential-protection        1m
# rate-limiting                1m
# prompt-elicitation           1m
```

## Step 7: Configure Moltbot/Clawdbot

Update your Clawdbot configuration to route through AgentGateway.

### Edit `~/.clawdbot/config.yaml`

```yaml
providers:
  # Route Anthropic through AgentGateway
  agentgateway-anthropic:
    baseUrl: "http://<GATEWAY_IP>:8080/anthropic"
    apiKey: "passthrough"  # Gateway injects real key
    
  # Route OpenAI through AgentGateway
  agentgateway-openai:
    baseUrl: "http://<GATEWAY_IP>:8080/openai"
    apiKey: "passthrough"
    
  # Route xAI through AgentGateway  
  agentgateway-xai:
    baseUrl: "http://<GATEWAY_IP>:8080/xai"
    apiKey: "passthrough"

# Map models to gateway providers
models:
  claude-sonnet-4-20250514:
    provider: agentgateway-anthropic
  claude-opus-4-5:
    provider: agentgateway-anthropic
  grok-3-mini-beta:
    provider: agentgateway-xai
  grok-3-beta:
    provider: agentgateway-xai
  gpt-4o:
    provider: agentgateway-openai
  gpt-4o-mini:
    provider: agentgateway-openai
```

### Restart Clawdbot

```bash
clawdbot gateway restart
```

## Step 8: Verify End-to-End

### Test via curl

```bash
# Test Anthropic routing
curl -X POST "http://$GATEWAY_IP:8080/anthropic/v1/messages" \
  -H "Content-Type: application/json" \
  -H "x-api-key: passthrough" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 50,
    "messages": [{"role": "user", "content": "Say hello!"}]
  }'
```

### Test via Clawdbot

```bash
clawdbot chat "Hello! Can you confirm you're working through AgentGateway?"
```

### Check Logs

```bash
# View agentgateway proxy logs
kubectl logs -n agentgateway-system -l app.kubernetes.io/name=agentgateway-proxy --tail=100
```

## Network Configurations

### Moltbot Outside Cluster

| Access Method | Configuration |
|--------------|---------------|
| **NodePort** | `http://<node-ip>:<nodeport>` |
| **LoadBalancer** | `http://<external-ip>:8080` |
| **Ingress** | `https://gateway.yourdomain.com` |

### Moltbot Inside Cluster

Use internal DNS:
```yaml
baseUrl: "http://agentgateway-proxy.agentgateway-system.svc.cluster.local:8080/anthropic"
```

### TLS Configuration

For production, configure TLS via Gateway API:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: gateway-tls-cert
```

## Cleanup

### Remove Everything

```bash
# Delete policies and backends
kubectl delete -f manifests/

# Uninstall helm charts
helm uninstall agentgateway agentgateway-crds -n agentgateway-system

# Delete namespace
kubectl delete namespace agentgateway-system

# Remove Gateway API CRDs (optional)
kubectl delete -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

### Delete Kind Cluster (if used)

```bash
kind delete cluster --name agentgateway-demo
```

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for common issues and solutions.

## Next Steps

- Review [POLICIES.md](POLICIES.md) for customizing security policies
- Run `./scripts/demo.sh` to see all features in action
- Check [agentgateway.dev](https://agentgateway.dev/docs/kubernetes/latest/) for advanced configuration
