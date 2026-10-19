---
name: implement-embedded
description: Implements a settled firmware plan, then has a subagent review and simplify the diff against that plan. Use when the user says implement the plan, build it, or implement-embedded.
disable-model-invocation: true
---

# Implement (embedded)

Turn a settled firmware plan into code. Then spawn a subagent to review and
simplify the diff. The subagent uses your default model unless the user names
one.

## Infer the project

1. Read `CLAUDE.md`, `AGENTS.md` and `.cursor/rules/` if they exist. Follow
   those over this skill when they conflict.
2. Skim `README.md` and `docs/` if present.

## Resolve the plan

The plan is the settled design in this conversation: a grilling decision log,
an agreed write-up, or a spec the user pasted. Extras they named on this
invocation (model, files to leave alone, "keep the ISR in one function") are
hard limits.

Stop and say what is missing when:

- There is no plan, only a vague idea
- Grilling is still open (frontier not empty)
- A silicon or architecture choice the implementer would have to invent is
  still unset

Do not invent the missing decision. Point at `/grilling-embedded` when the
design is the hole.

Apply-scope is what the plan names. Everything else is out of bounds.

## Implement

Implement only the plan; build and run the tests this tree already has. If you
hit an unstated design choice, stop and report it instead of guessing.

If nothing useful was written, report that and stop.

## Subagent review and simplify

Spawn the subagent. It should not edit firmware in this step. It should inspect
the complete implementer diff and enough surrounding code.

The following context should be supplied to it:

- Goal of the implementation in one sentence.
- Scope of the implemented feature.
- Original plan (can be concise).

Subagent should follow these steps:

1. Read sibling `review-embedded/SKILL.md` if it is installed. Follow it on
   this diff. Extra focus: the diff implements the plan only.
2. Read sibling `code-simplification-embedded/SKILL.md` if it is installed.
   Run its "identify opportunities" process on **added and changed code
   only**. Do not apply edits. Drop any simplification that would change the
   plan's idea, layering, timing, footprint, or register contract.
3. If those skills are missing, still walk sanity, defects, ISR/main races,
   complexity, duplication, peripheral edges, and docs; and scan new code for
   nesting, duplication, dead wrappers, and names that hide units or
   hardware.

Return the findings to the main agent. Do not paste a full review to the
user in this step.

If there are no findings and no simplifications worth making, skip the next step.

## Fix and simplify

Critically evaluate the review findings and implement whatever is necessary.
Prepare a concise report on the review - what was addressed and what was dropped.

## Final check

Re-read the resulting diff against the plan. Confirm claimed fixes landed
and that simplifications did not change behavior. Leftover concerns or issues go
in the report.

## Report

Lead with what landed. Then:

```
Implemented
One short paragraph. What was built vs the plan.

Review
Fixed findings, or No findings.

Simplifications
What got simpler, or None. Skip anything that would have left the plan.

Leftover
Remaining issues, or None. Use review-embedded priorities if you have them.

Verify
Build and tests that ran, or why they did not.
```

Do not close by offering more work unless the user asked.
