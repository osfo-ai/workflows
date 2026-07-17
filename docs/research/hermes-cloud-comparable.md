# Hermes Cloud as an osfo comparable

**Date:** 2026-07-17  
**Sources:** First-party Nous Research pages, terms, documentation, and repository only.

## Conclusion

Hermes Cloud validates the model osfo is converging on:

```text
durable agent identity + persistent workspace
                    |
          replaceable runtime lease
                    |
       inference and tools billed separately
```

It does **not** validate one permanently running container per consumer. Nous markets an always-available agent, but implements an idle path that can suspend its runtime while retaining its workspace. osfo should likewise make a durable logical Box the product entitlement, meter active compute independently, and reserve always-on or dedicated runtimes for workloads that need them.

## Verified product facts

- The product name is **Hermes Cloud**. Nous announced it on **July 8, 2026**, including organization agents with granular access controls and unified billing. It remains marked **preview** as of this note. Sources: [launch announcement](https://x.com/NousResearch/status/2074878754485043333), [Hermes Cloud page](https://portal.nousresearch.com/cloud).
- Nous describes each agent as a hosted, dedicated cloud instance with its own workspace and dashboard, and separately says each agent receives a hardened container. This establishes per-agent logical isolation, but does not establish a dedicated physical host. Source: [Nous Portal product information](https://portal.nousresearch.com/info), [Hermes Cloud page](https://portal.nousresearch.com/cloud).
- One agent exposes one memory across Telegram, Discord, Slack, Email, and CLI. Channels are access surfaces, not separate agent instances. Source: [Hermes Cloud page](https://portal.nousresearch.com/cloud).

### Pricing

The official page displays daily rates but says charges are deducted from Nous credit **hourly**. Running includes compute plus storage. Stopped includes storage only. Inference and tool usage are additional.

| Size | Published limit | Running | Hourly equivalent* | Stopped |
|---|---|---:|---:|---:|
| Small | 2 vCPU, 1 GB RAM, 5 concurrent sessions | $0.32/day | $0.0133/hour | $0.06/day |
| Medium | 4 vCPU, 2 GB RAM, 10 concurrent sessions | $0.59/day | $0.0246/hour | $0.06/day |
| Large | 8 vCPU, 4 GB RAM, 20 concurrent sessions | $1.12/day | $0.0467/hour | $0.06/day |

\*Derived by dividing the published daily rate by 24.

The stopped rate is the same for all sizes: **$0.0025/hour**, or **$1.80 per 30-day month**, derived from $0.06/day. This is a flat product price, not a usable $/GB comparison because Nous does not publish the included storage capacity. Source: [Nous Portal product information](https://portal.nousresearch.com/info).

Hermes defines concurrent sessions as simultaneously active chat sessions across its interfaces. At the cap, new sessions receive a limit response while existing sessions continue. The cap is a best-effort, single-host/profile guard, not a distributed concurrency guarantee. Source: [Hermes configuration reference](https://github.com/NousResearch/hermes-agent/blob/main/cli-config.yaml.example).

## Lifecycle and scale to zero

The public product claims that an agent scales to zero when idle. The merged gateway implementation specifies the mechanism:

```text
message edge buffers inbound
        |
        v
wakeUrl wakes suspended instance
        |
gateway reconnects and drains backlog
```

- Default idle timeout: five minutes.
- Dormancy requires no in-flight agent turn, no recent inbound message, and no live background work such as subagents, Kanban work, or background terminal processes.
- The gateway closes its relay connection without terminating its reconnect supervisor. Fly `autostop:"suspend"` freezes the machine and later resumes it; the implementation says this path preserves RAM.
- Scale to zero only arms when messaging is relay-only or absent and a wake URL exists. A directly connected Slack, Telegram, or Discord socket prevents suspension. This strongly supports osfo owning the shared messaging edge outside user runtimes.

Sources: [merged scale-to-zero PR](https://github.com/NousResearch/hermes-agent/pull/52243), [implementation at merge revision](https://github.com/NousResearch/hermes-agent/blob/d1cac0e5ef835eef691f33ebdc470905a78cfb38/gateway/scale_to_zero.py), [relay contract](https://github.com/NousResearch/hermes-agent/blob/d1cac0e5ef835eef691f33ebdc470905a78cfb38/docs/relay-connector-contract.md).

The repository proves the intended implementation, and the product page proves that scale to zero is offered. It does not publicly prove production wake latency, reliability, or that every preview instance uses this exact Fly path.

## Explicit unknowns and cautions

- Nous does not publish storage capacity, storage growth pricing, backup policy, snapshots, retention after exhausted credit, RPO, RTO, wake latency, or a Cloud-specific SLA.
- Public wording does not explicitly say whether automatic idle suspension immediately changes billing from Running to the Stopped storage-only rate. “Idle,” “suspended,” and manually “Stopped” should not be treated as identical without confirmation.
- The product pages conflict on deployment credit: `/info` says at least $2, while `/cloud` says $10 or an active subscription.
- Preview pricing may be promotional and should not determine osfo’s unit economics.
- Nous’s general terms promise commercially reasonable 24/7 availability but disclaim uninterrupted or error-free service and liability for failure to store data. They also prohibit competitors or anyone monitoring service performance from accessing the service without written consent. osfo may study public materials, but should not benchmark Hermes Cloud itself without permission. Source: [Nous Portal Terms of Service](https://portal.nousresearch.com/terms).
- Nous may collect AI workflow and operational data. Privacy Mode prospectively limits storage and use of some inference payloads, but operational metadata remains collectable. An enterprise comparison therefore requires a separate review of data residency, processing terms, and workspace-data coverage. Source: [Nous Privacy Policy](https://portal.nousresearch.com/privacy).

## Implications for osfo

### Consumer

Give a person one durable logical Box by default. The Box owns memory, files, permissions, and history. A phone message wakes temporary isolated compute, completes work, checkpoints state, and releases compute. Branches and task sandboxes remain invisible implementation details.

Consumer packaging should favor a predictable subscription with included storage and active-compute allowance. Internally, meter the real units so packaging can change:

```text
workspace storage byte-time
+ runtime vCPU/memory time
+ inference/tool units
+ optional warm or always-on premium
```

### Business and enterprise

Slack channels and threads should map to conversation scopes, not containers. Businesses can own multiple permission-bounded workspaces or agents under pooled billing. Dedicated, BYOC, warm, and always-on runtimes become premium deployment modes for compliance, private networking, hosted services, and strict latency.

### Architectural requirement

Do not encode pricing tiers into the workflow kernel. Keep these concepts independent:

```text
Box               durable identity and workspace
RuntimeLease      active compute
Execution         one agent/workflow run
ServiceMode       on-demand | warm | always-on | dedicated
MeterEvent        storage | compute | inference | tool
```

Hermes Cloud’s strongest lesson is product ergonomics: the customer experiences one continuous agent even though compute sleeps. osfo should copy that separation, not the specific preview prices or the assumption that every user needs a permanently running box.
