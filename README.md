# Development-Guidelines

A strict engineering specification and Claude Skill suite for **rapid prototyping in robotics and safety-critical systems using AI**.

AI coding agents accelerate iteration speed by an order of magnitude, but unconstrained generation introduces silent failure modes—unbounded recursion, hidden heap fragmentation, implicit type narrowing, monolithic hardware coupling, and fragile "happy-path" algorithms. In robotics, software bugs become kinetic hazards.

This specification enforces deterministic, bounded, and verifiable code patterns without sacrificing prototyping velocity.

---

## Core Philosophy: Speed Through Constraint

Rapid prototyping in physical systems fails when debugging happens on hardware. By enforcing strict structural boundaries upfront, algorithms stay decoupled from hardware I/O and can be generated, compiled, and adversarially stress-tested in milliseconds before touching an actuator.

---

## Mandatory Engineering Rules

### 1. Zero Recursion
* **Rule:** Direct, indirect, and mutual recursion are strictly prohibited.
* **Rationale:** Call stack depth must be statically provable at compile time. Unbounded stack growth on embedded microcontrollers or real-time threads causes silent memory corruption or hard faults.
* **Pattern:** Replace recursive traversals (trees, graphs, kinematics chains) with explicit, fixed-capacity stacks or bounded iterative loops.

### 2. Zero Dynamic Memory Allocation
* **Rule:** No heap allocation (`malloc`, `calloc`, `realloc`, `free`, `new`, `delete`) or dynamically growing containers (`std::vector::push_back`, `std::string`, dynamic maps) in runtime or control paths.
* **Rationale:** Eliminates out-of-memory crashes, allocator lock contention, and heap fragmentation over long duty cycles.
* **Pattern:** Pre-allocate fixed-capacity pools, ring buffers, and static arrays (`std::array`, fixed-size C structs) at compile time or during single-pass system initialization.

### 3. Strict Compilation & Zero Warnings
* **Rule:** All code must compile cleanly with maximum warnings enabled and warnings promoted to errors (`-Werror`).
* **Minimum C/C++ Baseline:**
  ```text
  -Wall -Wextra -Wpedantic -Werror -Wconversion -Wsign-conversion -Wshadow -Wnull-dereference -Wdouble-promotion -Wformat=2
  ```
* **Rationale:** AI-generated code frequently introduces subtle signed/unsigned mismatches, implicit float-to-double promotions, or shadowed variables. The compiler is the first line of defense.

### 4. High Modularity & Hardware Decoupling
* **Rule:** Separate pure algorithmic logic from hardware abstraction layers (HAL), middleware (ROS 2, DDS, CAN, serial), and global state.
* **Rationale:** Modularity is what makes rapid prototyping safe. When a controller, estimator, or planner is a pure state-transition function, AI can rewrite or swap it in seconds and test it headlessly without hardware dependencies.
* **Pattern:**
  * Every module exposes a narrow, explicit interface: `init()`, `step(const Input*, State*, Output*)`, and `reset()`.
  * Zero hidden global or `static` mutable state inside functions.
  * Target small, single-responsibility files with clear ownership boundaries.

### 5. Adversarial Verification Gate
* **Rule:** No algorithm is considered acceptable until it survives a dedicated adversarial test suite designed to break it. Happy-path unit tests are insufficient.
* **Mandatory Adversarial Vectors:**
  * Non-finite floats: `NaN`, `+Inf`, `-Inf`, subnormals, and negative zero (`-0.0`).
  * Temporal anomalies: `dt = 0`, negative `dt` (clock jump), and massive `dt` (sensor dropout).
  * Numeric extremes: `INT_MIN`, `INT_MAX`, unsigned wraparound, division by near-zero (`1e-12`).
  * Geometric/math singularities: zero-norm vectors, gimbal lock, collinear points, singular/ill-conditioned matrices.
  * Malformed I/O: truncated frames, out-of-order packets, saturated sensor rails, stuck-at faults.
* **Acceptance Criterion:** Under any malformed or adversarial input, the module must never crash, hang, divide by zero, access out-of-bounds memory, or emit an un-clamped/non-finite actuator command.

### 6. Fail-Safe Defaults (Never Fail-Silent)
* **Rule:** Every module must define an explicit `SafeState` (e.g., zero torque, passive damping, hold-position, or fault flag) and check all return/status codes.
* **Rationale:** On invalid inputs or internal fault detection, emitting an explicit fault status and commanding a safe physical state prevents runaway actuators.

### 7. Explicit SI Units & Boundary Clamping
* **Rule:** Encode physical units in variable and field names (`velocity_mps`, `yaw_rad`, `timeout_ms`, `current_a`). Validate inputs at module entry and clamp outputs to physical actuator limits at module exit.
* **Rationale:** Eliminates unit-conversion bugs (degrees vs. radians, mm vs. meters) during rapid AI code generation.

---

## Using the Claude Skills

This repository includes modular agent skills in `skills/` authored to the [Agent Skills specification](https://agentskills.io/specification).

### Setup for Claude Code
Symlink or copy the `skills/` directory into your project's `.claude/skills/` directory (or user-level `~/.claude/skills/`):

```bash
mkdir -p .claude
ln -s "$(pwd)/skills" .claude/skills
```

### Included Skills

| Skill | Purpose | When It Triggers |
| :--- | :--- | :--- |
| `safe-prototype-architect` | Scaffolds modular, static-memory, fail-safe robotics modules | Designing or scaffolding new controllers, estimators, parsers, or drivers |
| `safety-critical-audit` | Audits code for recursion, heap usage, unbounded loops, and warning hazards | Reviewing or hardening existing or AI-generated code before merge/flash |
| `adversarial-edge-tester` | Builds and runs edge-case, `NaN`/`Inf`, and fuzzing harnesses to break algorithms | Validating algorithms, writing tests, or running pre-deployment checks |
| `strict-build-hardening` | Configures compiler warning flags, sanitizers, and static analysis | Setting up CMake/Make/Cargo builds or fixing compiler warnings |

---

## Rapid Prototyping Workflow

1. **Scaffold (`safe-prototype-architect`):** Define module inputs, outputs, static state struct, unit suffixes, and safe fallback behavior.
2. **Implement:** Write pure, iterative, fixed-memory logic decoupled from hardware drivers.
3. **Compile (`strict-build-hardening`):** Build with `-Wall -Wextra -Wpedantic -Werror` and sanitizers enabled.
4. **Break (`adversarial-edge-tester`):** Hammer the module with `NaN`, dropouts, overflows, and malformed packets; patch until zero failures occur.
5. **Audit (`safety-critical-audit`):** Verify full spec compliance before deploying to simulation-in-the-loop or physical hardware.
