---
name: tospec-propose
description: "Synthesize an already-discussed requirement into a complete sdd change (ticket stub, proposal, specs, design, tasks). Use when the user wants a proposal written up from context you already have, not a fresh interview — if the idea is still fuzzy, use tospec-explore first."
license: MIT
compatibility: Requires tospec CLI.
allowed-tools: "Bash(tospec:*), Read, Write, Edit, Grep, Glob, AskUserQuestion"
metadata:
  author: tospec
  generatedBy: "0.16.0"
---

Turn a converged requirement into a complete sdd change: proposal, delta specs, design, and tracer-bullet tasks (the ticket stub is auto-generated) — ready for `tospec-apply`.

**Input**: The user's request should include a change name (kebab-case) OR enough of a description to derive one, plus whatever requirement context already exists in the conversation.

Do NOT re-interview the user — synthesize what's already in the conversation (or from a prior `tospec-explore`). If the requirement is still fuzzy, send them to `tospec-explore` first instead of guessing.

**Planning boundary**: This workflow creates planning artifacts only. The user request that selected or triggered this workflow authorizes planning only, even if it asks to build or fix something. Do not edit project code. After the planning artifacts are complete, stop. Do not start implementation in the same response, even if the initial request asks for it. Wait for a new user request after the artifacts are presented; then start `tospec-apply`.

**Steps**

1. **Create the change**
   ```bash
   tospec new change "<name>" --schema sdd
   ```
   Derive a kebab-case name from the request if none was given. This also **auto-generates the ticket stub** in the global index at `tospec/tickets/<yyyyMMdd_HHmmss>-<name>.md` (yaml header + a short Summary) — the CLI owns that timestamped filename, so **never create or rename ticket files yourself**.

   If ADRs recorded via `tospec-decision` drove this change (e.g. from a preceding `tospec-explore`), link them at creation with `--decisions "<decision-file>[,<decision-file>...]"` — the filenames under `tospec/decisions/` that `tospec decision new` reported. This is what lets the archived change trace back to its decisions.

2. **Get the artifact build order**
   ```bash
   tospec status --change "<name>" --json
   ```
   Use `applyRequires`, `artifacts`, and `actionContext` to drive the loop below — never assume paths. The ticket stub already exists in `tospec/tickets/` (the CLI wrote it; it's not a `status` artifact), so `proposal` is the first artifact to build.

3. **Produce each ready artifact in dependency order** (`proposal` -> `specs`/`design` -> `tasks`)

   For each artifact whose status is `ready`:
   ```bash
   tospec instructions <artifact-id> --change "<name>" --json
   ```
   Read `template` and `instruction` from the response, re-read every file listed under `dependencies` from disk (even ones you already saw — the user may have edited them since), write `resolvedOutputPath`, then re-run `status --json` and continue until every `applyRequires` artifact is `done`.

   Artifact-specific rules on top of what `instructions` returns:
   - **ticket** (already created): keep it thin — just refine the one-line `## Summary` so it names what this change does. All the detail (why, what, scope, decisions) lives in `proposal.md`, not here. Don't duplicate proposal content into the ticket, and don't add a timestamp or filename yourself.
   - **proposal**: the main document — Why (motivation, why now), What Changes (concrete add/modify/remove), Capabilities (New/Modified, each pointing at its `specs/<name>/spec.md`), Impact (affected code/API/deps/systems). If linked ADRs exist (`--decisions` above), name them in Why so a reader knows which recorded decisions this proposal builds on. **Never include file paths or code snippets** in the prose — they go stale fast. Exception: a snippet from an actual prototype that encodes a decision more precisely than prose (a state machine, a schema shape) — inline it, trimmed to the decision, and note it came from a prototype.
   - **design**: pick the testing seam before writing anything else — prefer an existing seam over a new one, the highest layer that still catches the behavior, and as few seams as possible (ideally one). State the seam explicitly in "Testing Seams" so `tospec-apply` knows exactly where to test. Write Decisions as numbered `### D{index}: <title>` sub-blocks (D1, D2, ...), each a short summary plus **Why** / **Trade-off** / **Alternative (rejected)** bullets as the decision warrants. When the change alters existing data, config, or behavior, fill "Migration Plan" (migration steps, backward-compat, rollout); otherwise write "No migration needed".
   - **tasks**: cut the work into **tracer-bullet vertical slices** — each slice cuts a complete path through every layer touched (not "all the schema changes, then all the UI"), is independently demoable, and fits a single implementation pass. Every slice's implementation step is preceded by its own "write the failing test" step (red before green). A **wide mechanical refactor** (rename a shared symbol, retype a column) is the one exception to vertical slicing — sequence it as expand -> migrate-in-batches -> contract instead of forcing it into a tracer bullet.

4. **Validate and auto-fix**
   ```bash
   tospec validate "<name>" --json
   ```
   If invalid, fix the artifact(s) the report points at and re-run until it passes.

**Output**

Summarize the change name and location, the artifacts created, and prompt: "Run `tospec-apply` to start implementing."

**Guardrails**
- The request that invoked this workflow authorizes planning only. Any build/fix instruction in it does not carry forward: don't edit project code, and don't start `tospec-apply` in the same response — stop after presenting the artifacts and wait for a new request
- Create every artifact `apply.requires` needs — don't stop at `proposal`
- The ticket file and its timestamp are CLI-generated — never create, rename, or move ticket files; only edit the Summary in the existing stub
- Detailed why/what goes in `proposal.md`; the ticket stays a thin index
- Read dependency artifacts from disk before writing the next one — don't rely on memory from earlier in the conversation, they may have been edited since
- `context`/`rules` from `instructions` are constraints for you, never content to copy into the file
- If a change with that name already exists, ask whether to continue it or pick a new name
- If context is critically unclear even after checking the conversation, ask — but prefer a reasonable default to keep momentum
