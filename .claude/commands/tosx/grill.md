---
name: "TOSPEC: Grill"
description: Pressure-test any artifact through a relentless four-quadrant Known/Unknown interrogation
category: Workflow
tags: [workflow, grill]
---

Grill any artifact the user hands you — a plan, decision, idea, or design — through the four-quadrant Known/Unknown frame until every assumption is named and the understanding is shared and confirmed.

**Input**: The argument after `tospec-grill` is whatever the user wants stress-tested — a plan file, a recorded decision, a raw idea, or a change name to interrogate in context of. May be empty, in which case resolve the target per "Pick the target first" below — never assume one.

**IMPORTANT: Grill is read-only pressure-testing, not building.** You may read files, search code, and run `tospec` read-only commands, but you must NEVER write code, edit sources, or create artifact files during a grill. Grill writes nothing itself — when the interrogation lands something worth persisting, it delegates to the skill that owns that write (`tospec-decision` for a settled decision, `tospec-update` for a change's artifacts).

This is a relentless interrogation of whatever is on the table — a plan, a decision, an idea, a design. Stress-test it until every assumption is named and every open question is either answered or explicitly parked.

**Pick the target first — never assume one**

Before asking a single question, establish *what* you are grilling. Resolve in this order:

1. **The user named it** — an explicit target in the request wins outright; use it.
2. **The conversation has an unambiguous subject** — an explore that just converged, or a proposal actively under discussion in this session. Only take this path when there is exactly one obvious candidate; a topic mentioned in passing does not count.
3. **Otherwise, ask.** Run `tospec list --json` and offer the unfinished changes (`status` is not `complete`, most recently modified first) via `AskUserQuestion`, each labelled with its name, status, and how recently it changed. **Never pick one yourself** — not even the top hit, and not even when only one candidate exists.
4. **Nothing to grill** — no explore in context and no unfinished changes: stop. Tell the user there is no subject available to dig into and suggest `tospec-explore` to start one. Do not invent a topic.

Note which kind of target you settled on — a raw idea/explore versus an existing written proposal. That distinction decides the handoff at the end.

**The four quadrants**

Sweep all four — the last two are where the real risk hides:

1. **Known Knowns — what we think we know.** Restate the claims the artifact rests on, then *verify* them. If it's checkable in the codebase, check it — never assume a "known" is true just because it was asserted. A false Known Known is the most dangerous kind.
2. **Known Unknowns — the open questions already on the table.** Enumerate them and put them to the user, each with your suggested answer and reasoning so they can just confirm.
3. **Unknown Knowns — the unspoken assumptions.** Surface the premises nobody said out loud and *name them* — the implicit "of course X" that the whole thing quietly depends on. Drag each into the light and confirm it holds.
4. **Unknown Unknowns — the blind spots.** Probe adjacent territory and reach for analogies that failed elsewhere, to unearth the questions the user didn't know to ask. Ask "what would have to be true for this to blow up?" and chase the answers.

**Interview mechanics**

- **Batch independent questions; sequence dependent ones.** Ask unrelated questions together in a single turn (use `AskUserQuestion` — up to 4 at once). A question whose answer depends on another still-open question isn't independent — hold it for a later turn, once the question it hangs off is settled.
- **Every question carries a suggested answer** and the reasoning behind it, so the user can confirm instead of writing an essay.
- **Look up what's checkable; ask what's a decision.** Factual questions ("does X already exist", "what does Y currently do") are yours to answer from the codebase — never ask the user something you can verify yourself. Decisions (behavior, scope, trade-offs) are always the user's: ask, and wait for the answer before proceeding.
- **Probe every answer for its hidden premise.** After each answer, ask what it implies that hasn't been said yet. Don't stop at the first plausible answer; keep pulling the thread until it stops giving new information.

**Converge**

You're done only when you can fully restate the shared understanding — the claims that held up, the assumptions now made explicit, the questions answered, and anything left deliberately open — and the user confirms it's right. Running out of questions is not convergence. If the user corrects your restatement, there's still a hidden branch; keep grilling.

**Hand off once the understanding is confirmed**

A grill that shifts the picture leaves work behind. Once the user confirms the restatement:

1. **Record any decision the grill produced — don't ask first.** A choice between real alternatives with lasting architecture/design impact belongs in an ADR before anything else: record it via `tospec-decision` with `--status proposed`. Don't offer this as an option — the confirmation you just got is the consent, and `proposed` is a draft status the user can still reject. Record nothing when nothing material moved; a confirmed assumption is not a decision. If you can't state the rationale and the alternatives weighed, don't invent them — ask the user, per `tospec-decision`'s own rule. Run it as a sub-step and skip `tospec-decision`'s own next-step handoff; step 2 covers it. Report the created filename.
2. **Then offer the next step,** routed by target kind. Write each option label from what this grill actually surfaced — a short, conversational phrase naming the concrete next action — then append the fixed action keyword in parentheses. The keyword is constant (`(update)`, `(propose)`/`(issue)`, `(grill)`); the words before it are not — they must describe *this* action in *this* context, never a canned phrase, and never the raw `tospec-*` skill name.
   - **Target was an existing written proposal** → **(update)** via `tospec-update` to fold the outcome into that change's artifacts. Name the change so update doesn't have to re-ask, and carry any recorded decision filename with it.
   - **Target was a raw idea or an explore** → nothing is written yet; offer **(propose)** (or **(issue)** when something is broken) via `tospec-propose`/`tospec-issue`, or **(grill)** if it's still fuzzy.

Apart from that ADR, grill writes nothing — `tospec-decision` and `tospec-update` own those files.

**Guardrails**
- **Don't act before consensus** — take no action, including any handoff, until the user has confirmed the restated understanding
- **Don't force closure** — if the user wants to keep grilling after you've restated where things stand, keep going
