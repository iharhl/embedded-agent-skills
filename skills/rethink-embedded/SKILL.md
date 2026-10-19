---
name: rethink-embedded
description: Critically evaluates a requested firmware feature, sub-feature, or change before any code is written. Recovers the core idea behind the code it touches, checks whether the request makes sense, fits the existing logic, and whether that logic is sound, then weighs simpler alternatives, including not doing it. Use when the user asks to rethink or sanity-check a change, re-evaluate a feature, asks "is this over-engineered", "step back", "does this still make sense", or says rethink-embedded.
disable-model-invocation: true
---

# Rethink (embedded)

Step back before building. Features grow by accretion: a good core idea, then
a flag, a mode, a retry, a callback, a special case for the special case. Each
step looked reasonable from the one before it. Your job is to look at the
request from the core idea, not from the last step, and give an honest
verdict before anything is implemented.

Read-only. Do not edit firmware, create commits, or start implementing. The
output is an assessment and a recommendation.

## Stance

- Treat earlier decisions in this conversation as evidence, not as settled.
  Re-questioning them is the point of this skill.
- Do not soften a verdict to match the user's enthusiasm or the momentum of
  the session. "Don't build this" and "fix the base first" are valid answers.
- Do not be contrarian for show. If the request is sound and fits, say so in
  a few lines and stop. Do not invent problems to look critical.
- Prefer subtraction. The best change often removes code, a mode, or a layer.
- Be specific. Every criticism points at a file, function, flag, or state, and
  says what it costs.

## Infer the project

1. Read `CLAUDE.md`, `AGENTS.md` and `.cursor/rules/` if they exist. Follow
   those over this skill when they conflict.
2. Skim `README.md` and `docs/` if present.
3. Keep the datasheet or reference manual in reach when the request touches
   silicon. A peripheral feature can replace a software layer.

Look facts up yourself. Ask the user only for intent the repo cannot tell you.

## 1. Pin down the request

Restate the request in one or two sentences: what the user wants to change and
what problem it is supposed to solve. If the problem is not stated, infer it
and mark it as inferred.

Separate the **problem** from the **proposed solution**. Users often arrive
with a solution. Evaluate the problem first.

## 2. Recover the core idea

Find the code the request touches and work out what it was originally for.

1. Read the module, its callers, and its header contract.
2. Read history where it helps: `git log --follow` on the main files,
   `git log -L` on the key function, the commit that introduced it, and
   recent commits in this area. Note what was layered on and why.
3. Read the conversation so far: what was added in this session on top of
   the original design.

Write the core idea as one sentence: the job this code does for the product.
If you cannot write that sentence, that is itself a finding.

Then list the layers that have grown on top of it, oldest first, one line
each with the reason it was added (from a commit, comment, or this chat).

## 3. Evaluate

Answer each question honestly. Skip one only if it truly does not apply.

### Does the request make sense?

- Does it solve a real, observed problem, or a hypothetical one ("in case",
  "might need", "for flexibility")?
- Is it treating a symptom? A retry for a bus error, a delay for a race, a
  flag to skip a broken path, a timeout to hide a missed event.
- Who needs it: a real caller, a real product requirement, or nobody yet?

### Does it fit the existing logic?

- Does it serve the core idea, or pull the module toward a second job?
- Does it need a new mode, flag, state, or branch that every other path must
  now respect?
- Does it cut across layers: application logic in a driver, driver knowledge
  in the application, new work in an ISR?
- Does it duplicate something the tree, the HAL, the RTOS, or the silicon
  already does?

### Is the existing logic good enough?

Look for over-engineering signals in the code the request touches:

- A flag, mode, or config option that is only ever set one way.
- An abstraction, vtable, or callback registry with one implementation or
  one caller.
- A generic framework built for two cases.
- States in a state machine that are unreachable or always pass straight
  through.
- Layers of recovery: a retry inside a retry, a timeout around a watchdog.
- The same state stored in two places and kept in sync.
- Features that exist to work around other features.
- Special cases that would vanish if an earlier choice were different.

If the request is hard only because the base is tangled, say so. Adding to a
tangled base makes the next change harder still.

### Embedded cost

State what the request costs, qualitatively unless you measured it:

- Flash, RAM, stack depth; a new libc pull or large buffer.
- ISR latency, critical-section length, timing and deadline risk.
- New concurrency between ISR and main or between tasks.
- Power: new wake-ups, clocks kept on, polling.
- Test surface: new states and combinations to verify on hardware.

Write byte counts only if you ran `size` or read a map file for this build.

## 4. Alternatives

Weigh these, roughly from least to most code. Drop the ones that clearly do
not apply; keep at least two real options including the request as asked.

1. **Do nothing.** The problem is rare, hypothetical, or acceptable.
2. **Remove instead.** Delete a mode, flag, or layer so the problem goes away.
3. **Change a value.** A constant, priority, buffer size, timeout, or config.
4. **Reuse.** An existing mechanism in the tree, the HAL, or the RTOS.
5. **Let the silicon do it.** DMA, timer capture/compare, hardware CRC, FIFO
   thresholds, peripheral interlocks, instead of software.
6. **Simplify first, then add.** Clean the base so the request becomes small.
7. **The request as asked.**
8. **A different design** for the same problem.

For each real option: what it is, what it costs, what it gives up.

## 5. Verdict

Pick exactly one:

- **Proceed as asked.** It makes sense and fits.
- **Proceed, adjusted.** Right goal, different shape. Say what changes.
- **Simplify first.** The base is the problem. Name what to cut before adding.
- **Do something else.** A different option solves the problem better.
- **Don't do it.** The problem does not justify the cost.

If the verdict depends on intent you could not find, ask questions and give
your recommended answer on each. Do not ask for facts the repo can answer.

## Write the result

Keep it short. Use simple words. No padding, no mannered prose.

```
Request
One or two sentences. Problem vs proposed solution. Mark inferred parts.

Core idea
One sentence. What this code is for.

How it grew
- layer - why it was added (source)
Skip if nothing has accreted.

Assessment
- Makes sense? one or two sentences
- Fits existing logic? one or two sentences
- Existing logic sound? concrete over-engineering signals with path:line,
  or "Sound."
- Cost: the embedded costs that matter here

Options
1. option - cost - what it gives up
2. ...

Verdict
<one of the five> - two or three sentences on why.

Questions
Only if the verdict depends on them. Number each, give a recommended answer.

Next step
One line: /grilling-embedded to settle the design, /implement-embedded if the
plan is already settled, /code-simplification-embedded to clean the base
first, or stop.
```

Do not start implementing. Wait for the user to accept, change, or reject
the verdict.
