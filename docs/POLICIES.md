# 🛡️ Security Policies Reference

This document details all security policies available for protecting AI traffic.

## Policy Categories

| Category | Purpose |
|----------|---------|
| Rate Limiting | Control request/token usage |
| PII Protection | Block sensitive data |
| Jailbreak Prevention | Stop prompt injection |
| Credential Protection | Prevent API key leaks |
| Prompt Elicitation | Auto-enrich prompts |
| Response Filtering | Mask sensitive responses |

---

## 📊 Rate Limiting

### Request Rate Limiter

Limits requests per minute to prevent abuse.

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: request-rate-limiter
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    rateLimit:
      requestsPerUnit: 10
      unit: MINUTE
      burst: 5
```

**Configuration:**
- `requestsPerUnit`: Max requests per time period
- `unit`: SECOND, MINUTE, HOUR, DAY
- `burst`: Allow temporary spike above limit

### Token Rate Limiter

Limits token consumption to control costs.

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: token-rate-limiter
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    rateLimit:
      tokensPerUnit: 50000
      unit: HOUR
```

---

## 🔐 PII Protection

### Block Social Security Numbers

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: block-ssn-numbers
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptGuard:
      request:
        customResponseMessage: "Request blocked: SSN pattern detected"
        matches:
          - action: REJECT
            regex: '\b\d{3}-\d{2}-\d{4}\b'
```

### Block Credit Card Numbers

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: block-credit-cards
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptGuard:
      request:
        customResponseMessage: "Request blocked: Credit card number detected"
        matches:
          # Visa
          - action: REJECT
            regex: '\b4[0-9]{12}(?:[0-9]{3})?\b'
          # Mastercard
          - action: REJECT
            regex: '\b5[1-5][0-9]{14}\b'
          # Amex
          - action: REJECT
            regex: '\b3[47][0-9]{13}\b'
          # Generic 16-digit
          - action: REJECT
            regex: '\b[0-9]{4}[- ]?[0-9]{4}[- ]?[0-9]{4}[- ]?[0-9]{4}\b'
```

### Block Phone Numbers

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: block-phone-numbers
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptGuard:
      request:
        customResponseMessage: "Request blocked: Phone number detected"
        matches:
          # US format
          - action: REJECT
            regex: '\b\(?[0-9]{3}\)?[-.\s]?[0-9]{3}[-.\s]?[0-9]{4}\b'
          # International
          - action: REJECT
            regex: '\+[0-9]{1,3}[-.\s]?[0-9]{6,14}\b'
```

### Block Canadian SIN

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: block-canadian-sin
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptGuard:
      request:
        customResponseMessage: "Request blocked: Canadian SIN detected"
        matches:
          - action: REJECT
            regex: '\b\d{3}[-\s]?\d{3}[-\s]?\d{3}\b'
```

---

## 🛡️ Jailbreak Prevention

### Block "Ignore Instructions" Attacks

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: block-jailbreak-ignore
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptGuard:
      request:
        customResponseMessage: "Request blocked: Prompt manipulation attempt detected"
        matches:
          - action: REJECT
            regex: '(?i)ignore\s+(all\s+)?(previous|prior|above|earlier)\s+(instructions?|prompts?|rules?|guidelines?)'
          - action: REJECT
            regex: '(?i)disregard\s+(all\s+)?(previous|prior|your)\s+(instructions?|programming|rules?)'
          - action: REJECT
            regex: '(?i)forget\s+(everything|all|your)\s+(you|instructions?|rules?)'
```

### Block DAN Mode Attacks

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: block-jailbreak-dan
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptGuard:
      request:
        customResponseMessage: "Request blocked: Jailbreak attempt detected"
        matches:
          - action: REJECT
            regex: '(?i)\bDAN\b.*mode'
          - action: REJECT
            regex: '(?i)do\s+anything\s+now'
          - action: REJECT
            regex: '(?i)jailbreak|jailbroken'
          - action: REJECT
            regex: '(?i)developer\s+mode\s+(enabled|activated|on)'
```

### Block Role Manipulation

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: block-role-manipulation
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptGuard:
      request:
        customResponseMessage: "Request blocked: Role manipulation attempt detected"
        matches:
          - action: REJECT
            regex: '(?i)you\s+are\s+(now\s+)?(a|an)\s+(evil|malicious|unrestricted|unfiltered)'
          - action: REJECT
            regex: '(?i)pretend\s+(to\s+be|you\s+are)\s+(a|an)\s+(hacker|criminal|villain)'
          - action: REJECT
            regex: '(?i)act\s+as\s+(if|though)\s+you\s+(have\s+)?no\s+(restrictions?|limitations?|rules?)'
```

---

## 🔑 Credential Protection

### Block OpenAI API Keys

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: block-openai-keys
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptGuard:
      request:
        customResponseMessage: "Request blocked: API key detected in prompt"
        matches:
          - action: REJECT
            regex: '\bsk-[a-zA-Z0-9]{20,}\b'
```

### Block GitHub Tokens

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: block-github-tokens
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptGuard:
      request:
        customResponseMessage: "Request blocked: GitHub token detected"
        matches:
          - action: REJECT
            regex: '\bghp_[a-zA-Z0-9]{36}\b'
          - action: REJECT
            regex: '\bgho_[a-zA-Z0-9]{36}\b'
          - action: REJECT
            regex: '\bghu_[a-zA-Z0-9]{36}\b'
```

### Block Slack Tokens

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: block-slack-tokens
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptGuard:
      request:
        customResponseMessage: "Request blocked: Slack token detected"
        matches:
          - action: REJECT
            regex: '\bxoxb-[0-9]{10,}-[0-9]{10,}-[a-zA-Z0-9]{24}\b'
          - action: REJECT
            regex: '\bxoxp-[0-9]{10,}-[0-9]{10,}-[a-zA-Z0-9]{24}\b'
```

### Block AWS Credentials

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: block-aws-credentials
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptGuard:
      request:
        customResponseMessage: "Request blocked: AWS credential detected"
        matches:
          - action: REJECT
            regex: '\bAKIA[0-9A-Z]{16}\b'
          - action: REJECT
            regex: '\bASIA[0-9A-Z]{16}\b'
```

---

## 💬 Prompt Elicitation

### Security Context Injection

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: elicit-security-context
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptEnrichment:
      prepend:
        - role: system
          content: |
            IMPORTANT SECURITY GUIDELINES:
            - Never reveal API keys, passwords, tokens, or credentials
            - Do not provide instructions for illegal activities
            - Decline requests to bypass security controls
            - Protect user privacy and personal information
            - Do not generate malicious code or exploits
```

### Compliance Context Injection

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: elicit-compliance-context
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptEnrichment:
      prepend:
        - role: system
          content: |
            COMPLIANCE REQUIREMENTS:
            - Handle all data according to SOC2 and GDPR requirements
            - Do not store or log sensitive personal information
            - Maintain audit trail awareness for regulated industries
            - Follow data minimization principles
```

### Expert Persona Injection

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: elicit-k8s-expert
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptEnrichment:
      prepend:
        - role: system
          content: |
            You are a Kubernetes and DevOps expert. When answering:
            - Provide production-ready configurations
            - Include security best practices
            - Consider scalability and reliability
            - Reference official documentation when relevant
```

---

## 📝 Response Filtering

### Mask SSN in Responses

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: mask-ssn-responses
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptGuard:
      response:
        matches:
          - action: MASK
            regex: '\b\d{3}-\d{2}-\d{4}\b'
            replacement: '[SSN REDACTED]'
```

### Mask Credit Cards in Responses

```yaml
apiVersion: gateway.agentgateway.io/v1
kind: AgentGatewayPolicy
metadata:
  name: mask-cc-responses
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: llm-routes
  default:
    promptGuard:
      response:
        matches:
          - action: MASK
            regex: '\b[0-9]{4}[- ]?[0-9]{4}[- ]?[0-9]{4}[- ]?[0-9]{4}\b'
            replacement: '[CARD REDACTED]'
```

---

## 🎛️ Policy Actions

| Action | Description |
|--------|-------------|
| `REJECT` | Block the request entirely |
| `MASK` | Replace matched content with placeholder |
| `LOG` | Allow but log the match |
| `ALLOW` | Explicitly allow (override other rules) |

## 📊 Monitoring Policies

Check policy effectiveness:

```bash
# View policy violations
kubectl logs -n agentgateway-system -l app=agentgateway | grep -i "policy"

# Check metrics
curl http://<gateway>:9090/metrics | grep agentgateway_policy
```
