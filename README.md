# Osfo Workflows

Osfo Workflows is defining a durable coordination layer for work executed across
replaceable processes, sandboxes, Boxes, and cloud resources.

## Current state

This repository is in architecture discovery. It contains product and engine
research, domain language, an initial problem statement, and interactive
architecture artifacts. It does not yet contain the Rust implementation.

The previous Python implementation remains useful evidence. It proved a
PostgreSQL-backed kernel with versioned definitions, serialized per-instance
transitions, durable Steps and Attempts, leases, fencing, Signals, Waits,
idempotency, and real transaction race tests. The new repository is not a port
until the engine choice is resolved.

## Decision posture

osfo should own its workflow product contracts and its integration with
heterogeneous execution resources. It has not yet established a need to own the
entire durable execution engine.

The next decision is whether the product contracts fit naturally over Temporal
or another mature engine. A narrow crash-recovery tracer and shared semantic
conformance tests should decide that before a custom scheduler, timer service,
history system, or sharding layer is built.

## Repository map

- [`CONTEXT.md`](./CONTEXT.md): canonical domain language
- [`docs/problem-statement.md`](./docs/problem-statement.md): the problem,
  required outcomes, boundaries, and decision test
- [`docs/adr/`](./docs/adr/): durable architecture decisions
- [`docs/research/`](./docs/research/): primary-source comparison and product
  research
- [`outputs/`](./outputs/): interactive architecture explanations
- `docs/agents/`: repository workflow conventions for agents

## Current evaluation

```text
Product boundary       strong
Domain model           strong, now being extracted from the Python proof
Correctness evidence   meaningful in the Python proof
Rust implementation    not started
Engine decision        open
Workload and SLOs      not yet specified
Production readiness   not started
```

The most important next artifact is an executable tracer that survives an
ambiguous external effect, an expired Attempt, control-plane restart, and
resumption on the exact pinned package version.
