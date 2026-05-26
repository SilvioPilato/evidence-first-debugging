# Structured Debugging — Special Patterns

Load this when an investigation is stuck, when a fix has failed, or when deciding whether to end a session.

## The two-bug pattern

When an agent keeps getting stuck on a bug despite multiple attempts, the likely explanation is not incompetence — it is **two bugs**:

- **Bug A** — the one you are trying to find.
- **Bug B** — a different bug poisoning the evidence you would use to find Bug A.

Common Bug Bs:
- A test harness that leaves orphan processes on panic, holding ports / inheriting stdout.
- Logging that buffers output, so failure messages are lost.
- A flaky setup step that occasionally fails silently, masquerading as the real bug.
- An environment difference (debug vs release, CI vs local, prod vs local) that changes the symptom.

**Protocol when you suspect two bugs:**
1. Name both explicitly in the map. Don't fold them together.
2. Resolve Bug B first — even just an interim mitigation — because **every experiment you run on Bug A while Bug B is active is contaminated.**
3. Restart the Bug A investigation from clean evidence.

This prevents the most common failure: a fix that works once, then the symptom returns because the second bug was the real cause.

**Trigger:** Multiple failed fixes is the signal. Three fixes aimed at one cause that all "should have worked" is evidence the evidence itself is wrong — not that you need a fourth, better-aimed fix.

## Before applying a fix — falsify, don't just confirm

A fix that passes the test is not the same as a fix that addresses the diagnosed cause. The first is enough to ship; the second is what you actually learned. When they come apart, you ship a fix that works by accident and carry the wrong lesson into the next bug.

Before applying a fix you believe in:

1. **Split the diagnosis into two claims.** Mechanism ("the bug is caused by X") and sufficiency ("therefore Y fixes it") have different falsifiers — evaluate them separately.
   - **Mechanism** is false if the fix works for a reason unrelated to X — classically, it perturbs timing or memory layout and masks the bug. Test passes, diagnosis wrong, bug returns under other conditions.
   - **Sufficiency** is false if there is a realistic case where X *is* the cause but Y does not address it — the fix closes the case in front of you and leaves the adjacent one open.

   Write both down: "I believe the bug is X (mechanism), therefore Y fixes it (sufficiency)."

2. **State what would falsify each.** Mechanism: what would you see if the fix worked but the diagnosis were wrong — is there a "lucky fix" path? Sufficiency: which adjacent cases or hypotheses does Y not cover? Name them concretely. Articulating these is half the value; many "obvious" diagnoses are obvious only because you have not yet asked what would disprove them.

3. **Decide whether to run an experiment — not every falsifier earns one.** Judge the candidate on two axes:
   - **Informative?** At least two of its outcomes must change your next decision. A test you constructed to produce one known outcome tells you nothing you didn't already know.
   - **Cheap?** Small setup, short runtime, unambiguous result.

   Then:
   - Informative **and** cheap → run it. Canonical form: the **controlled three-arm test** (baseline / fix / control with the same perturbation but no fix) — the Heisenbug guard, isolating the causal action from the side effects of measuring.
   - Informative but expensive → weigh against the cost of being wrong. Safety-critical code: usually worth it. A dev-only test harness: usually not.
   - Cheap but uninformative (a deterministic test rigged to a single outcome) → **do not run it.** Record the falsifier as a logical inference, note the limit, move on. Running it is ritual, not science.

4. **Document what falsification did and did not cover.** In the commit or fix description: the diagnosis (mechanism and sufficiency, separately); the falsification you ran and what it rules in or out; the falsification you *skipped* and why — especially for sufficiency against an adjacent case, because that is the case a future colleague will be hitting. This turns "the fix worked" into "the fix worked, and here is what we know vs. assumed." The first is folklore that erodes; the second is durable.

**Anti-patterns this section prevents:**
- **Falsification as ritual.** Running an experiment because the playbook says to, without asking whether any outcome would change what you do. The box gets ticked; no information is gained.
- **Confirmation disguised as falsification.** An experiment whose outcome is predetermined, its predicted result then treated as evidence. If you knew the answer before running, you ran a demonstration, not a test.
- **Sufficient mechanism, false confidence.** Confirming mechanism, then assuming sufficiency follows. Right cause, missed adjacent case. Evaluate sufficiency separately.
- **All-or-nothing falsification.** Treating "I can't cheaply falsify everything" as license to falsify nothing. Falsify the cheapest, most consequential claim; document the rest as known limitation.

**When to skip this entirely.** For simple bugs with a clear cause and a local fix, this protocol is overhead. Apply it only when:
- the bug went through the four phases (it was emergent or flaky),
- the fix was non-obvious enough that "it passed" is not by itself convincing, or
- a previous fix on this investigation already failed.

Otherwise: write the fix, run the test, ship it. Falsification discipline scales with the cost of being wrong.

## When a fix fails

You completed the phases, reached a confident diagnosis, applied the fix, and it *still* fails. This is a **diagnostic event, not a setback.**

The wrong response is to apply another fix immediately. That is the loop you were escaping.

The right response:
1. **Do not modify the failed fix yet.** Leave it as-is until you understand why it failed.
2. **Return to Phase 3.** A failed fix means a hypothesis was wrong. The evidence that revealed the failure is your new starting point for discriminating experiments.
3. **Instrument the fix itself.** If it added a probe, timer, or check, verify the new code actually executes and observe what it sees.
4. **Be willing to falsify your own diagnosis.** A failed fix usually means the diagnosis was *incomplete*, not that the fix was poorly executed.

A failed fix that produces good new evidence is more valuable than a successful fix built on incomplete understanding — the latter ships a hidden bug.

## When to stop a session

Long debugging sessions have diminishing returns: good-decisions-per-hour drops, compounding-error risk rises. Knowing when to stop is part of the protocol.

Signs the session has reached its useful end:
- The root cause lives in a different system / library / scope than you started in — the next phase needs fresh context.
- You ruled out the original family of hypotheses, but the new family needs more investigation than the remaining time allows.
- A fix failed and the new evidence points to a deeper issue than this session can address.
- Your own focus has dropped — you are accepting claims more readily and skipping the "evidence before diagnosis" discipline.

**Never stop empty-handed.** Capture:
- A structured issue documenting findings: what is confirmed, what is suspected, what remains open.
- A clean revert of any in-progress fixes that did not work.
- A pointer to where the next session should resume.

A session that ends with a well-written issue contributed permanently to the project. A session that ends with broken code and an unclear mental state evaporates overnight.
