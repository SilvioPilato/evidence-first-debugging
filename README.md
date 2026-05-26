# Evidence-First Debugging

A [SKILL.md](https://www.anthropic.com/news/skills) agent skill for the bugs that don't yield to a stack trace.

When you point an AI agent at a flaky, multi-layer, or already-fix-resisted bug, it has a
characteristic failure mode: it reads the code panoramically, locks onto the first plausible
cause, and produces confident fixes that don't fix anything — so you end up debugging the fixes
instead of the bug.

This skill imposes a four-phase protocol that prevents that:

1. **Build a map, don't read code.** Table the layers (app, runtime, OS, test harness,
   orchestration) before opening a single source file.
2. **Find the missing layers.** Your first map inherited the bug report's framing. Look below,
   above, and parallel to what you listed.
3. **Run discriminating experiments.** Experiments that *rule out a whole family* of causes —
   with the expected outcome under each hypothesis stated before you run. Report raw evidence,
   not interpretation.
4. **Narrow, scoped reading.** Only now read source — a named file, a named function, quoted
   with line numbers.

**Core principle: evidence before diagnosis, diagnosis before fix. The map is more valuable than the code.**

## When to use it

Use it when **two or more** hold: the symptom is indirect, multiple layers could be responsible,
it's flaky/intermittent, or previous fixes failed (the strongest signal). It is *not* for a clear
stack trace pointing at one file — that overhead isn't worth it.

## Install

```bash
# skills.sh
npx skills add <owner>/evidence-first-debugging

# agentskill.sh
ags install evidence-first-debugging
```

Or drop the folder into your agent's skills directory (e.g. `.claude/skills/evidence-first-debugging/`).

## What's in here

| File | Purpose |
|------|---------|
| `SKILL.md` | The protocol: the four phases, interaction rules, red flags, common rationalizations. |
| `patterns.md` | The two-bug pattern, falsify-before-you-fix, what to do when a fix fails, when to stop a session. |
| `prompts.md` | Copy-paste prompt templates for each phase — for when you're the human directing an agent. |

## License

MIT — see [LICENSE](LICENSE).
