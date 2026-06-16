---
layout: post
title: "TealTiger × Dakera: Governance State That Survives Restarts"
description: "How Dakera's persistent memory server gives TealTiger governance continuity across process restarts, agent migrations, and horizontal scaling."
date: 2026-06-16
permalink: /governance/integrations/tealtiger-dakera-governance-state/
category: governance
hub: integrations
author: Naga Satish Chilakamarti
author_github: https://github.com/nagasatish007
author_role: Maintainer

tags:
  - tealtiger
  - dakera
  - persistence
  - governance-state
  - cost-tracking
  - delegation
  - multi-agent
---

# TealTiger × Dakera: Governance State That Survives Restarts

TealTiger's governance middleware evaluates policies in under 5ms —
deterministic, no LLM in the path. But until now, governance state (cost
records, decision receipts, delegation chains) lived in-memory. Process
restarts, agent migrations, and horizontal scaling all lost that state.

For single-process prototyping, in-memory is fine. For production multi-agent
systems running across multiple processes, you need governance continuity.

## The Solution: Dakera as Governance State Backend

[Dakera](https://dakera.ai) is a self-hosted AI agent memory server with
agent-scoped state, decay-weighted retention, and a knowledge graph. It's
the first persistent backend for TealTiger's governance layer.

```bash
pip install dakera[tealtiger]
```

Three adapter classes shipped in `dakera v0.12.1`:

- **DakeraCostStorage** — implements all 8 methods of TealTiger's `CostStorage`
  ABC. Per-agent cost records persist across restarts.
- **DakeraDecisionStore** — governance decisions stored with importance-tiered
  retention (DENY=0.95, ALLOW=0.80). Critical denials outlive routine approvals.
- **DakeraDelegationHelper** — delegation chains stored as knowledge graph
  edges (`delegated_from`), enabling full audit-trail traversal.

![TealTiger × Dakera Architecture](/assets/images/blog/dakera-tealtiger-architecture.svg)

## Architecture Principle

A key design constraint (credit
[@rpelevin](https://github.com/rpelevin) from the AG2 governance discussion):

> **Storage = evidence/continuity, NOT authority.**

The storage layer answers:
- "Has this decision already reached a terminal state?" (idempotency)
- "What delegation chain was in force?" (evidence for new decision)

It does NOT answer: "Is this new action authorized?" — that always requires
a fresh policy evaluation over the current action envelope.

This means a stored ALLOW from yesterday cannot authorize today's action.
Every new request gets a fresh deterministic evaluation. Storage informs;
it doesn't permit.

## Usage

```python
from dakera.async_client import AsyncDakeraClient
from dakera.integrations.tealtiger import DakeraCostStorage

from tealtiger import TealOpenAI, TealOpenAIConfig

client = AsyncDakeraClient("http://localhost:3000", api_key="dk-mykey")
cost_storage = DakeraCostStorage(client)

# Every LLM call's cost is now persisted in Dakera
teal_client = TealOpenAI(config=TealOpenAIConfig(cost_storage=cost_storage))
```

## What This Enables

| Scenario | Before (in-memory) | After (Dakera) |
|---|---|---|
| Process restart | Cost budgets reset to zero | Budget state preserved |
| Agent migration | Decision history lost | Full receipt chain available |
| Horizontal scaling | Each instance has its own state | Shared governance state |
| Compliance audit | Manual log reconstruction | Query by agent, time, action type |
| Retry after timeout | Unknown if already executed | Idempotency check returns prior terminal state |

## The Three Adapters

### DakeraCostStorage

Implements TealTiger's `CostStorage` abstract base class — all 8 methods.
Cost records are scoped per agent and persist across process boundaries.

```python
from dakera.integrations.tealtiger import DakeraCostStorage

cost_storage = DakeraCostStorage(client)

# Budget enforcement continues across restarts
# If an agent spent $1.50 before a restart, it still knows that after restart
```

When an agent has a `$2.00/session` budget and the process crashes at `$1.80`
spent, the budget doesn't magically reset. The next process picks up where
the last one left off.

### DakeraDecisionStore

Governance decisions are stored with importance-weighted decay:

| Decision Type | Retention Weight | Rationale |
|---|---|---|
| DENY | 0.95 | Security-critical — must persist for compliance |
| ALLOW | 0.80 | Standard operations — retained but lower priority |
| MONITOR | 0.70 | Observational — useful for pattern detection |

This means critical security denials (secret detected, PII blocked, budget
exceeded) persist longer than routine approvals. Dakera's decay engine
naturally ages out low-importance decisions while keeping compliance-critical
evidence accessible.

### DakeraDelegationHelper

Delegation chains are modeled as knowledge graph edges with
`delegated_from` relationships. When Agent A delegates authority to Agent B,
and Agent B delegates to Agent C, the full chain is traversable:

```
Agent A → delegated_from → Agent B → delegated_from → Agent C
```

This enables:
- Full audit-trail traversal for compliance review
- Answering "who authorized this action?" with a complete chain
- Revoking delegations by removing graph edges

## Production Deployment Pattern

```python
import asyncio
from dakera.async_client import AsyncDakeraClient
from dakera.integrations.tealtiger import (
    DakeraCostStorage,
    DakeraDecisionStore,
    DakeraDelegationHelper,
)
from tealtiger import TealOpenAI, TealOpenAIConfig

async def create_governed_client():
    """Create a TealTiger client with full Dakera persistence."""
    dakera = AsyncDakeraClient(
        "http://dakera.internal:3000",
        api_key="dk-production-key",
    )

    return TealOpenAI(
        config=TealOpenAIConfig(
            cost_storage=DakeraCostStorage(dakera),
            decision_store=DakeraDecisionStore(dakera),
            delegation_helper=DakeraDelegationHelper(dakera),
        )
    )

# Governance state now survives:
# - Process restarts (container recycling, deployments)
# - Horizontal scaling (multiple replicas share state)
# - Agent migrations (move agent to different node)
```

## Links

- [Dakera Integration Page](https://dakera.ai/integrations/tealtiger)
- [Dakera Blog Post](https://dakera.ai/blog/dakera-tealtiger-integration)
- [TealTiger Integration Docs](https://github.com/agentguard-ai/tealtiger/blob/main/docs/integrations/dakera.md)
- [Design Discussion: Dakera-AI/dakera-deploy#169](https://github.com/Dakera-AI/dakera-deploy/discussions/169)
- PyPI: `pip install dakera[tealtiger]`

---

Governance state shouldn't be a single point of failure. With Dakera as the
backend, TealTiger's policy evaluation stays deterministic and fast, while
the evidence it produces survives anything the infrastructure throws at it.
