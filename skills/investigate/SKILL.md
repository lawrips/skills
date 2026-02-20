---
name: investigate
description: Debugging methodology that prevents speculation and enforces disciplined investigation. Use when tackling bugs, unexpected behavior, or broken features.
---

# Debugging

## Before Anything

**You are not allowed to edit code until you understand the problem.** No speculating, no "quick fixes," no theorizing about race conditions.

## Step 1: Orient

Read the affected code path. Not a deep investigation — just enough to understand what you're looking at, the flow, and the key state involved.

## Step 2: Ask

Use the AskUserQuestion tool to ask these three questions in a single call before proceeding:

1. **Do you have a repro?** (header: "Repro", single select)
   - No
   - Yes, consistent
   - Yes, intermittent

2. **Where should I start?** (header: "Start with", single select)
   - Code
   - Logs / error output
   - Both
   - Not sure

3. **Any additional context?** (header: "Context", single select)
   - I have a hunch where the problem is
   - I've seen a similar bug before
   - I've already tried some things
   - No, just the description above

The user can always select "Other" to provide free text on any question.

Then proceed. The principles below are your operating rules for the rest of the session.

---

## How to Investigate

- **Add logging FIRST, never speculate.** Check existing logs first. If they don't show enough, add targeted logging immediately. Don't theorize about browser versions, API compat, or environment differences. Instrument, repro, read output. Every time.
- **Start with the simplest explanation.** Check state values, feature flags, and code reachability before theorizing about race conditions or complex workflows.
- **Find a reliable repro, then simplify it.** When there is a reliable repro, debug from behavior, not code. Narrow the repro until it's minimal — that reveals the real cause.
- **Don't layer fix on fix.** When a fix doesn't work, understand WHY before adding another layer. If you're on your third patch for the same bug, you're fixing the wrong thing. You MUST STOP at this point and reengage the user to ask if this is the right approach.
- **Trace every mutation, not just final state.** For state bugs, add console.trace to the setter so every caller shows up with a stack trace. Printing the final value doesn't help when something overwrites it after your fix.

## How to Collaborate

- **User's observed reality wins over code analysis.** When the user says something is happening, don't argue it shouldn't based on code reading. There may be a second state variable, a parallel code path, or something you missed. Instrument and observe — don't theorize.
- **Ask the user what they see.** The user sees the phone, the terminal, the UX. They're a primary data source, not a last resort. One follow-up question about what's on screen often beats 4 code changes in isolation.
- **When the user remembers a similar bug, that's HIGH signal.** Their memory of "this behaves like that other bug" is usually pointing at the right area. Trust it and investigate there first.
- **Discuss before fixing.** When the user reports something broke, don't immediately edit the file. Stop, explain what you think happened, propose the fix, get agreement. Trace the full implications — "what else does this affect?"
