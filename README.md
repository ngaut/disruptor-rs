![Crates.io](https://img.shields.io/crates/v/disruptor)
![Crates.io](https://img.shields.io/crates/d/disruptor)
![Build](https://github.com/nicholassm/disruptor-rs/actions/workflows/rust.yml/badge.svg)

# Disruptor

This library is a low latency, inter-thread communication library written in Rust.

It's heavily inspired by the brilliant
[Disruptor library from LMAX](https://github.com/LMAX-Exchange/disruptor).

# Getting Started

Add the following to your `Cargo.toml` file:

    disruptor = "2.0.0"

To read details of how to use the library, check out the documentation on [docs.rs/disruptor](https://docs.rs/disruptor).

Here's a minimal example:

```rust
use disruptor::*;

// The event on the ring buffer.
struct Event {
    price: f64
}

fn main() {
    // Factory closure for initializing events in the Ring Buffer.
    let factory = || { Event { price: 0.0 }};

    // Closure for processing events.
    let processor = |e: &Event, sequence: Sequence, end_of_batch: bool| {
        // Your processing logic here.
    };

    let size = 64;
    let mut producer = disruptor::build_single_producer(size, factory, BusySpin)
        .handle_events_with(processor)
        .build();

    // Publish into the Disruptor via the `Producer` handle.
    for i in 0..10 {
        producer.publish(|e| {
            e.price = i as f64;
        });
    }
}// At this point, the Producer instance goes out of scope and when the
 // processor is done handling all events then the Disruptor is dropped
 // as well.
```

The library also supports pinning threads on cores to avoid latency induced by context switching.
A more advanced usage demonstrating this and with multiple producers and multiple interdependent consumers could look like this:

```rust
use disruptor::*;
use std::thread;

struct Event {
    price: f64
}

fn main() {
    let factory = || { Event { price: 0.0 }};

    // Closure for processing events.
    let h1 = |e: &Event, sequence: Sequence, end_of_batch: bool| {
        // Processing logic here.
    };
    let h2 = |e: &Event, sequence: Sequence, end_of_batch: bool| {
        // Some processing logic here.
    };
    let h3 = |e: &Event, sequence: Sequence, end_of_batch: bool| {
        // More processing logic here.
    };

    let mut producer1 = disruptor::build_multi_producer(64, factory, BusySpin)
        // `h2` handles events concurrently with `h1`.
        .pined_at_core(1).handle_events_with(h1)
        .pined_at_core(2).handle_events_with(h2)
            .and_then()
            // `h3` handles events after `h1` and `h2`.
            .pined_at_core(3).handle_events_with(h3)
        .build();

    // Create another producer.
    let mut producer2 = producer1.clone();

    // Publish into the Disruptor.
    thread::scope(|s| {
        s.spawn(move || {
            for i in 0..10 {
                producer1.publish(|e| {
                    e.price = i as f64;
                });
            }
        });
        s.spawn(move || {
            for i in 10..20 {
                producer2.publish(|e| {
                    e.price = i as f64;
                });
            }
        });
    });
}// At this point, the Producers instances go out of scope and when the
 // processors are done handling all events then the Disruptor is dropped
 // as well.
```

# Features

- [x] Single Producer Single Consumer (SPSC).
- [x] Multi Producer Single Consumer (MPSC).
- [x] Multi Producer Multi Consumer (MPMC) with consumer interdependencies.
- [x] Busy-spin wait strategies.
- [x] Batch consumption of events.
- [x] Thread affinity can be set for the event processor thread(s).
- [x] Set thread name of each event processor thread.

# Design Choices

Everything in the library is about low-latency and this heavily influences all choices made in this library.
As an example, you cannot allocate an event and *move* that into the ringbuffer. Instead, events are allocated on startup to ensure they are co-located in memory to increase cache coherency.
However, you can still allocate a struct on the heap and move ownership to a field in the event on the Ringbuffer.
As long as you realize that this can add latency, because the struct is allocated by one thread and dropped by another.
Hence, there's synchronization happening in the allocator.

There's also no use of dynamic dispatch - everything is monomorphed.

# Performance

The SPSC and MPSC Disruptor variants have been benchmarked and compared to Crossbeam. See the code in the `benches/spsc.rs` and `benches/mpsc.rs` files.

The results below of the SPSC benchmark are gathered from running the benchmarks on a 2016 Macbook Pro running a 2,6 GHz Quad-Core Intel Core i7. So on a modern Intel Xeon the numbers should be even better. Furthermore, it's not possible to isolate cores on Mac and pin threads which would produce even more stable results. This is future work.

If you have any suggestions to improving the benchmarks, please feel free to open an issue.

To provide a somewhat realistic benchmark not only burst of different sizes are considered but also variable pauses between bursts: 0 ms, 1 ms and 10 ms.

The latencies below are the mean latency per element with 95% confidence interval (standard `criterion` settings). Capturing all latencies and calculating misc. percentiles (and in particular the max latency) is future work. However, I expect the below measurements to be representative for the actual performance you can achieve in a real application.

## No Pause Between Bursts

*Latency:*

|  Burst Size | Crossbeam | Disruptor | Improvement |
|------------:|----------:|----------:|------------:|
|           1 |     53 ns |     35 ns |         34% |
|          10 |     71 ns |     29 ns |         59% |
|         100 |     34 ns |     31 ns |          9% |

*Throughput:*

|  Burst Size |  Crossbeam |   Disruptor | Improvement |
|------------:|-----------:|------------:|------------:|
|           1 |  19.0M / s |   28.7M / s |         51% |
|          10 |  14.2M / s |   34.8M / s |        145% |
|         100 |  29.7M / s |   31.9M / s |          7% |

## 1 ms Pause Between Bursts

*Latency:*

|  Burst Size | Crossbeam |  Disruptor | Improvement |
|------------:|----------:|-----------:|------------:|
|           1 |     52 ns |      35 ns |         32% |
|          10 |     71 ns |      31 ns |         56% |
|         100 |     33 ns |      29 ns |         12% |

*Throughput:*

|  Burst Size |  Crossbeam | Disruptor | Improvement |
|------------:|-----------:|----------:|------------:|
|           1 |  19.0M / s | 28.5M / s |         50% |
|          10 |  14.1M / s | 32.6M / s |        131% |
|         100 |  30.6M / s | 34.4M / s |         12% |

## 10 ms Pause Between Bursts

*Latency:*

|  Burst Size | Crossbeam | Disruptor | Improvement |
|------------:|----------:|----------:|------------:|
|           1 |     56 ns |     35 ns |         38% |
|          10 |     75 ns |     29 ns |         61% |
|         100 |     35 ns |     31 ns |         11% |

*Throughput:*

|  Burst Size | Crossbeam | Disruptor | Improvement |
|------------:|----------:|----------:|------------:|
|           1 | 18.0M / s | 28.8M / s |         60% |
|          10 | 13.3M / s | 35.0M / s |        163% |
|         100 | 28.7M / s | 32.4M / s |         13% |

## Conclusion

There's clearly a difference between the Disruptor and the Crossbeam libs. However, this is not because the Crossbeam library is not a great piece of software. It is. The Disruptor trades CPU and memory resources for lower latency and higher throughput and that is why it's able to achieve these results.

Both libraries greatly improves as the burst size goes up but the Disruptor's performance is more resilient to the pauses between bursts which is one of the design goals.

# Related Work

There are multiple other Rust projects that mimic the LMAX Disruptor library:
1. [Turbine](https://github.com/polyfractal/Turbine)
2. [Disrustor](https://github.com/sklose/disrustor)

A key feature that this library supports is multiple producers from different threads that neither of the above libraries support (at the time of writing).

# Contributions

You are welcome to create a Pull-Request or open an issue with suggestions for improvements.

Changes are accepted solely at my discretion and I will focus on whether the changes are a good fit for the purpose and design of this crate.

# Roadmap

1. Add a Sleeping Wait Strategy.
2. Support for batch publication.
