---
name: brainstorming
description: Use only when the user explicitly says "brainstorm" or "brainstorming", or directly invokes `$brainstorming`. Do not auto-apply this skill to general product questions, design questions, feature planning, implementation planning, or ordinary discussion even if the topic is exploratory.
---

# Brainstorming

Help turn ideas into concrete designs through natural dialogue, grounded in the current repo.

This skill assumes brainstorming happens inside an existing project. Context matters. Do not brainstorm in a vacuum when ticket history, recent commits, or current implementation already tell part of the story.

This skill is opt-in. Do not use it just because the user is discussing a feature, asking what to build, or asking a product or design question. Use it only when brainstorming is explicitly requested.

## Principles

- start with current project context, not a blank page
- one focused question at a time
- stay in discovery long enough to understand the real problem before recommending
- make success and failure explicit
- prioritize ruthlessly
- prefer simpler designs over more moving parts
- propose multiple approaches before settling
- pressure-test the recommended approach before moving on
- validate the design incrementally
- summarize current understanding as you go
- end in tickets

## Context First

Start in this order:

1. Run `tkt workflow` first to load the project's ticket workflow and conventions.
2. Search for relevant tickets on the topic:
   - matching feature names
   - related bugs
   - existing design or epic tickets
   - previously closed attempts in the same area
3. Read the most relevant ticket context:
   - current ticket if one exists
   - parent epic or design ticket
   - linked or related tickets when relevant
4. Read linked commits before broad code exploration:
   - if tickets reference commit IDs, inspect those commits first
   - use them to understand what changed recently and where the design is heading
5. Read code only where relevant:
   - files referenced by tickets or commits
   - current implementation surfaces that the idea would affect
   - avoid broad repo exploration unless the direction is still unclear

Before asking design questions, briefly summarize:
- what exists now
- what related work already happened
- what constraints or patterns seem to matter

## Conversation Flow

### 1. Understand the idea

Ask one focused question at a time.

Do not jump to a recommendation until you understand:
- who this is for
- what problem it solves
- what constraints matter
- what success looks like
- what failure would look like
- what should explicitly stay out of scope

After each user answer:
- briefly restate your current read of the problem
- identify the single biggest remaining unknown
- ask the next focused question

Prioritize:
- who is this for
- what problem it solves
- what constraints matter
- what success looks like
- what should explicitly stay out of scope

### 2. Explore approaches

Once the problem is clear:
- propose 2-3 approaches
- explain the tradeoffs
- lead with your recommended option and why
- keep the options compact and directly comparable
- call out what each option optimizes for

Do not stop after the first strong recommendation.
Pressure-test it with at least one follow-up question or tradeoff check before moving into the design.

Do not jump into implementation details too early if the product direction is still unsettled.

### 3. Present the design

When the direction is clear:
- present the design in small sections
- keep each section tight enough to react to easily
- validate after each section before continuing

Cover the parts that matter for the idea at hand, for example:
- architecture
- user flow
- component boundaries
- data flow
- edge cases
- rollout or migration considerations
- testing

Go back and clarify if something stops making sense.

Use a simple cadence:
1. current read
2. next question or option set
3. recommendation
4. section-by-section design

Do not collapse straight from "here's my recommendation" to a full design without giving the user a chance to redirect.

## After the Design

This workflow ends in tickets.

Once the design is settled:
- create or update the relevant tickets
- follow the `tkt workflow` instructions rather than inventing a separate ticketing flow
- make sure the resulting tickets reflect the agreed scope, constraints, and sequencing
