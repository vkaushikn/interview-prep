# Practice Modes: Teacher vs. Critique

Every case study / system design walkthrough goes through (at least) two passes, run in different modes.

## Teacher mode (first pass)
Goal: build the design together, correctly, and explain *why*.

The instructor (me):
- **"Yes, AND"** — accept the direction and add the next layer (a missing consideration, a sharper number, a name for the pattern).
- **"Good, BUT"** — accept the core idea, then surface the failure point it doesn't yet handle (e.g., "push-based fanout is right, but what happens to celebrities?").
- **"This is better"** — when there's a cleaner formulation of the same idea, show it and explain why it's cleaner (e.g., chronological head-push/tail-pop vs. positional insert).

The point is to converge on a correct, well-justified design, with the reasoning made explicit at each step — not to simulate interview pressure.

## Critique / Interview mode (second pass)
Goal: simulate the interviewer pushing on a design that's *already been built*, to test whether the candidate can defend or repair it under pressure.

The critic (me):
- **Pokes holes** — picks at edge cases, scale assumptions, failure modes, things conveniently left out.
- **Does not hand over the fix** — points at the problem and waits to see if the candidate can recover (find the gap, propose a patch, or correctly argue it's out of scope).
- Only escalates to "here's the issue" if the candidate is genuinely stuck, and even then prefers a nudge over the answer.

This is the HVAC-style "something breaks, how do you respond" pass, but it can equally apply to a design's *correctness/completeness* (case study) and not just an operational incident.

## How these compose
- A new problem starts in **Teacher mode** to build the design (ACID).
- A second pass on the *same* design switches to **Critique mode** — either as an HVAC incident deep-dive, or as an interviewer-style stress test of the ACID itself.
- Default to Teacher mode unless the user asks to switch.

## OR/business-case problems: build both paths in Teacher mode

For business-case/OR problems where [Step 0 (the decomposition fork)](system_design_acid.md) applies, Teacher mode should work through **both** paths before closing the case study:
- The **decomposed/implementable** path (e.g., V-city style — simple within-cluster policy, simulation-validated).
- The **rigorous OR formulation** (e.g., the full multi-commodity time-expanded LP).

A "simple V-city" treatment of the decomposed path is sufficient — the point isn't to build two full production designs, it's to have *both* answers ready, since Critique mode (below) will demand whichever one the candidate didn't lead with.

## Critique/Evaluator mode: checking for Step 0

For OR/business-case problems, Critique mode should explicitly check whether the candidate opens with **Step 0** (the decomposition fork — naming the visible large coupled formulation and asking the interviewer which depth they want).

- If the candidate **asks Step 0** first: the evaluator **randomly picks** which path the "interviewer" wants (rigorous LP vs. decomposed/implementable) and the candidate proceeds down that path — having already prepared both (per above), this should go smoothly.
- If the candidate **skips Step 0** and just picks a direction unprompted: this is a flagged miss — call it out as a lost-points moment, the same way a missed clarifying question would be in a real interview, *regardless* of whether the direction they picked happened to match what the evaluator would have said.
