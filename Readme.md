# pumps-rs

Eager streams for Rust. If a stream allows water to flow down the hill, a pump forces it up.

[![Crates.io](https://img.shields.io/crates/v/pumps)](https://crates.io/crates/pumps)
[![Documentation](https://docs.rs/pumps/badge.svg)](https://docs.rs/pumps)

<p align="center">
    <img src="https://github.com/user-attachments/assets/1b01e3a8-f1a6-47dd-8f0e-804ff3c9a32a">
</p>

[Futures stream api](https://docs.rs/futures/latest/futures/stream/index.html#) is awesome, but has unfortunate issues

- Futures run in surprising and unintuitive order. Read about [Barbara battles buffered streams](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_battles_buffered_streams.html)
- Prone to surprising deadlocks. [Fixing the Next Thousand Deadlocks: Why Buffered Streams Are Broken and How To Make Them Safer](https://blog.polybdenum.com/2022/07/24/fixing-the-next-thousand-deadlocks-why-buffered-streams-are-broken-and-how-to-make-them-safer.html)
- [Rust Stream API visualized and exposed](https://github.com/alexpusch/rust-magic-patterns/blob/master/rust-stream-visualized/Readme.md)
- Fixing these issues will require added features to the language - [poll_progress](https://without.boats/blog/poll-progress/)

This crate offers an alternative approach for rust async pipelining.

Main features:

- Designed for common async pipelining needs in heart
- Explicit concurrency, ordering, and backpressure control
- Eager - work is done before downstream methods consumes it
- builds on top of Rust async tools as tasks and channels.
- For now only supports the Tokio async runtime
- TBA
    - [ ] additional operators

Example:

```rust
let (mut output_receiver, join_handle) = pumps::Pipeline::from_iter(urls)
    .map(get_json, Concurrency::concurrent_ordered(5).backpressure(100))
    .map(download_heavy_resource, Concurrency::serial())
    .filter_map(run_algorithm, Concurrency::concurrent_unordered(5).backpressure(10))
    .map(save_to_db, Concurrency::concurrent_unordered(100))
    .build();

while let Some(output) = output_receiver.recv().await {
    println!("{output}");
}
```

## Pumps

A `Pump` is a wrapper around a common async programming (or rather multithreading) pattern - concurrent work is split into several tasks that communicate with each other using channels

```rust
let (sender0, mut receiver0) = mpsc::channel(100);
let (sender1, mut receiver1) = mpsc::channel(100);
let (sender2, mut receiver2) = mpsc::channel(100);

tokio::spawn(async move {
    while let Some(x) = receiver0.recv().await {
        let output = work0(x).await;
        sender1.send(output).await.unwrap();
    }
});

tokio::spawn(async move {
    while let Some(x) = receiver1.recv().await {
        let output = work1(x).await;
        sender2.send(output).await.unwrap();
    }
});

// send data to input channel
send_input(sender0).await;

while let Some(output) = receiver2.recv().await {
    println!("done with {}", output);
}
```

A 'Pump' is one step of such pipeline - a task and input/output channel. For example the `Map` Pump spawns a task, receives input via a `Receiver`, runs an async function, and sends its output to a `Sender`

A `Pipeline` is a chain of `Pump`s. Each pump receives its input from the output channel of its predecessor

### Creation

```rust
// from channel
let (mut output_receiver, join_handle) = Pipeline::from(tokio_channel);

// from a stream
let (mut output_receiver, join_handle) = Pipeline::from_stream(stream);

// create an iterator
let (mut output_receiver, join_handle) = Pipeline::from_iter(iter);
```

The `.build()` method returns a touple of a `tokio::sync::mpsc::Receiver` and a join handle to the internally spawned tasks

### Concurrency control

Each Pump operation receives a `Concurrency` struct that defines the concurrency characteristics of the operation.

- serial execution - `Concurrency::serial()`
- concurrent execution - `Concurrency::concurrent_ordered(n)`, `Concurrency::concurrent_unordered(n)`

#### Backpressure

Backpressure defines the amount of unconsumed data can accumulate in memory. Without back pressure an eger operation will keep processing data and storing it in memory. A slow downstream consumer will result with unbounded memory usage.

The `.backpressure(n)` definition limits the output channel of a `Pump` allowing it to stop processing data until the output channel have been consumed.

The default backpressure is equal to the concurrency number

### Visual comparison with streams
To understand the difference in concurrency characteristics between Pumps and Stream lets visualize similar pipelines in both frameworks.
We will visualize a series of 3 ordered concurrent async jobs. Each square in the animation represents a single unit of work that flows between the different pipeline stages. For a deeper dive into the visualization check out this [blog post](https://github.com/alexpusch/rust-magic-patterns/blob/master/rust-stream-visualized/Readme.md)

With streams the pipeline looks something like:
```rust
input_stream
  .map(async_job)
  .buffered(3)
  .map(async_job)
  .buffered(3)
  .map(longer_async_job)
  .buffered(2)
```
<p align="center">
    <img src="https://github.com/user-attachments/assets/07ef26e4-5d7b-482e-a38c-9499ae51088c">
</p>

With Pumps it looks like:
```rust
pumps::Pipeline::from_iter(input)
    .map(async_job, Concurrency::concurrent_ordered(3))
    .map(async_job, Concurrency::concurrent_ordered(3))
    .map(longer_async_job, Concurrency::concurrent_ordered(2))
    .build();
```
<p align="center">
    <img src="https://github.com/user-attachments/assets/1b01e3a8-f1a6-47dd-8f0e-804ff3c9a32a">
</p>

main differences:
- Using streams, futures from different stages of the pipeline do not run concurrently. Using Pumps everything runs concurrently.
- Using streams, each stage waits for a downstream method to `poll_next` it before taking new work. Using Pumps each stage takes new jobs eagerly.
- Pumps allows for a configurable backpressure. The effect of this can be seen when a heavy task is slow to take new work. The previous stages continue to work until it accumulates `backpressure` number of results. 