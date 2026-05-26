# Evidence-First Debugging — Prompt Templates

Copy-paste prompts for each phase and interaction rule. Written for a human directing an AI agent, but the wording also works as self-instructions when you are the agent.

## Phase 1 — Build the map (no code)

> Before proposing any fix or reading code, build a map. List the layers
> involved in this bug. For each layer, tell me: what must happen for the
> system to work, what could go wrong, and what evidence would localize the
> problem to that layer. Output a table of hypotheses, not a list of files.
> Do not read source files yet.

## Phase 2 — Integrate a missing layer

> The map is missing a layer: [describe it — e.g. the test harness, the OS
> scheduler, a second replica]. Integrate it into the map. Then reconsider
> the investigation steps with this layer included. Do not just append it —
> rethink whether the previous steps still make sense.

## Phase 3 — Discriminating experiment

> Run this experiment: [specify]. Before you run it, state the expected
> outcome under each competing hypothesis. Then report ONLY the raw output —
> values, timestamps, error messages. Do not propose a diagnosis or a fix
> until I ask. After the raw report, tell me which hypotheses the evidence
> rules in or out.

## Phase 4 — Scoped reading

> Read only [file] at [function or line range]. Tell me:
> 1. [specific question]
> 2. [specific question]
> Quote the relevant lines with line numbers. Do not summarize. Do not read
> other files unless you explicitly need them to answer — in which case ask
> me first.

## Rule 2 — Force explicit uncertainty (append to any prompt where confabulation is possible)

> If you don't know with certainty, say so instead of guessing. I'd rather
> hear "I'm not sure, this needs verification against the docs/runtime" than
> a confident answer that turns out wrong.

## Rule 4 — Pause to learn

> Pause the investigation. You mentioned [concept]. Explain it specifically
> in this context — not in general. What it does here, why it matters for
> this bug, and the wrong vs right way to use it. Keep it brief. When I
> confirm I've understood, we resume.

## Rule 5 — Orthogonal fix options

> Give me [N] orthogonal fix options — genuinely different angles, not
> variants of the same approach. For each: what changes concretely, what
> trade-off it carries, and what it does NOT solve. Then recommend one and
> explain why for my specific context.
