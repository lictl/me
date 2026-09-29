# 005: First agent runtime: a tool-calling loop

- Status: Accepted
- Recorded: 2026-09-29
- Decision date: 2026-09-29
- Decision maker: Lucas Li (direction given in the open-questions walkthrough chat, 2026-09-29); technical authorship and details: Claude
- Decision source: open-questions walkthrough chat, 2026-09-29 (not linkable)
- Human review: Approved by Lucas Li in chat on 2026-09-29 for revision 45bdeca (PR #4); the reviewed text is unchanged
- Related specs: [001-product.md](../specs/001-product.md), [002-walking-skeleton.md](../specs/002-walking-skeleton.md)
- Supersedes: None

## Context

The root spec gives the agent full control of its own workspace, shell, packages, files, skills and tools, and expects it to improve its prompts, skills, tools and orchestration ("Self-improvement and evaluation", "Sandbox and security boundaries"). [ADR 003](003-kernel-agent-interface.md) keeps the kernel interface neutral about how the agent thinks and acts, so the first real agent still needs a shape. The owner stated in the design walkthrough chat (2026-09-29) that the default model provider is DeepSeek, chosen for cost.

The shape affects three things: how much of the agent's behaviour is auditable, the API the model gateway must offer (with or without tool calls), and how large the agent's own execution surface is.

## Decision

The first real agent is a plain LLM **tool-calling loop**.

- Its tools are the kernel ports exposed as tools (`im.send`, `schedule.wake`, later `memory.*`), plus shell, file and web tools that run inside the agent's environment.
- Skills are documents or scripts the loop can load. Tool definitions, prompts and skills live in the evolvable layer ([ADR 006](006-evolvable-layer-repository.md)).
- The model gateway must pass tool definitions and tool calls through. Its exact schema is decided in the model-gateway spec.
- The interface stays runtime-neutral. A code-executing (CodeAct-style) tool may be added later as an evolvable change, provided any action involving platforms, memory or models still goes through ports.

## Alternatives considered

- **CodeAct-style from the start.** It can need fewer model round trips for multi-step actions. It adds a code-execution surface beyond ordinary tool calls, and each code block is an opaque action to audit. Rejected for the first agent.
- **Defer the choice.** Would leave the gateway's API shape (tool calls or not) to be decided by accident. Rejected.

## Consequences

- Benefit: every tool call is a discrete, named action. If the model gateway logs what the model sees and returns, as the root spec's "log everything the model sees" intends, the shell and file actions the model requests appear in the log even though the kernel does not mediate them. Actions taken inside bundled scripts, and ordinary web traffic, do not.
- Benefit: a smaller agent than a code-execution sandbox, and a familiar shape for the improvement loop to edit.
- Cost: one model round trip per tool step, so multi-step tasks cost more tokens than code that composes them. Parallel tool calls (where the model supports them) and skills that bundle steps reduce this.
- Cost: shell, file and web tools run unmediated inside the agent's environment, so their containment rests on the later container slice.

## Validation and revisit conditions

Unverified: how reliably the default model follows tool-calling in Chinese and English. This is tested in the model-gateway slice. Revisit if tokens per turn make the budget unworkable, or if a class of tasks clearly needs composition that code would do better.
