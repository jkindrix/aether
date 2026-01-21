# Aether Design Notes

This document captures architectural decisions, known constraints, and future directions for the Aether stream processing framework.

## Architecture Decision Records

### ADR-001: Effects as Stream Primitive

**Decision:** Use algebraic effects (`Emit<T>`, `Pull<T>`) as the fundamental streaming abstraction.

**Context:** Traditional streaming libraries use iterators (Rust), callbacks (JS), or monads (Haskell). Blood's effect system offers a unique alternative.

**Rationale:**
- Effects appear in function signatures, making data flow explicit
- Handlers compose naturally without monad transformers
- `resume()` provides delimited continuations for complex control flow
- Effects can be mocked for testing

**Consequences:**
- Stream pipelines are nested closures: `sink(|| operator(|| source()))`
- Early termination is elegant: simply don't call `resume()`
- Learning curve for developers unfamiliar with effect handlers

### ADR-002: Push vs Pull Model

**Decision:** Default to push-based (`Emit<T>`) with pull-based (`Pull<T>`) available for backpressure.

**Context:** Streams can be push-based (producer controls flow) or pull-based (consumer controls flow).

**Rationale:**
- Push is more natural for most transformations
- Pull enables true backpressure without buffering
- Effect handlers can convert between models
- Most operators work identically in either model

**Trade-offs:**
- Push: Simpler implementation, potential buffer overflow without backpressure
- Pull: Natural backpressure, more complex implementation

### ADR-003: Fixed-Size Arrays for Bounded Collections

**Decision:** Use fixed-size arrays (`[T; N]`) for bounded collections like worker pools and channel selectors.

**Context:** Blood's current type system supports fixed-size arrays. Dynamic collections would require heap allocation.

**Rationale:**
- Predictable memory usage (no dynamic allocation)
- Compile-time bounds checking where possible
- Matches Blood's systems programming philosophy
- Sufficient for typical use cases (16 workers, 8 sensors, etc.)

**Constraints:**
- Maximum sizes are compile-time constants
- Some patterns (unbounded windows) require truncation or wrapping

**Future:** When Blood supports `Vec<T>` or dynamic arrays, these can be upgraded.

### ADR-004: Channel IDs vs Typed Handles

**Decision:** Provide both raw `i32` channel IDs (for runtime flexibility) and typed `ChannelHandle<T>` wrappers (for compile-time safety).

**Context:** Blood's `Channel<T>` effect uses `i32` identifiers for runtime dispatch.

**Rationale:**
- Raw IDs enable dynamic channel creation and selection
- Typed handles prevent cross-type channel errors at compile time
- Wrapper adds zero runtime overhead (same underlying ID)
- Both patterns have valid use cases

**Implementation:** See `channels.blood` for `ChannelHandle<T>`, `Sender<T>`, `Receiver<T>`.

### ADR-005: Self-Contained Examples

**Decision:** Each example file redefines core types (`Emit<T>`, `Option<T>`, etc.) rather than importing.

**Context:** Blood's module system is not yet complete.

**Rationale:**
- Examples can be compiled and run independently
- No circular dependency issues during development
- Clear documentation of what types each example uses
- Easy to copy-paste into other projects

**Future:** When Blood has `use aether::prelude::*`, examples will import instead of redefine.

## Known Constraints

### Constraint 1: Maximum Window Size

**Limitation:** Sliding windows are limited to 64 elements due to fixed-size ring buffers.

**Workaround:** For larger windows, use tumbling windows or sampling.

**Root Cause:** Fixed-size arrays in Blood require compile-time sizes.

**Impact:** Affects `windowing.blood` sliding window implementations.

### Constraint 2: Worker Pool Size

**Limitation:** Parallel operators support maximum 32 workers.

**Workaround:** For higher parallelism, use nested parallelism or multiple pools.

**Root Cause:** Fixed-size arrays for worker ID storage.

**Impact:** Affects `concurrent.blood` `FiberPool` and `parallel_map`.

### Constraint 3: Channel Selector Limit

**Limitation:** `ChannelSelector` supports maximum 8 channels.

**Workaround:** For more channels, use hierarchical selection or polling.

**Root Cause:** Fixed-size array for channel tracking.

**Impact:** Affects `channels.blood` select operations.

### Constraint 4: No I/O Effects Yet

**Limitation:** Print functions are placeholder stubs awaiting Blood's I/O effect implementation.

**Current State:**
```blood
fn println_str(s: &str) {
    // Placeholder - awaits Blood IO effects
}
```

**Impact:** Examples can be compiled but won't produce console output until Blood's I/O system is complete.

### Constraint 5: Generic Type Bounds

**Limitation:** Some operators require specific type constraints that Blood's type system expresses differently than Rust.

**Example:** `Ord` bound for `max`/`min` operations is implicit rather than explicit.

## Performance Considerations

### Effect Handler Overhead

Effect handlers have near-zero overhead when:
- The handler is statically known (common case)
- No deep recursion in handler bodies
- `resume()` is tail-position

Potential overhead when:
- Dynamic effect dispatch is required
- Handler captures large closures
- Multiple nested handlers (stack depth)

### Memory Layout

Aether is designed for predictable memory:
- No hidden allocations in operators
- Fixed-size buffers with known bounds
- Value semantics by default (copies, not references)
- Generational pointers for safe inter-stage communication

### Parallelism Granularity

Optimal parallelism depends on:
- Work per element (CPU-bound vs I/O-bound)
- Channel buffer sizes (affects latency vs throughput)
- Number of pipeline stages (deeper = more parallelism opportunities)

Recommendation: Start with 4 workers, 256 buffer, tune based on profiling.

## Future Directions

### F1: Module System Integration

When Blood's module system is complete:
- Single `use aether::prelude::*` import
- Proper visibility modifiers (`pub`, `pub(crate)`)
- Namespace organization

### F2: Async I/O Integration

Connect to Blood's io_uring/kqueue runtime:
- File-based stream sources
- Network sockets as streams
- Timer-based windowing

### F3: Distributed Streams

Cross-node stream processing:
- Serialization of stream elements
- Network channels
- Partition-aware operators
- Fault tolerance (checkpointing)

### F4: SQL-Like Operators

Higher-level query operations:
- `join(stream1, stream2, key_fn)` - Stream joins
- `group_by(stream, key_fn)` - Grouping
- `order_by(stream, compare_fn)` - Sorting (bounded)
- `distinct(stream)` - Deduplication (hash-based)

### F5: Schema Evolution

Type-safe stream versioning:
- Content-addressed schema definitions
- Automatic migration operators
- Backwards compatibility checking

### F6: Observability

Built-in monitoring:
- Throughput metrics per stage
- Latency percentiles
- Backpressure indicators
- Dead letter queues for failed elements

## Testing Strategy

### Unit Tests

Each operator should have tests for:
- Empty stream input
- Single element input
- Normal operation
- Boundary conditions (e.g., window boundaries)
- Early termination

### Property Tests

Invariants to verify:
- `count(stream) == fold(stream, 0, |acc, _| acc + 1)`
- `take(n) ∘ drop(n)` is identity for streams longer than 2n
- `map(f) ∘ map(g) == map(g ∘ f)` (functor law)
- `filter(p) ∘ filter(q) == filter(p && q)`

### Benchmark Tests

Performance baselines:
- Throughput: elements/second for simple pipelines
- Latency: time from emit to sink receipt
- Memory: peak allocation during processing
- Parallelism: scaling with worker count

## Glossary

| Term | Definition |
|------|------------|
| **Source** | A function that produces elements via `Emit<T>` effect |
| **Sink** | A handler that consumes elements to produce a final value |
| **Operator** | A handler that transforms elements (intercepts and re-emits) |
| **Pipeline** | Composition of source → operators → sink |
| **Backpressure** | Mechanism for slow consumers to signal fast producers |
| **Window** | Bounded subset of stream elements for aggregation |
| **Fiber** | Lightweight cooperative thread in Blood's runtime |
| **Channel** | Typed queue for inter-fiber communication |
| **Effect** | Typed operation that may have side effects |
| **Handler** | Code that interprets effect operations |

---

*Document Version: 1.0*
*Last Updated: January 2026*
