# 002: IM connectors as trusted services with a curated typed interface

- Status: Accepted
- Recorded: 2026-09-29
- Decision date: 2026-09-29
- Decision maker: Lucas Li (direction given in the design walkthrough chat, 2026-09-29); technical authorship: Claude
- Decision source: design walkthrough chat, 2026-09-29 (not linkable)
- Human review: Approved by Lucas Li in chat on 2026-09-29 for revision a0a0081 (PR #2); the reviewed text is unchanged
- Related specs: [001-product.md](../specs/001-product.md), [002-walking-skeleton.md](../specs/002-walking-skeleton.md)
- Supersedes: None

## Context

The root spec starts on Telegram through a userbot on a dedicated account and expects other platforms later ("Key decisions", "First platform"). It requires that outward messages be limited by policy that the agent cannot change. [ADR 001](001-trust-tiers.md) places platform sessions in trusted services.

Two forces shape the connector interface:

- **Platform independence.** Platform quirks such as peer resolution, reconnects, backfill and flood waits must not leak into the agent or the kernel.
- **Unbypassable send policy.** One comparable agent project reviewed during this design exposes its platform client through a raw pass-through, so any caller can ban, kick or message people outside policy. The kernel can enforce limits only if the connector offers nothing else.

## Decision

1. **A connector is a separate trusted service** that holds one platform's credentials and session and accepts calls only from the kernel. Telegram (userbot, dedicated account) is the first. A loopback command-line connector is the second implementation, used in spec 002 to keep the interface platform-neutral.
2. **The interface is curated and typed, with no raw pass-through.**
   - Events (connector to kernel): `message` (connector id, chat id, sender id, text, media references, reply-to, mentions-agent flag, direct flag, backfill flag, platform message id, timestamp), edited, deleted, reaction, chat joined or left, and connection status.
   - Operations (kernel to connector): `send` (with an idempotency key), react, typing, mark read, join (by invite), leave, backfill (from a watermark), get status.
   - Adding an operation is a reviewed code change by a human.
3. **Ids** are `platform:rawId`, for both chats and users.
4. **Policy lives in the kernel, not the connector**: whitelist, blacklist, caps, mute and per-chat approval. The connector delivers events for every chat it belongs to and performs the operations the kernel requests.
5. **Per-chat approval.** A chat is *ignored* until the operator approves it as *observe-only* or *active* (spec 002). The kernel drops an ignored chat's content before anything is stored, keeping only a pending-chat notice. The kernel requests backfill only for chats that are observe-only or active.
6. **Which operations the agent can reach.** In spec 002 the agent's ports expose `send` only. Join, leave and backfill are kernel- or operator-initiated. Other operations are exposed by later specs, under the send policy.
7. **Login** is a one-time interactive command on the connector host. The session is kept in the connector's private storage. An invalidated or banned session appears as a status error and notifies the operator.
8. **The Telegram client library is not chosen here.** [ADR 004](004-typescript-node.md) names a leading candidate to validate.

## Alternatives considered

- **Connector inside the kernel.** Simpler to run, but the kernel absorbs a large, fast-changing platform adapter and stops being small enough to audit.
- **Raw pass-through of the platform client.** Maximum flexibility, but any caller can act outside the send policy. Rejected.
- **Bot API instead of a userbot.** The root spec selects a userbot; this ADR does not reopen that.
- **No shared interface, with platform code in the agent.** Puts platform quirks and credentials on the agent's side. Rejected.
- **Connector drops ignored chats itself.** Would keep content away from the kernel, but splits policy across two processes. Rejected for now; it can be revisited if kernel exposure to ignored content is a concern.

## Consequences

- Benefit: policy is in one place, and a new platform means a new connector.
- Cost: administrative needs beyond the curated set (for example editing a profile) require a code change.
- Cost: the kernel handles the content of every chat the connector belongs to, including ignored chats, before it drops them. "Ignored" therefore depends on kernel code, and the kernel must not persist that content anywhere, including its own logs.
- Cost: one more process per platform.

## Validation and revisit conditions

Unverified: how a Telegram userbot behaves on a dedicated account (flood waits, session invalidation, ban risk). The account does not exist yet, so this waits for the Telegram connector spec. The loopback connector tests platform neutrality only weakly; a second real platform would test it better. Revisit if a platform needs an operation the curated set cannot express, or if the number of connectors makes operations burdensome.
