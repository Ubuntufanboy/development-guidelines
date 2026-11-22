---
name: adversarial-edge-tester
description: Generates and runs adversarial test suites with purposefully crafted edge cases, non-finite numbers, sensor dropouts, numeric overflows, and fuzz inputs to break robotics and safety-critical algorithms. Use when testing code, validating controllers or parsers, or hardening algorithms before deployment.
---

# Adversarial Edge Tester

Act as an adversary trying to crash, hang, or trick the target algorithm into emitting unsafe commands. Code is not acceptable until zero adversarial inputs can violate its safety invariants.

## Mandatory Attack Vectors

Construct test cases covering every applicable category below:

1. **Non-Finite & Subnormal Floats:**
   - `NAN`, `INFINITY`, `-INFINITY`, `-0.0`, `DBL_MIN` / `FLT_MIN`, `DBL_MAX` / `FLT_MAX`, and subnormal values (`1e-310`).
   - Inject `NaN` into one sensor channel while keeping all other channels valid.
2. **Time & Step Anomalies:**
   - `dt = 0.0`, `dt = -0.01` (clock regression), `dt = 1e-9` (micro-step), `dt = 100.0` (long sensor blackout), and sudden step-size jitter.
3. **Numeric Boundaries & Overflows:**
   - `0`, `-1`, `1`, `INT_MIN`, `INT_MAX`, `UINT_MAX`, wraparound timestamps, and near-zero denominators (`1e-15`, `-1e-15`).
4. **Domain & Geometric Singularities:**
   - Zero-length vectors, unnormalized quaternions, antipodal rotations (`180 deg`), gimbal lock (`pitch = +-90 deg`), collinear points, singular/zero matrices, and empty/full ring buffers.
5. **Sensor & Protocol Faults:**
   - Stuck-at-constant values, rail saturation (`+MAX` / `-MAX` spikes), step discontinuities, out-of-order sequence numbers, truncated byte buffers, and corrupted checksums.
6. **Deterministic Property-Based Fuzzing:**
   - Run at least `10,000` seeded pseudo-random and boundary-biased input sequences (fixed PRNG seed for reproducibility) verifying invariants on every step.

## Safety Invariants (Must Hold on Every Step)

- Process never crashes (`SIGSEGV`, `SIGFPE`, `SIGABRT`) or triggers sanitizer errors.
- Execution completes within bounded iteration limits (no infinite loops).
- Every floating-point output satisfies `isfinite(val)` unless explicitly flagged invalid.
- Every actuator output stays strictly within `[min_limit, max_limit]`.
- Invalid inputs transition output to the module's defined `SafeState`.

## Execution Workflow

1. **Identify Invariants:** Read the module's interface, physical limits, and `SafeState` definition.
2. **Write Adversarial Harness:** Create a standalone test file (`test_<module>_adversarial.*`) exercising all 6 attack vectors above plus invariant assertions after every `step()` call.
3. **Compile & Run with Sanitizers:** Build with `-fsanitize=address,undefined` (when supported by the toolchain) and execute the test suite.
4. **Fix & Iterate:** For any failure discovered:
   - Fix the root cause in the module (not in the test).
   - Add the exact breaking input as a permanent regression case.
   - Re-run until 100% of adversarial vectors and fuzz iterations pass with zero sanitizer findings.
