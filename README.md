# Embedded agent skills and rules

Agent skills and standing rules for MCU firmware. Install them into a firmware repo so the agent there follows the same review, grilling, and C conventions.

## Install

Any agent that the [skills CLI](https://github.com/vercel-labs/skills) supports:

```bash
npx skills add iharhl/embedded-agent-skills
```

Pick which skills to take, and which agents to install them on. That copies each skill folder into the path that agent reads (`.claude/skills/`, `.agents/skills/`, `.opencode/skills/`, and so on).

Native plugin install ships the whole pack as a managed plugin.

**Claude Code**

```text
/plugin marketplace add iharhl/embedded-agent-skills
/plugin install embedded-agent-skills@embedded-agent-skills
```

**Codex**

```bash
codex plugin marketplace add iharhl/embedded-agent-skills
codex plugin add embedded-agent-skills@embedded-agent-skills
```

**Cursor**

Copy skills into `.cursor/skills/` and the `.mdc` files into `.cursor/rules/`. Do not paste skill bodies into rules.

```bash
mkdir -p your-firmware/.cursor/skills your-firmware/.cursor/rules
cp -R skills/. your-firmware/.cursor/skills/
cp .cursor/rules/*.mdc your-firmware/.cursor/rules/
```

**OpenCode**

Copy skills into `.opencode/skills/` (project) or `~/.config/opencode/skills/` (global).

```bash
mkdir -p your-firmware/.opencode/skills
cp -R skills/. your-firmware/.opencode/skills/
```

Agents that ignore `.cursor/rules/` need the rule bodies pasted into that project's `AGENTS.md`.

## Layout

```text
skills/                  # source of truth (all agents)
  review-embedded/
  grilling-embedded/
  code-simplification-embedded/
  memory-optimization-embedded/
.cursor/rules/           # Cursor rules (.mdc)
.claude-plugin/          # Claude Code plugin + marketplace
.codex-plugin/           # Codex plugin
.agents/plugins/         # Codex marketplace
```

Skills are slash-only: `/skill-name` in Cursor, `$skill-name` in Codex. Each skill has `agents/openai.yaml` so Codex will not attach it unless you invoke it. Before judging firmware, the skills read `CLAUDE.md`, `AGENTS.md`, and `.cursor/rules/` if those exist.

In Cursor, the `.mdc` rules apply when matching `*.c` / `*.h` files are in context. This library's own `AGENTS.md` is for maintaining the library, not firmware style.

## What is here

**Rules**

- **commenting.** `/* */` on non-trivial logic. Why, not restating the next line, not leftover fix narrative. `*.c` / `*.h`.
- **c-conventions.** Quote includes vs compiler headers, host-test stub include order, `FILENAME_H` guards, `ptr == 0`, unsigned `U`. `*.c` / `*.h`.

**Skills**

- **review-embedded.** Read-only review of a firmware diff. Sanity, defects, ISR/main races, stack/RAM/flash, duplication, peripheral edge cases, docs. Silicon-facing changes need the datasheet or reference manual. `/review-embedded` or `$review-embedded`.
- **grilling-embedded.** Architecture interview before code. Rounds of frontier questions with a recommended answer on each, then a decision log. Skips settled low-level branches and can drill into application logic. `/grilling-embedded` or `$grilling-embedded`.
- **code-simplification-embedded.** Clarify working C/C++ firmware without changing behavior, including timing, footprint, register order, and errata workarounds. `/code-simplification-embedded` or `$code-simplification-embedded`.
- **memory-optimization-embedded.** Profile the image with size/map/disassembly, apply safe RAM and Flash cuts on the requested feature, and propose neighboring or whole-image wins. `/memory-optimization-embedded` or `$memory-optimization-embedded`.

These are firmware-specific. For general agent workflow (how/why a subsystem works, review, verification), [pstack](https://github.com/cursor/plugins/tree/main/pstack) is a good companion in Cursor (`/add-plugin pstack`).

## Add a skill

1. Create `skills/my-skill/SKILL.md` with `name` and `description` frontmatter. `name` must match the folder.
2. Keep the body short. Put long tables in `references/`.
3. For slash-only skills, add `agents/openai.yaml` with `allow_implicit_invocation: false`.
4. Link it from this README.

Add a rule as `.cursor/rules/my-rule.mdc` with `description` and either `alwaysApply: true` or `globs`. See `AGENTS.md` for the split between rules and skills.

## Acknowledgements

`grilling-embedded` is adapted from Matt Pocock's [grill-me / grilling](https://github.com/mattpocock/skills/tree/main) skills.

`code-simplification-embedded` is adapted from Addy Osmani's [code-simplification](https://github.com/addyosmani/agent-skills/tree/main) skill.

## License

MIT, see `LICENSE`.
