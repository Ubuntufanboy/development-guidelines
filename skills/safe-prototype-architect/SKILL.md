---
name: safe-prototype-architect
description: Designs and scaffolds modular, deterministic, static-memory software modules for robotics and safety-critical rapid prototyping. Use when creating new control loops, state estimators, sensor parsers, motion planners, or refactoring monolithic code into hardware-decoupled components.
---

# Safe Prototype Architect

Scaffold new robotics and safety-critical modules so they are modular, statically bounded, and immediately testable in headless simulation before hardware deployment.

## Architectural Rules

Enforce these constraints on every generated module:

1. **Zero Heap & Zero Recursion:**
   - All state lives in caller-owned structs (`ModuleState`) or compile-time fixed buffers (`MAX_*` constants).
   - Never emit `malloc`, `new`, dynamic STL containers, or recursive function calls.
2. **Pure Step Architecture (Sim-to-Real Seam):**
   - Separate algorithmic logic from hardware I/O (CAN, UART, SPI, ROS 2 publishers/subscribers).
   - Expose a clean deterministic lifecycle:
     - `module_init(ModuleState* state, const ModuleConfig* config)`
     - `module_reset(ModuleState* state)`
     - `ModuleStatus module_step(ModuleState* state, const ModuleInput* in, ModuleOutput* out)`
3. **Explicit SI Units & Physical Bounds:**
   - Suffix all physical quantities with SI units (`_rad`, `_radps`, `_m`, `_mps`, `_mps2`, `_s`, `_ms`, `_nm`, `_a`).
   - Include min/max saturation limits in `ModuleConfig`.
4. **Fail-Safe Default Output:**
   - The first action inside `module_step()` must populate `*out` with the module's deterministic safe state (e.g., zero velocity/torque, `valid = false`) before any validation or computation runs.

## Scaffolding Workflow

1. **Clarify Boundaries:** Identify inputs, outputs, update rate (`dt_s`), fixed buffer capacities, and the physical safe state on fault.
2. **Define Data Contracts First:** Write the header/interface file containing `ModuleConfig`, `ModuleInput`, `ModuleOutput`, `ModuleState`, and `ModuleStatus` enum before writing implementation logic.
3. **Implement Bounded Logic:**
   - Guard entry with null-pointer checks, `isfinite()` checks on all floating-point inputs, and `dt_s` range checks (`dt_min_s <= dt_s <= dt_max_s`).
   - Bound every loop with an explicit `MAX_*` constant.
   - Clamp all outputs to configured physical limits before returning `STATUS_OK`.
4. **Isolate Hardware Adapters:** If hardware or middleware glue is needed, place it in a separate `*_adapter` file that translates raw bus/topic messages into `ModuleInput` and dispatches `ModuleOutput`.

## Output Checklist

Before completing a scaffold, verify:
- [ ] Module compiles standalone with zero hardware or OS headers included in the core algorithm.
- [ ] Maximum memory footprint is statically known from `sizeof(ModuleState)`.
- [ ] Any early return leaves `*out` in a verified safe physical state.
