---
name: grilling-embedded
description: Interview the user relentlessly about a plan, decision, or idea. Use when the user wants to stress-test their thinking, wants an architecture interview before coding, wants to interrogate a firmware design, or uses any 'grill' trigger phrases.
disable-model-invocation: true
---

# Grilling (embedded)

Your goal is to interview the user relentlessly about their embedded system
architecture, resolving timing dependencies, memory boundaries, and failure modes
(in the context of the user-prompted idea) before any code is written.

## Infer the project

Before asking any questions, learn the chip and constraints from the environment.
Never ask the user for a fact the repo or datasheet already states. If a fact is
needed, inspect the environment:

1. Read `CLAUDE.md`, `AGENTS.md` and `.cursor/rules/` if they exist.
2. Skim `README.md` and documentation (e.g., `docs/`) if present.
3. Inspect existing code, linker scripts, and datasheets if reachable.
4. Look facts up yourself. Dispatch a sub-agent if the host can run one in the
   background; only questions that depend on that fact should wait. If the host
   cannot run background sub-agents, look the fact up in this turn before asking
   the user a decision.

## Resolve the scope

- If the user named a feature, driver, peripheral, or protocol, that is the
  apply-scope. Grill that feature and only the hardware, timing, memory, faults,
  etc. it actually touches.
- Whole-product interview only for a greenfield system or when the user asks to
  grill the architecture as a whole.
- If they did not specify, take the feature or idea under discussion, not the
  entire firmware tree.

## Grilling Protocol

Interview the user relentlessly until the named scope is settled. Map this as a
design tree: every decision branches into the decisions that hang off it.

Work the tree in rounds. The frontier is every decision whose prerequisites are already
settled: the questions you can ask now without guessing at answers you haven't heard
yet. Ask the whole frontier in one round: number each question and give your recommended
answer. Then wait for the user's answers before the next round.

Format a round like so:

```text
❓ **Q1** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>

---

❓ **Q2** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>
```

Each round the user answers reshapes the tree: settled decisions push the frontier
outward and unblock questions that depended on them. Recompute the frontier and ask
the next round. A question whose answer depends on another question still open in
this round belongs to a later round, not this one.

Finding facts is your job, never the user's. When a frontier question needs a fact
from the environment (filesystem, tools, etc.), dispatch a sub-agent to find it;
don't ask the user for anything you could look up yourself. Don't block on it: a
running exploration is an unsettled prerequisite, so only the questions downstream of
it wait for the sub-agent to report; ask the rest of the frontier now. The decisions
are the user's: put each to them and wait.

The session is done when the frontier is empty: every branch of the design tree visited,
nothing left silently assumed. Do not act on it until the user confirms you have reached
a shared understanding.

## Interrogation Branches

The branches below are a structural guide, not a rigid exam. How deep you go into each is entirely dependent on the project's maturity and the apply-scope.

- For a bare-metal greenfield project, focus heavily on hardware, timing, and memory.
- For a project using a mature BSP or RTOS, briefly validate the low-level setup, then
  pivot to grilling the application logic and system architecture.
- For a named feature, skip whole-chip pinmux, clock trees, and buses the feature does
  not touch.
- Skip any branch that doesn't apply or was already answered by the inferred context;
  feel free to drill down into domain-specific application logic if the lower layers
  are already settled.

### 1. Hardware and peripherals

- **Pin / bus allocation:** Multiplexing conflicts, shared SPI/I2C buses, DMA
  channel assignments.
- **Clocking and power:** System clock, active vs sleep, wake-up latency, supply
  stability.
- **Physical transceivers:** Driver modes, termination, voltage levels, bus speed
  bounds. Only if there is a real bus.

### 2. Determinism and timing

- **Execution model:** Bare-metal super-loop, cooperative state machine, or
  preemptive RTOS?
- **Interrupt handling:** ISR depth, nested priorities, deferred work (PendSV,
  queues, a main-loop flag), critical-section duration.
- **Deadlines:** Bounded execution, what happens if a task or loop misses a deadline.

### 3. Memory and resource budgets

- **Static vs dynamic:** All-static, fixed pools, or heap (malloc / TLSF / vendor
  heap). If dynamic: who allocates, from which context (task vs ISR), what happens
  on OOM, and when (if ever) memory is freed.
- **Footprint:** Flash (`.text`, `.rodata`), RAM (`.bss`, `.data`), peak stack.
- **Concurrency protection:** Only if there is concurrent access: mutex vs spinlock
  vs disable-IRQ; priority inheritance or a ceiling protocol if a preemptive kernel
  shares a resource.

### 4. Safety, telemetry, and fault recovery

- **Watchdog:** Reset strategy, who pets it, windowed refresh, heartbeat vs
  single main-loop kick.
- **Boot / update / rollback:** How new firmware is loaded (IAP, MCUboot, vendor
  bootloader), what happens on a failed or interrupted update, and the safe state
  if the image is invalid. Do not expand this into a full security exam unless the
  user asked.
- **Fault handling:** Handler fallback, what gets logged, safe hardware state
  (pins, drivers, motors) after a panic.
- **Diagnostics:** Trace (SWO, RTT, serial), logging vs latency, how you will
  inspect registers on a live target.

### 5. Application architecture (if low-level is settled)

- **State machines:** Hierarchical vs flat, state transitions, handling invalid/
  transient states.
- **Data flow & buffering:** Ring buffers, packet parsing, endianness, handling
  partial data or framing errors.
- **Control logic:** PID loops, sensor fusion, actuator saturation, open-loop vs
  closed-loop fallbacks.
- **Protocol handling:** Command dispatching, timeouts, retries, backpressure
  mechanisms.

## When done

The session is done when the named scope is settled and the frontier is empty: every
applicable branch is visited, and nothing is left silently assumed.

When done, write a short Decision Log: each choice that was made, the recommendation
you gave, and what was skipped as N/A. Do not start implementing from that log unless
the user asks.
