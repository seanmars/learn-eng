---
name: "TOSPEC: Apply (TDD)"
description: Implement tasks from a change test-first (red → green)
category: Workflow
tags: [workflow, tdd, implementation]
---

Implement a change's tasks test-first (red → green), one slice at a time, branching by schema — ready for `tospec-archive` once the tasks are done and the tests are green. The two-axis review is opt-in: it runs only if the user asks for it.

**Input**: Optionally specify a change name after `tospec-apply-with-tdd` (e.g., `tospec-apply-with-tdd add-auth`). If omitted, check if it can be inferred from conversation context. If vague or ambiguous you MUST prompt for available changes.

**Task tracking is file-based only.** The tracks file's checkboxes (`- [ ]` / `- [x]`) are the single source of truth for progress. Never use a built-in todo or task-tracking tool — editing the checkbox in the file is the only way progress gets recorded.

**Steps**

1. **Select the change**

   If the user named one, use it. Otherwise infer it from the conversation. If it is still ambiguous, run `tospec list --json` and let the user pick with `AskUserQuestion` — never guess.

   Announce `Using change: <name>` and how to override.

2. **Get apply instructions**
   ```bash
   tospec instructions apply --change "<name>" --json
   ```
   One call gives you the gate and the briefing: `schemaName` (decides which branch below applies), `contextFiles` (artifact id -> concrete paths), `progress`, `tasks`, and a dynamic `instruction`.

   Act on `state`:
   - `blocked` — required artifacts or the tracks file are missing; show `instruction` and STOP, sending the user back to `tospec-propose`/`tospec-issue`.
   - `all_done` — everything is checked off already; say so and stop. A change being complete is not by itself a reason to review it.
   - `ready` — proceed.

3. **Read the context files**

   Read every path listed under `contextFiles` before writing any code. Take the ids and paths from the CLI output — never assume filenames.

4. **Show current progress**

   Schema in use, `N/M tasks complete`, the remaining items, and the CLI's `instruction`.

5. **Work the tracks file one item at a time**

   The tracks file is `tasks.md` (sdd) or `task.md` (issue). Take the next unchecked item, implement it, check it off (`- [ ]` → `- [x]`) only once its test is green, then move to the next. Never batch multiple items into one pass.

   Before starting an item, re-read the parts of design/specs that cover its scope — don't rely on memory from earlier in the conversation, context may have been compressed since.

   Before checking an item off, re-read its description and confirm every part of it is covered by your changes and the behavior it describes actually works. An item checked off half-finished is the most expensive mistake in this loop.

   **Pause if** the item is unclear, implementation reveals a design issue (suggest `tospec-update`), or you hit an error or blocker — report it and wait for guidance. Don't guess.

   **Pause too if** the item turns out to need work beyond what the specs and the tracks file describe, or you catch yourself wanting to drop, narrow, defer, or make an exception to specified behavior so the item fits — say what the extra scope is and ask. Absorbing it silently is worse than the delay: it turns "done" into a claim the user has no way to check.

### sdd branch — TDD per slice

For each tracer-bullet slice in `tasks.md`:
1. Write the failing test first (red) — **only at the seam `design.md` named**. Don't test internals, don't add implementation-coupled assertions (mocking internal collaborators, reaching into private state, checking the database instead of the interface).
2. Implement the minimum to pass (green). Don't anticipate future slices or add speculative flexibility.
3. Verify the result matches the behavior the slice's specs describe.
4. Check off the slice, then move to the next one — one slice per cycle. Refactoring is not part of this loop; raise it separately once the slice is green.

### issue branch — three-way test split

Read `task.md`'s feedback loop. Then:
- **Existing test, failing** → the regression test from diagnosis already goes red. Find out why it fails and fix root cause (grep every caller of the function you're touching — the bug may hit more than the one path the ticket names).
- **Existing test, passing** → the bug is an edge case the existing test doesn't cover. Add a new failing test for that edge case first, then fix.
- **No existing test** → write the red regression test from `task.md`'s feedback loop first, then fix.

Fix root cause, not symptom — if the same defect is reachable through multiple callers of a shared function, the fix belongs in the shared function, not patched into just the caller the ticket named.

6. **Re-validate artifacts**
   ```bash
   tospec validate "<name>" --json
   ```
   Implementation sometimes touches tasks.md/specs directly (adding a discovered edge case, correcting an estimate) — confirm the change's artifacts are still structurally valid before handing off. Fix and re-run until it passes.

## Verify (opt-in — only when the user asks)

The two-axis review does **not** run on its own. Apply finishes on the full test suite and `tospec validate`; a change with those green is implemented, review or no review.

When the user asks for a review, run the methodology below in full. Don't offer it unprompted either — no "shall I review this?" at the end of a run. If a review does run, the change stays unfinished — and must not go to `tospec-archive` — until it comes back clean.

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

**Output**

Report which tasks/checkboxes completed this pass, current progress (`tospec status --change "<name>"`), and whether all required artifacts are checked off. When everything is done and the test suite is green, tell the user the change is ready for `tospec-archive`. If they asked for a review and it surfaced findings it defines as blocking, keep working here to resolve them.

**Guardrails**
- Red before green, always — never write the implementation before its failing test
- One slice/task at a time — no batching
- Keep going through items until done or blocked; pause on errors, blockers, or unclear requirements — don't guess
- Read the context files before starting, and use the paths from `tospec instructions apply` — don't assume filenames
- Check the box in the tracks file immediately after finishing each item; no external task tracker
- Keep code changes minimal and scoped to the item at hand — but never buy that scope by narrowing, deferring, or simplifying away behavior the specs call for; surface it and pause instead
- Any test you do write goes at the pre-agreed seam (design.md for sdd; the existing/new seam identified for issue) — never against internals
- Fix root cause: check every caller of a shared function before declaring a fix complete
- Don't hand off to archive until the full test suite is green — and, when a review was requested, until it comes back clean
