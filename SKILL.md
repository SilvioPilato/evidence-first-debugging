---
name: evidence-first-debugging
description: Use when a bug's symptom is indirect (appears layers away from the cause), spans multiple system layers, is flaky/intermittent/timing-dependent, or when previous fix attempts failed and the investigation seems to cycle through variations of one wrong hypothesis. Covers race conditions, distributed-system bugs, platform-specific failures, and any investigation where exploring the wrong direction costs more than upfront structure. NOT for simple bugs with a clear stack trace and a single suspect file.
---

# Evidence-First Debugging (with an AI agent)

## Overview

Debugging emergent bugs with an AI agent has its own failure mode: the agent reads code panoramically, locks onto one plausible cause, and produces fixes that don't fix anything — while you debug the fixes instead of the bug.

**Core principle: evidence before diagnosis, diagnosis before fix. The map is more valuable than the code.**

**Violating the letter of these phases is violating the spirit of debugging.** Skipping a phase doesn't save time — it degenerates the next phase into the anti-pattern it was meant to prevent.

This skill is for the *agent doing the debugging*. It is also worth using when you are the human directing another agent — the prompt templates in `prompts.md` are written for that case.

**Scope:** General root-cause discipline (read the error, reproduce it, test one hypothesis at a time) handles most bugs. This skill is for the harder case: **emergent / flaky / multi-layer** bugs where the hard part is *which* of many layers and *which family* of causes — and where reading code first actively misleads you.

## The Iron Law

```
NO READING CODE BEFORE THE MAP. NO FIX BEFORE DISCRIMINATING EVIDENCE.
```

If you have not built the hypothesis map (Phase 1), you may not read source files.
If you have not run an experiment that rules out a family of causes (Phase 3), you may not propose a fix.

## When to Use

Use when **two or more** of these hold:
- Symptom is **indirect** — you see "rows missing at the end," not "the queue dropped a job."
- **Multiple layers** could be responsible (app, runtime, OS, hardware, test harness, orchestration).
- **Flaky / intermittent** — "sometimes," "20% of the time," "only in prod," "only on Windows," "no pattern."
- **Previous fixes failed** — you (or another agent) tried things and the symptom persists. This is the strongest signal.

**Do NOT use** for a clear stack trace pointing at one file, a compiler error, a feature request, or code explanation. The overhead isn't worth it. Just read the error, reproduce it, and fix it.

When in doubt, ask the user one clarifying question before activating: *"This sounds like it might benefit from the evidence-first protocol — want me to apply it, or attempt a direct fix first?"*

## The Four Phases

Sequential. Each produces an artifact the next consumes. Do not skip ahead.

### Phase 1 — Build a map, do NOT read code

Before opening **any** source file, produce a table of the layers involved. For each layer:

| Layer | What must happen for it to work | What could go wrong here | What evidence would localize the problem to this layer |
|-------|--------------------------------|--------------------------|--------------------------------------------------------|

This is a map of the **search space**, not of the code. It tells you *where reading would be worthwhile* — which you cannot know yet.

**Write the map down — even a quick three-row table — before doing anything else.** When the discriminating experiment is obvious (e.g. "loop the flaky test"), running it is legitimate as your first *evidence* step — but you still write the map first. Its real job is to surface the layers you are NOT thinking about: the test harness, a second bug, a wrong framing in the bug report. Skipping the map because the experiment is obvious is how you run a great experiment against the wrong test.

Your default is to read code. **Resist it.** Reading now produces a confident-looking diagnosis built on whichever file you happened to open first. That is the failure this phase prevents.

### Phase 2 — Find the missing layers

Your first map is incomplete — it inherited the framing of the bug report. Read it critically:
- Layers **below** what you listed? (OS primitives, runtime/memory semantics, hardware, scheduler.)
- Layers **above**? (Test harness, CI vs local, orchestration, deploy/shutdown signals.)
- **Parallel** layers touching the same data? (Other threads, other replicas, dedup logic.)

When you find one, **integrate it structurally** and re-examine whether the earlier investigation steps still make sense — don't append it as a footnote.

### Phase 3 — Discriminating experiments, not inspection

Run experiments that **separate families of hypotheses**, not ones that confirm a favorite. A discriminating experiment rules out at least one whole family.

Every experiment must specify, *before running it*, the **expected outcome under each competing hypothesis**. An experiment whose result you can't interpret in advance is useless.

- Loop the failing test 20–50× → deterministic failure rules out timing races; flaky failure rules out static logic bugs.
- Timestamped instrumentation at two boundaries → relative ordering tells you which mechanism fires first.
- Change exactly one variable (timeout, ordering primitive, platform) → tells you whether it's in the causal chain.

**Report raw evidence — values, timestamps, error text — not interpretation.** State which hypotheses the evidence rules in or out *only after* the raw report. The pull toward skipping to "so the bug is X" is constant and corrosive.

### Phase 4 — Narrow, scoped reading

Only now may you read source. Scope it explicitly: which file, which function, which lines. Forbid "reading for context." **Quote the relevant lines with line numbers — do not summarize.** A summary here hides the specific detail that makes the diagnosis precise.

## Critical Interaction Rules (all phases)

1. **Permit disagreement.** Ask "is this consistent with the evidence?" / "what would falsify this?" — not "confirm this." A precise counter-argument from the agent is often right.
2. **Force explicit uncertainty.** On platform-specific or undocumented behavior: *"If you don't know with certainty, say so instead of guessing."* This kills a large fraction of false diagnoses.
3. **Bound the scope of reading.** "Read only function X." "Don't read other files unless required — ask first." Broad reading → panoramic summaries → hidden details.
4. **Pause to learn.** When a concept appears that the human may not fully grasp, stop and explain it *in this specific context* before continuing. Don't nod through.
5. **Demand orthogonal options.** For fixes, require genuinely different angles, not incremental variants. For each: what changes, what trade-off, what it does NOT solve — then recommend one with reasoning.

## Special patterns

See `patterns.md` for the full treatment of:
- **The two-bug pattern** — when stuck despite multiple attempts, suspect Bug B poisoning the evidence for Bug A. Resolve B first.
- **Before applying a fix** — a passing fix is not a confirmed diagnosis. Split mechanism ("the cause is X") from sufficiency ("therefore Y fixes it"), state what would falsify each, and run a discriminating (often three-arm) test only when an outcome would actually change your decision.
- **When a fix fails** — a failed fix is a diagnostic event, not a setback. Do not pile on another fix; return to Phase 3 and be willing to falsify your own diagnosis.
- **When to stop a session** — signs to stop, and never stopping empty-handed (capture a structured issue + clean revert + resume pointer).

Prompt templates for every phase and rule: see `prompts.md`.

## Red Flags — STOP and return to Phase 1

If you catch yourself doing any of these, you are in the failure mode:

- Opening source files before the layer map exists ("let me just look at the code first").
- Running an experiment or reading code before the layer map is written down ("the experiment is obvious, I'll skip the map").
- Writing "Why I'm confident (not guessing)" or "this confirms the bug precisely" — **from static reading, with no runtime evidence.**
- Producing a fix or a diff before any experiment has ruled out a family of causes.
- Applying a fix to a Phase 1–4 bug without stating what would falsify it — treating "the test passed" as proof the diagnosis was right.
- Holding a single hypothesis and gathering support for it instead of evidence that would falsify it.
- Treating prior failed fixes as "wrong target" rather than as a signal of contaminated evidence or a second bug.
- Reporting a diagnosis where you were asked for raw evidence.
- Stating platform/runtime behavior as fact when you have not verified it.

**All of these mean: stop, build (or return to) the map, design a discriminating experiment.**

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "I'll read the code first to understand it, then map." | Reading first anchors you to the first file you open. The map tells you *which* file is worth reading. Map first. |
| "The architecture confirms the bug precisely." (from static reading) | Static reading shows what code *can* do, not what *did* happen. No runtime evidence = no confirmation. Run the experiment. |
| "I'm confident, not guessing." | Confidence from reading is still a hypothesis. Falsify it before you fix it. |
| "Let me just propose the fix, we can verify after." | Verifying-after means debugging your fix. The 3 fixes that already failed were proposed this way. Evidence first. |
| "The prior fixes were aimed at the wrong thing; mine is right." | Three failed fixes is a signal your *evidence* is contaminated — suspect a second bug (see patterns.md), don't just aim better. |
| "It's flaky, there may be no root cause." | 95% of "no root cause" is incomplete investigation. Loop the test 50× and instrument before concluding. |
| "Building a map is overhead, the bug is probably simple." | If it were simple it wouldn't be flaky/multi-layer/already-failed-fixes. The map costs minutes; a wrong-direction fix costs hours. |
| "I'll report the diagnosis since I already see it." | You were asked for raw evidence. Diagnosis-before-evidence is exactly the bias the phases prevent. |
| "The test passes, so the diagnosis was right." | A fix can pass by perturbing timing/layout and masking the bug — mechanism unconfirmed. Add a control arm (same perturbation, no fix); if it also passes, your fix is an accident. See `patterns.md`. |
| "The experiment is obvious, no need to write the map." | The map's job is the layers you're NOT thinking about — the wrong test, the second bug. Write it; it takes a minute and catches the framing error. |

## Real-World Impact

Developed during a real debugging session where the agent had cycled through three failed fixes on the wrong hypothesis family. Structured investigation reached a confirmed diagnosis in five focused exchanges after the agent had been stuck for hours.
