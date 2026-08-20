---
name: tospec-update
description: "Update a change by revising its existing planning artifacts and keeping them coherent with one another. Use when the user wants to revise a change's plan, fold new decisions into it, or reconcile its artifacts after an edit. Never edits code."
license: MIT
compatibility: Requires tospec CLI.
allowed-tools: "Bash(tospec:*), Read, Write, Edit, Grep, Glob, AskUserQuestion"
metadata:
  author: tospec
  generatedBy: "0.16.0"
---

Revise a change's existing planning artifacts and keep them coherent. Never edit code.

**Input**: Optionally specify a change name. If omitted, check if it can be inferred from conversation context. If vague or ambiguous you MUST prompt for available changes.

**Steps**

1. **Select the change**

   If the user named one, use it. Otherwise run `tospec list --json` and let the user pick with `AskUserQuestion` — never guess, never auto-select. Offer the 3-4 most recently modified, each labelled with its name, its status (from `completedTasks`/`totalTasks`/`status`) and how recently it changed (from `lastModified`), with the most recent marked "(Recommended)".

2. **Get the change's artifacts**
   ```bash
   tospec status --change "<name>" --json
   ```
   Parse the JSON to understand current state. The response includes:
   - `schemaName`: The workflow schema being used (e.g., "sdd")
   - `artifacts`: Array of artifacts with their status ("done", "ready", "blocked", "skipped")
   - `isComplete`: Boolean indicating if all required artifacts are complete
   - `planningHome`, `changeRoot`, `artifactPaths`, and `actionContext`: path and scope context. Use these instead of assuming repo-local paths.

   The artifact ids and paths come from the active schema — never assume them, and never branch on hardcoded artifact names. Custom schemas must work unchanged.

   The files to edit are `artifactPaths.<id>.existingOutputPaths` — the concrete files that exist on disk, already glob-expanded for glob artifacts (e.g. `specs/*/spec.md`). Do NOT write to `resolvedOutputPath`: for a glob artifact it is still the glob pattern, not a real file.

3. **Understand the request**
   - If the user asked for a specific revision ("the design now uses X"), that is the starting edit.
   - If they only said "update" / "make this coherent", treat it as a coherence review: read the existing artifacts once and check them against each other for contradictions, gaps, and duplication. This is a single read-through, not a recursive audit — don't re-review your own reconciliation after making it.

4. **Read and reconcile**
   - Read the artifact(s) the request touches and the change's other existing artifacts.
   - Apply the requested edit. Then check every other existing artifact against it — in ANY direction: an edit to a later artifact may require revising an earlier one, not only the other way around. Build order is a useful reading order, not a constraint on which artifacts may be revised.
   - Note everything that is now inconsistent, missing, or contradictory.
   - Revise only files that already exist (`existingOutputPaths`). Creating an artifact that doesn't exist yet, or a new file under a glob artifact, is `tospec-propose`/`tospec-issue`'s job — note what's missing and point the user there.
   - If the change is already coherent, say so and make no edits.

5. **Confirm and apply, one artifact at a time**
   - Show each proposed revision and why. Write only after the user confirms.
   - If the user rejects a revision, leave that artifact unchanged.
   - When a substantial rewrite is needed, get that artifact's rules and template first:
     ```bash
     tospec instructions <artifact-id> --change "<name>" --json
     ```
     Read `rules` and `template` from the response.

6. **Point to the next step (guidance only - NEVER act on it)**
   Write each option label from where this change actually stands — a short, conversational phrase naming the concrete next action — then append the fixed action keyword in parentheses. The keyword is constant (`(propose)`/`(issue)`, `(apply)`, `(archive)`); the words before it are not — they must describe *this* action in *this* context, never a canned phrase, and never the raw `tospec-*` skill name.
   - Artifacts still missing → suggest **(propose)** (sdd) or **(issue)** via `tospec-propose`/`tospec-issue` to create them.
   - Change already implemented (tasks checked off / already applied) → the code may no longer match the revised plan; suggest **(apply)** via `tospec-apply`.
   - Everything done and implemented → suggest **(archive)** via `tospec-archive`.

**Output**

After each invocation, show:
- Which artifacts were revised (and which proposed revisions were rejected)
- Anything deferred to artifact creation (not-yet-created artifacts or files → `tospec-propose`/`tospec-issue`)
- Where the change stands and the recommended next command

**Guardrails**
- Planning artifacts only — NEVER edit implementation code. If the revised plan implies code changes, stop and point to `tospec-apply`.
- Use the artifact ids and paths reported by `tospec status`; never branch on hardcoded artifact names.
- Edit only the concrete files in `existingOutputPaths`; never write to a glob `resolvedOutputPath`.
- Stay behind the build frontier: revise what exists, and leave new artifacts and new glob files to `tospec-propose`/`tospec-issue`.
- Confirm every edit with the user before writing.
- If the request changes the change's *intent* rather than refining it, recommend starting fresh with `tospec-propose`/`tospec-issue` (the "Update vs. Start Fresh" heuristic).
