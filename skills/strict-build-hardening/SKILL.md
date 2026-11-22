---
name: strict-build-hardening
description: Configures strict compiler warning flags, warning-as-error enforcement, sanitizers, and static analysis for C, C++, and Rust builds. Use when initializing a repository, editing CMakeLists.txt, Makefiles, or Cargo.toml, setting up CI, or resolving compiler warnings.
---

# Strict Build Hardening

Configure build systems so compiler warnings, type conversions, and undefined behavior fail the build immediately during rapid prototyping.

## Compiler Flag Baselines

### C / C++ (GCC & Clang)
Apply these flags to all project targets (excluding third-party vendor headers via `-isystem`):

```cmake
set(STRICT_COMMON_FLAGS
  -Wall
  -Wextra
  -Wpedantic
  -Werror
  -Wconversion
  -Wsign-conversion
  -Wdouble-promotion
  -Wshadow
  -Wnull-dereference
  -Wformat=2
  -Wundef
  -Wcast-align
  -Wuninitialized
  -fno-common
)

# C-specific additions
set(STRICT_C_FLAGS
  -Wstrict-prototypes
  -Wmissing-prototypes
  -Wold-style-definition
)

# C++-specific additions
set(STRICT_CXX_FLAGS
  -Wnon-virtual-dtor
  -Woverloaded-virtual
  -Wold-style-cast
  -fno-exceptions
  -fno-rtti
)
```

### Debug & Test Sanitizers (Host Simulation Builds)
Enable sanitizers on host test targets to catch out-of-bounds access, stack issues, and undefined behavior immediately:

```cmake
set(SANITIZER_FLAGS
  -fsanitize=address,undefined
  -fno-omit-frame-pointer
)
```

### Rust (`Cargo.toml` / `lib.rs`)
For Rust safety-critical modules, enforce `#![no_std]` where applicable and deny warnings:

```toml
[lints.rust]
warnings = "deny"
unsafe_code = "forbid"

[lints.clippy]
all = { level = "deny", priority = -1 }
pedantic = { level = "deny", priority = -1 }
float_cmp = "deny"
cast_possible_truncation = "deny"
cast_sign_loss = "deny"
```

## Hardening Workflow

1. **Inspect Build Files:** Locate `CMakeLists.txt`, `Makefile`, `meson.build`, `Cargo.toml`, or CI workflows.
2. **Apply Strict Flags:** Add the language-appropriate strict warning set and `-Werror` to all first-party targets. Isolate external dependencies with `-isystem` so third-party warnings do not block builds.
3. **Add Sanitized Test Target:** Ensure host unit and adversarial tests build with AddressSanitizer and UndefinedBehaviorSanitizer enabled.
4. **Verify Clean Build:** Run a full clean build and resolve every compiler warning properly (explicit casts with range checks, proper types, initialized variables)—never suppress warnings with pragmas unless documented and justified.
