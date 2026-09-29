# 001: Trust tiers: frozen kernel, trusted services, evolvable agent

- Status: Proposed
- Recorded: 2026-09-29
- Decision date: Pending
- Decision maker: Lucas Li (direction given in the design walkthrough chat, 2026-09-29); technical authorship: Claude
- Decision source: design walkthrough chat, 2026-09-29 (not linkable); the text below awaits human review
- Human review: Pending
- Related specs: [001-product.md](../specs/001-product.md), [002-walking-skeleton.md](../specs/002-walking-skeleton.md)
- Supersedes: None

## Context

The root spec splits the system into a frozen kernel and an evolvable layer, and requires that the kernel be small enough to audit ("Build versus adopt"). It also requires that the agent never sees raw secrets, that private information never surfaces in the wrong chat, and that every action is logged with the harness version ("Constraints", "Sandbox and security boundaries", "Memory framework" §7). Its "supervisor" is the kernel in this ADR.

Three things must sit where the agent cannot reach them, yet do not belong in a small kernel:

- **Chat-platform sessions.** A Telegram userbot session is a stateful connection. Unlike an HTTP API key, it cannot simply be added to each request by a proxy, so whoever holds the connection holds the credential.
- **Model and embedding API keys**, and the metering of every call.
- **The memory store.** Visibility must be applied at query time. If the agent process can open the database file, it can skip the filter, and that filter is code the improvement loop may edit.

Putting all of these in the kernel makes it large and ties it to fast-moving libraries. Leaving them with the agent breaks the constraints above.

## Decision (proposed)

Adopt three tiers.

| Tier | Holds | Changed by |
| --- | --- | --- |
| Kernel | Policy (whitelist, blacklist, caps, chat approvals, budgets); sole writer of the append-only event log; version store and agent launcher (stamps harness version and run id); the port API and router ([ADR 003](003-kernel-agent-interface.md)); the admin API; later, evaluator and rollback | Humans only |
| Trusted services | One class of secret or data each: connectors ([ADR 002](002-connector-plane.md)), the model gateway (LLM and embedding keys), the memory service (derived memory data and the invariants on it) | Humans only |
| Agent (evolvable layer) | Prompts, skills, tools, orchestration, and the memory policies (extraction, consolidation, retrieval). No credentials and no database files | The agent proposes; the kernel evaluates |

Rules:

1. A service accepts calls only from the kernel. The agent reaches services only through the kernel's port API. A service that needs another service (for example the memory service calling the model gateway) does so with its own kernel-issued credential over a route the kernel defines; the path is decided in the memory slice.
2. The kernel is the only writer of the event log. The memory service opens it read-only and owns everything derived from it (claims, graph, personas, embeddings), which remain rebuildable projections as in the root spec. The kernel, the log and the memory service share one host; embeddings are reached over HTTP.
3. The kernel binds each turn to a visibility scope derived from the event that triggered it. The kernel enforces the scope on sends, and the memory service enforces it on reads. The agent cannot choose or widen it. *(New relative to the root spec; the reviewer should confirm it.)*
4. Memory policy code stays evolvable and runs on the agent side. The memory service stores and serves, and it enforces the invariants that policy code cannot override: append-only history, provenance on every claim, and visibility that never widens. Agent-authored records reach it as typed proposals.
5. Kernel and services are both outside the evolvable layer. They differ by responsibility and rate of change, not by trust in the agent: the kernel stays small enough to audit, and services absorb external libraries.

The operator dashboard is a separate UI process outside the agent's reach. It holds no secrets or data and calls the kernel's admin API with the admin token.

The model gateway and memory service are built in later slices; this ADR fixes only where they sit. Where coding workers (delegation) sit is left to a later ADR.

## Alternatives considered

- **Two tiers, as drawn in the root spec.** The kernel holds everything trusted. Rejected: the kernel grows with each platform library and store, which defeats the audit-size constraint.
- **The agent holds credentials and data; the kernel only observes.** Rejected: it violates the secrets and privacy constraints, since the improvement loop can edit the code that enforces them.
- **The kernel issues scoped, short-lived credentials so the agent calls services directly.** Fewer hops, but many enforcement points and no single audit chokepoint for port calls. Not chosen for a single deployment; it can be revisited if latency demands it.

## Consequences

- Benefit: by the design of the interfaces, credentials and stored data are out of the agent's reach once the processes are isolated. Spec 002 proves this at the protocol level only; OS-level isolation is a later slice. The agent still sees the event content it is given, and open egress means content could leave through ordinary web access.
- Benefit: one chokepoint for policy, port-call audit and version stamping.
- Cost: more processes to run (kernel, each service, agent) on modest hardware, and more operational surface.
- Cost: the kernel is on the path of every call, which adds latency.
- Risk: a service that fails to authenticate the kernel would undo the boundary, so each service's caller check needs its own test.
- Tension: the root spec's manual deletion of a person's data conflicts with an append-only log. An operator-only redaction procedure outside the ports is needed and is left to the log and memory slices.
- Effect: the two-layer diagram in the root spec becomes a simplification. This ADR does not edit it.

## Validation and revisit conditions

Nothing is built yet. Spec 002 checks the protocol-level boundary (AC-002, AC-006); a later container slice must check the OS-level one. Not addressed here: storage, lifetime and revocation of the kernel-to-service credentials, and log tamper-evidence. Revisit if the process count proves too heavy for the target machines, if the kernel stays small enough to host a service in-process, or if the chokepoint latency becomes a problem.
