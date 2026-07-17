# Durable workflow kernel and agent runtime comparables

Status: Decision research  
Research date: 2026-07-17  
Decision owner: osfo  
Scope: Product architecture, execution semantics, package model, and build-versus-adopt direction

## Ranked comparables

Scores are 0 to 5. The seven criteria are domain fit (D), target language and stack fit (L), production maturity (M), architecture clarity (A), infrastructure and operations relevance (O), testing quality (T), and documentation and maintainability (Doc).

| Rank | Comparable | D | L | M | A | O | T | Doc | Total |
|---:|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 | Temporal | 5 | 4 | 5 | 5 | 5 | 5 | 5 | 34/35 |
| 2 | Restate | 5 | 5 | 4 | 5 | 5 | 5 | 5 | 34/35 |
| 3 | Inngest | 5 | 3 | 4 | 4 | 5 | 5 | 5 | 31/35 |
| 4 | Cloudflare Workflows, Agents, and Dynamic Workflows | 5 | 2 | 5 | 4 | 5 | 3 | 5 | 29/35 |
| 5 | DBOS Transact for Go | 5 | 2 | 3 | 4 | 4 | 5 | 5 | 28/35 |
| 6 | Obelisk | 5 | 5 | 2 | 4 | 3 | 4 | 4 | 27/35 |
| 7 | LangGraph and Agent Server | 3 | 1 | 4 | 4 | 3 | 5 | 5 | 25/35 |

The score is not an adoption recommendation. Licensing, closed-source components, hosting model, and product fit are treated separately. Temporal wins on semantic and operational maturity. Restate is the strongest Rust implementation reference. Cloudflare most closely validates osfo's proposed product boundary. LangGraph is the best agent-runtime comparison, but the weakest model for a general workflow kernel.

### Best comparable by question

| Question | Best reference | Why |
|---|---|---|
| Is the overall osfo product shape valid? | Cloudflare | General Workflows, durable Agents, and tenant-loaded Dynamic Workflows are separate but composable products. |
| What is the enterprise durability and operations bar? | Temporal | It exposes the full cost of histories, task queues, sharding, replay, compatibility, failover, and operator tooling. |
| What should a serious Rust service look like? | Restate | The server is Rust, componentized, heavily tested, and explicit about storage, invocation, queues, timers, and release safety. |
| How can dynamic agent decisions remain durable? | Inngest | LLM and tool results become recorded step results, so later control flow can be replayed without making the engine agent-specific. |
| What is the smallest Postgres-backed shape worth prototyping? | DBOS | It demonstrates replay plus memoized steps without requiring a Temporal-sized server. |
| How might third-party packages be isolated and pinned? | Obelisk | Content-addressed WebAssembly components and typed WIT contracts make artifact identity explicit. |
| What belongs in the osfo Agent Runtime? | LangGraph | Thread checkpoints, long-term memory, graph execution, interrupts, and agent-specific control flow belong above the general kernel. |

### Adoption and license caveats

This is an engineering summary, not legal advice.

| Comparable | Relevant posture | Adoption implication |
|---|---|---|
| Temporal | Server and SDK are MIT | Strongest direct-adoption candidate, subject to operational and Rust SDK fit. |
| Restate | Server uses BSL 1.1 | Its public-platform restriction needs product-specific legal review. |
| Inngest | Server uses SSPL, with delayed Apache 2.0 conversion by release | Not a straightforward base for an immediate proprietary hosted service. |
| Cloudflare | Public SDKs and adapters are MIT; hosted engine is proprietary | Excellent product reference, not a self-hostable kernel implementation. |
| DBOS Go | MIT | Legally simpler reference, but its Go and library-coupled model do not match the target package architecture. |
| Obelisk | AGPL 3.0 | Architecture reference unless osfo deliberately accepts the license obligations. |
| LangGraph | Core runtime is MIT; production Agent Server and LangSmith have separate commercial requirements | The open graph library should not be confused with the complete hosted control plane. |

## Executive verdict

The current osfo direction is valid:

1. **osfo Workflows should be a general durable execution product.** It should run deterministic business processes, agent-initiated work, and long-running event-driven tasks without knowing what an agent, prompt, model, phone number, Box, or memory provider is.
2. **The osfo Agent Runtime should be a separate consumer of Workflows.** It owns durable agent identity, conversations, memory access, model and tool loops, streaming, and turn-level recovery.
3. **Oz is the first business-domain application, not the platform kernel.** It can onboard, sell, charge, provision, and operate user Boxes by starting and signaling generic workflows.
4. **TryAgent's Go infrastructure remains the authority for host and Box mutation.** A platform workflow calls it through a small typed executor or `RuntimeControl` boundary. There is no reason to rewrite proven cloud provisioning code in Rust.
5. **Application Packages are the reusable deployment unit.** A package can declare agents, workflows, schemas, capabilities, and immutable artifact versions. A tenant deployment resolves to an exact package digest.

What is not yet validated is whether osfo should implement every durable execution primitive itself. The product boundary should be committed now. Engine ownership should be decided after a crash-safe tracer implementation is compared with at least one mature backend.

The principal scaling risk is structure before latency. A system with unclear ownership, unfenced retries, ambiguous side effects, or unpinned code cannot be made correct by lowering response time. Once the structure is sound, the real workload and SLOs determine the latency architecture.

## Decision frame

### The proposed product

osfo aims to let its own team and other companies build reusable agentic applications. Each application may contain:

- one or more durable agents;
- deterministic or agent-driven workflows;
- typed activities against external systems;
- tenant configuration and capability bindings;
- durable memory or Box access;
- versioned code and schemas;
- operator-visible execution history.

Oz is the first application. The same platform should later support another company's agent without embedding Oz's sales, phone, memory, or Box assumptions into the workflow engine.

### Questions this research answers

1. Should the workflow layer be general or agent-specific?
2. Where is the boundary between an Agent Runtime and a workflow kernel?
3. Can dynamic agent plans coexist with deterministic durability?
4. What should an Application Package contain?
5. What parts of TryAgent should be reused?
6. What does "millions of users" require from the architecture?
7. Which implementation patterns should osfo copy, and which systems could plausibly be adopted?

### Questions that remain empirical

- active workflow starts and runnable steps per second;
- p95 and p99 queue delay, signal latency, and wake latency;
- required recovery point and recovery time objectives;
- average and worst-case execution history size;
- tenant fairness and burst limits;
- cost per active step, waiting instance, GB-month, and Box wake;
- whether third-party code must be supported in the first product version;
- whether Rust-authored workflows alone are initially sufficient.

Registered users are not a useful capacity unit. One million durable identities with one percent active is a different system from one million concurrently runnable workflows.

## The boundary validated by the comparables

The following is the recommended ownership model.

```text
Phone, WhatsApp, Slack, API, SDK
                |
                v
       Tenant and agent routing
                |
                v
        osfo Agent Runtime
        - durable agent identity
        - conversation and streaming
        - model and tool loop
        - memory and Box access
        - turn-level recovery
                |
                | start, signal, query, cancel
                v
     osfo Workflows Control Plane
     - generic workflow instances
     - steps, attempts, waits, signals
     - retries, leases, fencing
     - timers, scheduling, quotas
     - package and version pinning
                |
        +-------+--------+
        |                |
        v                v
  Package runner    Typed executors
  - trusted worker  - TryAgent RuntimeControl in Go
  - isolated worker - payments and billing
  - future WASM     - model and data providers
        |                |
        +-------+--------+
                |
                v
       External durable resources
       Boxes, Postgres, filesystems,
       object storage, provider APIs
```

The workflow kernel owns the lifecycle of a **Workflow Instance**. It does not own the lifecycle or implementation of every resource a workflow touches.

A platform workflow may orchestrate Box creation, wake, pause, snapshot, or deletion. The Go provisioning service still owns the actual Box mutation and its host-level invariants. Oz may decide that a user needs a Box and start that workflow. These are three different authorities.

### Domain boundary

| Concept | Durable identity | Owner | Important distinction |
|---|---|---|---|
| Agent | Yes | Agent Runtime | A long-lived participant with policy, tools, and memory. |
| Conversation | Yes | Agent Runtime | A channel-specific or cross-channel interaction history. |
| Agent Run or turn | Usually | Agent Runtime | A bounded reasoning and tool-use execution that may recover independently. |
| Workflow Definition | Yes, versioned | Workflows | Executable durable process contract. |
| Workflow Instance | Yes | Workflows | A discrete run-to-completion or cancellable process. |
| Step | Yes within an instance | Workflows | A durable boundary around computation, waiting, or an external effect. |
| Attempt | Yes | Workflows | One leased try at executing a step. Attempts are never the step identity. |
| Box | Yes | Provisioning domain | A user's durable compute and storage identity. It need not remain continuously running. |
| Application Package | Yes, immutable version | Package Registry | The deployable bundle that connects agents, workflows, schemas, and capabilities. |

### Choosing agent-local execution or a workflow

Use an Agent Runtime primitive when the work is part of the agent's own lifecycle:

- continue a model and tool loop;
- checkpoint a turn;
- stream intermediate output;
- update conversational memory;
- schedule a small reminder owned by that agent;
- resume after a transient model or tool failure.

Use a Workflow Instance when the process should have an independent identity:

- it can continue after the initiating conversation ends;
- each stage needs its own retry or timeout policy;
- it waits for human, payment, webhook, or infrastructure events;
- operators need to pause, cancel, retry, or reconcile it;
- another agent or channel may query or signal it;
- it coordinates multiple external systems;
- it has business consequences such as billing, provisioning, or account changes.

Cloudflare now makes essentially this distinction between work internal to an Agent and a separate Workflow. LangGraph, by contrast, places most durable work inside an agent-oriented graph. Both are coherent, but only the Cloudflare shape satisfies osfo's requirement to host deterministic non-agent processes too.

## Three meanings of dynamic workflows

"Dynamic" must not be one undifferentiated capability.

### 1. Dynamic control flow inside a pinned definition

A branch, loop, fan-out, or tool choice can depend on recorded inputs and recorded step results. This is compatible with durable execution when:

- the workflow package version is immutable and pinned;
- nondeterministic values enter through durable commands or steps;
- completed command results are memoized or replayed;
- stable command identities prevent accidental duplicate effects;
- retries re-observe the same durable values.

This is the Inngest, DBOS, Temporal, and LangGraph lesson in different forms.

### 2. Dynamic package loading

A multitenant control plane may resolve:

```text
tenant + application + deployment
                 |
                 v
package id + exact content digest + runtime compatibility
```

The runner must load that exact artifact again after a crash or long wait. Cloudflare Dynamic Workflows validates this product need, but its public implementation is young. Tenant-routing metadata must not carry credentials or become authorization by assertion.

### 3. Model-generated plans

A model may produce plan data such as:

```text
research -> compare -> ask for approval -> purchase -> report
```

The plan is not executable authority. The runtime validates each requested operation against:

- a registered typed operation;
- the agent and tenant's capabilities;
- budgets and rate limits;
- approval policy;
- schema and parameter constraints;
- package version compatibility.

The model chooses among allowed operations. It does not invent privileged code, storage mutations, or kernel commands.

## Repository architecture extracts

### 1. Temporal

Repository state inspected: [`temporalio/temporal@3aec474`](https://github.com/temporalio/temporal/tree/3aec474f48ab52a2f8299bdfdf030143f8514800) and [`temporalio/sdk-rust@3dac901`](https://github.com/temporalio/sdk-rust/tree/3dac9013b9031e5ffd51d7335838585b2db42efb).

Temporal is the most mature reference for a general durable workflow service. Workflow code runs in user-owned workers. The server stores an event history and schedules workflow and activity tasks. Workers replay history to reconstruct workflow state, while side effects happen in Activities.

Its service split shows the operational scope hidden behind a simple SDK:

- Frontend handles public APIs and routing.
- History owns per-workflow state transitions and history shards.
- Matching owns task queues and worker dispatch.
- internal workers handle system workflows and maintenance.
- persistence supports multiple durable stores.

The Rust repository is especially useful even though the direct Rust workflow SDK remains Public Preview. Temporal Core powers several language SDKs and deliberately separates polling, protocol state machines, command handling, and replay from language-specific invocation and concurrency.

Concrete paths:

- `docs/architecture/README.md`
- `service/history/`
- `service/matching/`
- `service/frontend/`
- `schema/postgresql/`
- `tests/xdc/`, `tests/ndc/`, and `tests/mixedbrain/`
- Rust SDK `ARCHITECTURE.md`
- Rust SDK crates `core/`, `client/`, `sdk/`, and `workflow/`

Adopt:

- explicit workflow versus activity boundary;
- deterministic or memoized reconstruction;
- versioned worker compatibility;
- task queues as a control-plane boundary;
- replay and upgrade tests using real histories;
- operator-visible history and repair tooling.

Avoid:

- copying the full distributed architecture before osfo's workload proves it;
- assuming a mature Go server means the direct Rust authoring experience is mature;
- claiming exactly-once external side effects.

### 2. Restate

Repository state inspected: [`restatedev/restate@061157b`](https://github.com/restatedev/restate/tree/061157b44049b51c0f87ed1050f88b3c8797fc41).

Restate is the strongest open Rust implementation reference. It exposes general Services, keyed Virtual Objects, and Workflows. User services run outside the durable server. The server journals invocations, timers, promises, and calls so that handlers can recover.

Its internals are already separated around real operational responsibilities:

- `crates/bifrost/` for the replicated log;
- `crates/partition-store/` for partitioned durable state;
- `crates/invoker-impl/` for execution;
- `crates/queue/` and `crates/timer/` for runnable and delayed work;
- `crates/service-protocol/` for the service boundary;
- `crates/storage-query-datafusion/` for introspection;
- `crates/node/` for composition.

The release process is unusually relevant. It includes SDK integration, end-to-end tests, Jepsen, long-running load, rolling upgrades, rollback, snapshots, compatibility, and cluster operations. Its Rust guidelines recommend deep components with an `Options` configuration, an event-loop `Service`, a cheap-clone `Handle`, and typed errors. API and implementation crates are split where that protects the seam.

Adopt:

- Rust module and service patterns;
- a language-neutral invocation protocol;
- durable promises, timers, and keyed serialization;
- first-class introspection;
- fault, upgrade, and compatibility testing as release gates.

Avoid:

- adopting the server for a public multitenant workflow platform without a license review. Restate uses BSL 1.1 and contains a specific restriction concerning a public Restate platform service. A proprietary abstraction may be treated differently, but counsel must evaluate the actual osfo product and deployment model.

### 3. Inngest

Repository state inspected: [`inngest/inngest@bf307f5`](https://github.com/inngest/inngest/tree/bf307f517950fae9440d9823079990a680c20396).

Inngest begins with general event-driven durable functions and layers durable agents on top. User functions run on customer compute through SDKs. The service coordinates events, steps, pauses, queues, retries, and flow control.

Its agent design is important for osfo. An LLM or tool call runs as a memoized step. When execution is reconstructed, previous results are returned rather than repeated. Later branches can therefore depend on model output without teaching the engine what a prompt, model, or agent is.

Concrete paths:

- `pkg/execution/execution.go`
- `pkg/execution/executor/`
- `pkg/execution/queue/`
- `pkg/execution/state/`
- `pkg/execution/state/redis_state/`
- `pkg/execution/pauses/`
- `pkg/execution/checkpoint/`

The state documentation treats the function hash and version as correctness-critical because replay and cross-language migration depend on it. The implementation also exposes how much multi-tenant flow control is required beyond durable storage.

Adopt:

- general durable functions beneath agent products;
- memoized model and tool steps;
- explicit flow control, concurrency limits, throttling, and fairness;
- stable function and step identity.

Avoid:

- coupling the kernel to Redis-specific state structures;
- using SSPL-licensed server code in a proprietary hosted service without a deliberate license strategy. Individual releases are scheduled to move to Apache 2.0 after three years, which does not solve immediate adoption.

### 4. Cloudflare Workflows, Agents, and Dynamic Workflows

Repository states inspected: [`cloudflare/agents@e906760`](https://github.com/cloudflare/agents/tree/e906760381bef1458b758260ac41dc0f7bd921e3) and [`cloudflare/dynamic-workflows@9726985`](https://github.com/cloudflare/dynamic-workflows/tree/9726985af886698e85e90e3f044daf6dfcea2c1d).

Cloudflare provides the closest product-level validation:

- Workflows is a general durable multi-step product with retries, sleeps, and external events. AI is one use case, not the type system.
- Agents provides durable identity, local SQL state, direct messaging, schedules, and agent lifecycle primitives.
- `AgentWorkflow` adapts a generic `WorkflowEntrypoint` to bidirectional agent callbacks.
- Dynamic Workflows loads tenant-specific code and reloads the correct version on resume.

The public Dynamic Workflows implementation has three notable parts:

1. a Worker Loader;
2. a Dynamic Worker containing tenant code;
3. a Dynamic Workflow entrypoint that reloads the correct tenant worker during execution and resume.

Concrete paths:

- Agents `packages/agents/src/workflows.ts`
- Agents `packages/think/src/workflows.ts`
- Agents `packages/think/src/e2e-tests/workflow-recovery.test.ts`
- Dynamic Workflows `packages/dynamic-workflows/src/binding.ts`
- Dynamic Workflows `packages/dynamic-workflows/src/entrypoint.ts`
- Dynamic Workflows `packages/dynamic-workflows/src/types.ts`
- Dynamic Workflows `packages/tests/src/binding.test.ts`
- Dynamic Workflows `packages/tests/src/entrypoint.test.ts`

The public Workflows engine is not open source, so source-level confidence covers the SDKs and adapters rather than the durable kernel. Dynamic Workflows was also new at the time of inspection, with a small history and no tagged release. It validates the shape more strongly than the implementation maturity.

Cloudflare's rules also make an essential point: workflow steps can retry, so calls to external systems must be idempotent. Durable orchestration is not magical exactly-once execution.

Adopt:

- the product split among generic workflows, durable agents, and dynamic tenant packages;
- an adapter from Agent Runtime to Workflows rather than a merged engine;
- durable identity that sleeps when inactive;
- package routing metadata separated from secrets and authorization.

Avoid:

- treating published account limits as osfo SLOs;
- treating a durable identity as an always-running process;
- assuming the closed platform's semantics can be inferred completely from its SDK.

### 5. DBOS Transact for Go

Repository state inspected: [`dbos-inc/dbos-transact-golang@d5c96f3`](https://github.com/dbos-inc/dbos-transact-golang/tree/d5c96f3f6c0d98cf593d88e57f4d822222162ae4).

DBOS shows the smallest credible general workflow architecture in this comparison. The application links a language SDK and uses Postgres as the system database. On recovery, workflow code re-enters from the beginning while completed steps return stored results. A separate Conductor can provide centralized recovery and operations.

Concrete paths:

- `dbos/workflow.go`
- `dbos/recovery.go`
- `dbos/queue.go`
- `dbos/scheduler.go`
- `dbos/internal/sysdb/`
- `chaos_tests/chaos_test.go`
- `chaos_tests/cancellation_chaos_test.go`

The repository's real Postgres, CockroachDB, SQLite, cancellation, and chaos tests are more valuable than its benchmark headline. The project is young, and its language-linked application model is less suitable for a hosted, multi-language package platform.

Adopt:

- a Postgres-first tracer implementation;
- memoized step results;
- explicit recovery tests at crash boundaries;
- simple APIs whose durable behavior can be explained without a large server.

Avoid:

- making all workflows live in one language process;
- treating a single-database throughput claim as an osfo capacity plan;
- omitting the operator and recovery control plane because the library API looks small.

### 6. Obelisk

Repository state inspected: [`obeli-sk/obelisk@cb3f26d`](https://github.com/obeli-sk/obelisk/tree/cb3f26de7b53180e6f2022abd9023d2e9e0c83c0).

Obelisk is a Rust and WebAssembly durable execution system. It uses WIT contracts, content-addressed components, execution logs, deterministic workflows, and idempotent activities. It is the best concrete reference for immutable third-party workflow artifacts.

Concrete paths:

- `crates/concepts/src/storage.rs`
- `crates/executor/`
- `crates/db-postgres/`
- `crates/db-sqlite/`
- `crates/wasm-workers/`
- `crates/workflow-js-runtime/`
- `crates/testing/db-tests/`
- `wit/`

Its separation between AI-assisted authoring and deterministic deployed execution is directionally correct. The project explicitly warns that the CLI, APIs, WIT contracts, and schemas are still changing.

Adopt:

- content digest as artifact identity;
- typed host capabilities;
- process or WebAssembly isolation as a package-runner concern;
- authoring-time AI separated from runtime authority.

Avoid:

- committing to WebAssembly before the worker and package protocol is proven;
- using a pre-release system as the production foundation without a long validation cycle;
- adopting AGPL server code without legal review.

### 7. LangGraph and Agent Server

Repository state inspected: [`langchain-ai/langgraph@49ae27c`](https://github.com/langchain-ai/langgraph/tree/49ae27c2ae983cfb92091b0dea9f7bc37a716479).

LangGraph describes itself as a low-level orchestration runtime for long-running, stateful agents. It can implement predetermined workflows, but its primary domain is agent execution.

The open-source runtime uses a StateGraph and Pregel-style supersteps. Reducers merge state updates. Conditional edges and `Send` support dynamic control flow. Persistence has two distinct scopes:

- a thread checkpointer for short-term graph state and recovery;
- a Store for long-term memory shared across threads.

Concrete paths:

- `libs/langgraph/langgraph/graph/state.py`
- `libs/langgraph/langgraph/pregel/`
- `libs/checkpoint/`
- `libs/checkpoint-postgres/`
- `libs/checkpoint-sqlite/`
- `libs/prebuilt/`

The production Agent Server adds assistants, threads, runs, cron, queues, Postgres, Redis, and deployment modes. That production implementation is not the same thing as the MIT runtime repository. Current deployment documentation requires a LangGraph Cloud license key, and self-hosted LangSmith is an enterprise product.

Adopt for the Agent Runtime:

- explicit thread and run identities;
- graph or loop checkpointing;
- separate short-term checkpoint and long-term memory concepts;
- interrupts, resumability, and agent-specific observability;
- a high-level agent harness above a low-level runtime.

Avoid for the workflow kernel:

- making every business process an agent thread;
- using conversation checkpoints as workflow authority;
- embedding prompts, messages, models, or tool semantics in the generic kernel;
- assuming the open-source graph library includes the hosted server's operational guarantees.

## Cross-comparable findings

### 1. General workflows and agents are layers, not rivals

Cloudflare and Inngest put general durable execution underneath their agent features. LangGraph intentionally optimizes for the agent layer. osfo needs both, so it should preserve the lower general layer and build a richer Agent Runtime above it.

### 2. Waiting identity and active compute are different resources

Durable workflows, Agents, and Boxes can exist while no CPU is assigned to them. Capacity planning needs separate counts for:

- registered identities;
- open but waiting workflow instances;
- runnable steps;
- executing attempts;
- active Boxes;
- stored history and artifacts.

This is why a single osfo phone number can route millions of users without keeping one process or VM active per user.

### 3. Per-instance serialization is the default correctness boundary

Temporal History shards, Restate keyed objects and workflows, LangGraph's one-run-per-thread deployment rule, and the previous workflows-python ADR all converge on one idea: state transitions for one durable identity need a single effective authority.

Physical work may run concurrently. The acceptance of a state transition must still be serialized and fenced.

### 4. External effects are at-least-once or uncertain

No durable engine can atomically commit both its own database and an arbitrary external provider. The safe contract is:

- a stable effect idempotency key;
- a fenced attempt;
- an atomic local record before or with dispatch;
- provider-specific deduplication where available;
- an explicit `uncertain` outcome when the process crashes between effect and acknowledgement;
- reconciliation rather than blind retry for non-idempotent effects.

The public API must not claim exactly-once external execution.

### 5. Code version is part of state

A workflow that sleeps for six months must resume against compatible semantics. Every serious comparable has some form of history compatibility, function hash, worker version, or content-addressed component.

An osfo Workflow Instance should pin:

- package id and exact content digest;
- workflow definition key and version;
- input schema version;
- runtime protocol version;
- capability and executor contract versions;
- migration policy.

Deployment metadata should never be the only record of which code an old instance requires.

### 6. The operational plane is most of the system

The happy-path interpreter is not the hard part. Mature systems also implement:

- queues, timers, leases, and stale-worker fencing;
- tenant fairness, concurrency limits, and backpressure;
- history retention and compaction;
- rolling upgrades and mixed-version compatibility;
- backup, restore, failover, and repair;
- operator queries, cancellation, pause, retry, and reconciliation;
- metrics, traces, logs, and audit history;
- poison-work detection and dead-letter handling.

This is the main reason to prove build-versus-adopt before expanding the Rust implementation.

## Mapping to the previous workflows-python decisions

The previous Python work reached several decisions that remain strong:

- runtime and application packages have one-way dependencies;
- definitions and packages are immutable and digest-pinned;
- each instance has serialized durable state;
- attempts use leases and fencing;
- signals are deduplicated and race-safe;
- external effects expose uncertainty and idempotency;
- operators can inspect the current state and ordered trace.

Those should be preserved.

The following constraints were reasonable for the insurance-company proof but are too narrow as the sole model for a Cloudflare-like platform:

- only finite named routes;
- no general loops or sleeps;
- no child workflows or buffered events;
- no hosted third-party authoring;
- no runtime-loaded application package.

The correct move is not to discard the old ADRs. It is to supersede their scope:

```text
Old conclusion:
  A closed finite manifest is sufficient for the reference application.

New conclusion:
  The durable kernel needs a broader command model, while every deployed
  workflow still belongs to an immutable, validated, capability-limited package.
```

## Recommended osfo shape

### Product boundary to commit now

1. `osfo workflows` is a general durable execution platform.
2. `osfo agents` is an agent runtime and application framework built on that platform.
3. Oz is the first osfo Agent Application Package.
4. A platform-owned package implements onboarding, billing, entitlements, and Box lifecycle workflows.
5. TryAgent remains the Go implementation of provisioning and Box control behind a typed protocol.
6. Third-party application packages use the same public contracts as Oz.

This lets Oz start a provisioning workflow without making either Oz or provisioning part of the workflow kernel.

### Proposed Rust module seams

The names are provisional. The dependency direction is the important part.

```text
workflows-core
  Pure domain types, commands, state transitions, IDs, and typed errors.
  No Postgres, network server, agent, Box, or application dependencies.

workflows-protocol
  Client, worker, package, and executor wire contracts.

workflows-store
  Narrow persistence transaction and query interfaces.

workflows-store-postgres
  PostgreSQL implementation, migrations, and lock strategy.

workflows-scheduler
  Runnable work, timers, leases, fencing, quotas, and fairness.

workflows-worker
  Worker loop, package resolution, durable command exchange, and heartbeats.

workflows-server
  API, authentication integration, control-plane composition, and operator APIs.

workflows-sdk
  Rust authoring API with stable durable commands.

workflows-testkit
  Deterministic clock, fake executors, conformance suites, and crash injection.

workflows-telemetry
  Stable metrics, traces, logs, and audit-field vocabulary.
```

`workflows-server` composes modules. It should not become the place where every module imports every other module. `osfo agents`, Oz, and the platform application depend on the public client and package contracts, never on store internals.

### Application Package contract

An immutable Application Package version should declare:

- package id, semantic version, and content digest;
- compatible workflow protocol versions;
- agent entrypoints and workflow entrypoints;
- input, output, signal, and query schemas;
- required typed capabilities;
- trusted executor bindings by logical key;
- runtime and isolation requirements;
- resource, concurrency, and retention limits;
- migration and deprecation metadata;
- human-readable documentation.

Secrets and tenant authorization do not belong in the package artifact or routing metadata. Deployment binds a package's declared capability to a tenant-authorized provider.

### Initial trust tiers

The protocol should support isolation without requiring the most complex isolation mechanism on day one.

| Tier | Code owner | Initial execution | Direction |
|---|---|---|---|
| 1 | osfo | Trusted Rust worker process | Ship first. |
| 2 | selected partners | Separate worker process or Box with scoped credentials | Add after protocol conformance. |
| 3 | arbitrary tenants | Hardened sandbox, microVM, or WebAssembly runtime | Add only when the product requires untrusted code. |

The kernel process should never load arbitrary tenant code into its own address space.

### Initial durability contract

The first public contract should include:

- idempotent start using a caller-supplied request key;
- immutable instance identity;
- serialized state transitions per instance;
- durable step results;
- attempt identity separate from step identity;
- expiring leases and monotonically fenced writes;
- durable sleep and timer wake;
- signal deduplication and safe early arrival;
- cancellation propagation with explicit terminal states;
- package digest pinning;
- stable error classes and retry policies;
- effect idempotency keys;
- explicit uncertain effects and reconciliation;
- operator query, pause, cancel, retry, and audit history.

Do not promise:

- exactly-once external side effects;
- transparent arbitrary code changes for sleeping instances;
- unlimited history;
- arbitrary model-generated executable code;
- million-user capacity before workload-specific testing.

### Authoring model to prototype

The old closed route manifest should not be the only authoring surface. A practical Rust prototype should test a replayable durable-command API. The following is API pseudocode, not promised Rust syntax:

```text
let profile = ctx
    .step("load-profile", || load_profile(user_id))
    .await?;

if profile.needs_box {
    let box_id = ctx
        .step("provision-box", || runtime_control.provision(user_id))
        .await?;

    ctx.wait_for_signal::<PaymentConfirmed>("payment-confirmed").await?;
    ctx.sleep("settlement-delay", two_hours).await?;
    ctx.complete(box_id).await
} else {
    ctx.complete(profile.box_id).await
}
```

This example is illustrative, not an API decision. The prototype must answer:

- whether stable explicit command keys are sufficient for loops and fan-out;
- how completed commands are memoized;
- how source changes are detected against the pinned package digest;
- whether resumption re-enters Rust code or interprets a compiled state model;
- how child workflows and parallel branches are represented;
- which values must be serializable;
- how cancellation and compensation appear to authors.

The semantic contract matters more than surface syntax.

## Build-versus-adopt options

Scores are decision utility out of 10, not comparable maturity scores.

| Option | Utility | Strength | Main cost or risk |
|---|---:|---|---|
| A. Own the osfo product contracts, keep an engine seam, and prove a Rust kernel with a tracer slice | 9 | Preserves strategic ownership without assuming the implementation answer | Requires disciplined scope and a real bake-off |
| B. Implement the osfo API over Temporal first | 8 | Fastest route to mature durability and operations | Large operational system, direct Rust SDK maturity, and abstraction leakage |
| C. Implement the osfo API over Restate first | 7 | Excellent Rust fit and external worker model | BSL terms require product-specific legal review |
| D. Build an agent-specific LangGraph-like runtime as the platform kernel | 4 | Fast for Oz's first agent loops | Fails deterministic and non-agent workflows, mixes memory with business authority |
| E. Keep the previous finite-route manifest as the complete workflow model | 5 | Constrained and easy to reason about | Too weak for reusable dynamic workflows and third-party packages |

### Recommendation

Choose Option A at the product and interface level.

Do not interpret that as approval to build a Temporal-sized system. Build one vertical slice in Rust and keep the persistence and execution seams narrow enough that the same public contract can be exercised against a reference backend or simulator.

The ownership decision should be revisited after the slice survives the following adversarial scenario:

```text
1. Start one workflow idempotently.
2. Lease a step to a worker.
3. Call TryAgent's Go RuntimeControl to create or wake a Box.
4. Kill the worker after the external effect but before result acknowledgement.
5. Expire the lease and issue a new fenced attempt.
6. Reconcile rather than duplicate the external effect.
7. Persist the result.
8. Sleep or wait for a signal.
9. Restart the control plane.
10. Resume against the exact pinned package version.
11. Query the complete operator-visible trace.
```

If the homegrown slice cannot make this behavior simple, inspectable, and testable, osfo should adopt a mature execution substrate while retaining its own product API and package model.

## Enterprise verification bar

### Correctness tests

- model or property tests for every workflow state transition;
- duplicate start, signal, timer, completion, and cancellation tests;
- stale lease and stale fence rejection;
- crash injection before and after every durable write and external call boundary;
- explicit ambiguous-effect and reconciliation tests;
- replay or memoization tests against saved execution histories;
- package digest and incompatible-version rejection;
- early, late, duplicate, and racing signal tests;
- child and parent cancellation propagation;
- real PostgreSQL isolation and lock tests.

### Compatibility and release tests

- old instance resumed by new control-plane code;
- new and old workers operating during rolling deployment;
- database migration forward and rollback behavior;
- package protocol compatibility matrix;
- snapshot, restore, and point-in-time recovery;
- mixed-version cluster and worker tests;
- retained histories from previous releases as regression fixtures.

### Load and operations tests

- sustained starts and runnable steps per second;
- burst traffic by one tenant without starving others;
- timer storms and mass wakeups;
- long histories and history compaction;
- unavailable worker pools and poison work;
- queue recovery after control-plane restart;
- storage growth and retention deletion;
- failover, split-brain, and stale-writer rejection;
- multi-hour and multi-day soak tests.

### CI and release gates

- formatting, Clippy, documentation, and dependency policy checks;
- unit, property, integration, and protocol conformance suites;
- real PostgreSQL jobs, not only mocks;
- sanitizer or Miri coverage for sensitive unsafe or concurrency code where practical;
- fault-injection and crash-recovery suite;
- package SDK examples compiled as tests;
- release candidate upgrade, rollback, and restore rehearsal.

### Required observability vocabulary

Every trace, log, metric, and operator record should consistently identify:

- tenant and application;
- package id and digest;
- workflow definition and instance;
- step and attempt;
- lease and fence;
- signal or effect idempotency key;
- queue and worker pool;
- execution status and terminal reason.

Key measurements include queue delay, wake latency, step duration, attempts, retries, lease loss, stale-write rejection, uncertain effects, reconciliation age, signal latency, retained history size, and cost by tenant.

## SLOs to define before claiming million-user scale

An SLO is a measurable reliability or performance target over a time window. The first osfo capacity plan should state targets such as:

- 99.9 percent of accepted workflow starts become durable within X milliseconds;
- p99 runnable-step queue delay stays below X seconds at the contracted load;
- p99 signal-to-resume latency stays below X seconds;
- no acknowledged state transition is lost within the stated recovery objective;
- 99.9 percent of timers wake within X seconds of their due time;
- one tenant's burst cannot consume more than its allocated share;
- uncertain external effects are surfaced to operators within X seconds;
- restore from backup meets a stated RPO and RTO.

The product may eventually support millions of durable Agent, Box, and Workflow identities. The initial engineering target should be expressed in active starts, runnable work, wakeups, bytes retained, and recovery time.

## Language and architecture guidance

### Rust API Guidelines

Apply the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) to the public SDK and protocol crates:

- use newtypes for semantically different identifiers;
- implement expected common traits;
- expose typed, stable errors;
- document panic, error, and safety behavior;
- make conversions and ownership predictable;
- compile public examples in CI;
- treat naming and feature flags as compatibility commitments.

### PostgreSQL transaction and locking rules

The [PostgreSQL transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html) and [explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html) documentation should govern the first store implementation:

- use unique constraints for idempotency claims;
- keep authority-changing reads and writes in one transaction;
- use row locks only around the smallest authoritative state;
- understand serialization failures and deadlocks as expected retryable outcomes;
- never use a process-local mutex as cross-replica authority.

### Idempotent API design

The AWS Builders' Library article [Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) reinforces:

- caller-provided request identity;
- atomic storage of token and accepted mutation;
- rejection when the same token is reused with different parameters;
- deterministic retrieval of the prior semantic result;
- explicit handling of late requests.

### Workflow Patterns

The [Workflow Patterns](http://www.workflowpatterns.com/) catalog is a useful completeness check. A product should not call itself a general workflow engine until it has an explicit answer for the control-flow patterns it claims, including sequence, splits, joins, deferred choice, cancellation, loops, and persistent triggers.

## Final recommendation

Commit the following architecture:

```text
General osfo Workflows
        ^
        | public client and package contracts
        |
osfo Agent Runtime
        ^
        |
Oz and third-party agents

Platform workflows -> typed RuntimeControl -> existing TryAgent Go infrastructure
```

Then execute a narrow decision program:

1. write ADRs for the domain boundary, durability contract, package identity, and engine adapter;
2. build the crash-safe TryAgent tracer scenario;
3. run the same semantic conformance tests against the Rust tracer and at least one mature comparable;
4. have two adversarial reviewers attack correctness and operations assumptions;
5. define the first workload-based SLO and cost model;
6. only then choose the long-term execution substrate and expand scheduling or sharding.

The idea is not crazy. It is close to where the strongest platforms are converging. The enterprise-standard version is a cleanly layered system where agents are important clients of durable workflows, not the definition of workflows themselves.

## Sources

All sources were accessed on 2026-07-17. Repository links are pinned to the inspected commit where possible.

### Cloudflare

- [Cloudflare Workflows overview](https://developers.cloudflare.com/workflows/)
- [Cloudflare Rules of Workflows](https://developers.cloudflare.com/workflows/build/rules-of-workflows/)
- [Cloudflare Workflows limits](https://developers.cloudflare.com/workflows/reference/limits/)
- [Using Workflows from Agents](https://developers.cloudflare.com/agents/concepts/workflows/)
- [Long-running Agents](https://developers.cloudflare.com/agents/concepts/agentic-patterns/long-running-agents/)
- [Durable execution for Agents](https://developers.cloudflare.com/agents/runtime/execution/durable-execution/)
- [Dynamic Workflows](https://developers.cloudflare.com/dynamic-workers/usage/dynamic-workflows/)
- [cloudflare/agents repository at inspected commit](https://github.com/cloudflare/agents/tree/e906760381bef1458b758260ac41dc0f7bd921e3)
- [cloudflare/dynamic-workflows repository at inspected commit](https://github.com/cloudflare/dynamic-workflows/tree/9726985af886698e85e90e3f044daf6dfcea2c1d)

### LangGraph

- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview)
- [Workflows and agents](https://docs.langchain.com/oss/python/langgraph/workflows-agents)
- [LangGraph persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- [Agent Server](https://docs.langchain.com/langsmith/agent-server)
- [Standalone server deployment](https://docs.langchain.com/langsmith/deploy-standalone-server)
- [langchain-ai/langgraph repository at inspected commit](https://github.com/langchain-ai/langgraph/tree/49ae27c2ae983cfb92091b0dea9f7bc37a716479)

### Temporal

- [Temporal server architecture](https://github.com/temporalio/temporal/blob/3aec474f48ab52a2f8299bdfdf030143f8514800/docs/architecture/README.md)
- [Temporal server repository at inspected commit](https://github.com/temporalio/temporal/tree/3aec474f48ab52a2f8299bdfdf030143f8514800)
- [Temporal Rust SDK repository and status](https://github.com/temporalio/sdk-rust/tree/3dac9013b9031e5ffd51d7335838585b2db42efb)
- [Temporal Rust Core architecture](https://github.com/temporalio/sdk-rust/blob/3dac9013b9031e5ffd51d7335838585b2db42efb/ARCHITECTURE.md)

### Restate

- [Restate workflows use case](https://docs.restate.dev/use-cases/workflows)
- [Restate key concepts](https://docs.restate.dev/foundations/key-concepts)
- [restatedev/restate repository at inspected commit](https://github.com/restatedev/restate/tree/061157b44049b51c0f87ed1050f88b3c8797fc41)
- [Restate BSL license](https://github.com/restatedev/restate/blob/061157b44049b51c0f87ed1050f88b3c8797fc41/LICENSE)
- [Restate Rust guidelines](https://github.com/restatedev/restate/blob/061157b44049b51c0f87ed1050f88b3c8797fc41/docs/dev/rust-guidelines.md)
- [Restate release testing](https://github.com/restatedev/restate/blob/061157b44049b51c0f87ed1050f88b3c8797fc41/docs/dev/release-testing.md)

### DBOS

- [DBOS architecture](https://docs.dbos.dev/architecture)
- [DBOS Go repository at inspected commit](https://github.com/dbos-inc/dbos-transact-golang/tree/d5c96f3f6c0d98cf593d88e57f4d822222162ae4)

### Inngest

- [Inngest functions](https://www.inngest.com/docs/learn/inngest-functions)
- [Inngest durable agents](https://www.inngest.com/docs/learn/durable-agents)
- [How Inngest functions execute](https://www.inngest.com/docs/learn/how-functions-are-executed)
- [inngest/inngest repository at inspected commit](https://github.com/inngest/inngest/tree/bf307f517950fae9440d9823079990a680c20396)
- [Inngest license](https://github.com/inngest/inngest/blob/bf307f517950fae9440d9823079990a680c20396/LICENSE.md)

### Obelisk

- [Obelisk documentation](https://obeli.sk/)
- [Obelisk WebAssembly getting started](https://obeli.sk/docs/latest/wasm/getting-started/)
- [obeli-sk/obelisk repository at inspected commit](https://github.com/obeli-sk/obelisk/tree/cb3f26de7b53180e6f2022abd9023d2e9e0c83c0)
- [Obelisk license](https://github.com/obeli-sk/obelisk/blob/cb3f26de7b53180e6f2022abd9023d2e9e0c83c0/LICENSE-AGPL)

### Standards and references

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [Rust API Guidelines checklist](https://rust-lang.github.io/api-guidelines/checklist.html)
- [PostgreSQL transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [AWS Builders' Library: Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- [Workflow Patterns](http://www.workflowpatterns.com/)
- [Workflow control-flow patterns](http://www.workflowpatterns.com/patterns/control/)

### Previous osfo workflow decisions

- [ADR 0002: Closed declarative workflow definitions](https://github.com/heyimcarlos/openmagic/blob/aa8bc45b7c438c909b351126bd7feef86c72a64f/docs/adr/0002-closed-declarative-workflow-definitions.md)
- [ADR 0003: Per-instance serialized durable kernel state](https://github.com/heyimcarlos/openmagic/blob/aa8bc45b7c438c909b351126bd7feef86c72a64f/docs/adr/0003-per-instance-serialized-durable-kernel-state.md)
- [ADR 0004: Runtime and reference Application Packages](https://github.com/heyimcarlos/openmagic/blob/aa8bc45b7c438c909b351126bd7feef86c72a64f/docs/adr/0004-rebuild-as-runtime-and-reference-application-packages.md)
- [Issue 56: Generic durable workflow kernel](https://github.com/heyimcarlos/openmagic/issues/56)
- [Issue 62: Comparable production systems](https://github.com/heyimcarlos/openmagic/issues/62)
- [Issue 64: Agent and non-agent reuse](https://github.com/heyimcarlos/openmagic/issues/64)
