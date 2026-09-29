# 006: The evolvable layer lives in a separate private git repository

- Status: Accepted
- Recorded: 2026-09-29
- Decision date: 2026-09-29
- Decision maker: Lucas Li (direction given in the open-questions walkthrough chat, 2026-09-29); technical authorship and details: Claude
- Decision source: open-questions walkthrough chat, 2026-09-29 (not linkable)
- Human review: Approved by Lucas Li in chat on 2026-09-29 for revision 45bdeca (PR #4); the reviewed text is unchanged
- Related specs: [001-product.md](../specs/001-product.md), [002-walking-skeleton.md](../specs/002-walking-skeleton.md)
- Supersedes: None

## Context

The root spec has every change to the agent's prompts, skills, tools, orchestration and memory policy versioned as a patch in git, with rollback, and every record tagged with the harness version ("Self-improvement and evaluation"). This repository is public. The real persona, learned skills and memory policies can contain personal details derived from chats, and they must not be published. Spec 002 defines the harness version as the kernel version plus a revision identifier of the evolvable-layer files the kernel launched.

## Decision

1. **The real evolvable layer** (persona, prompts, skills, tools, memory policies) lives in a **separate private git repository**. Its name is chosen at implementation.
2. **This repository ships a generic seed** of the evolvable layer: default prompts and tools with no personal content. It is the starting point for the private repository, and later seed changes are merged into it deliberately.
3. **The kernel launches the agent from a clean checkout of a specific commit.** The harness version is the kernel version plus that commit SHA. The kernel refuses to launch from a checkout with uncommitted or untracked files, so the SHA identifies the launched content. Until the private repository exists, a content hash of the launched local directory serves as the revision identifier, as spec 002 allows.
4. **Runtime data never goes in git**: the event log, memory database, embeddings, connector sessions and credentials. They live on the deployment's volumes with their own backup.
5. How the agent's proposed changes reach the repository, and how promotion and rollback are done, is decided in the self-improvement spec.

## Alternatives considered

- **A local git repository on the Linux box only.** It exposes nothing to a third party, but there is no off-box backup, and a disk loss loses everything the agent has learned.
- **A directory in this public repository.** The simplest start, but every persona edit and learned skill would be published, including anything personal that ends up in them. Rejected.

## Consequences

- Benefit: the harness version is a commit SHA, and rollback is a git operation on the layer's history.
- Benefit: personal content stays out of the public repository.
- Cost: two repositories to keep in step, and seed changes must be merged by hand.
- Cost: a private hosted repository (for example on GitHub) still stores chat-derived personal content with a third party. Privacy scrubbing of what the agent writes into skills and prompts is needed, and so is scanning for credentials and personal data before commits.
- Risk: runtime data and the layer can drift apart, because a record's harness version points at a commit while the data lives elsewhere. Backups of both must be taken together.

## Validation and revisit conditions

Nothing is built yet. Revisit if chat-derived content in the layer makes even a private third-party host unacceptable, in which case self-hosted git is the fallback, or if keeping the seed in step proves burdensome.
