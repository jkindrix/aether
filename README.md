# Aether: Reactive Stream Processing for Blood

**Aether** is a reactive stream processing framework written in Blood, demonstrating the language's unique capabilities for building type-safe, concurrent data pipelines.

## Overview

Aether showcases Blood's five core innovations:

1. **Algebraic Effects** — Streams are defined using typed effects (`Emit<T>`, `Pull<T>`, `Accumulate<S>`), making side effects explicit and composable
2. **Fiber Concurrency** — Parallel stream processing using lightweight fibers with channel-based communication
3. **Multiple Dispatch** — Polymorphic operators that work across different element types
4. **Content-Addressed Code** — Pipeline definitions can be hashed for caching and incremental computation
5. **Generational Memory Safety** — Safe references between processing stages without garbage collection

## Core Concepts

### Effects-Based Stream Model

Unlike traditional iterator-based streams, Aether uses algebraic effects to model data flow:

```blood
effect Emit<T> {
    op emit(value: T) -> ();
}

// A source is a function that performs Emit effects
fn range(start: i32, end: i32) / {Emit<i32>} {
    let mut i: i32 = start;
    while i < end {
        perform Emit.emit(i);
        i = i + 1;
    }
}
```

### Operators as Effect Handlers

Stream transformations are implemented as effect handlers that intercept and re-emit values:

```blood
// Transform each element
fn map<T, U>(stream: fn() / {Emit<T>}, f: fn(T) -> U) / {Emit<U>} {
    handle stream() {
        Emit.emit(value) => {
            perform Emit.emit(f(value));
            resume(());
        }
    };
}

// Filter elements by predicate
fn filter<T>(stream: fn() / {Emit<T>}, pred: fn(T) -> bool) / {Emit<T>} {
    handle stream() {
        Emit.emit(value) => {
            if pred(value) {
                perform Emit.emit(value);
            }
            resume(());
        }
    };
}
```

### Composable Pipelines

Pipelines are built by nesting effect-producing functions:

```blood
fn pipeline_example() -> i64 {
    // Compute sum of squares of even numbers from 1-100
    sum(|| {
        map(|| {
            filter(|| {
                range(1, 101);
            }, |x| x % 2 == 0);
        }, |x| x * x);
    })
}
```

## Project Structure

```
aether/
├── src/
│   ├── prelude.blood      # Unified import module (API reference)
│   ├── core.blood         # Core effects and types
│   ├── sources.blood      # Stream source generators
│   ├── sinks.blood        # Terminal operations (fold, collect, etc.)
│   ├── operators.blood    # Stream transformations
│   ├── concurrent.blood   # Fiber-based parallel processing
│   ├── backpressure.blood # Flow control mechanisms
│   ├── channels.blood     # Typed channel abstractions
│   └── windowing.blood    # True sliding window support
├── examples/
│   ├── fibonacci_stream.blood   # Simple, compilable example
│   ├── basic_streams.blood      # Fundamental stream operations
│   ├── concurrent_pipeline.blood # Parallel processing demo
│   └── event_processing.blood   # Complex event processing (CEP)
├── DESIGN.md              # Architecture decisions and constraints
└── README.md
```

## Modules

### `core.blood` — Effect Definitions

Defines the fundamental effects for stream processing:

- `Emit<T>` — Push-based value emission
- `Pull<T>` — Pull-based value consumption (backpressure)
- `Accumulate<S>` — Stateful transformations
- `StreamError<E>` — Typed error handling
- `Lifecycle` — Stream lifecycle hooks

### `sources.blood` — Stream Sources

Generators that produce stream data:

- `range(start, end)` — Integer sequences
- `fibonacci(n)` — Fibonacci numbers
- `primes_up_to(limit)` — Prime number generation
- `repeat_n(value, count)` — Repetition
- `generate(count, f)` — Custom generators
- `unfold(state, f)` — Stateful unfolding

### `sinks.blood` — Terminal Operations

Consumers that reduce streams to values:

- `sum`, `product` — Arithmetic reductions
- `min`, `max`, `average` — Statistics
- `count`, `count_if` — Counting
- `fold`, `fold_indexed` — General reduction
- `first`, `last`, `nth` — Element access
- `all`, `any`, `none` — Boolean predicates
- `for_each`, `drain` — Side-effect execution

### `operators.blood` — Transformations

Intermediate operations that transform streams:

- `map`, `map_indexed` — Element transformation
- `filter`, `filter_not`, `filter_map` — Selection
- `take`, `take_while` — Prefix operations
- `drop`, `drop_while` — Skip operations
- `scan`, `running_sum` — Running accumulations
- `dedup`, `dedup_by` — Deduplication
- `enumerate` — Index pairing
- `flat_map` — Nested stream flattening
- `inspect` — Debugging/side effects

### `concurrent.blood` — Parallel Processing

Fiber-based concurrency primitives:

- `parallel_map` — Multi-worker transformation
- `parallel_filter` — Concurrent filtering
- `fan_out`, `fan_in` — Stream splitting/merging
- `batch` — Element batching
- `rate_limit`, `throttle` — Timing control
- `FiberPool` — Worker pool management

### `backpressure.blood` — Flow Control

Mechanisms for handling producer/consumer speed mismatches:

- `buffer` — Bounded buffering with strategies
- `conflate` — Latest-only sampling
- `sample` — Periodic sampling
- `debounce` — Quiet-period filtering
- `windowed_sum` — Time/count windows

### `channels.blood` — Typed Channel Abstraction

Type-safe wrappers around Blood's channel primitives:

- `ChannelHandle<T>` — Typed channel handle
- `Sender<T>`, `Receiver<T>` — Split channel halves
- `channel_pair<T>` — Create sender/receiver pair
- `BroadcastChannel<T>` — One-to-many delivery
- `RingChannel<T>` — Ring buffer (overwrites oldest)

### `windowing.blood` — True Sliding Windows

Proper windowed aggregation with element history:

- `tumbling_window` — Non-overlapping fixed windows
- `sliding_window` — Overlapping windows with configurable slide
- `session_window` — Gap-based dynamic windows
- `moving_average` — Sliding average computation
- `exponential_moving_average` — EMA with configurable smoothing

### `prelude.blood` — Unified API Reference

Single-file reference for all public types and effects. When Blood's module system is complete, this will be the import target:
```blood
use aether::prelude::*;
```

## Examples

### Basic Stream Processing

```blood
// Sum of even numbers squared
fn example() -> i64 {
    sum(|| {
        map(|| {
            filter(|| {
                range(1, 11);
            }, |x| x % 2 == 0);
        }, |x| x * x);
    })
    // range: 1,2,3,4,5,6,7,8,9,10
    // filter: 2,4,6,8,10
    // map: 4,16,36,64,100
    // sum: 220
}
```

### Concurrent Sensor Processing

```blood
// Process sensor data with parallel normalization
fn process_sensors() -> PipelineSummary {
    collect_summary(|| {
        aggregate_by_sensor(|| {
            filter_alerts(|| {
                parallel_normalize(|| {
                    sensor_source(4, 100);
                }, 4);  // 4 worker fibers
            });
        });
    })
}
```

### Complex Event Processing

```blood
// Financial event monitoring with pattern detection
fn monitor_trading() -> AlertSummary {
    summarize_alerts(|| {
        detect_price_spikes(|| {
            compute_metrics(|| {
                trades_only(|| {
                    market_event_source(4, 200);
                });
            });
        });
    })
}
```

## Design Philosophy

### Why Effects?

Traditional stream libraries use callbacks, iterators, or monads. Aether uses algebraic effects because:

1. **Type Safety** — Effects appear in function signatures, making data flow explicit
2. **Composability** — Handlers compose naturally without monad transformers
3. **Control Flow** — Effects provide delimited continuations for complex patterns
4. **Testability** — Effects can be mocked by providing alternative handlers

### Why Blood?

Blood's unique combination of features makes it ideal for stream processing:

- **No GC pauses** — Generational references provide predictable latency
- **Typed effects** — Stream operations are type-checked at compile time
- **Fiber concurrency** — Lightweight parallelism for high-throughput pipelines
- **Value semantics** — Safe data passing between pipeline stages
- **Multiple dispatch** — Operators work polymorphically across types

## Performance Considerations

- **Zero-allocation operators** — Transformations don't allocate intermediate collections
- **Early termination** — Operations like `take`, `first`, `any` short-circuit
- **Parallel execution** — Multiple fibers process elements concurrently
- **Backpressure** — Bounded buffers prevent memory exhaustion
- **Batching** — Amortize overhead for high-throughput scenarios

## Future Directions

- **Async I/O integration** — Connect to Blood's io_uring/kqueue runtime
- **Distributed streams** — Cross-node stream processing
- **Persistent streams** — Integration with content-addressed storage
- **SQL-like operators** — Join, group-by, window functions
- **Schema evolution** — Type-safe stream versioning

## Compiler Compatibility

| Feature | Status | Notes |
|---------|--------|-------|
| Generic effects `Emit<T>` | ✓ Supported | Core streaming primitive |
| Effect handlers | ✓ Supported | `handle stream() { ... }` |
| Closures `\|\| { ... }` | ✓ Supported | Pipeline composition |
| Generic structs | ✓ Supported | `ChannelHandle<T>`, etc. |
| Enums with payloads | ✓ Supported | `Option<T>`, `Result<T,E>` |
| Fixed-size arrays | ✓ Supported | `[i32; 64]` for buffers |
| Index-based loops | ✓ Supported | All loops use `while` |
| I/O effects | Awaiting | Print stubs placeholder |

**Note:** Each example redefines core types for self-containment. When Blood's module system is complete, examples will use `use aether::prelude::*`.

## Known Constraints

See [DESIGN.md](DESIGN.md) for detailed architecture decisions and constraints:

- **Window size:** Sliding windows limited to 64 elements (fixed-size ring buffers)
- **Worker pools:** Maximum 32 parallel workers
- **Channel selectors:** Maximum 8 channels in select operations

## Documentation

- **[DESIGN.md](DESIGN.md)** — Architecture decisions, constraints, and future directions
- **[prelude.blood](src/prelude.blood)** — Complete API reference with all public types

## License

Aether is part of the Blood ecosystem and shares its licensing terms.

---

*Aether demonstrates how Blood's algebraic effects enable elegant, type-safe stream processing with first-class support for concurrency and backpressure.*
