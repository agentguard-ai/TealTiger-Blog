---
layout: post
title: "The Two-Layer Authorization Problem in AI Agents"
date: 2026-06-03
author: Naga Satish
tags: [governance, security, mcp, authorization, production]
description: "Governance hooks decide if an agent can call a tool. Credential brokers decide if the agent can use the secret the tool needs. Neither layer alone is sufficient for production."
---

# The Two-Layer Authorization Problem in AI Agents

Every production AI agent deployment faces a question that most frameworks don't answer: **who decides what the agent is allowed to do?**

The obvious answer — "the tool allowlist" — is wrong. Or rather, it's half the answer. And half-answers in security are worse than no answer at all.

## The Problem

Consider an agent with access to a Stripe tool. The governance layer says: "this agent is allowed to call `stripe.refund`." So the call goes through. But the agent was running with production Stripe keys, and it just issued a $50,000 refund to a test account.

The tool authorization was correct. The credential scope was not.

This is the two-layer problem. And it shows up everywhere once you start looking:

- An agent authorized to call `aws.terminate_instance` — but holding read-only IAM credentials
- An agent authorized to call `postgres.execute_sql` — but connected with a production connection string that has write access
- An agent authorized to call `github.merge` — but holding a PAT with `repo:delete` scope

In each case, the **governance decision** (should this agent call this tool?) and the **credential decision** (should this agent hold these secrets?) are answered by different systems. Neither system sees the other's context.

## Why One Layer Fails

**Governance-only** (what most SDKs provide):

```python
# "Is this agent allowed to call stripe.refund?"
# Answer: Yes, per policy.
# But: with what credential scope? Unknown.
```

The governance hook evaluates agent identity, tool name, arguments, budget — but it has no visibility into what credentials the agent is holding. A tool that's safe for a sandbox-scoped agent is dangerous for one holding production keys.

**Credential-only** (what vaults provide):

```python
# "Is this agent allowed to use the Stripe production key?"
# Answer: Yes, it's on the allowlist.
# But: for what operation? Unknown.
```

The credential broker knows which secrets the agent can access, but has no visibility into what the agent is actually doing with them. Issuing a key doesn't constrain how it's used.

## The Architecture That Works

Production deployments need both layers, and they need the layers to see each other's context:

```
┌─────────────────────────────────────────────────────┐
│                    Agent Request                      │
│         "Call stripe.refund($50,000)"                │
└──────────────────────┬──────────────────────────────┘
                       │
          ┌────────────▼────────────────┐
          │   Layer 1: Governance Hook   │
          │   "Is this agent allowed     │
          │    to call this tool,        │
          │    with this credential      │
          │    scope, at this budget?"   │
          └────────────┬────────────────┘
                       │ ALLOW / DENY / REQUIRE_APPROVAL
                       │
          ┌────────────▼────────────────┐
          │   Layer 2: Credential Broker │
          │   "Is this agent allowed     │
          │    to use this secret for    │
          │    this specific operation?" │
          └────────────┬────────────────┘
                       │ ISSUE / DENY
                       │
          ┌────────────▼────────────────┐
          │      Tool Execution          │
          └──────────────────────────────┘
```

The key insight: **Layer 1 consumes metadata from Layer 2.** The governance hook doesn't need to see the actual credentials — it needs to know the *scope* of what the agent is holding. A field like `credential_context.tier: production` or `credential_context.scope: read-only` is enough to make an informed governance decision.

## What This Looks Like in Policy

With scope-aware governance, policies can express conditions that were previously impossible:

```yaml
rules:
  # Safe in sandbox, dangerous in production
  - tool: stripe.refund
    when:
      credential_context.tier: { eq: 'production' }
      args.amount: { gt: 1000 }
    action: REQUIRE_APPROVAL
    reason: "Production refunds over $1000 require human sign-off"

  # Read-only credentials can't run write queries
  - tool: postgres.execute_sql
    when:
      credential_context.scope: { eq: 'read-only' }
      args.query: { pattern: "INSERT|UPDATE|DELETE|DROP" }
    action: DENY
    reason: "Write operations blocked for read-only credential scope"

  # Sandbox credentials have lower limits
  - tool: aws.terminate_instance
    when:
      credential_context.tier: { eq: 'sandbox' }
      args.instance_id: { pattern: "^prod-" }
    action: DENY
    reason: "Sandbox credentials cannot terminate production instances"
```

None of these policies are possible if the governance layer can't see credential scope. And none of them require the governance layer to hold or manage the actual credentials.

## The Interface Between Layers

The integration point is lightweight. The governance engine doesn't become a credential manager — it receives a metadata envelope from whatever credential system is in use:

```typescript
// TealTiger receives scope metadata from the credential broker
const engine = new TealEngine({
  credentialContext: {
    provider: async (agentId, toolName) => ({
      scope: 'read-only',        // what the credentials allow
      tier: 'production',        // environment level
      tools_authorized: ['search_docs', 'lookup_customer'],
      expires_at: '2026-06-03T18:00:00Z'
    })
  }
});
```

This works with any credential system — HashiCorp Vault, AWS Secrets Manager, 1Claw, Azure Key Vault, or a static JSON config for simple deployments. The governance layer is a consumer of scope metadata, not a credential manager.

## The Audit Trail

When both layers produce evidence, the TEEC (Typed Evidence & Evidence Contract) receipt captures the full authorization picture:

```json
{
  "decision": "ALLOW",
  "agent_id": "support-bot",
  "tool": "stripe.refund",
  "credential_context": {
    "tier": "sandbox",
    "scope": "payments-limited",
    "tools_authorized": ["stripe.refund", "stripe.lookup"]
  },
  "policy_rule": "stripe-sandbox-limit",
  "reason": "Refund under $100 with sandbox credentials — auto-approved",
  "issued_at": "2026-06-03T14:30:00Z"
}
```

An auditor reviewing this receipt can answer: "What did the agent try to do? Was it authorized? With what credential scope? Under what policy?" All in one artifact, without trusting the system that produced it.

## What's Missing in Most Frameworks

Most AI agent frameworks today have exactly one of these layers:

| Framework | Layer 1 (Governance) | Layer 2 (Credentials) |
|-----------|---------------------|----------------------|
| LangChain | Tool allowlists only | None |
| CrewAI | Role-based tool assignment | None |
| AutoGen | None built-in | None |
| OpenAI Assistants | Function definitions | API key per assistant |
| TealTiger (v1.4) | Full policy engine + credential context | Consumer interface |
| 1Claw Vault | None | HSM-backed credential broker |
| HashiCorp Vault | None | Secret management |

The production-grade answer is **governance + credential broker, connected**. Not one or the other.

## Building This Today

If you're deploying AI agents in production and want both layers:

**Step 1**: Instrument with governance (zero-config start):
```python
from tealtiger import observe
client = observe(openai)  # immediate visibility, nothing blocked
```

**Step 2**: Add credential scope awareness:
```python
from tealtiger import TealEngine

engine = TealEngine({
  credentialContext: {
    provider: vault.get_scope  # your credential broker
  },
  policies: company_policies
})
```

**Step 3**: Write policies that reference both layers:
```yaml
- tool: stripe.refund
  when:
    credential_context.tier: { eq: 'production' }
    args.amount: { gt: 1000 }
  action: REQUIRE_APPROVAL
```

The governance hook handles authorization. The vault handles credentials. The policy sees both. The receipt proves what happened.

Neither layer alone is sufficient. Together, they close the gap.

---

*TealTiger v1.4 ships credential context awareness as part of the MCP Governance module. Open source, Apache 2.0.*

*GitHub: [github.com/agentguard-ai/tealtiger](https://github.com/agentguard-ai/tealtiger)*
