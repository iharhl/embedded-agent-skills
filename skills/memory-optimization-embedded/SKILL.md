---
name: memory-optimization-embedded
description: Optimize the memory footprint of a named function, feature, or module in embedded firmware. Builds the project, profiles it, then applies safe RAM and Flash cuts in that scope and proposes neighboring or whole-image wins. Use when the user asks to shrink a driver, ISR, function, or feature, free RAM, fit an image into a smaller part, or audit memory usage of embedded C/C++ software.
disable-model-invocation: true
---

# Memory optimization for embedded software

Modifies relevant files in the apply-scope and proposes improvements to cut RAM and
Flash use. Never trade behavior for bytes: do not delete features, drop error paths,
or weaken contracts to save space. Do not introduce `malloc` to "fix" a memory problem.
Do not create commits, push branches, or delegate the run. If the user asked for an
audit only, make no edits and report findings instead.

If the user names constraints - a part to fit into, a byte budget, "keep `-Og`, I
debug this" - treat them as hard limits on every change below.

## Infer the project

Before measuring anything, learn the target and the build:

1. Read `CLAUDE.md`, `AGENTS.md` and `.cursor/rules/` if they exist. Follow those
   over this skill when they conflict.
2. Skim `README.md`, check documentation in folders like `docs/` if applicable.
3. Find the build command, the ELF and `.map` output paths, and the linker script
   (`.ld`, `.icf`, or scatter file).
4. Learn the part: architecture (AVR / Cortex-M / RISC-V / etc.),
   Flash and RAM sizes, alignment rules, whether unaligned access faults.
   Check the datasheet or reference manual when the repo has one.

## Resolve the target

Decide the apply-scope before measuring:

- If the user named a function, module, driver, or feature, that is the apply-scope.
- If they asked to fit a part, shrink the image, or audit the firmware, the
  apply-scope is the whole tree.
- If they did not specify, take the feature under discussion.

Apply only edits whose measured saving is in that scope: the feature's `.text` /
`.rodata` / `.data` / `.bss` / stack frames, or a type or buffer it owns, plus code
reachable only through it - a library function the feature alone pulls in is the
feature's footprint; code shared with other callers is not.

Everything else is propose, do not apply: neighboring modules, shared tables,
a project-wide `-Os` or libc switch, restructuring a layer that the feature only
touches. A P0 image overflow or colliding stack is still reported first, even if
it sits outside the feature.

## Profile the build

Use the project's size, map, and disassembly tools. If the build does not emit a
map, a size report, listings, or stack-usage files, run the equivalent commands on
the ELF for this pass. Put wiring that into the build under `Proposed` so later
size work and ordinary debugging do not start from a bare ELF.

Build and size the whole ELF. You cannot measure a function without that. What
changes with scope is what you are allowed to edit.

1. Build with the project's standard command. If it fails, stop and report - do not
   optimize a broken build.
2. Run the project's `size -A` on the ELF for a section breakdown - only
   the one-line `size` summary is not enough. Record `.text`, `.data`, `.bss`, and
   any stack/heap reservations the linker script carves out. `.data` is paid twice:
   Flash for the initializer, RAM at runtime.
3. Parse the map file (`nm --print-size --size-sort --radix=d` and the tail of that
   list help). List symbols that belong to the apply-scope and what it calls:
   functions and tables in `.text`/`.rodata`, variables and buffers in `.data`/`.bss`.
   Then glance at the global top consumers so neighboring or whole-image wins are
   not missed. Note object files or libraries that are surprisingly large, and
   sections kept despite being unused - a sign `--gc-sections` is missing.
4. Disassemble the biggest functions in the apply-scope (`objdump -d -S`). Look for
   literal pools, register spills, and heavyweight library calls - floating-point
   formatting in `printf` is the classic kilobyte sink. Glance at the global giants
   only to fill proposals.
5. If the build can emit `-fstack-usage`, read the `.su` files for frames in the
   apply-scope. A large frame is not necessarily a defect - judge frames against the
   reserved stack and the worst-case call path, never against an invented cutoff.
6. Read the source behind the apply-scope's biggest consumers before proposing
   anything. Continue through that list after the first find, not just the first.
   Do not walk the rest of the firmware as if every file were in play.

Numbers come from the tools, never from eyeballing source.

## Hunt for the waste

These are starting points, not a checklist. Skip what does not apply. The list is
not complete: if the size report, the map, or the source shows another way to
save bytes, pursue that too. Measure it. Apply it only if it is in the
apply-scope; otherwise put it under `Proposed`. Glance at the rest of the image
only to fill proposals. Then apply any extra focus the user named.

1. Is never-written data sitting in RAM? Without `const` it goes in `.data`: a
   Flash copy to hold the initializer, then RAM at runtime. Mark it `const`
   (C++: `constexpr` / `consteval`) so it stays in Flash (`.rodata`). On Harvard
   parts (AVR and similar), `const` is not enough - use `PROGMEM` / `__flash` and
   the matching read helpers, or the bytes still land in RAM.
2. Are types right-sized? `int` counters that fit `uint8_t`, `bool` arrays that are
   really bitmasks, 32-bit indexes into a 200-byte table.
3. Are structs padded needlessly? Sorting members largest-alignment-first beats
   `__attribute__((packed))`, which can fault on unaligned access. Reorder first;
   pack only with a comment and a reason.
4. Are static buffers sized to need? Evidence required: protocol limits,
   datasheet FIFO depths, comments, call sites. A hunch is not a bound.
5. Is the RAM placement load-bearing? DMA buffers in non-cacheable regions, retention
   RAM for standby, relocated vector tables, RAM-run functions. Moving or merging
   these saves bytes and breaks silicon - check the reference manual before touching.
6. Is dead code being linked? `-ffunction-sections -fdata-sections` with
   `-Wl,--gc-sections`, LTO where the toolchain supports it, and library hygiene:
   newlib-nano over newlib, no `%f` in printf, exceptions and RTTI off unless used.
7. Is the optimization level right for size? `-Os` (or `-Oz` where the compiler has
   it). Check the current flags first; a deliberate `-O0`/`-Og` debug config is a
   constraint, not an oversight.
8. Is the stack understood? Worst-case path, recursion, big local arrays, pass-by-value
   of fat structs. On an RTOS there are many reserved stacks, not one. A stack overflow
   is a memory bug that lands far from its cause - use `-fstack-usage` or the call
   graph, and check the reserved size against reality. Do not make a local mutable
   `static` just to shrink the stack - a local is one copy per call while `static`
   is one buffer for the whole program. A second entry (another caller, recursion,
   an ISR) overwrites the first call's data. Propose that move; apply it only if the
   user asked and the function cannot overlap with itself.
9. Is there a heap? Do not add one. If one already exists, who allocates, from which
   context (task vs ISR), what happens on OOM, and whether anything is freed. Follow
   the project's allocator rules if they already allow one. Do not rip `malloc` out
   unless the user asked.
10. Does the feature's own control flow bloat the size? Duplicate functions the
    linker kept, two buffers for one frame, a table that should be a few lines
    (or the reverse), a library call this path does not need. Same contract,
    fewer bytes. Glance at neighbors only to propose. This is not a
    simplify-the-logic pass.

## Apply only what is safe

Apply an optimization only when all of these are true:

1. The waste is real and visible in the map or size output, not guessed.
2. The measured saving is in the apply-scope.
3. The fix preserves behavior: same outputs, same peripheral sequences, same memory
   contracts.
4. The build is clean afterward - no new warnings.
5. The saving is measured after rebuild. If it saved nothing - revert it.
6. The change is one relatively small reviewable edit. If the fix is more intrusive -
   control flow, new layering, a diff a reviewer would have to sit with - do not apply
   it. Put an entry under `Proposed` with the approach, the map numbers you have, and
   both sides of the tradeoff.
7. It is not mixed with a feature or a behavior change in the same edit.

Work from least risk to most:

1. Build and link flags - no code changes, usually the biggest single win. Apply
   them when the measured saving is in the apply-scope. A project-wide policy
   switch (`-Os` vs a deliberate `-Og`, newlib-nano) goes under `Proposed` unless
   the user asked to shrink the whole image.
2. `const` / Flash migration of data the apply-scope owns.
3. Pass large structs by `const` pointer (or C++ reference) instead of by value, when
   that is the measured cost.
4. Buffer right-sizing, with the bound cited in the report entry.
5. Type narrowing and bitmask conversions.
6. Struct member reordering.
7. In-scope logic cuts that keep the contract: collapse duplicate functions, share
   a buffer the feature owns twice, drop a path that only exists to hold RAM.
   Revert if `size` does not move. A neighboring restructure, a new shared layer,
   or anything that changes timing or protocol goes under `Proposed`.
8. Anything touching ISRs, DMA, boot, or moving a mutable stack buffer into static
   storage: propose it, do not apply it, unless the user asked for exactly that -
   even when it sits inside the apply-scope.

## Verify and report

Rebuild, re-run size evaluation, compare against the baseline. Lead with the
totals (bytes of RAM and Flash saved, baseline and final), then one entry per
applied change. Applied entries are in-scope only.

```
Imperative change title - path/to/file.c:line

What changed
Plain language. What moved, shrank, reordered, or got deleted.

Why it is safe
One or two sentences. Cite the bound or contract that holds behavior:
the protocol limit behind a smaller buffer, the datasheet behind a
register read, the flags diff behind a smaller library.

What it saved
-N bytes in .section, from the map file - measured, not estimated.
```

In-scope waste that was not applied, and is not already under `Proposed`, goes
under `Found, not fixed`, one entry each, tagged `[P0]`-`[P3]`, with what is
wasted, why it matters, and what to do - naming both sides of any tradeoff.
Too-large diffs, ISR/DMA/boot, stack-to-static, and out-of-scope wins belong in
`Proposed`, not here.

Out-of-scope waste, and in-scope fixes that are too large to apply unasked, go
under `Proposed`: a neighboring module, shared table, a project-wide `-Os` or libc
switch, a restructure that would help this feature, a missing size, map, listing, or
stack-usage step in the build, or an in-scope cut whose diff would not read at a
glance. Same `[P0]`-`[P3]` tags, what is wasted, why it matters, and what to do.
Cite map numbers when you have them.

Priorities for those findings:

- `P0`: the image does not fit - link error, region overflow, or a stack that
  provably collides. Report it first and immediately, even when it sits outside
  the apply-scope.
- `P1`: firmware fits but sits within a few percent of a limit; the next feature
  breaks the build.
- `P2`: real waste with a known fix that needs an author decision - shrinking needs
  a product call, or removal touches a feature.
- `P3`: worth fixing: minor waste, or docs missing on a memory contract like an
  undocumented buffer bound.

Map severity by bytes and headroom, not by category. A missing `const` is `P3`
unless that table is what keeps the image from fitting. Six bytes saved by packing
a struct is not worth an unaligned fault.

If nothing in the apply-scope can be safely changed, write `No safe changes.` and
give the audit as findings and proposals instead. Do not invent churn to fill the
report.

After the changes (or after `No safe changes.`), add a short overall:

- One sentence per hunt that had a signal, including extras you tried. Skip what
  had nothing useful to say.
- The named feature (or inferred apply-scope): whether it was the thing that moved.
- Extra focus the user asked for, even if it came up empty.
- Leftover risk: unknown worst-case stack, heap fragmentation, regions now near a
  limit, memory contracts that need re-documenting.

Do not report savings that were not measured, and do not close by offering more
work unless the user asked. `Proposed` is the place for those wins, not a chat
closer.
