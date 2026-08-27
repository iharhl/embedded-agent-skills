---
name: review-embedded
description: Review changes in embedded firmware projects. Checks sanity and defects, complexity tradeoffs, duplication, edge cases, and documentation. Use when the user asks for a firmware review, or a review of embedded C/C++.
disable-model-invocation: true
---

# Review (embedded)

Read-only review. Do not modify files, create commits, push branches, post review
comments, or delegate the review to another agent.

If the user names extra things to focus on, treat those as additional required checks
after the questions below.

## Infer the project

Before judging the diff, learn the chip and the constraints from the repo:

1. Read `CLAUDE.md`, `AGENTS.md` and `.cursor/rules/` if they exist.
2. Skim `README.md`, check documentation in folders like `docs/` if applicable.

## Review the change

1. Resolve the target: uncommitted diff, a commit, or the merge that would land
   (merge-base against the base branch, not a raw branch-tip diff).
2. Inspect the complete diff and enough surrounding code to understand each
   changed path.
3. Identify problems introduced by the change - defects and the design issues in the
   questions below. Continue through the whole diff after finding the first issue.
4. Check the relevant tests and call sites to confirm that each finding is
   real and actionable.
5. If the diff programs silicon (clocks, registers, flash, DMA, interrupts, boot
   sequences), check the datasheet or reference manual when it is available. Tests
   and call sites are not enough for that.

Walk the whole change against these questions. Then apply any extra focus the user named.

1. Is implementation sane, has no defects or potential issues? Pay special attention
   to concurrency safety: race conditions between ISRs and main code, proper use of
   `volatile`, and critical sections.
2. Is it optimal - are tradeoffs considered and is it too complex for the problem that is
   being solved, and does it respect resource limits (Stack/RAM/Flash footprint)?
3. Can it be flattened or organized in a better way (e.g., there is duplicate or
   similar logic in another method or file - can it be reconciled)?
4. Are there edge cases worth addressing (e.g., peripheral initialization order,
   clock gating, interrupt clearing, and boundary conditions on ADC/Timers)?
5. Is it well documented (in-code comments, doc files)? If it is straightforward,
   then self-documenting code is enough.

Flag an issue only when all of these are true:

1. It affects correctness, security, performance, or maintainability in a meaningful way.
2. It is discrete and actionable.
3. It was introduced by the reviewed change.
4. The affected scenario or call path can be demonstrated from the code.
5. The author would probably fix it if they knew about it.

Complexity, duplication, and missing hardware contracts count as maintainability when
the author would likely change them. Do not file style nits the project's rules already
cover unless they hide a hardware contract.

## Write the result

Lead with findings, highest severity first. One entry per issue:

```
[P1] Imperative finding title - path/to/file.c:line

What goes wrong
Plain language. Who hits this, when, and what the code does instead of
what it should do.

Why it matters
One or two sentences. Tie it to a real effect: wrong behavior, bricked
update, stack smash, silent data loss, extra flash use, or a contract
the next reader cannot see.

What to do
The smallest concrete fix, or a clear question if a tradeoff needs an
author decision. Name both sides of a tradeoff when there is one.
```

Keep the cited range small and overlapping the reviewed diff. Use simple
words. Prefer a short scenario over jargon. Do not pad with extra headings
or severity lectures.

Priorities:

- `P0`: release blocker or likely brick / data loss / memory corruption / hard fault.
- `P1`: urgent defect; should be fixed next.
- `P2`: real defect, missing edge case, or complexity/duplication that
will cause mistakes.
- `P3`: worth fixing: weak docs on a non-obvious contract, a smaller
structure issue, or a low-impact edge case.

Map the five questions onto severity by impact, not by category. A missing
comment is `P3` unless it hides a flash/boot/ISR contract. Duplication is
`P3` unless it has already drifted or is likely to.

If nothing qualifies, write `No findings.` Do not invent an issue to fill
the list.

After findings (or after `No findings.`), add a short overall:

- One sentence per review question that had a signal. Skip questions with
nothing useful to say.
- Extra focus the user asked for, even if clean.
- Material test gaps or leftover risk.

Do not close by offering to patch unless the user asked.
