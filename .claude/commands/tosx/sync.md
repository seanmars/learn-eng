---
name: "TOSPEC: Sync"
description: "Sync specs against the implementation, code as source of truth"
category: Workflow
tags: [workflow, sync, archive-gate]
---

Sync `specs/` against the implementation for every Requirement this change touched — code wins on any mismatch. Run this before `tospec-archive` when the change's specs may have drifted from the code.

**Input**: The argument after `tospec-sync` is the change name to sync.

Whether a spec still matches the source code is a semantic judgment no CLI can make — that's why this is agent work, not a `tospec` subcommand. The CLI's job is the part that IS machine-checkable: the synced spec must still pass full validation, and `tospec archive --require-sync` refuses unless `sync-report.md` reports PASS.

**Steps**

1. **List the Requirements this change touches**

   For sdd: every Requirement in `specs/<capability>/spec.md` under the change directory. For issue: the Requirement(s) referenced in the ticket/`task.md`, plus any delta specs the change produced.

   **No-delta case**: an issue fix with no behavior change has no delta specs at all. Don't invent Requirements to check — instead confirm `task.md`'s described fix matches what the code actually does, and note this as the no-delta case in the report (see format below).

2. **Judge each Requirement against the actual implementation — writing nothing**

   Steps 2 and 3 are the judge phase: they produce verdicts only. **Do not edit a spec file yet, however obviously stale it looks.** Record each verdict and move on; step 4 does all the writing, once every Requirement has a verdict.

   For each Requirement, locate the corresponding code (the files apply touched are a good starting point) and judge: does the implemented behavior match what the Requirement's scenarios describe?

   **Can't find any corresponding implementation at all**: that's a CODE-BUG, same severity as a behavioral mismatch — the Requirement was agreed to and never built. Don't wave it through as MATCH.

3. **Sweep the other direction — behavior nobody asked for**

   Step 2 walks Requirements to code, so it can only find what is *missing*. It cannot see the opposite: behavior in the diff that **maps to no Requirement** at all. Walk the change's diff once and flag anything the specs never asked for.

   **Report it, never spec it.** "Code wins" settles how a Requirement is *worded*, not whether behavior *belongs* — it is not a licence to make unrequested work look agreed after the fact. Never add a Requirement to cover scope creep: tell the user what you found and let them decide to keep it, drop it, or plan it properly.

4. **Write the resolutions — code wins**

   Start this phase **only once steps 2 and 3 are complete and no Requirement is a CODE-BUG.** That gate is what makes it safe for `tospec-archive` to run this workflow without announcing it: because every defect surfaces while the judge phase is still writing nothing, walking away at this point leaves every spec file unmodified.

   - **Any CODE-BUG** (the code doesn't do what was agreed in ticket/design, or was never built): **stop before writing anything.** Do not touch the spec to paper over a bug, and do not write the SPEC-UPDATED entries you already judged — they keep until the defect is resolved. Report the mismatch and go to step 5 with `Conclusion: FAIL`.

     **Resuming after the defect is fixed** (this is how `tospec-archive` re-enters on its "fix now" branch): come back to *this step*, not to step 1. Re-judge only the Requirements that were CODE-BUG; every other verdict still stands. Once none of them is a defect any more, fall through to the bullet below and write **every** SPEC-UPDATED verdict — the ones held back here included, or their staleness ships into `tospec/specs/` on archive. Then rewrite `sync-report.md` (step 5) with the new conclusion; a report left at FAIL blocks the archive gate.
   - **No CODE-BUG**: work the SPEC-UPDATED verdicts one at a time. For each, update the delta spec's Requirement with **MODIFIED**, pasting the complete, corrected Requirement block. Then:
     ```bash
     tospec validate "<name>" --json
     ```
     Must pass before writing the next one — one write, one validate, so a failure names the Requirement that caused it instead of leaving you to bisect a bulk edit. Record the validate result in that Requirement's report entry.

     **If that validate fails**, the MODIFIED block you just wrote is malformed — an artifact problem, not a code defect. Fix that block and re-run `tospec validate` until it passes, then carry on with the remaining verdicts. Only if you cannot make it pass do you stop and report `Conclusion: FAIL`, naming the Requirement; that FAIL is not a CODE-BUG and archive handles it as an artifact fix, not a return to `tospec-apply`.

5. **Write the sync report**

   Create `sync-report.md` in the change directory in exactly this format — `tospec archive --require-sync` parses the first `Conclusion:` line, so the heading and label text must match:

   ```markdown
   # Sync Report: <change-name>
   ## Summary
   Conclusion: <PASS or FAIL>
   ## Requirements
   ### <Requirement name>
   - Implementation: <module/function description, not a hard path>
   - Verdict: MATCH | SPEC-UPDATED (updated to match code) | CODE-BUG (stops archiving)
   - Notes: <the difference and how it was handled; SPEC-UPDATED must include the validate result>
   ```

   Fill the `Conclusion:` line with exactly one literal — `Conclusion: PASS` or `Conclusion: FAIL` (FAIL may append a `(reason)` note); never leave both options in one line, or `tospec archive --require-sync` can't parse it. `Conclusion:` is **FAIL** if even one Requirement is CODE-BUG — a single unresolved defect blocks the whole change, not just that Requirement. It's **PASS** only when every Requirement resolved to MATCH or SPEC-UPDATED (with validate passing). For the no-delta case, still emit the Summary and a single Requirements entry describing the task.md-vs-code comparison (module described, not a Requirement name).

6. **Final artifact validation**
   ```bash
   tospec validate "<name>" --json
   ```
   Even when no spec needed updating, run the full change validation before finishing. If it reports an artifact problem, fix the affected artifact and re-run until it passes. Do not write `Conclusion: PASS` while validation is failing.

**Output**

Summarize how many Requirements matched, how many specs were updated, whether anything was flagged as a code defect, and anything found that no Requirement asked for.

Where that summary goes depends on who called you. **Run from `tospec-archive`** (the usual case): hand the summary straight back — archive continues on `Conclusion: PASS`, passing `--require-sync` for free, and on `Conclusion: FAIL` archive owns the single question about fixing the defect. Do not prompt for archiving yourself; you are already inside it. **Run standalone**: on `Conclusion: PASS` prompt "Run `tospec-archive` to finish up", and on `Conclusion: FAIL` prompt to return to `tospec-apply` first.

**Guardrails**
- Code is the source of truth for behavior — spec updates always follow code, never the other way around
- Scope creep is reported, never specced — "code wins" settles a Requirement's wording, never whether unrequested behavior belongs
- Never edit source code from this workflow — a code-side mismatch is diagnosis, not a license to patch here
- Judge every Requirement before writing any spec update; inside the write phase, every MODIFIED delta must re-pass `tospec validate` before writing the next one
- `sync-report.md` is required output, in the exact format above — `tospec archive --require-sync` depends on both its existence and its `Conclusion:` line
- One comparison pass per Requirement — check it against the code once and record the verdict. `sync-report.md` is the deliverable, not another review target: don't re-audit the report after writing it, and don't re-check a Requirement already judged MATCH
