# Embedded agent skills and rules

Agent skills and standing rules for MCU firmware. Copy them into a firmware repo so the agent in that tree follows the same review, grilling, and C conventions.

This is a library, not a board support package. It has been used in Cursor. Paths below also match what Codex and Claude Code look for, but those have not been the daily driver here.

Copy into a firmware repo, then tailor. Put project-only bullets there (`-nostdlib`, Unity host tests, allocator `NULL` exceptions). Skills are generic on purpose; still tighten examples and skip-lists to the project's kernel and memory map.

## Layout

```text
.agents/skills/          # skills (Cursor, Claude Code, Codex)
  review-agent-embedded/
  grilling-embedded/
  code-simplification-embedded/
.cursor/rules/           # Cursor rules (.mdc); Codex does not load these
.cursor-plugin/          # optional Cursor plugin manifest
```

Skills are slash-only: `/skill-name` in Cursor, `$skill-name` in Codex. Each skill has `agents/openai.yaml` so Codex will not attach it unless you invoke it. Before judging firmware, the skills read `CLAUDE.md`, `AGENTS.md`, and `.cursor/rules/` if those exist.

In Cursor, the `.mdc` rules apply when matching `*.c` / `*.h` files are in context. Other agents do not load `.cursor/rules/`. For those, paste the rule bodies into the firmware repo's `AGENTS.md` (drop the YAML frontmatter). This library's own `AGENTS.md` is for maintaining the library, not firmware style.

## What is here

**Rules**

- **commenting.** `/* */` on non-trivial logic. Why, not restating the next line, not leftover fix narrative. `*.c` / `*.h`.
- **c-conventions.** Quote includes vs compiler headers, host-test stub include order, `FILENAME_H` guards, `ptr == 0`, unsigned `U`. `*.c` / `*.h`.

**Skills**

- **review-agent-embedded.** Read-only review of a firmware diff. Sanity, defects, ISR/main races, stack/RAM/flash, duplication, peripheral edge cases, docs. Silicon-facing changes need the datasheet or reference manual. `/review-agent-embedded` or `$review-agent-embedded`.
- **grilling-embedded.** Architecture interview before code. Rounds of frontier questions with a recommended answer on each, then a decision log. Skips settled low-level branches and can drill into application logic. `/grilling-embedded` or `$grilling-embedded`.
- **code-simplification-embedded.** Clarify working C/C++ firmware without changing behavior, including timing, footprint, register order, and errata workarounds. `/code-simplification-embedded` or `$code-simplification-embedded`.

These are firmware-specific. For general agent workflow (how/why a subsystem works, review, verification), [pstack](https://github.com/cursor/plugins/tree/main/pstack) is a good companion in Cursor (`/add-plugin pstack`).

## Use in a firmware project

Copy the pieces you want:

```text
your-firmware/
  .agents/skills/     <-- from this repo (Cursor, Codex, Claude Code)
  .cursor/rules/      <-- from this repo (Cursor only)
  AGENTS.md           <-- standing guidance for agents that ignore .cursor/rules
```

Or copy a single skill folder into `.agents/skills/<name>/`.


## Add a skill

1. Create `.agents/skills/my-skill/SKILL.md` with `name` and `description` frontmatter. `name` must match the folder.
2. Keep the body short. Put long tables in `references/`.
3. Link it from this README.

Add a rule as `.cursor/rules/my-rule.mdc` with `description` and either `alwaysApply: true` or `globs`. See `AGENTS.md` for the split between rules and skills.

## Acknowledgements

`grilling-embedded` is adapted from Matt Pocock's [grill-me / grilling](https://github.com/mattpocock/skills/tree/main) skills.
`code-simplification-embedded` is adapted from Addy Osmani's [code-simplification](https://github.com/addyosmani/agent-skills/tree/main) skill.

## License

MIT, see `LICENSE`.
