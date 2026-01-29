# 🔧 Troubleshooting Guide

Common issues and solutions when running Moltbot with AgentGateway.

## Connection Issues

### "Connection refused" from Moltbot

**Symptoms:**
- Moltbot shows connection errors
- `curl` to gateway times out

**Solutions:**

1. **Check gateway is running:**
```bash
kubectl get pods -n agentgateway-system
# Should show agentgateway-xxx in Running state
```

2. **Verify service is exposed:**
```bash
kubectl get svc -n agentgateway-system
# Check NodePort or LoadBalancer IP
```

3. **Check network path:**
```bash
# From Moltbot host
curl -v http://<gateway-ip>:30890/healthz
```

4. **Firewall rules:**
```bash
# Ensure port 30890 is open
sudo iptables -L -n | grep 30890
```

### "401 Unauthorized" responses

**Symptoms:**
- Gateway rejects requests with 401

**Solutions:**

1. **Check API key secrets exist:**
```bash
kubectl get secrets -n agentgateway-system
# Should see: anthropic-api-key, openai-api-key, etc.
```

2. **Verify secret has correct key:**
```bash
kubectl get secret anthropic-api-key -n agentgateway-system -o jsonpath='{.data.api-key}' | base64 -d
```

3. **Check backend references secret correctly:**
```bash
kubectl get llmbackend -n agentgateway-system -o yaml
```

---

## Policy Issues

### Legitimate requests being blocked

**Symptoms:**
- Valid requests return "Request blocked" errors
- False positives from PII/jailbreak detection

**Solutions:**

1. **Check which policy triggered:**
```bash
kubectl logs -n agentgateway-system -l app=agentgateway --tail=100 | grep -i "blocked\|policy"
```

2. **Review policy regex:**
```bash
kubectl get agentgatewaypolicy <policy-name> -n agentgateway-system -o yaml
```

3. **Temporarily disable suspect policy:**
```bash
kubectl delete agentgatewaypolicy <policy-name> -n agentgateway-system
```

4. **Adjust regex to reduce false positives:**
   - Make patterns more specific
   - Add word boundaries (`\b`)
   - Use negative lookahead for exceptions

### Policies not taking effect

**Symptoms:**
- Requests that should be blocked are passing through
- No policy enforcement visible

**Solutions:**

1. **Verify policy is applied:**
```bash
kubectl get agentgatewaypolicies -n agentgateway-system
```

2. **Check policy targets correct route:**
```bash
kubectl get agentgatewaypolicy <name> -o yaml | grep -A5 targetRefs
```

3. **Verify HTTPRoute exists:**
```bash
kubectl get httproute -n agentgateway-system
```

4. **Check for policy errors:**
```bash
kubectl describe agentgatewaypolicy <name> -n agentgateway-system
```

---

## Performance Issues

### High latency on requests

**Symptoms:**
- Requests take much longer through gateway
- Timeouts occurring

**Solutions:**

1. **Check gateway resource usage:**
```bash
kubectl top pods -n agentgateway-system
```

2. **Review policy complexity:**
   - Multiple complex regex patterns add latency
   - Consider combining related patterns

3. **Scale gateway if needed:**
```bash
kubectl scale deployment agentgateway -n agentgateway-system --replicas=3
```

4. **Check network latency to LLM providers:**
```bash
curl -w "@curl-format.txt" -o /dev/null -s https://api.anthropic.com
```

### Rate limiting unexpectedly

**Symptoms:**
- "Rate limit exceeded" errors
- Requests being throttled

**Solutions:**

1. **Check current rate limit settings:**
```bash
kubectl get agentgatewaypolicy -l category=rate-limiting -o yaml
```

2. **Increase limits if needed:**
```yaml
spec:
  default:
    rateLimit:
      requestsPerUnit: 100  # Increase from 10
      unit: MINUTE
```

3. **Check token usage:**
```bash
# Review metrics for token consumption
kubectl port-forward svc/agentgateway -n agentgateway-system 9090:9090
curl localhost:9090/metrics | grep token
```

---

## LLM Provider Errors

### "Model not found" errors

**Symptoms:**
- 404 errors from LLM providers
- Model name mismatches

**Solutions:**

1. **Verify model name in request:**
   - Anthropic: `claude-sonnet-4-20250514`
   - OpenAI: `gpt-4o`
   - xAI: `grok-3-mini-beta`

2. **Check backend model list:**
```bash
kubectl get llmbackend -o yaml | grep -A10 models
```

3. **Update to valid model names in backend config**

### "Insufficient quota" errors

**Symptoms:**
- 429 errors from providers
- Quota exceeded messages

**Solutions:**

1. **Check provider dashboard for usage**

2. **Implement request queuing**

3. **Add failover to alternative provider:**
```yaml
spec:
  failover:
    targets:
      - name: openai-backend
      - name: anthropic-backend  # Fallback
```

---

## Moltbot Configuration Issues

### Moltbot not using gateway

**Symptoms:**
- Requests go directly to LLM providers
- Gateway logs show no traffic

**Solutions:**

1. **Check Moltbot config:**
```bash
cat ~/.clawdbot/config.yaml | grep -A5 providers
```

2. **Verify baseUrl points to gateway:**
```yaml
providers:
  agentgateway-anthropic:
    baseUrl: "http://172.16.10.162:30890/anthropic"  # ← Must be gateway URL
```

3. **Restart Clawdbot:**
```bash
clawdbot gateway restart
```

4. **Test with curl to confirm gateway path:**
```bash
curl http://<gateway>:30890/anthropic/v1/messages \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-sonnet-4-20250514","max_tokens":10,"messages":[{"role":"user","content":"Hi"}]}'
```

---

## Debugging Commands

### Useful kubectl commands

```bash
# All gateway resources
kubectl get all -n agentgateway-system

# Gateway logs (follow)
kubectl logs -f -n agentgateway-system -l app=agentgateway

# Policy list with details
kubectl get agentgatewaypolicies -n agentgateway-system -o wide

# Describe specific policy
kubectl describe agentgatewaypolicy block-credit-cards -n agentgateway-system

# Check events for errors
kubectl get events -n agentgateway-system --sort-by='.lastTimestamp'

# Get gateway endpoints
kubectl get endpoints -n agentgateway-system
```

### Test request through gateway

```bash
# Simple test
curl -X POST "http://<gateway>:30890/anthropic/v1/messages" \
  -H "Content-Type: application/json" \
  -H "x-api-key: test" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 50,
    "messages": [{"role": "user", "content": "Say hello"}]
  }'

# Test PII blocking
curl -X POST "http://<gateway>:30890/anthropic/v1/messages" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "My SSN is 123-45-6789"}]}'
# Should return 403 with blocking message
```

---

## Getting Help

If you're still stuck:

1. Check [AgentGateway docs](https://docs.solo.io/agentgateway)
2. Search [GitHub issues](https://github.com/agentgateway/agentgateway/issues)
3. Join Solo.io community Slack
4. Open an issue with:
   - Gateway version
   - Kubernetes version
   - Relevant logs
   - Steps to reproduce
