---
name: safety-critical-audit
description: Audits C, C++, Rust, and embedded code against safety-critical rapid-prototyping rules including zero recursion, zero dynamic memory allocation, bounded loops, compiler warning hygiene, and fail-safe error handling. Use when reviewing code, auditing AI-generated implementations, or verifying pre-deployment readiness.
---

# Safety-Critical Audit

Perform a zero-tolerance compliance audit against the repository's safety-critical rapid-prototyping specification.

## Audit Checklist

Inspect every file in scope against these seven rules:

1. **No Recursion (Direct or Indirect):**
   - Flag any function calling itself directly or participating in a call cycle (`A -> B -> A`).
   - Flag function pointers or callbacks inside control loops unless target sets are statically bounded and acyclic.
2. **No Dynamic Memory Allocation:**
   - Flag `malloc`, `calloc`, `realloc`, `free`, `aligned_alloc`, `new`, `delete`, `std::vector`, `std::string`, `std::map`, `std::unordered_map`, `std::function`, or heap-allocating smart pointers (`std::make_unique`, `std::make_shared`).
   - Verify all arrays and ring buffers use compile-time capacity constants.
3. **Bounded Loops:**
   - Flag `while (true)`, `for (;;)`, or `while` loops lacking a guaranteed monotonic termination counter bounded by a `MAX_*` constant.
4. **Compiler Warning Hazards:**
   - Flag implicit narrowing conversions (`double` to `float`, `int64_t` to `int32_t`), signed/unsigned comparisons, uninitialized variables, shadowed declarations, and unused return values.
5. **Input & Numeric Safety:**
   - Flag unguarded divisions (`a / b` without checking `fabs(b) > EPSILON`), `sqrt` or `acos`/`asin` on unclamped domains, and missing `isfinite()` checks at module entry.
6. **Modularity & Global State:**
   - Flag mutable file-scope or function-local `static` variables that break reentrancy or prevent multi-instance simulation.
   - Flag direct hardware/middleware calls embedded inside algorithmic modules.
7. **Fail-Safe Behavior:**
   - Flag silent error swallowing, unchecked status returns, or fault paths that leave output structs uninitialized or holding stale actuator commands.

## Audit Procedure

1. Scan all target source and header files systematically rule by rule.
2. For each violation, record:
   - **Severity:** `CRITICAL` (recursion, heap allocation, unbounded loop, unsafe fault output) or `WARNING` (unit naming, modularity leak, implicit conversion).
   - **Location:** `file:line`
   - **Violation & Failure Mode:** Concise explanation of how it can fail in runtime/hardware.
   - **Concrete Fix:** Exact replacement pattern.
3. If violations are found, offer or apply fixes directly, then re-verify the modified sections.

## Report Format

```markdown
## Safety-Critical Audit Summary
- **Status:** PASS | FAIL (<N> critical, <M> warnings)
- **Files Audited:** <list>

### Findings
1. **[CRITICAL] `<file>:<line>` — <Rule Name>**
   - **Issue:** <what violates the spec>
   - **Fix:** <concrete remediation>
```
