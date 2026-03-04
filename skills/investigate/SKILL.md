---
name: investigate
description: Debugging methodology that prevents speculation and enforces disciplined investigation. Use when tackling bugs, unexpected behavior, or broken features.
disable-model-invocation: true
---

# Debugging

## Principles

These are your operating rules for the rest of the session. Internalize them before doing anything.

### How to Investigate

- **Add logging FIRST, never speculate.** Check existing logs first. If they don't show enough, add targeted logging immediately. Don't theorize about browser versions, API compat, or environment differences. Instrument, repro, read output. Every time.
- **Instrument the full path, not just the convenient layer.** These logs are temporary — they exist to prove or disprove hypotheses, and they'll be removed once the root cause is found. So over-instrument. If a request flows through multiple layers, log every layer. If it's state, log every mutation. The log you skip is usually the one that would have revealed the root cause.
- **Start with the simplest explanation.** Check state values, feature flags, and code reachability before theorizing about race conditions or complex workflows.
- **Find a reliable repro, then simplify it.** When there is a reliable repro, debug from behavior, not code. Narrow the repro until it's minimal — that reveals the real cause.
- **Don't layer fix on fix.** When a fix doesn't work, understand WHY before adding another layer. If you're on your third patch for the same bug, you're fixing the wrong thing. You MUST STOP at this point and reengage the user to ask if this is the right approach.
- **Trace every mutation, not just final state.** For state bugs, add console.trace to the setter so every caller shows up with a stack trace. Printing the final value doesn't help when something overwrites it after your fix.

### How to Collaborate

- **User's observed reality wins over code analysis.** When the user says something is happening, don't argue it shouldn't based on code reading. There may be a second state variable, a parallel code path, or something you missed. Instrument and observe — don't theorize.
- **Ask the user what they see.** The user sees the phone, the terminal, the UX. They're a primary data source, not a last resort. One follow-up question about what's on screen often beats 4 code changes in isolation.
- **When the user remembers a similar bug, that's HIGH signal.** Their memory of "this behaves like that other bug" is usually pointing at the right area. Trust it and investigate there first.
- **Discuss before fixing.** When the user reports something broke, don't immediately edit the file. Stop, explain what you think happened, propose the fix, get agreement. Trace the full implications — "what else does this affect?"

---

## What to Do

**Do not read code, check logs, or take any action yet.** First, present the following to the user exactly as written:

> What's going on? Pick the one that best describes where you're at:
>
> 1. **Regression** — this was working before, something recently broke - lets focus on understanding what changed
> 2. **Add logs** — we've been going back and forth, stop guessing or trying to solve it through code reviews. We need to systematically instrument / add logs, repro, repeat. 
> 3. **I have a lead** — The direction we're going isn't right. I may know what this is, let me point you somewhere
> 4. **Step back** — stop everything, this isn't going to work. We need to take a step back, rethink the approach and discuss before doing anything else
> 5. **Something else** — I'll explain

Wait for the user to respond.

---

## After the User Picks

**Still do not read code, check logs, or take any action.** Based on the user's choice and any additional context they provide, do the following:

1. Briefly restate your understanding of the problem so far and the user's chosen direction
2. Propose your specific approach — what you'll look at first, what you expect to find, and what you'll do based on what you find
3. Ask the user to confirm before proceeding

Only after the user agrees do you begin. The principles above, especially those detailed in "How to Investigate" govern everything you do from this point forward.
