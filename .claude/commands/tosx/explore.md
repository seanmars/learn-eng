---
name: "TOSPEC: Explore"
description: Interview the user to converge a fuzzy idea or problem into a concrete requirement
category: Workflow
tags: [workflow, explore]
---

Interview the user about their idea or problem until it converges into something concrete enough to become a `tospec-propose` or `tospec-issue`.

**Input**: The argument after `tospec-explore` is whatever the user wants to think through — a vague idea, a specific problem, or a change name to explore in context of. May be empty.

**IMPORTANT: Explore mode is for thinking, not implementing.** You may read files, search code, and run `tospec` read-only commands, but you must NEVER write code or create artifact files during explore. When the requirement converges, record any material decisions via `tospec-decision` first, then hand off to `tospec-propose` or `tospec-issue` — those write the files.

This is a relentless interview, not a freeform chat. Interrogate the idea or problem until you and the user share the same understanding, walking every branch of the decision tree.

**Interview mechanics**

- **Batch independent questions; sequence dependent ones.** Ask unrelated questions together in a single turn (use `AskUserQuestion` — up to 4 at once). A question whose answer depends on another still-open question isn't independent — hold it for a later turn, once the question it hangs off is settled.
- **Every question carries a suggested answer** and the reasoning behind it, so the user can confirm instead of writing an essay.
- **Look up what's checkable; ask what's a decision.** Factual questions ("does X already exist", "what does Y currently do") are yours to answer from the codebase — never ask the user something you can verify yourself. Decisions (behavior, scope, trade-offs) are always the user's: ask, and wait for the answer before proceeding.
- **Probe every answer for its hidden premise.** After each answer, ask what it implies that hasn't been said yet. Don't stop at the first plausible answer; keep pulling the thread until it stops giving new information.

**Steps**

1. **Build known boundaries first**
   ```bash
   tospec list --json
   ```
   Read `tospec/specs/` for the capabilities this touches and `tospec/config.yaml` for project context. Do this BEFORE asking anything — questions you can answer yourself by reading the codebase are questions you should never ask the user.

2. **Work the question priority order**

   Take these in order, skipping anything the conversation or the codebase already answered:
   1. Goal and success condition — what does "done" look like?
   2. Who the user/actor is
   3. Boundaries — what's explicitly out of scope
   4. Exceptions and error paths
   5. Data and state — what persists, what's transient
   6. Unstated assumptions — the things nobody said out loud

3. **Converge**

   You're done when you can fully restate the requirement back to the user in your own words and they confirm it's correct — not when you run out of questions. If they correct your restatement, that's a sign there's still a hidden branch; keep going.

4. **Restate**

   Deliver the converged understanding back in the conversation — the problem, the decisions made (with reasoning), and anything explicitly ruled out — so `tospec-propose`/`tospec-issue` can synthesize it straight from context (that's exactly the input they expect).

5. **Record any settled decision — don't ask first**

   If the interview settled a **material** decision — a choice between real alternatives with lasting architecture/design impact that will get re-litigated if left in chat — record it now via `tospec-decision` with `--status proposed`. Don't put this to the user as an option: the restatement they just confirmed is the consent, and `proposed` is a draft status they can still reject. An answered factual lookup or a trivial preference is not material — record nothing when nothing material was settled. If you can't state the decision's rationale and the alternatives weighed, don't invent them — ask the user, per `tospec-decision`'s own rule. Run it as a sub-step and skip `tospec-decision`'s own next-step handoff; step 6 covers it. Report the created filename.

6. **Always ask for the next step — never pick it yourself**

   Once you've restated the understanding (and recorded any decision), you MUST put the next step to the user with `AskUserQuestion`. This is unconditional: apart from that ADR, explore never proceeds to the next action on its own, and never ends without asking.

   One question, always these options, with your recommended one **first** and labelled "(Recommended)". Write each option label from the situation you just explored — a short, conversational phrase naming the concrete next action — then append the fixed action keyword in parentheses. The keyword is constant (`(propose)`/`(issue)`, `(grill)`); the words before it are not — they must describe *this* action in *this* context, never a canned phrase, and never the raw `tospec-*` skill name. E.g. after settling a retry policy the first option might read "Write up the retry-limit change (propose)":
   - **(propose)** (or **(issue)** when something is broken) — start writing the proposal via `tospec-propose`/`tospec-issue`.
   - **(grill)** — keep digging with a four-quadrant deep dive on what was just explored via `tospec-grill`.

   Recommend `tospec-propose`/`tospec-issue` when the requirement is ready to build, and `tospec-grill` when it still feels fuzzy or under-tested.

   `tospec-propose`/`tospec-issue` synthesize the requirement straight from the conversation and write the change artifacts; apart from the ADR in step 5, explore persists nothing itself. Carry the recorded decision filename(s) (reported by `tospec decision new`) into that handoff: the propose/issue step passes them via `tospec new change --decisions` so the change's metadata links back to its ADRs.

**Guardrails**
- **Always end with the next-step question** — never choose the follow-up action yourself, and never finish explore without asking. Recording a settled decision (step 5) is the one exception: that one you do unasked
- **Don't implement** — never write application code or edit source files during explore
- **Record decisions before proposing** — a settled material decision gets recorded via `tospec-decision` before `tospec-propose`/`tospec-issue`, not offered as a choice; a real decision must not survive only in chat
- **Don't force convergence** — if the user wants to keep exploring after you've restated where things stand, keep going
