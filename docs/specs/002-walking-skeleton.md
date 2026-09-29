# 002: Walking skeleton: kernel, agent and loopback chat

- Status: Approved
- Owner: Lucas Li
- Intent source: human decisions in the design walkthrough chat, 2026-09-29, building on [001-product.md](001-product.md)
- Human review: Approved by Lucas Li in chat on 2026-09-29 for revision a0a0081 (PR #2); the reviewed text is unchanged. The open questions below remain open.
- Parent: [001-product.md](001-product.md)

## Problem

The root spec's safety structure (a frozen kernel judging a low-authority agent, outward-message limits, an audited record) is only a claim until it runs. If memory, self-improvement and Telegram are built first, each will rest on an untested boundary. The operator (super admin) needs to see the boundary working with a fake agent and a fake chat before any intelligence is added.

## Desired behavior

A person chats through a loopback command-line connector. A stub agent replies. Everything passes through the kernel, which launches the agent, wakes it with one event at a time under a budget, applies the send policy, and records what happens. The operator manages the lists through a small web dashboard.

- **Chat states.** A chat the operator has not approved is *ignored*: no turn starts, its content is not stored, and the dashboard shows only that a chat (id and name) is pending, with a message count. The operator can set a chat to *observe-only* (events reach the agent; sends are blocked) or *active* (sends are allowed, subject to the policy below). Approval applies to messages that arrive afterwards.
- **Send policy**, as in the root spec: whitelisted people are unlimited. Everyone else has a per-person cap and shares an aggregate cap. An aggregate cap of 0 stops all interaction with non-whitelisted people, in direct messages and in direct mentions, and their messages are dropped, not queued. Blacklisted people are never answered.
- **Failure behavior.** With the kernel stopped, the agent cannot send. A turn that overruns its budget is killed and logged. Retrying a send with the same key delivers it once.

Example: a new chat "test-room" sends "hi". Nothing is stored and the dashboard lists the chat as pending. The operator approves it as observe-only, and the next message reaches the stub agent, whose reply is blocked. The operator sets the chat to active, and the message after that is answered.

## Scope

- The kernel: policy, the append-only event log, the agent launcher, the port API ([ADR 003](../adr/003-kernel-agent-interface.md)) with the `inbox`, `im.send` and `schedule.wake` ports, and the admin API.
- A loopback connector ([ADR 002](../adr/002-connector-plane.md)) that can simulate several chats, several senders, direct messages and mentions, and a stub agent.
- A minimal dashboard: view and edit the whitelist, blacklist, caps and chat states; view pending chats; read-only log tail.
- Separate operating-system processes, run on a developer machine and on the Linux box.

## Non-goals

- A real LLM and the model gateway (the next spec), the Telegram connector and verified-chat admin, and the memory service.
- Container isolation. This spec proves mediation by protocol only. It does not prove that the agent cannot read files or reach the network. It departs from the root spec's "in containers" constraint for this slice only; a later slice adds containers.
- Logging of ordinary web egress, self-improvement (proposals, evaluation, rollback), delegation, and multiple admins.
- Dashboard features beyond those listed under Scope.

## Constraints

- Trust tiers, the connector interface, the port API and the language follow [ADR 001](../adr/001-trust-tiers.md) to [ADR 004](../adr/004-typescript-node.md). They are Accepted.
- The admin API and dashboard bind to loopback or a private interface by default, and require a single admin token, stored only as a hash (walkthrough decision).
- *Harness version* means the kernel version plus a revision identifier of the evolvable-layer files the kernel launched. In this slice those files are a local directory.
- The system must run on a MacBook Pro (M1 Max, 32 GB) and on an Ubuntu Linux machine (walkthrough decision).
- Lists, caps and approvals are edited only by the super admin through authenticated channels, never through the agent ([001-product.md](001-product.md), "Outward-action policy").
- Development artifacts are in English; the agent's user-facing language is Chinese (walkthrough decision).

## Acceptance criteria

- AC-001: Given the kernel, loopback connector and stub agent running and an active chat with a whitelisted sender, when the sender types a message, the stub agent's reply appears in the loopback CLI, and the message, the agent's port calls and the reply are all in the log. Verify with an automated end-to-end script, run on macOS and on Ubuntu.
- AC-002: Given the same setup, when the kernel is stopped, an agent send does not reach the connector. The connector also rejects any call that lacks the kernel's credential (a shared secret configured at start-up). Verify with a script that stops the kernel and calls the connector directly.
- AC-003: Given a stub agent that always replies, a whitelisted sender is answered without limit; a non-whitelisted sender with a per-person cap of *N* replies (counted since the cap was last set or reset) gets exactly *N*; a blacklisted sender gets none. Verify with policy tests.
- AC-004: Given an aggregate cap of 0, when a non-whitelisted person sends a direct message or mentions the agent, no turn starts and no reply is sent; raising the cap afterwards does not release those messages. Verify with a test.
- AC-005: Every inbound event from a non-ignored chat, every port call, every policy decision (with its reason) and every outbound send is written to the append-only log with a run id, a turn id where one applies, and the harness version, all stamped by the kernel. The log offers no update or delete through the ports or the admin API. A port call that claims a different version is logged under the kernel's version. Ignored chats leave only the pending-chat counter (see AC-009). Verify by inspecting the log and attempting the false claim and the edits.
- AC-006: The agent process is started with no admin token, no platform or model credential, and no log or database path in its arguments, environment or working directory; its only credential is the per-run port token. The port API has no operation that reads the log or changes lists, caps or approvals, and every admin route rejects the agent's token. Verify by inspecting the launch parameters and calling the routes. This does not prove OS-level isolation (see Non-goals).
- AC-007: Given a budget of *S* steps (port calls) and *T* seconds, a stub agent that never ends its turn is killed by the kernel within one second of the budget, the log records the kill and its reason, a repeated kill has no further effect, and the next event starts a fresh turn. Verify with a test.
- AC-008: Given two `im.send` requests with the same idempotency key, the connector delivers one message. Verify with a test.
- AC-009: Given an ignored chat, a message from it starts no turn, and its content appears nowhere the kernel writes (the log, the database, output logs, files); the dashboard shows the chat as pending with its id, name and message count only. Verify with a test that searches the kernel's data directory and captured output for the content.
- AC-010: Given an ignored chat set to observe-only, the agent receives its later events and its sends are blocked with a logged reason. Given the chat is then set to active, the next send is delivered. Verify with a test.
- AC-011: Reading or changing the whitelist, blacklist, caps, chat states, pending chats or log requires the admin token; a missing or wrong token is rejected. A change takes effect without a restart, and the log records the previous and new values. The admin listener binds only to loopback or a private interface unless configured otherwise, and the configuration holds only a hash of the token. Verify with API tests and by inspecting the configuration.
- AC-012: The dashboard lets the operator do everything in AC-011, see pending chats, and read the log tail (read-only). Verify with a manual walk-through recorded in the PR.
- AC-013: Given a stub agent that requests `schedule.wake` for a later time, the kernel wakes it then, in the scope of the turn that scheduled it; requests beyond the configured cap are refused and logged. Verify with a test.

## Open questions

1. **Default limits.** What are the initial per-person and aggregate caps and list contents, and is a cap counted over a lifetime or per period? Recommendation: start closed, with an empty whitelist and an aggregate cap of 0, counted per lifetime until the operator resets. Blocks only the defaults, not the criteria. Owner: Lucas Li.
2. **Dropped senders' messages.** Should messages from blacklisted people, or dropped under an aggregate cap of 0, be recorded as observations? Recommendation: record the event and its policy decision, and do not start a turn. This decides what the later memory service can learn from. Until answered, AC-005 records the decision and its reason. Whether the content is kept is open. Owner: Lucas Li.
3. **Per-chat approval states.** The root spec has person-level tiers but no chat-level state. Should [001-product.md](001-product.md) be updated to include them, or do they live only in this spec? Owner: Lucas Li.
4. **Agent runtime.** Is the first real agent a tool-calling loop or a code-executing (CodeAct-style) agent? Blocks the first real agent, not this spec or the model gateway spec.
5. **Where the evolvable layer lives.** This repository is public, and a real persona, skills and data must not be committed here. Where do they live (a separate private repository or a local volume)? Blocks real persona work and the self-improvement slice.
6. **Admin transport security.** The admin token travels over plain HTTP on a private network. Should the dashboard add TLS, login rate limiting, and Host and Origin checks against CSRF and DNS-rebinding? Recommendation: Host and Origin checks and rate limiting in this slice; TLS through the operator's own tunnel or proxy. Owner: Lucas Li.
