# Verify — apply's opt-in review

Run this **when the user asks for a review**, once every task in the tracks file is checked off and `tospec validate` passes. It is not a step apply reaches on its own.

Two-axis review of the implementation, run as **independent parallel subagents** so neither pollutes or reranks the other's findings — a change can pass one axis and fail the other, and reporting them separately is what keeps that visible.

**Scope — two subagents, one pass each, and that is the whole review.** The subject under review is always the *code diff*, never another review's output. Do not spawn a third agent to grade the two axes' findings, do not re-review a report you have already written, do not audit the audit. Right-sized review beats deep review: enough eyes to catch real problems, no recursion for its own sake.

**What counts as blocking.** Exactly two kinds of finding block: a **hard violation** of a convention this repo documents, and a **spec-correctness** problem — a requirement missing, implemented wrongly, or contradicting a decision `design.md` records. Everything else — judgement calls, style preferences, and every code smell named below — is **report-only**: report it, do not fix it, and do not re-review because of it. A report-only finding is the user's call to make later, not this run's work. Leave this unstated and the safe reading is to fix everything, which makes "re-run only the affected axis" fire on every axis, every time.

**Steps**

1. **Gather context**
   ```bash
   tospec status --change "<name>" --json
   ```
   Note `schemaName` (sdd → compare against `specs/`; issue → compare against `task.md`) and the files this change touched (from `tasks.md`/`task.md`, or `git diff` against the commit before the change started).

2. **Spawn both axes in parallel** — a single message with two `Agent` tool calls, both `general-purpose`:

   **Standards axis prompt** — include the diff and the brief: "Report every place the diff violates this repo's documented conventions (cite the file/rule), plus any of Fowler's code smells you spot (Mysterious Name, Duplicated Code, Feature Envy, Data Clumps, Primitive Obsession, Repeated Switches, Shotgun Surgery, Divergent Change, Speculative Generality, Message Chains, Middle Man, Refused Bequest) — name the smell, quote the hunk, and treat documented repo conventions as overriding the baseline. Distinguish hard violations from judgement calls. Skip anything tooling already enforces."

   **Spec axis prompt** — include the diff plus, for sdd, the relevant `specs/<capability>/spec.md` delta content (`tospec show "<name>" --json`) and `design.md`'s decisions; for issue, `task.md`'s root cause and fix plan. Brief: "Report (a) requirements/task items that are missing or partially implemented; (b) behavior in the diff that wasn't asked for (scope creep); (c) requirements that look implemented but the implementation looks wrong, or that deviate from design.md's stated decisions. Quote the spec/task line for each finding."

3. **Run the full test suite.** This is a hard gate alongside both axes — a change with green axes but red tests isn't verified.

4. **Re-validate artifacts**
   ```bash
   tospec validate "<name>" --json
   ```
   If validation reports an artifact-format or structural problem, fix that artifact and re-run the command until it passes. A review is not complete while artifact validation is red.

5. **Aggregate — do not merge or rerank**

   Present the two reports under `## Standards` and `## Spec` headings. End with one line per axis: total findings and the worst issue within that axis. Do not pick a single overall verdict across axes.

   This is a **mechanical collation you do yourself** — collect, label, count. It is not another review pass and not a job for a subagent: nothing reviews these two reports.

**Verify output**

Report the two-axis result plus test suite status. If both axes are clean and tests are green, the change is verified — tell the user it's ready for `tospec-archive`. If either axis has a blocking finding or tests are red, do **not** treat apply as done: fix the issue here (this is still apply), then send **only the hunk that fixed it** back to the axis that reported it — never the whole axis, and never the other one. You are reviewing the changed code, not the previous report. A clean axis is not re-run for extra confidence.

**Verify guardrails**
- Standards and Spec axes are independent — never let one axis's findings influence the other's report
- Full test suite must be green; a clean review with red tests is not verified
- Quote the specific spec/task line or hunk for every finding — no unsupported claims
- **One review pass per axis — never review the review.** No agent audits another agent's findings, no re-reviewing a report you already wrote; a re-run after a fix sees only the hunk that fixed it, and a clean axis is not re-run for extra confidence
