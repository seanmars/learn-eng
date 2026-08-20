---
name: tospec-archive
description: "Finish and archive a completed tospec change via `tospec archive`, running a spec/code sync first when the change has spec deltas. Use when the user wants to finalize an implemented change."
license: MIT
compatibility: Requires tospec CLI.
allowed-tools: "Bash(tospec:*), Read, Write, Edit, Grep, Glob, AskUserQuestion"
metadata:
  author: tospec
  generatedBy: "0.16.0"
---

Finish a completed change: run `tospec-sync` when the change has spec deltas, then `tospec archive` — the only path to a merged, archived change.

**Input**: The user's request should name the change to archive.

**Finishing a change: sync, then archive — in that order.** Do not run `tospec archive` speculatively "to see what happens"; each step below must actually hold first.

**Step 1 — run the sync, don't ask about it**

Archive merges the change's delta specs into `tospec/specs/` permanently, so if those specs have drifted from what the code actually does, archiving locks in a description that no longer matches reality. Decide **proactively** whether that risk applies here:

Take these in order; the first that matches wins.

1. **The user's request unambiguously names sync as what to skip** ("archive without sync", "skip the sync") — honor that. Skip the sync and archive **without** `--require-sync` (Step 2, not-synced branch). This is checked first on purpose: an explicit instruction outranks the default, and what was removed is the unsolicited question, not the user's authority. A request that merely contains the word "archive" (e.g. "just archive this") does not count on its own — that word is also the ordinary verb for the whole workflow, so matching on it alone risks skipping a sync the user never asked to skip.
2. **No delta specs at all** (e.g. a purely cosmetic issue fix with no behavior change) — there is no spec-vs-code drift to find. Skip straight to Step 2 and archive directly. Under a schema that requires deltas (sdd), such a change also needs `skip_specs: true` in its `.tospec.yaml` or validation will refuse it — set that once, and only when the change truly has no externally observable effect. Never write a delta spec just to get past the gate.
3. **Otherwise the change produced delta specs** (a non-empty `specs/<capability>/spec.md` under the change dir), so behavior was added or changed and drift is possible. Run the `tospec-sync` workflow now, without asking. There is no question to put: declining would mean knowingly publishing a spec that may not describe the code, so a prompt here spends the user's attention without buying a decision.

Running it unannounced is safe because of how `tospec-sync` is ordered: it judges every Requirement before writing anything, so a defect surfaces while the working tree is still untouched.

**Step 1a — read the sync's verdict and branch**

- **Every Requirement MATCH, or SPEC-UPDATED** (the spec was stale, sync corrected it to match the code and re-validated) → continue to Step 2, synced branch. **Ask nothing** — a corrected spec is the workflow working as intended, not a decision point.
- **Any CODE-BUG** (the implementation doesn't do what a Requirement describes, or was never built) → `sync-report.md` says `Conclusion: FAIL` and **no spec file was written**. This is the one place archive stops. **Ask the user exactly once:**

  > "Sync found <N> Requirement(s) the code doesn't satisfy: <names>. Fix them now, or leave it for later?"

  - **Fix now** → hand off to `tospec-apply` for the fix. Archive does not edit source itself. On return, resume `tospec-sync` **at step 4** with the verdicts it already produced — do not re-run steps 1-3. Concretely: re-judge **only the Requirements that were reported as defects** (not the whole set, and not the ones already judged MATCH), and once none of them is still a CODE-BUG, run step 4's write phase over **every** SPEC-UPDATED verdict, including the ones sync held back when it stopped. Sync then rewrites `sync-report.md` with `Conclusion: PASS`. Only then continue to Step 2 — the gate reads that report, so an un-rewritten FAIL will (correctly) refuse.
  - **Later** → stop. Do not archive; the change stays active with its spec files unmodified.

  If the re-judge finds a Requirement is *still* a CODE-BUG, do **not** put the question again — it was already answered. Report which Requirement did not come clean and stop, as for "Later". The one-question rule is per archive run, not per attempt.

- **`Conclusion: FAIL` with no CODE-BUG** — validation failed somewhere sync could not resolve: either on a spec update it had just written (the report names that Requirement) or on the final artifact check, which can fail on an artifact no Requirement touched. Either way this is an artifact problem, not a code defect: fix whatever `tospec validate --json` points at — the report names it when a spec update caused it — then re-run validation and have sync complete any remaining writes and rewrite the report to PASS. Do not archive on a FAIL report, and do not reach for `--require-sync`'s absence to route around it.

**Step 2 — archive**

Synced branch (a sync ran and `sync-report.md` says `Conclusion: PASS`):
```bash
tospec archive "<name>" --json --require-sync
```
`--require-sync` is the CLI's own machine-checkable half of the sync gate: it refuses unless `sync-report.md` exists with a `Conclusion: PASS` line (`SYNC_REPORT_MISSING` / `SYNC_REPORT_FAILED`). Since the sync just ran and passed, this succeeds for free — a CLI-enforced backstop behind the instruction-level order above.

Not-synced branch (no delta specs, or the user asked to skip):
```bash
tospec archive "<name>" --json
```
Only omit `--require-sync` on this branch; passing it without a sync having run will (correctly) refuse.

Either way, the CLI then: re-validates the delta specs (blocking), checks tasks/task completion (with `--json`, incomplete tasks block unless you pass `--yes` after the user confirms), does a two-phase atomic merge into `tospec/specs/` (dry-run + re-validate every target first, writes only if all pass), and moves the change directory — including `sync-report.md` when present — into `tospec/changes/archive/` under a `yyyyMMdd_HHmmss-<name>` timestamp the CLI generates itself.

**Reading the result**: the `--json` output is `{archive, status}`. Success → `archive` holds `{change, archivedAs, path, specsUpdated}`. Failure → `archive` is `null` and `status[0].code` tells you which ending you got: `archive_validation_failed` (fix artifacts, re-validate), `archive_tasks_incomplete` / `archive_tasks_missing` (ask the user, then rerun with `--yes` to override), `SYNC_REPORT_MISSING` / `SYNC_REPORT_FAILED` (the sync gate refused). Each status entry carries `{code, message, fix}` — follow `fix`.

**Guardrails — explicit and non-negotiable**
- Run the sync when the change has spec deltas; do not ask permission for it. Honor an explicit request to skip, and skip it outright when there are no delta specs
- A CODE-BUG is the only outcome that may interrupt — MATCH and SPEC-UPDATED continue silently. Never edit source from here; a fix goes through `tospec-apply`
- Only pass `--require-sync` when `tospec-sync` actually produced a passing `sync-report.md` this run
- **`tospec archive` owns the archive directory** — it generates the `yyyyMMdd_HHmmss-<name>` name and moves the change there. Let the command do it: the timestamp comes from internals you cannot reproduce, so a hand-built name risks a collision or a wrong sort order
- If `tospec archive` reports a validation failure or asks for confirmation on incomplete tasks, resolve the underlying issue (or explicitly confirm with the user) — do not route around it with manual file operations

**Output**

Report the archived name and path from `tospec archive`'s output, whether a sync ran and what it concluded, and **anything the sync found that no Requirement asked for** — that is reported here, never written into a spec, and never a reason to block. Confirm `tospec/specs/` now reflects the merged change.
