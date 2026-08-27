---
name: code-simplification-embedded
description: Simplifies embedded C/C++ firmware for clarity without changing behavior. Use when refactoring drivers, ISRs, HAL layers, or state machines that work but are hard to read, maintain, or extend. Use when reviewing firmware that has accumulated unnecessary complexity.
disable-model-invocation: true
---

# Code simplification (embedded)

Simplify firmware by reducing complexity while preserving exact behavior - functional and
non-functional. The goal is not fewer lines; it's code that is easier to read, understand,
modify, and debug. A refactor that produces identical outputs but grows the image past the
flash budget, deepens the stack, or pushes an ISR past its latency budget is a regression,
not a simplification.

## Infer the project

1. Read `CLAUDE.md`, `AGENTS.md` and `.cursor/rules/` if they exist. Follow those over this
   skill when they conflict.
2. Check the coding style.
3. If the code programs silicon, keep the datasheet or reference manual in reach. A cleaner
   control flow that writes registers in a new order is a behavior change.

## When to use

- Feature works, tests pass, the code is heavier than the problem.
- Review flagged code complexity, nesting, duplication, or names that hide the hardware.
- Related logic is copy-pasted across files.

**Do not use** when:
- Code is already clean and readable - don't simplify for the sake of it.
- You don't understand what the code does yet - comprehend before you simplify.
- The code is performance-critical and the "simpler" version would be measurably slower.
- You're about to rewrite the module entirely - simplifying throwaway code wastes effort.
- The code is timing-critical and hand-tuned (ISR prologues, bit-banged buses, control
  loops) - restructuring can change measured cycle counts.

## Principles

### 1. Preserve behavior

Don't change what the code does - only how it expresses it. In embedded systems the observable contract includes:

- **Functional behavior:** outputs, side effects, error paths, edge cases
- **Timing:** execution cycles, interrupt latency, determinism, jitter
- **Footprint:** flash, RAM, stack depth, memory sections
- **Hardware interaction:** register access order and side effects (some registers clear on
  read; some peripherals require exact write sequences)

If a test would need to change, you changed behavior. Revert.

### 2. Respect the hardware contract

Embedded Chesterton's Fence is stronger than the software version: the fence is often a
silicon erratum or datasheet requirement with no comment. Magic delays, dummy reads, odd
write-then-read sequences, and volatile qualifiers that "do nothing" are frequently the
most load-bearing lines in the file.

```text
BEFORE REMOVING ANYTHING "POINTLESS":
-> Check the reference manual for required sequences and timing
-> Check the silicon errata sheet for the exact part and revision
-> Check git blame - was this added after a field failure?
-> Try other compiler optimization levels
```

When you find an undocumented workaround, the simplification is often adding the missing
why-comment - not deleting the code:

```c
/* KEEP - and document if the comment is missing:
   Dummy read required: RCC_CFGR can return stale data right after
   PLL lock (silicon errata ES0xxx rev 7, ch 2.4). Do not remove. */
(void)RCC->CFGR;
```

### 3. Follow Project Conventions

Keep the code consistent with the codebase. Before simplifying study how neighboring
drivers handle similar patterns. Match project's style for register access idioms, error
handling patterns, naming conventions (prefixes, number postfix like `0U` ...), etc.

### 4. Clarity over cleverness

Explicit code is better than compact code when the compact version requires a mental pause
to parse. Named guards and `static` helpers in the same file stay cheap if the compiler can
inline them.

```c
// UNCLEAR: magic value - hard to verify this against the datasheet
UART1->CR1 = 0x202C;

// CLEAR: bit names mirror the reference manual
UART1->CR1 = USART_CR1_UE | USART_CR1_RXNEIE | USART_CR1_TE | USART_CR1_RE;
```

### 5. Maintain Balance

- Splitting hot paths into helper functions - each call adds stack depth and cycles;
  inside an ISR this can break a latency budget or a project rule about ISR complexity.
  Split when it aids comprehension and budgets allow.
- Deduplicating code across execution contexts - near-identical ISR and main-loop versions
  sometimes exist deliberately, for reentrancy or to keep interrupt paths independent.
  Understand the context boundaries first.
- Removing "unnecessary" error handling - peripheral calls fail in the field (brown-out,
  EMI, disconnected sensor);
failure paths are behavior.
- Inlining too aggressively - removing a helper that gave a concept a name makes call sites
  harder to read.
- Removing abstractions that exist for testability or portability - a HAL seam that lets
  drivers run in host unit tests is not accidental complexity.
- Optimizing for line count - comprehension speed is the goal, not brevity.

### 6. Stay in scope.

Default to simplifying recently modified code. Unscoped refactors create noisy diffs and
regression risk. Keep changes minimal, traceable, and separate from feature/bugfix work.

## Process

### 1. Understand Before Touching (Chesterton's Fence)

Before changing or removing anything, understand why it exists. In embedded work, "why it
exists" often lives in a datasheet, not the code.

```text
BEFORE SIMPLIFYING, ANSWER:
- What is this code's responsibility?
- Which contexts execute it - boot, ISR, RTOS task, main loop? Is reentrancy assumed?
- What calls it? What does it call? What hardware does it touch?
- What are the error paths, and what does the hardware do on failure?
- Are there tests (host unit tests, HIL, on-target) defining expected behavior?
- Why might it be written this way? (Datasheet sequence? Errata workaround? Timing? Stack budget? Historical accident?)
- Check git blame AND the part's errata sheet before assuming complexity is accidental.
- Check the map file: is this code in a special section (RAM function, bootloader region)? Is its size load-bearing?
```

### 2. Identify Simplification Opportunities

Scan for these patterns - each one is a concrete signal, not a vague smell:

**Structural complexity**

| Pattern | Signal | Simplification |
|---------|--------|----------------|
| Deep nesting (3+ levels) | Hard to follow control flow | Guard clauses or helper functions - check stack/cycle budgets first |
| Flag soup | Multiple interacting booleans encode hidden states | Explicit state machine: `enum` states + `switch` |
| Long functions (50+ lines) | Multiple responsibilities | Split into focused functions - mind stack depth and ISR rules |
| Boolean parameter flags | `init(true, false, true)` | Config struct or separate functions |
| Repeated conditionals | Same check in multiple places | Well-named predicate function |

**Naming and readability**

| Pattern | Signal | Simplification |
|---------|--------|----------------|
| Generic names | `data`, `tmp`, `val`, `buf2` | Rename to content: `rx_frame`, `cell_voltage_mv` |
| Missing units | `timeout`, `rate`, `period` | `timeout_ms`, `baud_hz`, `period_ticks` |
| Abbreviated names | `cfg`, `pkt`, `dev`, `u8x` | Match project convention - many embedded style guides mandate these; don't "fix" them |
| Misleading names | `read_status()` that also clears flags | Rename to actual behavior |

**Redundancy**

| Pattern | Signal | Simplification |
|---------|--------|----------------|
| Duplicated logic | Same 5+ lines in multiple places | Extract shared function - unless duplication separates ISR/main contexts |
| Dead code | Unreachable branches, unused variables | Remove (after confirming it's truly dead) |
| Unnecessary abstraction | Wrapper that adds no value | Call the underlying function directly |
| Over-engineered patterns | Callback indirection with one callback, factory-of-one | Direct approach |
| "Redundant" volatile, casts, dummy reads | Code looks over-cautious | Verify against datasheet/errata/compiler docs - usually load-bearing |

**Do not "fix" by**

- Replacing a bounded wait with `while (!(REG & BIT)) {}`
- Moving work into an ISR because the task path looked verbose
- Introducing `malloc`, VLAs, or a new global buffer "to clean the signature"
- Reordering reference manual-mandated register writes for a prettier function
- Extracting from an ISR if that adds a call/stack you have not accounted for

See examples below.

```c
/* UNCLEAR: nested success path */
if (spi_init(&spi) == 0) {
    if (spi_xfer(&spi, buf, n, SPI_TIMEOUT_CYCLES) == 0) {
        return 0;
    }
    return -EIO;
}
return -ENODEV;

/* CLEAR: same codes, guards first */
if (spi_init(&spi) != 0) {
    return -ENODEV;
}
if (spi_xfer(&spi, buf, n, SPI_TIMEOUT_CYCLES) != 0) {
    return -EIO;
}
return 0;
```

```c
/* KEEP: bound is the contract, not noise */
uint32_t n = 0U;
while ((SPI1->SR & SPI_SR_RXNE) == 0U) {
    if (++n >= SPI_TIMEOUT_CYCLES) {
        return -ETIMEDOUT;
    }
}
```

```c
/* AVOID: nested conditionals and duplicated bodies */
/* Forced request listens forever; otherwise a short window if the slot is valid. */
if (iap_forced) {
    if (iap_listen(0U)) {
        status         = fw_verify(&handoff);
        report_to_host = (iap_update(status == FW_OK) == FW_OK);
    }
} else if (iap_listen(IAP_LISTEN_MS)) {
    status         = fw_verify(&handoff);
    report_to_host = (iap_update(status == FW_OK) == FW_OK);
}

/* USE: ternary gets you less code; intent is still clear */
/* Forced request listens forever; otherwise a short window if the slot is valid. */
if (iap_listen(iap_forced ? 0U : IAP_LISTEN_MS)) {
    status = fw_verify(&handoff);
    report_to_host = (iap_update(status == FW_OK) == FW_OK);
}
```

### 3. Apply Changes Incrementally

Make one simplification, then run the static analysis and tests this tree already has (host
Unity, on-target, or both). If tests fail, revert that step. Do not mix simplification with
a feature or a bug fix in the same step unless the user asked for one patch.

If a pass would touch hundreds of lines, stop and split. Do not invent a new framework to "help."

### 4. Verify the Result

```text
COMPARE BEFORE AND AFTER:
- Genuinely easier to understand - including for someone holding the datasheet?
- No new patterns inconsistent with the codebase?
- Footprint delta explained from the map file?
- Timing unchanged on measured paths?
- Diff clean, reviewable, traceable?
- Would a firmware teammate approve this as a net improvement?
```

If the "simplified" version is harder to understand, doesn't fit, or shifts timing - revert.
Not every simplification attempt succeeds.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
|"It's working, no need to touch it" | Working code that's hard to read will be hard to fix when it breaks. Simplifying now saves time on every future change. |
| "Fewer lines is always simpler" | A dense bit-twiddling one-liner isn't simpler than named constants. Simplicity is comprehension speed, not line count. |
| "I'll just quickly simplify this unrelated code too" | Unscoped simplification creates noisy diffs and risks regressions in code you didn't intend to change. Stay focused. |
| "This abstraction might be useful later" | Speculative abstractions cost flash, RAM, and review effort. Remove and re-add when a real need appears. |
| "The original author must have had a reason" | Check git blame and the errata sheet - apply Chesterton's Fence. But some complexity is just residue from iteration under pressure. Evidence decides which. |
| "I'll refactor while adding this feature" | Separate refactoring from feature work. Mixed changes are harder to review, revert, and understand in history. |
| "This magic delay / dummy read does nothing" | Check the datasheet timing specs and the silicon errata sheet first. Unexplained hardware workarounds are the most load-bearing lines in the file - delete them last, not first. |
| "I'll split this long function into helpers" | Each call adds stack depth and cycles; in an ISR it may break the latency budget or project style rules. Split when it aids comprehension and budgets allow. |
| "Those `#if 0` blocks are dead code" | They often document an alternate board config or an errata workaround. Confirm before deleting. |

## Red flags

- Simplification that requires modifying tests to pass (you likely changed behavior)
- "Simplified" code that is longer and harder to follow than the original
- Renaming things to match your preferences rather than project conventions
- Removing/reducing error handling, timeouts, IRQs because "it makes the code cleaner"
- Removing `volatile` qualifiers, dummy reads, or required delays
- Simplifying code you don't fully understand
- Batching many simplifications into one large, hard-to-review commit
- Refactoring code outside the scope of the current task without being asked
- Noticeable memory-related changes after the refactor

## Verification

After completing a simplification pass:

- [ ] Build succeeds with no new warnings; static analysis checks are clean
- [ ] Existing tests pass without editing them
- [ ] Register order, timeouts, and ISR work match the before
- [ ] No noticeable effect on the memory footprint + stack usage checked (-fstack-usage, high-water marks, puncover or equivalent) — no unexplained changes, delta explained
- [ ] Timing-sensitive paths not affected
- [ ] Simplified code follows project conventions (CLAUDE.md, AGENTS.md coding standard, HAL idiom, cursor rules, etc.)
- [ ] No error handling removed or weakened
- [ ] No dead code was left behind (unused imports, unreachable branches)
- [ ] Diff is clean (no unrelated changes mixed in), reviewable and on-scope
