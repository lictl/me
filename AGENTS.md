# Project guidance

## Project map

Self-Improving Social Agent Harness: an AI persona that lives in Telegram group and private chats over the long term, learns each person, and improves its own prompts, skills, tools and orchestration from real interaction, while a small frozen kernel that it cannot edit judges and limits it. One operator, one deployment, local hardware. The repository is new: there is no code, stack or build yet, only the design doc.

- Product and design intent: [docs/specs/001-product.md](docs/specs/001-product.md) is the root spec: goals, requirements, key decisions, architecture (with a Mermaid diagram), memory framework, evaluation, sandbox boundaries, and the list deferred to implementation.
  - It is a human-authored design doc with no recorded approval status, and it does not follow the spec headings below. Treat it as the human's current direction. Do not rewrite, rename or restructure it without human direction.
  - Its "Key decisions" table records choices made by the human but is not an ADR. Cite it, and write an ADR when a decision needs its own rationale and alternatives.
- Specs: `docs/specs/`, as `NNN-descriptive-slug.md`. Link the parent spec from each child. The walking-skeleton spec `002-walking-skeleton.md` is the first child of the root spec.
- Decisions: `docs/adr/`, as `NNN-descriptive-slug.md`. ADRs and specs have separate number sequences. Take the next number above the largest existing one in each, and never reuse or renumber. The trust tiers, connector plane, kernel–agent interface and language are proposed in ADRs `001` to `004`.
- Project memory: `docs/memory/` (empty so far). Read only the notes relevant to the task.
- Current work and delivery evidence: GitHub issues and PRs in `lictl/me` (see GitHub delivery below). Templates are in `.github/`.

## Commands

Not established: no manifest, CI or tests exist, and the language and stack are undecided. When the first slice fixes them, add setup, build and focused test commands here with working directories and prerequisites, and mark which ones have been run. Do not invent commands or assume a stack from `.gitignore` patterns.

## Working agreement

- Intent comes from current explicit human direction and approved specs. Parent specs constrain child specs. Accepted ADRs record architectural constraints and reasons. Tests, code and runtime are evidence of implemented behavior. Memory and earlier agent output are fallible context, not authority.
- When these sources conflict, name the discrepancy. Follow an explicit human resolution if one exists. Otherwise surface the material choice and continue only unaffected work or a bounded exploratory check. Do not silently rewrite requirements, weaken tests, or replace an accepted decision.
- Human chat and comments set intent, but they do not review generated documents. Creating or editing an ADR or spec, including editorial edits, renames, deletions and mixed PRs, requires human review of the actual text before merge. Record who approved, the source, and the document revision. Task authorization, agent review and passing checks do not count. Keep new ADRs Proposed and new specs Draft until then. Changed reviewed content needs renewed review; unrelated code commits or adding the approval record do not. Once approval is recorded, moving Proposed to Accepted or Draft to Approved needs no further review if the text is unchanged. Ordinary code-only merges stay autonomous.
- Before each slice, check the current request, the relevant branch of the spec tree, relevant accepted ADRs and interfaces. Reuse unchanged context and recheck revisions when work resumes.
- Work in independently verifiable slices, preferably a thin observable behavior through the necessary layers. An enabling or spike slice needs a concrete verification result and a named dependent behavior. A spike is evidence, not approval to ship its assumptions.
- The design doc's "Deferred to implementation" items (embedding model, decay rates, canary size, egress rules, dashboard details and so on) are decided when their slice is built, using the evidence the doc names. Record significant choices in an ADR or spec change. Do not pick silent defaults.
- Tie completion claims to acceptance criteria and actual evidence. Report failed or unrun checks and environment limits. Do not weaken a check to get a pass.
- Prefer focused tests: unit tests for logic, boundary/contract tests where components meet, and a few e2e tests for important workflows. For a bug, add a regression test that fails before the fix and passes after, including relevant negative cases. Reuse existing coverage, and avoid redundant or implementation-mirroring tests and tests for documentation-only edits. Not every change needs every layer. The design doc rules out synthetic benchmark suites for evaluating the agent; that concerns the product's self-evaluation, not ordinary tests of this codebase.
- Specs stay compact, with seven headings: Problem, Desired behavior, Scope, Non-goals, Constraints, Acceptance criteria, Open questions. Add metadata for status (Draft/Approved), owner, source and parent. Unresolved behavior stays an open question, not a default. Keep decisions in ADRs and executable constraints in tests, types, schemas or configuration.
- An ADR records one significant decision: date, status (Proposed/Accepted), decision maker and source, context, decision, credible alternatives, consequences. A changed decision gets a new ADR linked to the one it supersedes. Do not turn implementation archaeology into accepted ADRs.
- Save a project memory note when verified evidence contradicts a plausible assumption, or a costly investigation yields a reusable lesson. One short file per topic in `docs/memory/`, named for the topic, with: observed date, what it applies to, evidence, why it is worth remembering, and when to revalidate. Then state the tempting assumption or trap and the verified finding. Update or retire an existing note rather than adding a duplicate. No secrets, transcripts, speculative requirements or obvious facts.
- Continue autonomously through the authorized task, including checks and agent review. Finish a reviewable ADR/spec change before requesting its human review, and leave it unmerged until approval is recorded. For other real blockers or unresolved material choices, ask the human and keep going on unaffected work.

## Local constraints

The design doc governs. These are the constraints most likely to be violated by accident:

- **Frozen kernel.** The evaluator, safety and permission policy, budget limits, rollback, version store, event log and persona charter live in a separate supervisor process and are changed only by humans. Nothing in the evolvable layer may be able to write to them.
- **Brokered secrets.** The agent runtime never sees raw LLM or chat-platform credentials. The supervisor or a proxy injects them.
- **Outward messages.** Every message the agent sends goes through the supervisor's whitelist, blacklist and rate-limit enforcement. No code path may send around it.
- **Event log and provenance.** The event log is append-only. Every event and action is tagged with the harness version that produced it. Every memory records its producer and source trust, and low-trust content never becomes a durable fact without verification.
- **Privacy.** Visibility and sensitivity are enforced at query time, so information from a private chat cannot surface in the wrong chat.
- **Fit.** The root spec requires both Chinese and English. Development is in English (code, docs, commits), and the agent talks to users in Chinese. The target hardware is modest local machines (an 8 GB Mac mini plus a 32 GB box), in containers.
- **Secrets and data in git.** The GitHub repo `lictl/me` is public. `.env*` (except `.env.example`), `.dev.vars*`, `/local` and `/output` are gitignored. Never commit or post to an issue or PR any secrets, real chat content or personal data. Use synthetic or scrubbed fixtures.
- **Workstation-local paths.** `.claude/` and `.agents/` are gitignored, so any skills there (for example `write-spec`, `write-adr`, `github-orchestrate`) exist only on some workstations. This file does not depend on them. `/.ai/worktree/` is gitignored and holds agent worker checkouts.

## Commit messages

Git history is part of the traceable work log. Keep messages brief:

- Aim for a subject of about 50 characters or fewer. A subject alone is enough when no explanation or reference is needed.
- If a body is useful, separate it from the subject with a blank line and wrap prose around 72 characters. Explain the problem and why the change is needed, plus non-obvious consequences. Do not narrate the diff or copy the PR/test log. Separate paragraphs with blank lines; short bullets are fine.
- Put real issue references at the bottom. Use `Resolves: #123` only when the change completes that issue; use `See also: #456, #789` for related or partial work. Omit unused references.
- Add `Changelog: <category>` only for a changelog-worthy change, choosing one of `added`, `fixed`, `changed`, `deprecated`, `removed`, `security`, `performance`, or `other`. Omit the trailer entirely for documentation-only changes and small refactors.
- Preserve this format and useful references in the final squash/merge commit; do not concatenate worker messages into a long summary.

Message shape (replace example references and omit optional parts):

```text
Short summary of the change

Optional explanation of the problem and why this change is needed.
Mention non-obvious consequences only when useful.

Resolves: #123
See also: #456, #789
Changelog: fixed
```

## GitHub delivery

Tasks are delivered through GitHub issues and PRs in `lictl/me`, base branch `main`. Branch protection is not enabled and no GitHub Actions workflow exists, so "required checks" are the ones the project's commands define once they exist. Do not change branch protections or add CI to get a merge through.

- All agents use the shared repository-owner GitHub account, `lictl`. Several `gh` accounts can be logged in on one machine, so run `gh auth status` and confirm `lictl` is active before writing to GitHub.
- The parent coordinates issues, delegation, review and merge. Delegate implementation by default: fan out independent slices, serialize dependent ones. Use one worker for a small task.
- Create or reuse an issue before implementation, using [.github/ISSUE_TEMPLATE/task.md](.github/ISSUE_TEMPLATE/task.md). Link intent, scope, acceptance criteria, planned verification and dependencies. Link specs instead of copying them. For a larger task, link child issues from a parent. Each implementation issue has one active writer and normally one PR.
- Assign each worker its own branch `agent/issue-<number>-<slug>` and worktree `<fixed-project-root>/.ai/worktree/<issue-number>-<slug>`. The fixed project root is the primary checkout (`git rev-parse --show-toplevel` from it). Pass it explicitly and never nest orchestration roots inside a worker checkout. Fetch the base, record its branch and SHA, and create the worktree from it. Preserve existing work and never force-reset an occupied branch or overwrite a directory. The base must already be on `origin`, so publish local commits before dispatching. When adding recursive build, test, lint or watch tools, exclude `.ai/worktree/`.
- Hand the worker the issue URL, exact worktree path, branch, starting base SHA and verification expectations, and record the assignment on the issue. Workers implement, verify, push, and open or update a PR using [.github/pull_request_template.md](.github/pull_request_template.md). Keep incomplete work in draft. Link issue and PR both ways. Use `Closes #N` only for an issue the PR completes and `Related to #N` for partial or parent work. Coordinate shared test resources and numbered documents so slices do not collide.
- The orchestrator reviews the actual diff and evidence against the issue and relevant specs/ADRs, and posts findings on the PR naming the reviewed head SHA. Label comments with a role such as `Worker for #N` or `Orchestrator review of <sha>`. Workers fix on the same branch and reply with the commit and evidence. Re-review every new PR head.
- For ADR/spec changes, finish checks and agent review first, then ask the human to review the actual document. Leave the PR open with auto-merge disabled until approval is recorded in the PR's "Human document review" section. Unchanged ADR/spec references alone do not trigger the gate.
- When criteria are met, findings are resolved and applicable checks pass, the orchestrator merges with the shared account, using the reviewed head SHA as the precondition. Merge by squash, so history stays one commit per slice, and write a concise final message that follows the commit convention above with the issue references. Confirm GitHub reports the PR merged and close the completed issue. If GitHub blocks the merge, report the actual blocker.
- Fix ordinary failures and missing evidence autonomously. Escalate only a blocker that cannot be resolved within scope or needs human action, stating what is needed, and continue unaffected slices. A follow-up issue needs an origin link, a clear remaining outcome and acceptance criteria. It never excuses unmet agreed behavior.
- Issues, commits and PRs are the durable work log. Record assignments, material blockers, review and fix results, and the merge there, not only in agent messages. Before creating or retrying anything, check GitHub's actual state and resume the existing issue, branch, PR and worktree instead of duplicating them.
- At completion, record the outcome, tested revision, verification, caveats and follow-ups on the issue/PR with secret-free evidence, attached or linked. Then the parent removes the finished worktree, verifies it is gone from `git worktree list` and the filesystem, and records the result before the final report. Do not keep worktrees or local recovery copies as the record. Before removal, confirm nothing still uses the checkout, resolve unpublished work or private files, and protect primary, pinned, shared and in-use checkouts. Use the app's archive tool for a managed worktree and `git worktree remove` for an unmanaged one. Pending review or approval means the work is unfinished. Record any cleanup blocker with its reason and owner.

## Handoff

Report the behavior changed, evidence obtained, remaining uncertainty, and any human decision still needed, in the issue and PR. Update a spec when intent changes; correct editorial errors or stale links without adding implementation history. Record useful discoveries in their durable home (spec, ADR, test, schema, configuration or memory note) and do not duplicate them elsewhere.
