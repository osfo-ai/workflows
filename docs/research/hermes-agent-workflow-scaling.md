# Hermes Agent, TryAgent, and workflow scaling

**Date:** 2026-07-17  
**Hermes revision:** [`7cb2d2c`](https://github.com/NousResearch/hermes-agent/tree/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9)  
**TryAgent revision:** [`64fdadc`](https://github.com/heyimcarlos/tryagent/tree/64fdadcbfc207a77390ed69c96824a9eb04cb351)

## Conclusion

Hermes has its own agent orchestration and several workflow-like subsystems. It does **not** have one general, horizontally scalable durable-workflow engine comparable to Temporal or Cloudflare Workflows.

That distinction matters for OSFO:

- Millions of registered users do not, by themselves, require every chat turn to be a durable workflow.
- A TryAgent-style service needs a scalable messaging edge, identity router, box placement and lifecycle control plane, and many isolated Hermes boxes.
- OSFO Workflows is valuable for durable operations that cross process restarts, waits, retries, machines, or external side effects.
- The LLM and tool loop should normally stay inside Hermes. Putting every model iteration or tool call into OSFO Workflows would add coupling and state volume without solving the main scaling problem.

## What Hermes actually provides

| Mechanism | Hermes implementation | Boundary |
|---|---|---|
| Agent turn | `AIAgent` is the synchronous orchestration engine. `run_conversation` drives model calls, tool dispatch, retries, compression, callbacks, and persistence. | One process and session, not replayable distributed execution. |
| Sessions | Full histories live in `~/.hermes/state.db`; messaging keys deterministically isolate users, including one WhatsApp DM session per canonical user identity. SQLite uses WAL with concurrent readers and one writer. | Appropriate per box or profile. It is not a shared multi-tenant database. |
| Goals | `/goal` persists a goal and wait barrier in session metadata, judges each turn, and automatically continues within a turn budget. | A persistent Ralph loop, not a general workflow definition and execution service. |
| Cron | Built-in cron stores jobs in JSON and starts a fresh `AIAgent` for each due job. Hosted Chronos moves timers outside the gateway so a box may scale to zero. | Useful scheduling, but not a complete workflow runtime. |
| Delegation | Child `AIAgent` loops run in isolated contexts, with three concurrent children by default. | Hermes explicitly says completion delivery can be durable, but execution is not: a restart does not resume an in-flight child. |
| Kanban | SQLite-backed task state machine and work queue with dependencies, atomic claims, heartbeats, stale-worker reclamation, run history, and worker profiles. | Closest Hermes mechanism to durable workflows, but documented as single-host by design. |
| Relay connector | Experimental shared-bot edge with authenticated account linking, author-first routing, Redis cross-instance routing, durable per-instance buffering, acknowledgements, and wake signals. | A promising multi-instance messaging topology, not evidence of production scale or a general workflow engine. |
| Tool sandbox | Docker execution can drop capabilities, disable privilege escalation, apply resource limits, and disable networking. | Hermes uses one persistent tool container across the process, not one sandbox per user inside a shared gateway. In TryAgent, the Agent Box should remain the tenant boundary. |

Sources: Hermes [architecture](https://github.com/NousResearch/hermes-agent/blob/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9/website/docs/developer-guide/architecture.md), [conversation loop](https://github.com/NousResearch/hermes-agent/blob/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9/agent/conversation_loop.py), [sessions](https://github.com/NousResearch/hermes-agent/blob/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9/website/docs/user-guide/sessions.md), [goals](https://github.com/NousResearch/hermes-agent/blob/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9/website/docs/user-guide/features/goals.md), [cron](https://github.com/NousResearch/hermes-agent/blob/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9/website/docs/user-guide/features/cron.md), [Chronos contract](https://github.com/NousResearch/hermes-agent/blob/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9/docs/chronos-managed-cron-contract.md), [delegation](https://github.com/NousResearch/hermes-agent/blob/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9/website/docs/user-guide/features/delegation.md), [Kanban](https://github.com/NousResearch/hermes-agent/blob/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9/website/docs/user-guide/features/kanban.md), [relay contract](https://github.com/NousResearch/hermes-agent/blob/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9/docs/relay-connector-contract.md), and [Docker tool backend](https://github.com/NousResearch/hermes-agent/blob/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9/website/docs/user-guide/features/tools.md).

This is an architectural inference from the documented components and source. The repository contains no general workflow-definition/versioning API, distributed history service, activity queue, or deterministic replay model that would make Hermes itself a Temporal or Cloudflare Workflows equivalent.

## Shared-number TryAgent topology

Hermes's experimental relay contract describes almost exactly the reusable topology a shared entry point needs:

```text
shared WhatsApp number or iMessage identity
                  |
        provider-specific connector
  authentication, normalization, dedupe,
  account linking, buffering, rate limits
                  |
          user_id -> box_id router
                  |
        wake or ensure assigned box
                  |
       Hermes Gateway inside that box
                  |
       session -> AIAgent -> tools
                  |
       reply through the connector
```

The connector, not Hermes's conversation loop, must own the shared platform identity and resolve each authenticated sender to exactly one agent instance. Hermes's relay design uses `/link <code>`, author-first resolution, fail-closed handling for unlinked users, and a durable per-instance buffer before waking a sleeping gateway. It also routes across connector instances through Redis. See the official [relay connector contract](https://github.com/NousResearch/hermes-agent/blob/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9/docs/relay-connector-contract.md).

Two cautions:

1. The contract is explicitly experimental and says Discord and Telegram are its initial validation platforms. It is useful design evidence, not proof that one WhatsApp or iMessage identity has been validated at million-user scale.
2. Current TryAgent product truth says channel identities are user-owned and TryAgent does not provide a shared first-party identity. A single shared number therefore changes the accepted product model and needs a superseding product decision and ADR, not merely an implementation ticket. See TryAgent [CONTEXT.md](https://github.com/heyimcarlos/tryagent/blob/64fdadcbfc207a77390ed69c96824a9eb04cb351/CONTEXT.md) and [ADR 0007](https://github.com/heyimcarlos/tryagent/blob/64fdadcbfc207a77390ed69c96824a9eb04cb351/docs/adr/0007-messaging-platform-control-through-hermes-gateway.md).

Hermes supports WhatsApp and BlueBubbles for iMessage as direct gateway platforms, but its own deployment documentation warns against running two gateway containers against one data directory. The scalable unit is therefore one box or profile state partition, not one global Hermes instance and SQLite file. See [sessions](https://github.com/NousResearch/hermes-agent/blob/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9/website/docs/user-guide/sessions.md) and [Docker deployment](https://github.com/NousResearch/hermes-agent/blob/7cb2d2cd4acf0c452349968ce8d90908e8fb4cd9/website/docs/user-guide/docker.md).

## Where OSFO Workflows belongs

```text
Messaging edge       Box control plane         Agent box
---------------      -----------------         ----------------
accept + dedupe  --> locate/provision/wake --> Hermes session
buffer + ack         suspend/migrate/retry      AIAgent loop
route + rate limit   reconcile uncertainty      tools + local state
       |                      |
       +------ durable OSFO workflows ----------+
```

Use OSFO Workflows for:

- Agent Box allocation, provisioning, wake, suspend, migration, and reconciliation.
- Long-running user automations that wait on timers, approvals, external callbacks, or unavailable boxes.
- Multi-step external effects where retry policy, idempotency, compensation, and an inspectable execution history matter.
- Scheduled workflow instances and cross-box control-plane operations.

Keep outside the workflow engine:

- Token streaming and individual model iterations.
- Ordinary in-box tool dispatch and short retries.
- Session transcript storage.
- High-volume message transport itself. Use a durable queue or inbox/outbox for transport, then start a workflow only when delivery requires a durable state transition such as waking or repairing a box.

TryAgent already separates durable placement from host-local execution: the allocation is the placement source of truth, while a scheduler/binder selects a host and the host agent performs and reconciles provisioning. That is a natural OSFO Workflows consumer without putting TryAgent domain types into the reusable Rust kernel. See [ADR 0014](https://github.com/heyimcarlos/tryagent/blob/64fdadcbfc207a77390ed69c96824a9eb04cb351/docs/adr/0014-agent-box-allocation-scheduler-binder.md) and [ADR 0011](https://github.com/heyimcarlos/tryagent/blob/64fdadcbfc207a77390ed69c96824a9eb04cb351/docs/adr/0011-go-runtime-host-agent-and-provisioning-worker.md).

## Scale verdict

There is no primary-source evidence in the Hermes repository of a million-user benchmark, production SLO, or a single gateway intended to own millions of active sessions. Its local cache, thread-pool controls, SQLite state, and single-host Kanban design point to a per-box runtime. Hermes does provide the right shape for connecting many such runtimes behind a shared connector.

For “millions of users,” structure is the first-order problem:

- partition tenants into cells;
- shard connector and routing state;
- isolate state and compute per box;
- enforce admission control and per-tenant quotas;
- make placement and lifecycle operations recoverable;
- define failure domains and backpressure.

Latency then becomes an SLO on that structure. The decisive inputs are not registered users, but peak inbound messages per second, concurrently active agents, tool concurrency, cold-wake rate and latency, model latency, response-stream duration, and acceptable message-loss or duplication bounds.

**Decision:** build OSFO Workflows as a reusable durable orchestration package, but do not make it a prerequisite for every Hermes chat turn. Prove it first on box lifecycle plus one genuinely long-running user automation. Treat the shared-number connector as a separate, provider-aware edge service that routes into isolated boxes.
