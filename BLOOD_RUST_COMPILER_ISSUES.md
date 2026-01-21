# Blood-Rust Compiler Issues Report

**Date:** 2025-01-21
**Compiler:** `/home/jkindrix/blood-rust/target/release/blood`
**Test Project:** Aether (reactive stream processing framework using algebraic effects)

---

## Executive Summary

The blood-rust compiler successfully compiles and runs the core Aether test suite (`streams.blood`), demonstrating that algebraic effects, closures, and effect handlers work correctly for most use cases. However, testing revealed:

1. **One critical closure bug** - Closures returned from functions lose their captured environment
2. **Unsupported `handle` syntax** - 7 Aether files use `handle expr { }` instead of `try { } with { }`
3. **Builtin type conflicts** - Files that redefine `Option`/`Result` fail

---

## Issue #1: Closures Returned from Functions Lose Captured Environment

### Severity: HIGH

### Description

When a function returns a closure that captures one of its parameters, the captured value is not preserved when the closure is later called. The closure appears to read garbage or zero instead of the captured value.

### Minimal Reproduction

```blood
fn make_adder(n: i32) -> fn(i32) -> i32 {
    |x: i32| -> i32 { x + n }
}

fn main() -> i32 {
    let f: fn(i32) -> i32 = make_adder(100);
    f(1)  // Expected: 101, Actual: 1
}
```

**Expected:** Exit code 101 (1 + 100)
**Actual:** Exit code 1 (captured `n` is effectively 0)

### Additional Test Case

```blood
fn make_adder(n: i32) -> fn(i32) -> i32 {
    |x: i32| -> i32 { x + n }
}

fn main() -> i32 {
    let add5: fn(i32) -> i32 = make_adder(5);
    let r1: i32 = add5(3);   // Expected: 8, Actual: 3
    if r1 != 8 { return r1 + 100; }

    let add10: fn(i32) -> i32 = make_adder(10);
    let r2: i32 = add10(7);  // Expected: 17, Actual: 7
    if r2 != 17 { return r2 + 200; }

    0
}
```

**Expected:** Exit code 0
**Actual:** Exit code 103 (r1 = 3, so returns 3 + 100)

### What Works (for comparison)

Closures that are **used within the same scope** or **passed to another function** work correctly:

```blood
fn apply(f: fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}

fn main() -> i32 {
    let n: i32 = 5;
    let adder = |x: i32| -> i32 { x + n };

    // Direct call - WORKS
    let r1: i32 = adder(10);  // Returns 15 ✓

    // Passed to another function - WORKS
    let r2: i32 = apply(adder, 3);  // Returns 8 ✓

    0
}
```

### Root Cause Hypothesis

When a closure is returned from a function:
1. The closure's environment pointer references stack memory from the creating function
2. When the creating function returns, that stack frame is deallocated
3. The closure's environment now points to invalid/stale memory
4. When called, the closure reads garbage values for captured variables

The fix likely requires:
- Heap-allocating the closure environment when the closure escapes its creating scope
- Or copying captured values into the closure's fat pointer structure

### Impact

This bug prevents common functional programming patterns:
- Factory functions that return closures (`make_adder`, `make_multiplier`)
- Currying
- Partial application
- Any pattern where closures outlive their creating scope

---

## Issue #2: `handle` Syntax Not Supported

### Severity: MEDIUM (syntax mismatch, not compiler bug)

### Description

Several Aether source files use `handle expr { ... }` syntax for effect handlers, but the compiler only supports `try { } with { }` syntax.

### Files Affected

| File | Line | Code |
|------|------|------|
| `backpressure.blood` | 139 | `handle stream() { Emit.emit(value) => { ... } }` |
| `channels.blood` | ~100+ | Multiple `handle` blocks |
| `concurrent.blood` | ~100+ | Multiple `handle` blocks |
| `operators.blood` | ~50+ | Multiple `handle` blocks |
| `prelude.blood` | ~100+ | Multiple `handle` blocks |
| `sinks.blood` | ~50+ | Multiple `handle` blocks |
| `windowing.blood` | ~100+ | Multiple `handle` blocks |

### Example Error

```
Error: [E0100] expected literal, identifier, `(`, `[`, `{`, `if`, or `match`, found `=>`
   ╭─[src/backpressure.blood:140:26]
   │
140 │         Emit.emit(value) => {
   │                          ^^
```

### Syntax Comparison

**Currently supported (`try/with`):**
```blood
try { stream(); }
with {
    Emit.emit(value) => {
        // handle emission
        resume(());
    }
};
```

**Not supported (`handle`):**
```blood
handle stream() {
    Emit.emit(value) => {
        // handle emission
        resume(());
    }
}
```

### Recommendation

Either:
1. **Add support for `handle` syntax** as syntactic sugar for `try/with`
2. **Document that only `try/with` is supported** and update Aether files accordingly

The `handle` syntax is more concise and commonly used in effect systems literature (Koka, Eff, etc.), so supporting it would improve ergonomics.

### Workaround

Convert `handle expr { handlers }` to `try { expr; } with { handlers };`

---

## Issue #3: Builtin Type Name Conflicts

### Severity: LOW

### Description

Files that define their own `Option<T>` or `Result<T, E>` types fail because these names conflict with builtin types.

### Files Affected

- `core.blood` - Defines `Option<T>` and `Result<T, E>`

### Error

```
Error: [E0214] the name `Option` is defined multiple times
   ╭─[src/core.blood:62:1]
   │
62 │ ╭─▶ enum Option<T> {
   ┆ ┆
65 │ ├─▶ }
   │ │
   │ ╰─────── the name `Option` is defined multiple times
```

### Recommendation

1. **Allow shadowing of builtins** in user code (with warning)
2. **Or require explicit import** of builtins, allowing user definitions by default
3. **Or document** that `Option`, `Result`, `String`, `Vec` are reserved names

---

## Issue #4: Type `never` Not Found

### Severity: LOW

### Description

The `never` type (for functions that don't return) is referenced but not available.

### File Affected

- `core.blood` line 42

### Code

```blood
effect Fail<E> {
    op fail(error: E) -> never;  // Error: cannot find type `never`
}
```

### Recommendation

Either:
1. Add `never` as a builtin type (like Rust's `!`)
2. Or document the correct syntax for non-returning operations

---

## Issue #5: Sources.blood Type Errors

### Severity: LOW

### Description

`sources.blood` has 23 type checking errors, primarily related to iterator patterns and generic type inference.

### Sample Errors

```
Error: [E0203] cannot find type `Iterator` in this scope
Error: [E0201] type mismatch in generic instantiation
```

### Recommendation

Review the iterator abstraction design and ensure the necessary traits/types are available.

---

## Test Results Summary

### Passing Tests (Core Functionality Works)

| Test | Description | Result |
|------|-------------|--------|
| `streams.blood` | Full Aether test suite | ✅ PASS |
| `test_lambda_direct.blood` | Inline lambda passed to function | ✅ PASS |
| `test_inline_lambda.blood` | Lambda as filter predicate | ✅ PASS |
| `test_capture_in_filter.blood` | Closure capturing local variable | ✅ PASS |
| `test_param_capture_filter.blood` | Closure capturing function parameter | ✅ PASS |
| `bug_closure_capture.blood` | Previous closure bug regression test | ✅ PASS |

### Passing Edge Cases

| Test | Description |
|------|-------------|
| Nested closures | Closure inside closure with multiple captures |
| Multi-op effects | Effect with both `get()` and `set(v)` operations |
| Early handler return | `take` operator (stop processing early) |
| Generic effects | `emit_twice<T>` with type parameter in effect |

### Failing Tests

| Test | Issue |
|------|-------|
| `make_adder` pattern | Closure returned from function (Issue #1) |
| `backpressure.blood` | Uses `handle` syntax (Issue #2) |
| `channels.blood` | Uses `handle` syntax (Issue #2) |
| `concurrent.blood` | Uses `handle` syntax (Issue #2) |
| `operators.blood` | Uses `handle` syntax (Issue #2) |
| `prelude.blood` | Uses `handle` syntax (Issue #2) |
| `sinks.blood` | Uses `handle` syntax (Issue #2) |
| `windowing.blood` | Uses `handle` syntax (Issue #2) |
| `core.blood` | Builtin name conflict (Issue #3) |
| `sources.blood` | Type errors (Issue #5) |

---

## Reproduction Steps

### Setup

```bash
# Ensure runtime is built
cd /home/jkindrix/blood-rust/runtime && make

# Clear build cache before each test
rm -rf ~/.blood/cache
```

### Run Aether Test Suite

```bash
cd /home/jkindrix/blood-test/aether
/home/jkindrix/blood-rust/target/release/blood build src/streams.blood
./src/streams
echo "Exit code: $?"  # Should be 0
```

### Reproduce Issue #1 (Closure Return Bug)

```bash
cat > /tmp/test_closure_return.blood << 'EOF'
fn make_adder(n: i32) -> fn(i32) -> i32 {
    |x: i32| -> i32 { x + n }
}

fn main() -> i32 {
    let f: fn(i32) -> i32 = make_adder(100);
    let result: i32 = f(1);
    if result == 101 { 0 } else { result }
}
EOF

rm -rf ~/.blood/cache
/home/jkindrix/blood-rust/target/release/blood build /tmp/test_closure_return.blood
/tmp/test_closure_return
echo "Exit code: $?"  # Bug: returns 1, should return 0
```

---

## Priority Recommendation

1. **HIGH: Fix Issue #1** - Closure return bug breaks common patterns
2. **MEDIUM: Add `handle` syntax** - Would allow 7 more Aether files to compile
3. **LOW: Document builtin conflicts** - Easy workaround (rename user types)
4. **LOW: Add `never` type** - Only affects advanced effect patterns

---

## Appendix: Working Aether Code Pattern

For reference, here's the pattern that works correctly:

```blood
effect Emit<T> {
    op emit(value: T) -> ();
}

fn range(start: i32, end: i32) / {Emit<i32>} {
    let mut i: i32 = start;
    while i < end {
        perform Emit.emit(i);
        i = i + 1;
    }
}

fn filter<T>(stream: fn() / {Emit<T>}, pred: fn(T) -> bool) / {Emit<T>} {
    try { stream(); }
    with {
        Emit.emit(value) => {
            if pred(value) { perform Emit.emit(value); }
            resume(());
        }
    };
}

fn sum(stream: fn() / {Emit<i32>}) -> i64 {
    let mut total: i64 = 0;
    try { stream(); }
    with {
        Emit.emit(value) => {
            total = total + (value as i64);
            resume(());
        }
    };
    total
}

fn main() -> i32 {
    // This works: closure captures local variable, passed to function
    let limit: i32 = 11;
    let result: i64 = sum(|| / {Emit<i32>} {
        filter(|| / {Emit<i32>} { range(1, limit); }, |x: i32| -> bool { x % 2 == 0 });
    });
    if result == 30 { 0 } else { 1 }
}
```

This compiles and runs correctly, demonstrating that the core effect system works.
