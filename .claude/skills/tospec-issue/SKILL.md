---
name: tospec-issue
description: "Diagnose a reported bug down to a verified root cause and turn it into an issue change (ticket stub + task). Use when the user reports something broken, throwing, or behaving unexpectedly and wants it tracked and fixed."
license: MIT
compatibility: Requires tospec CLI.
allowed-tools: "Bash, Read, Write, Edit, Grep, Glob, AskUserQuestion"
metadata:
  author: tospec
  generatedBy: "0.16.0"
---

Diagnose a bug to a verified root cause and produce an issue change (ticket stub + task) — ready for `tospec-apply`.

**Input**: The user's request should describe the symptom (what's broken) and, ideally, how to reproduce it.

Diagnose before you propose a fix. Jumping to a hypothesis before you can reproduce the bug on command is the exact failure this workflow prevents.

**Steps**

1. **Create the change**
   ```bash
   tospec new change "<name>" --schema issue
   ```
   Derive a kebab-case name from the symptom if none was given. This also **auto-generates the ticket stub** in the global index at `tospec/tickets/<yyyyMMdd_HHmmss>-<name>.md` (yaml header + a short Summary, `ref` pointing at `task.md`) — the CLI owns that timestamped filename, so **never create or rename ticket files yourself**.

2. **Refine the ticket summary**
   ```bash
   tospec status --change "<name>" --json
   ```
   The ticket stub already exists in `tospec/tickets/` (CLI-written; not a `status` artifact). Edit its one-line `## Summary` to name the symptom in a sentence. Keep it thin — the full symptom/reproduction/expected-vs-actual and the whole diagnosis go into `task.md` (step 6), not the ticket.

3. **Build a tight feedback loop before hypothesizing**

   This is the core of the workflow. Find or build **one command** that reliably goes red on this exact bug: a failing test at the right seam, a CLI invocation diffed against expected output, a curl against a running dev server — whatever's fastest. It must be:
   - **Red-capable** — actually drives the bug's code path and catches the user's exact symptom, not "runs without erroring"
   - **Deterministic** — same verdict every run (flaky bugs: raise the reproduction rate until it's debuggable)
   - **Fast** — seconds, not minutes
   - **Already run at least once** — you have pasted its invocation and output before moving on

   Do not skip this to go read code and form a theory. No red-capable command, no hypotheses.

4. **Rank 3-5 falsifiable hypotheses**

   Each hypothesis must predict something checkable: "if X is the cause, changing Y makes the bug disappear." A hypothesis you can't falsify is a vibe — sharpen it or drop it. Show the ranked list to the user before testing; they often have context that instantly re-ranks it ("we just touched that module").

5. **Verify hypotheses one at a time**

   Change one variable per probe. Prefer a debugger/REPL breakpoint over logs; if you must log, tag every debug line with a unique prefix (`[DEBUG-xxxx]`) so cleanup is one grep. Stop at the first hypothesis the loop from step 3 confirms.

6. **Write the diagnosis into task.md**
   ```bash
   tospec instructions task --change "<name>" --json
   ```
   Include: the symptom and how to reproduce it, the verified root cause (not a guess — how you verified it), why existing tests didn't already catch this, the fix plan, the test plan, and a checkbox task list for `tospec-apply`. This is where the issue's detail lives (the ticket only summarizes).

7. **Delta specs — only if behavior changes**

   `specs` is optional in the issue schema. Only produce it if the fix changes externally observable behavior (skip it and `status` will show `skipped`, not `blocked`). If the bug is the spec itself being wrong, use MODIFIED to correct the Requirement instead of writing new tests against a wrong spec.

8. **Validate and auto-fix**
   ```bash
   tospec validate "<name>" --json
   ```
   Fix whatever the report points at and re-run until it passes.

**Output**

Summarize the confirmed root cause, the fix plan, and prompt: "Run `tospec-apply` to fix it."

**Guardrails**
- The ticket file and its timestamp are CLI-generated — never create, rename, or move ticket files; only edit the Summary in the existing stub
- No hypotheses before a red-capable feedback loop exists
- 3-5 ranked, falsifiable hypotheses — never a single guess
- Root cause must be verified, not assumed — state how you confirmed it
- Don't write `specs` unless the fix actually changes external behavior
