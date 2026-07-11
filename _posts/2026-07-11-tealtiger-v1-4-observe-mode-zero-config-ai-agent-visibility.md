---
title: "TealTiger v1.4: observe() — One Line of Code, Full Agent Visibility"
description: "TealTiger v1.4 introduces observe() — a zero-config, single-function wrapper that gives developers instant cost tracking, audit logging, behavioral baselines, and PII detection for any LLM agent. No policies, no configuration, no proxy. Just one line."
tags: [release, observe, zero-config, cost-tracking, audit, pii, behavioral-baseline, kill-switch, security, governance]
---

Every governance platform eventually faces the same adoption challenge: developers won't add security tooling if it requires configuration upfront. They want to see value *before* they invest effort.

TealTiger v1.4 solves this with `observe()` — a single function that wraps any LLM client and immediately delivers full-stack visibility. No config files. No policy definitions. No external services. One line of code, and you're instrumented.

```typescript
import { observe } from 'tealtiger';
import OpenAI from 'openai';

const client = observe(new OpenAI());
// That's it. Cost tracking, audit logging, PII detection — all active.
```

---

## The Problem: Day-Zero Visibility

Most teams discover governance needs *after* an incident. The pattern is predictable:

1. Ship an agent with no instrumentation
2. Something goes wrong — cost spike, data leak, runaway loop
3. Scramble to add observability after the fact
4. Realize you have no baseline of "normal" to compare against

`observe()` eliminates steps 1-4 by making visibility the default state. You add it on day one, before you know what your policies should be. TealTiger learns your agent's behavior, and when you're ready for enforcement, you already have the data to write informed policies.

---

## What observe() Delivers

### Automatic Cost Tracking

Every request through the observed client accumulates cost — per request, per session, per agent. No budget policies needed. Just visibility.

```typescript
const client = observe(new OpenAI(), { agentId: 'research-bot' });

// After some requests...
const costs = client.getCost();
// { session: 0.47, agent: 12.83, lastRequest: 0.003 }
```

Supports all 12 providers with accurate pricing data: OpenAI, Anthropic, Gemini, Bedrock, Azure OpenAI, Cohere, Mistral, DeepSeek, Groq, xAI, Together, and HF-TGI.

### Complete Audit Trail

Every request, response, tool call, and error is logged automatically using TealTiger's existing `TealAudit` system. PII in logs is hashed by default — security-by-default, not security-by-configuration.

Each audit event includes:
- Agent ID, session ID, request ID (UUID v4)
- Provider, model, timestamps
- Token counts, cost, latency
- Tool call metadata (name, argument count, argument hash)
- PII detection results (types found, never the values)

### Behavioral Baseline Construction

The first 100 requests automatically build a statistical baseline of your agent's normal behavior. After the baseline window completes, you get p50, p95, and p99 metrics for:

- Request latency
- Input/output token distribution
- Cost per request
- Tool call frequency

```typescript
const baseline = client.getBaseline();
// {
//   complete: true,
//   requestCount: 100,
//   latency: { p50: 340, p95: 890, p99: 1240 },
//   cost: { p50: 0.002, p95: 0.008, p99: 0.015 },
//   ...
// }
```

This baseline becomes your reference point when you're ready to set alert thresholds or define anomaly detection policies.

### PII Detection (Report-Only)

Every request and response is scanned for PII — email addresses, phone numbers, SSNs, credit card numbers. Detections are logged to the audit trail but *never block or modify traffic*. You see where sensitive data flows without any production risk.

---

## The Kill Switch: freeze() and unfreeze()

Even without policies, operators need an emergency stop. `freeze()` immediately blocks all requests for a given agent — or all agents with the wildcard `'*'`.

```typescript
import { freeze, unfreeze } from 'tealtiger';

// Security incident detected — stop everything
freeze('*');

// Investigate, then restore specific agents
unfreeze('research-bot');
```

No policy files needed. No configuration. Just an in-memory kill switch that works in any environment, including air-gapped deployments.

Key properties:
- **Idempotent**: Calling `freeze()` twice is the same as calling it once
- **Round-trip safe**: `freeze()` then `unfreeze()` restores normal operation
- **In-process**: No external service, no database, no network call
- **Immediate**: Takes effect on the very next request

---

## Architecture: Zero Dependencies, Zero Latency

`observe()` is designed for production from day one:

- **In-process only** — no proxy, no sidecar, no SaaS dependency
- **Air-gap compatible** — works with no internet connectivity
- **< 5ms overhead** — P99 latency added is under 5 milliseconds
- **Deterministic** — no LLM in the governance path
- **Thread-safe** — supports concurrent async calls
- **Fail-open on audit** — if the log target is unavailable, requests continue

This is the same architectural philosophy that underpins all of TealTiger: governance should never be the thing that breaks your production system.

---

## New in v1.4: Multi-Stage Defense Pipeline

Beyond observe mode, v1.4 introduces configurable defense depth for TealGuard:

| Depth | Stages | Target Latency |
|-------|--------|---------------|
| `fast` | Pattern scan (PII, secrets, injection) | < 5ms |
| `standard` | + Structural analysis (AST intent, sentence anomaly) | < 20ms |
| `deep` | + Local binary classifier (harmful content) | < 50ms |

All stages are deterministic. No generative LLM is used anywhere in the pipeline. If Stage 1 produces a DENY, subsequent stages are short-circuited.

```typescript
const guard = new TealGuard({ depth: 'standard' });
```

---

## New in v1.4: Post-Execution Response Governance

Previous versions only scanned inputs. v1.4 adds bidirectional governance — scan LLM *outputs* for secrets, PII, and harmful content before they reach your application code.

```typescript
const policy = {
  guardrails: {
    pre: { pii: 'block', injection: 'block' },
    post: { secrets: 'block', pii: 'monitor' }
  }
};
```

The `post` scan executes after the provider response is received but before it's returned to the caller. Blocked content never reaches application code.

---

## New in v1.4: Role-Based Per-Agent Governance

Multi-agent systems need differentiated policies. v1.4 introduces role-based governance — each agent operates with minimum privilege for its function.

```typescript
const client = observe(new OpenAI(), { role: 'researcher' });

// Researcher role: read-only tools, no PII access, $5/session budget
// Writer role: content tools, PII allowed, $2/session budget
// Reviewer role: all tools, full access, $10/session budget
```

---

## Progressive Disclosure: The Four Levels

TealTiger v1.4 completes the progressive adoption path:

| Level | What You Do | What You Get |
|-------|-------------|-------------|
| **0** | `observe(client)` | Cost, audit, baseline, PII visibility |
| **1** | Add policies | Enforcement, budgets, tool restrictions |
| **2** | Add TealProof | Cryptographic governance receipts |
| **3** | Add TealFlow | Declarative workflows, drift detection |

Level 0 requires zero configuration. Each subsequent level adds capability without breaking the previous level. You adopt governance at your own pace.

---

## Python Parity

Everything works identically in Python:

```python
from tealtiger import observe, freeze, unfreeze
from openai import OpenAI

client = observe(OpenAI())

# Same cost tracking, audit logging, PII detection
cost = client.get_cost()
baseline = client.get_baseline()

# Same kill switch
freeze("*")
unfreeze("research-bot")
```

Both SDKs produce structurally equivalent audit events with equivalent metadata fields.

---

## 12 Providers Supported

observe() works out of the box with:

OpenAI · Anthropic · Gemini · Bedrock · Azure OpenAI · Cohere · Mistral · DeepSeek · Groq · xAI · Together · HF-TGI

That's 95%+ of production LLM traffic covered by a single `observe()` call.

---

## Get Started

```bash
# TypeScript
npm install tealtiger

# Python
pip install tealtiger
```

```typescript
import { observe } from 'tealtiger';
const client = observe(new OpenAI());
// You now have full visibility. Zero config. One line.
```

---

## What's Next

v1.4 is available now on npm and PyPI. The observe() quickstart examples ship with the package at `examples/observe-quickstart.ts` and `examples/observe_quickstart.py`.

For teams already on TealTiger v1.3, the upgrade is non-breaking. `observe()` is additive — your existing policies, TealProof receipts, and TealFlow workflows continue to work unchanged.

The gap between "no governance" and "full governance" used to be a configuration cliff. With `observe()`, it's a gentle slope. Start with visibility. Add enforcement when you're ready.

---

*TealTiger is open-source AI agent security. Deterministic governance, no LLM in the governance path, 95%+ provider coverage. [GitHub](https://github.com/agentguard-ai/tealtiger) · [Docs](https://docs.tealtiger.ai)*
