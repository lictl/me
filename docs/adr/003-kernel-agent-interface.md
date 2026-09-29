# 003: Kernel–agent interface: a single port API with kernel-driven turns

- Status: Accepted
- Recorded: 2026-09-29
- Decision date: 2026-09-29
- Decision maker: Lucas Li (direction given in the design walkthrough chat, 2026-09-29); technical authorship: Claude
- Decision source: design walkthrough chat, 2026-09-29 (not linkable)
- Human review: Approved by Lucas Li in chat on 2026-09-29 for revision a0a0081 (PR #2); the reviewed text is unchanged
- Related specs: [001-product.md](../specs/001-product.md), [002-walking-skeleton.md](../specs/002-walking-skeleton.md)
- Supersedes: None

## Context

The root spec describes the interface as narrow: append an event, propose a change, request a delegation ("Sandbox and security boundaries"). It also requires that all outbound traffic be logged and that the agent have full freedom inside its own environment. [ADR 001](001-trust-tiers.md) puts credentials and data in trusted services, so the agent needs a defined way to reach them.

The interface must give the kernel control of budget, audit and version stamping. It must also stay usable while the agent runtime (a tool-calling loop or code execution) and the agent's language remain open.

## Decision

1. **One endpoint.** The agent's only channel to the kernel and, through it, to every service is the kernel's port API. Ordinary web access and package installs remain open, as the root spec decides. Ports carry everything involving credentials, chat platforms, memory, models, proposals, delegation and schedules.
2. **Schema-first, transport-neutral contract**, with a protocol version in the handshake. The first transport is HTTP for calls plus one streaming channel (WebSocket) for the inbox, on a private network. The kernel issues a bearer token for each run. It is valid only for the port API, and it is revoked when the run ends or is killed.
3. **Kernel-driven turns.** A *run* is one launch of the agent process. A *turn* is one wake within a run. The kernel wakes the agent with one event (or a scheduled wake) and a budget (steps, tokens, wall time), where a step is one port call. The agent works through ports. A turn ends when the agent finishes or the kernel kills it, and a kill must be safe to repeat. Proactive behaviour uses `schedule.wake`, capped by kernel policy.
4. **The kernel stamps the harness version, run id and turn id** on every record. *Harness version* is the kernel version plus a revision identifier of the evolvable layer the kernel launched. The run id comes from the token. Version and id claims made by the agent are not accepted; records use the kernel's own values.
5. **Turn scope** is bound by the kernel as in [ADR 001](001-trust-tiers.md), rule 3. A wake inherits the scope of the turn that scheduled it and cannot widen it.
6. **Side effects carry idempotency keys.** Inbox events are delivered at least once, with ids, and the agent acknowledges them.
7. **No append-event port.** The root spec's "append an event" is replaced by the kernel logging every port call itself. Agent-authored records reach the memory service as typed proposals (`memory.propose`), not as log entries.
8. **Ports.**

   | Port | Slice |
   | --- | --- |
   | `inbox` (wake events and acknowledgements), `im.send`, `schedule.wake` | Spec 002 |
   | `model.complete`, `model.embed` | Model gateway slice |
   | `memory.search`, `memory.propose` | Memory slice |
   | `changes.propose`, `delegate.*` | Self-improvement and delegation slices |

   `im.send` maps to the connector's `send` operation after policy ([ADR 002](002-connector-plane.md)). Admin operations (lists, caps, approvals) are a separate API with a separate credential and are not ports.
9. **Runtime-neutral.** The interface assumes neither a tool-calling loop nor code execution.

## Alternatives considered

- **Unix domain socket.** Good permission semantics, but the agent and kernel will be separate containers, and Docker on a Mac runs in a VM. It stays a possible later transport behind the same schema.
- **Agent-driven long-running loop.** Suits long thinking, but the kernel then controls only timeouts and kills, and "one turn is one audited unit" is lost.
- **Scoped credentials so the agent calls services directly.** See [ADR 001](001-trust-tiers.md).
- **MCP as the agent-facing interface.** Familiar to tools, but it carries no turn budget or kernel-bound scope. It could be offered later as an adapter over the same ports.

## Consequences

- Benefit: budget, port-call audit, scope and version stamping are enforced in one place.
- Benefit: the schema is independent of language, so the agent runtime can be replaced without changing the kernel.
- Cost: every model call and memory query passes through the kernel, adding a hop, and the gateway must stream tokens through it.
- Cost: breaking changes to the schema need kernel and agent released together.
- Gap: the root spec logs all egress, but nothing in spec 002 logs ordinary web traffic. Egress logging is unbuilt and unverified.
- Gap: ports have step budgets but no per-port rate limits beyond the send policy.

## Validation and revisit conditions

Spec 002 verifies the parts in its slice: mediation (AC-001, AC-002), version stamping (AC-005), budget kill (AC-007), idempotent send (AC-008) and scheduled wakes (AC-013). Streaming model output and memory scoping are unverified until their slices. Revisit if hop latency is noticeable in real conversations, if a runtime needs long autonomous loops, or if the port set grows past what a small kernel can carry.
