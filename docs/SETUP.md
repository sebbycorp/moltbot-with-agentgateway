# 🔧 Setup Guide

This guide walks through setting up Moltbot with AgentGateway step-by-step.

## Prerequisites

- **Kubernetes cluster** (v1.28+)
  - Tested on: Talos Linux, K3s, EKS, GKE, AKS
- **kubectl** configured with cluster access
- **Helm** v3.x installed
- **API Keys** for at least one LLM provider:
  - Anthropic (Claude)
  - OpenAI (GPT)
  - xAI (Grok)

## Step 1: Install AgentGateway

### Option A: Helm (Recommended)

```bash
# Add Solo.io Helm repository
helm repo add solo-io https://storage.googleapis.com/solo-public-helm
helm repo update

# Create namespace
kubectl create namespace agentgateway-system

# Install AgentGateway
helm install agentgateway solo-io/agentgateway \
  --namespace agentgateway-system \
  --set gateway.service.type=NodePort \
  --set gateway.service.nodePort=30890
```

### Option B: Manual Manifests

```bash
kubectl apply -f https://raw.githubusercontent.com/agentgateway/agentgateway/main/deploy/agentgateway.yaml
```

### Verify Installation

```bash
kubectl get pods -n agentgateway-system
# Expected: agentgateway-xxx Running

kubectl get svc -n agentgateway-system
# Expected: agentgateway service with NodePort/LoadBalancer
```

## Step 2: Configure API Key Secrets

Create Kubernetes secrets for your LLM provider API keys:

```bash
# Create secrets (replace with your actual keys)
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

## Step 3: Deploy Backend Configurations

Apply the LLM backend configurations:

```bash
kubectl apply -f manifests/backends/
```

This creates:
- `anthropic-backend` - Routes to Anthropic Claude API
- `openai-backend` - Routes to OpenAI API  
- `xai-backend` - Routes to xAI Grok API

## Step 4: Deploy Gateway and Routes

```bash
kubectl apply -f manifests/gateway/
```

This creates:
- `Gateway` resource exposing the service
- `HTTPRoute` with paths `/anthropic`, `/openai`, `/xai`, `/grok`

### Verify Gateway

```bash
# Get gateway IP/hostname
kubectl get gateway -n agentgateway-system

# Test connectivity
curl http://<gateway-ip>:30890/anthropic/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: test" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"claude-sonnet-4-20250514","max_tokens":10,"messages":[{"role":"user","content":"Hi"}]}'
```

## Step 5: Apply Security Policies

Apply all security policies:

```bash
kubectl apply -f manifests/policies/
```

### Verify Policies

```bash
kubectl get agentgatewaypolicies -n agentgateway-system

# Expected output:
# NAME                         AGE
# block-credit-cards           1m
# block-ssn-numbers            1m
# block-jailbreak-patterns     1m
# ...
```

## Step 6: Configure Moltbot/Clawdbot

Update your Clawdbot configuration to use AgentGateway:

### Edit `~/.clawdbot/config.yaml`

```yaml
providers:
  # Route Anthropic through AgentGateway
  agentgateway-anthropic:
    baseUrl: "http://172.16.10.162:30890/anthropic"
    apiKey: "demo"  # Gateway handles real keys
    
  # Route xAI through AgentGateway  
  agentgateway-xai:
    baseUrl: "http://172.16.10.162:30890/xai"
    apiKey: "demo"

  # Route OpenAI through AgentGateway
  agentgateway-openai:
    baseUrl: "http://172.16.10.162:30890/openai"
    apiKey: "demo"

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
```

### Restart Clawdbot

```bash
clawdbot gateway restart
```

## Step 7: Verify End-to-End

Test that Moltbot requests flow through AgentGateway:

```bash
# Send a test message through Clawdbot
clawdbot chat "Hello, can you confirm you're working?"

# Check AgentGateway logs for the request
kubectl logs -n agentgateway-system -l app=agentgateway --tail=50
```

## Network Considerations

### If Moltbot runs outside the cluster:

1. **NodePort**: Use `<node-ip>:30890`
2. **LoadBalancer**: Use the external IP
3. **Ingress**: Configure with TLS termination

### If Moltbot runs inside the cluster:

Use the internal service DNS:
```yaml
baseUrl: "http://agentgateway.agentgateway-system.svc.cluster.local:8080/anthropic"
```

## Troubleshooting

### Gateway not responding

```bash
# Check pods
kubectl get pods -n agentgateway-system

# Check logs
kubectl logs -n agentgateway-system -l app=agentgateway

# Check service endpoints
kubectl get endpoints -n agentgateway-system
```

### Policies not working

```bash
# Verify policies are applied
kubectl get agentgatewaypolicies -n agentgateway-system -o yaml

# Check policy status
kubectl describe agentgatewaypolicy <policy-name> -n agentgateway-system
```

### API errors from providers

```bash
# Check backend configuration
kubectl get llmbackend -n agentgateway-system -o yaml

# Verify secrets exist
kubectl get secrets -n agentgateway-system
```

## Next Steps

- Review [POLICIES.md](POLICIES.md) for policy customization
- Run the [demo script](../scripts/demo.sh) to see features in action
- Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for common issues
