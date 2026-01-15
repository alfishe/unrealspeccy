# Z80 T-State Management in UnrealSpeccy

This document describes how CPU cycles (T-states) are tracked and managed for cycle-accurate emulation.

## Overview

The ZX Spectrum uses a Z80 CPU where timing is measured in **T-states** (clock cycles). UnrealSpeccy maintains two classes of timing counters:

1. **Frame-relative** — reset each video frame
2. **Monotonic** — never reset, always increasing

---

## Key Variables

### Frame-Relative Counters

| Variable | Type | Location | Description |
|----------|------|----------|-------------|
| `cpu.t` | `unsigned` | `z80/defs.h` (`TZ80State`) | **Primary T-state counter within the current frame.** Incremented by each instruction, reset at frame boundary. |
| `cpu.eipos` | `unsigned` | `z80/defs.h` | T-state position when `EI` instruction was executed (for interrupt timing). |
| `cpu.haltpos` | `unsigned` | `z80/defs.h` | T-state position when `HALT` was entered (for performance monitoring). |

### Monotonic Counters

| Variable | Type | Location | Description |
|----------|------|----------|-------------|
| `comp.t_states` | `__int64` | `emul.h` (`COMPUTER`) | **Absolute T-state counter.** Incremented by `conf.frame` at end of each frame. Never reset (except on hard reset). |
| `comp.frame_counter` | `unsigned` | `emul.h` (`COMPUTER`) | Frame number since emulation start. Incremented each frame. |

### Frame Timing Configuration

| Variable | Location | Description |
|----------|----------|-------------|
| `conf.frame` | `emul.h` (`CONFIG`) | Total T-states per frame (e.g., 71680 for Pentagon 50Hz). |
| `conf.t_line` | `emul.h` (`CONFIG`) | T-states per scanline (e.g., 224). |
| `conf.paper` | `emul.h` (`CONFIG`) | T-state offset before screen paper area begins. |
| `conf.intlen` | `emul.h` (`CONFIG`) | Duration of INT signal in T-states. |
| `cpu.tpi` | `z80/defs.h` (`Z80`) | "T-states per interrupt" — threshold for frame end detection (set via `cpu.SetTpi(conf.frame)`). |

---

## Timing Model

### Within a Frame

```
Frame Start (cpu.t = 0 or negative carryover)
    │
    ├── Instruction execution increments cpu.t
    ├── I/O operations use cpu.t for timing
    ├── Beam position = cpu.t / conf.t_line
    │
Frame End (cpu.t >= conf.frame)
```

### Frame Boundary Transition

At the end of each frame in `spectrum_frame()` ([mainloop.cpp](file:///Users/dev/Projects/GitHub/unrealspeccy/mainloop.cpp#L60-L63)):

```cpp
comp.t_states += conf.frame;   // Advance monotonic counter
cpu.t -= conf.frame;           // Wrap frame counter (may be negative if overshoot)
cpu.eipos -= conf.frame;       // Adjust EI position
comp.frame_counter++;          // Increment frame number
```

### Absolute Time Calculation

To get absolute T-state time (e.g., for tape timing, debugger):

```cpp
__int64 absolute_t = comp.t_states + cpu.t;
```

This pattern appears throughout the codebase for timing comparisons across frame boundaries.

---

## Cycle-Accurate Emulation

### Instruction Timing

Each Z80 opcode adds its cycle count to `cpu.t`. The opcode implementations in `z80/op_*.cpp` increment the counter appropriately.

### Interrupt Timing

The `tpi` field gates interrupt behavior:

```cpp
// In op_ed.cpp (HALT handling)
if (cpu->iff1 && (cpu->t + 10 < cpu->tpi))
```

This ensures interrupts fire at the correct frame position.

### Beam Racing

For effects requiring sub-frame accuracy (contention, border effects):

```cpp
unsigned current_line = cpu.t / conf.t_line;
unsigned line_offset = cpu.t % conf.t_line;
```

---

## GS (General Sound) CPU Timing

The optional General Sound co-processor has its own parallel timing:

| Variable | Description |
|----------|-------------|
| `gs_t_states` | Monotonic T-state counter for GS Z80 |
| `gscpu.t` | Frame-relative T-state for GS |
| `gscpu_t_at_frame_start` | Synchronization anchor |

Sound chip updates use:
```cpp
unsigned t = (gs_t_states + gscpu.t) - gscpu_t_at_frame_start;
```

---

## Common Usage Patterns

### Timing an Event

```cpp
// Record when tape edge changed
comp.tape.edge_change = comp.t_states + cpu.t;

// Later, check elapsed time
__int64 current = comp.t_states + cpu.t;
__int64 delta = current - comp.tape.edge_change;
```

### Debugger Delta Time

```cpp
// In dbgoth.cpp
i64 delta() {
    return comp.t_states + cpu.t - cpu.debug_last_t;
}
```

### Sound Chip Updates

Sound renderers receive the current T-state to position samples accurately within the audio buffer:

```cpp
sound.update(cpu.t, left_sample, right_sample);
```

---

## Summary

| Counter | Scope | Reset When | Primary Use |
|---------|-------|------------|-------------|
| `cpu.t` | Frame | Each frame boundary | Instruction timing, beam racing, contention |
| `comp.t_states` | Global | Hard reset only | Cross-frame timing, absolute timestamps |
| `comp.frame_counter` | Global | Hard reset only | Frame counting, animation timing |
| `cpu.tpi` | Config | Configuration change | Frame-end interrupt detection |
