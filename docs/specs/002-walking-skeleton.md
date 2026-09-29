# 002: Walking skeleton: kernel, agent and loopback chat

- Status: Approved for revision a0a0081; the revision on this branch is pending review
- Owner: Lucas Li
- Intent source: human decisions in the design walkthrough chat, 2026-09-29, building on [001-product.md](001-product.md); the owner's resolutions of the open questions, 2026-09-29 (issue #3)
- Human review: Approved by Lucas Li in chat on 2026-09-29 for revision a0a0081 (PR #2). This revision (issue #3): Pending
- Parent: [001-product.md](001-product.md)

## Problem

The root spec's safety structure (a frozen kernel judging a low-authority agent, outward-message limits, an audited record) is only a claim until it runs. If memory, self-improvement and Telegram are built first, each will rest on an untested boundary. The operator (super admin) needs to see the boundary working with a fake agent and a fake chat before any intelligence is added.

## Desired behavior

A person chats through a loopback command-line connector. A stub agent replies. Everything passes through the kernel, which launches the agent, wakes it with one event at a time under a budget, applies the send policy, and records what happens. The operator manages the lists through a small web dashboard.

- **Chat states.** A chat the operator has not approved is *ignored*: no turn starts, its content is not stored, and the dashboard shows only that a chat (id and name) is pending, with a message count. The operator can set a chat to *observe-only* (events reach the agent; sends are blocked) or *active* (sends are allowed, subject to the policy below). Approval applies to messages that arrive afterwards.
- **Send policy**, as in the root spec: whitelisted people are unlimited. Everyone else has a per-person cap and shares an aggregate cap. An aggregate cap of 0 stops all interaction with non-whitelisted people: none of their messages starts a turn, whether it is a direct message, a mention of the agent, or an ordinary message in a group. Their messages are dropped, not queued. Blacklisted people are never answered. Caps count the replies sent since the cap was last set or reset.
- **Recording.** Chat approval decides what the agent may see, and the person tier decides whom it may answer. In an approved chat, every message is recorded, including those from blacklisted people and those dropped under an aggregate cap of 0, together with the policy decision. No turn starts for a dropped message.
- **Closed start.** A fresh installation has empty whitelist and blacklist, an aggregate cap and a per-person cap of 0, and no approved chats. The operator opens things up deliberately.
- **Failure behavior.** With the kernel stopped, the agent cannot send. A turn that overruns its budget is killed and logged. Retrying a send with the same key delivers it once.

Example: on a fresh installation the operator whitelists themself. A new chat "test-room" sends "hi". Nothing is stored and the dashboard lists the chat as pending. The operator approves it as observe-only, and the operator's next message reaches the stub agent, whose reply is blocked. The operator sets the chat to active, and the message after that is answered.

## Scope

- The kernel: policy, the append-only event log, the agent launcher, the port API ([ADR 003](../adr/003-kernel-agent-interface.md)) with the `inbox`, `im.send` and `schedule.wake` ports, and the admin API.
- A loopback connector ([ADR 002](../adr/002-connector-plane.md)) that can simulate several chats, several senders, direct messages and mentions, and a stub agent.
- A minimal dashboard: view and edit the whitelist, blacklist, caps (including resetting a cap's reply count) and chat states; view pending chats; read-only log tail.
- Separate operating-system processes, run on a developer machine and on the Linux box.

## Non-goals

- A real LLM and the model gateway (the next spec), the Telegram connector and verified-chat admin, and the memory service.
- Container isolation. This spec proves mediation by protocol only. It does not prove that the agent cannot read files or reach the network. It departs from the root spec's "in containers" constraint for this slice only; a later slice adds containers.
- Per-period cap windows: caps count replies since set or reset in this slice, and a later spec adds windows before real Telegram use.
- Logging of ordinary web egress, self-improvement (proposals, evaluation, rollback), delegation, and multiple admins.
- Dashboard features beyond those listed under Scope.

## Constraints

- Trust tiers, the connector interface, the port API and the language follow [ADR 001](../adr/001-trust-tiers.md) to [ADR 004](../adr/004-typescript-node.md). They are Accepted.
- The admin API and dashboard bind to loopback or a private interface by default, and require a single admin token, stored only as a hash (walkthrough decision). The token is sent in an `Authorization` header, never in a cookie or query string. Requests with an unexpected `Host` or `Origin` are rejected, and repeated failed attempts are rate limited. TLS is not built in; the operator provides it through their own tunnel or VPN (owner decision, 2026-09-29).
- *Harness version* means the kernel version plus a revision identifier of the evolvable-layer files the kernel launched ([ADR 006](../adr/006-evolvable-layer-repository.md)). In this slice those files are a local directory.
- The system must run on a MacBook Pro (M1 Max, 32 GB) and on an Ubuntu Linux machine (walkthrough decision).
- Lists, caps and approvals are edited only by the super admin through authenticated channels, never through the agent ([001-product.md](001-product.md), "Outward-action policy").
- Development artifacts are in English; the agent's user-facing language is Chinese (walkthrough decision).

## Acceptance criteria

- AC-001: Given the kernel, loopback connector and stub agent running and an active chat with a whitelisted sender, when the sender types a message, the stub agent's reply appears in the loopback CLI, and the message, the agent's port calls and the reply are all in the log. Verify with an automated end-to-end script, run on macOS and on Ubuntu.
- AC-002: Given the same setup, when the kernel is stopped, an agent send does not reach the connector. The connector also rejects any call that lacks the kernel's credential (a shared secret configured at start-up). Verify with a script that stops the kernel and calls the connector directly.
- AC-003: Given a stub agent that always replies, a whitelisted sender is answered without limit; a non-whitelisted sender with a per-person cap of *N* replies (counted since the cap was last set or reset) gets exactly *N*; a blacklisted sender gets none. Verify with policy tests.
- AC-004: Given an approved chat and an aggregate cap of 0, when a non-whitelisted person sends a direct message, mentions the agent, or posts an ordinary message in a group, no turn starts and no reply is sent; the message is still recorded in the log with its policy decision, and raising the cap afterwards does not release it. Verify with a test.
- AC-005: Every inbound event from a non-ignored chat (including messages from blacklisted people and messages dropped under an aggregate cap of 0), every port call, every policy decision (with its reason) and every outbound send is written to the append-only log with the harness version, and with the run id and turn id where they apply, all stamped by the kernel. The log offers no update or delete through the ports or the admin API. A port call that claims a different version is logged under the kernel's version. Ignored chats leave only the pending-chat counter (see AC-009). Verify by inspecting the log and attempting the false claim and the edits.
- AC-006: The agent process is started with no admin token, no platform or model credential, and no log or database path in its arguments, environment or working directory; its only credential is the per-run port token. The port API has no operation that reads the log or changes lists, caps or approvals, and every admin route rejects the agent's token. Verify by inspecting the launch parameters and calling the routes. This does not prove OS-level isolation (see Non-goals).
- AC-007: Given a budget of *S* steps (port calls) and *T* seconds, a stub agent that never ends its turn is killed by the kernel within one second of the budget, the log records the kill and its reason, a repeated kill has no further effect, and the next event starts a fresh turn. Verify with a test.
- AC-008: Given two `im.send` requests with the same idempotency key, the connector delivers one message. Verify with a test.
- AC-009: Given an ignored chat, a message from it starts no turn, and its content appears nowhere the kernel writes (the log, the database, output logs, files); the dashboard shows the chat as pending with its id, name and message count only. Verify with a test that searches the kernel's data directory and captured output for the content.
- AC-010: Given an ignored chat set to observe-only, the agent receives later events from whitelisted senders and its sends are blocked with a logged reason. Given the chat is then set to active, the next send is delivered. Verify with a test.
- AC-011: Reading or changing the whitelist, blacklist, caps (including resetting a count), chat states, pending chats or log requires the admin token; a missing or wrong token is rejected. A change takes effect without a restart, and the log records the previous and new values. The admin listener binds only to loopback or a private interface unless configured otherwise, and the configuration holds only a hash of the token. Verify with API tests and by inspecting the configuration.
- AC-012: The dashboard lets the operator do everything in AC-011, see pending chats, and read the log tail (read-only). Verify with a manual walk-through recorded in the PR.
- AC-013: Given a stub agent that requests `schedule.wake` for a later time, the kernel wakes it then, in the scope of the turn that scheduled it; requests beyond the configured cap are refused and logged. Verify with a test.
- AC-014: Given a fresh installation, the whitelist and blacklist are empty, both caps are 0 and no chat is approved; a message from any chat starts no turn and nothing is sent until the operator changes the configuration through the admin API. Verify with a test on an empty data directory.
- AC-015: Given a request to an admin route with the correct token, it is rejected when its `Host` is not in the configured allowlist (by default the address the listener is bound to) or when it carries a browser `Origin` that is not allowlisted; requests without an `Origin` are judged by `Host` alone. A token supplied in a cookie or query string is not accepted. After the configured number of failed attempts within the configured window, further attempts from that client are refused until the window passes, and the log records the start of each lockout. Verify with API tests.

## Open questions

None. The questions from earlier revisions were resolved by the owner on 2026-09-29: the limits, recording rule, group messages under a cap of 0 and admin security above; chat approval in [001-product.md](001-product.md); the agent runtime in [ADR 005](../adr/005-agent-runtime.md); and the evolvable layer's location in [ADR 006](../adr/006-evolvable-layer-repository.md).
