---
name: tospec-decision
description: "Record an architecture decision as a permanent ADR. Use when a design/architecture choice has been made or is ready to adopt and should be documented with its rationale, alternatives, and impact instead of left in chat."
license: MIT
compatibility: Requires tospec CLI.
allowed-tools: "Bash(tospec:*), Read, Write, Edit, Grep, Glob, AskUserQuestion"
metadata:
  author: tospec
  generatedBy: "0.16.0"
---

Turn a made (or ready-to-adopt) decision into a permanent ADR record via the `tospec decision` command.

**Input**: The user's request should describe the decision and, ideally, the reasoning and the alternatives that were considered.

Capture a decision that has already been made or is ready to be adopted — the reasoning, the alternatives weighed, and what it affects. A decision left only in chat is a decision that gets re-litigated next week.

**Steps**

1. **Check the ledger for prior decisions**
   Read `tospec/decisions/index.md` (if it exists) — its timestamp / title / summary rows are the history of what was already decided. Use it to spot a decision this one extends or supersedes, and to avoid contradicting an accepted one.

2. **Converge on the decision first — and keep the exchange**
   Before writing anything, make sure you can state: what was decided, why (the driving forces), which alternatives were evaluated, and what it impacts. If any of these is fuzzy, ask the user — don't invent them. As you interview, keep the material questions you asked and the user's answers; you'll record them in Decision Process (step 4) so the reasoning trail survives the chat.

3. **Create the record**
   ```bash
   tospec decision new "<topic>" --title "<title>" --summary "<one-line summary>" --status proposed
   ```
   Derive a kebab-case `<topic>` from the subject. The CLI produces both files: it writes `tospec/decisions/<yyyyMMdd_HHmmss>-<topic>.md` from its template (Status/Date/title pre-filled) and appends a row to `tospec/decisions/index.md` (seeding that ledger on first use). **The CLI owns the dated filename and the index — never create/rename decision files or hand-edit `index.md` yourself.** Use `--status accepted` if already agreed, or `--status superseded` (and name the successor) when replacing an older decision; pass `--date` only to backfill a past decision.

4. **Fill the sections**
   Open the created file. Six sections are required — they are the structure the schema validates against:
   - **Status** — proposed / accepted / superseded (+ Date); if superseded, name the decision that replaces it.
   - **Context** — the situation and forces that made this decision necessary.
   - **Decision** — what was chosen, the canonical terms, and why this over the alternatives.
   - **Impact** — the modules / APIs / docs / tests this touches.
   - **Alternatives** — each option considered, its trade-offs, and the conclusion.
   - **Follow-up** — the follow-up implementation items, phased if useful.

   The template carries two more that are optional:
   - **Decision Process** — the key questions you asked and the user's answers that shaped this decision. Format each pair as a `**Q:** question` line followed by the answer in the next paragraph, one pair per block. Summarize each Q&A; don't paste a full transcript. Omit the section entirely if there was no interactive discussion.
   - **Related Changes** — leave it empty. It fills in later, as changes link back via `tospec new change --decisions`.

5. **Verify**
   ```bash
   tospec decision list
   ```
   Confirm the record shows up with the right date, status, and title, and that its ledger row landed in `tospec/decisions/index.md`.

**Output**

Summarize the decision, its status, and the follow-up work. If it supersedes an older decision, state which one and flip that older file's Status to `superseded`. Report the created filename explicitly — when a change later implements this decision, that filename is what `tospec new change --decisions` records to link the change back here (the file's optional `## Related Changes` section can list those changes in return).

Then point at the next step. Write each option label from the decision you just recorded — a short, conversational phrase naming the concrete next action — then append the fixed action keyword in parentheses. The keyword is constant (`(update)`, `(propose)`/`(issue)`); the words before it are not — they must describe *this* action in *this* context, never a canned phrase, and never the raw `tospec-*` skill name. If the decision came out of grilling or exploring an **existing** change, offer **(update)** via `tospec-update` to fold it into that change's artifacts; if no change exists yet, offer **(propose)** / **(issue)** via `tospec-propose`/`tospec-issue` and carry the filename into `--decisions`.

**Guardrails**
- The decision file, its date-prefixed filename, and the ledger row are all CLI-generated — pass `--summary`, and never create/rename decision files or hand-edit `index.md` yourself; only edit the decision file's section contents
- Don't record a decision whose rationale or alternatives you can't state — clarify with the user first
- `superseded` must name its successor; the superseded record's Status must be updated too
