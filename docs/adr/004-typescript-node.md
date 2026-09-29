# 004: TypeScript on Node.js for the kernel, services and connectors

- Status: Proposed
- Recorded: 2026-09-29
- Decision date: Pending
- Decision maker: Lucas Li (direction given in the design walkthrough chat, 2026-09-29); technical authorship: Claude
- Decision source: design walkthrough chat, 2026-09-29 (not linkable); the text below awaits human review
- Human review: Pending
- Related specs: [001-product.md](../specs/001-product.md), [002-walking-skeleton.md](../specs/002-walking-skeleton.md)
- Supersedes: None

## Context

The root spec sets no language. Constraints from the walkthrough and the root spec:

- The agent runs on an Ubuntu Linux box. Embeddings are served natively on macOS. Development happens on a MacBook Pro (M1 Max, 32 GB) that must be able to run everything.
- Storage is SQLite with FTS5 and a vector extension ("Storage decisions"). Telegram is reached through MTProto as a userbot.
- English is the development language; the agent talks to users in Chinese.
- The kernel must stay small enough to audit ("Build versus adopt").

Observed on 2026-09-29 in a local checkout of a comparable TypeScript agent project (CyberGroupmate, AGPL-3.0; ideas only, no code copied): its `package.json` declares `@mtcute/node`, `better-sqlite3` and `sqlite-vec`, and its tests include a Telegram adapter and sqlite-vec. This is weak evidence that the libraries can coexist. It does not show that they fit this system, and this repository cannot verify it.

## Decision (proposed)

- The kernel, trusted services, connectors and dashboard are written in **TypeScript on Node.js 22 or newer**. The dashboard is a separate trusted UI service that calls the kernel's admin API ([ADR 001](001-trust-tiers.md)).
- Port and connector schemas are defined once and shared as types, and they are **validated at runtime at every process boundary**.
- SQLite is used through `better-sqlite3`, with the vector extension behind an interface as the root spec requires.
- The Telegram client library is chosen in the Telegram connector spec. `mtcute` is the leading candidate; pin its exact version.
- The **agent runtime's language is not fixed** by this ADR. Its first implementation is TypeScript, and [ADR 003](003-kernel-agent-interface.md)'s schema-first interface allows another runtime later.

## Alternatives considered

- **Python.** Mature Telegram libraries and the strongest LLM and ML ecosystem, but embeddings sit behind an HTTP API here, which shrinks that advantage. A strongly typed interface shared across processes takes more discipline.
- **Go.** Small static binaries and a small, auditable kernel. It was not evaluated in depth; the MTProto and SQLite-vector ecosystem is believed to be thinner, and iterating on prompts, skills and tools is likely slower.
- **Mixed languages by tier** (for example a Go kernel and a TypeScript agent). Possible because the interface is schema-first, but it costs two toolchains before there is evidence that the kernel needs it.

## Consequences

- Benefit: one language, one toolchain and shared schema types across tiers, and fast iteration on the evolvable layer.
- Cost: types vanish at runtime, so boundary validation is mandatory and adds code.
- Cost: the audit surface includes the Node runtime and npm dependencies. Dependencies of the kernel need to be few and pinned.
- Risk: a fast-moving Telegram library can break on updates; pinning and a small connector limit the damage.

## Validation and revisit conditions

Unverified: `mtcute` on a dedicated userbot account, FTS5 behaviour for Chinese text (the default tokenizer segments it poorly), and loading `sqlite-vec` in the chosen container images. Each is checked in the slice that first needs it. Revisit if the Telegram spike shows the library is unsuitable, if the kernel's footprint is a problem on the target machines, or if auditing the kernel's dependency tree proves impractical.
