---
title: Lock Free Data Structure for High Speed Data Processing
description: Mutex works well for protecting data access in multi-threading, however, it slows down the system when one thread locks the data, which is essential for high frequency data and it does not allow multiple access simutaneously, like digital signal processing or real-time radio data. This article provides one solution for this purpose with a lock-free data structure, with Atomic.
layout: post
author: kewei
date: 2026-05-24 22:37:00 +0000
categories:
  - Rust
  - DSP
tags:
  - Lock-free
  - Threading
  - Atomic
  - Mutex
pin: true
math: true
mermaid: true
comments: true
image:
  path: /assets/img/2026-05-24/lock_vs_mutex.png
  alt: Lock-free vs Mutex
---

# Lock Free Data Structure for High Speed Data Processing
{: .mt-4 .mb-0 }
Mutex or RwLock is a good choice for sharing data across threads in safe Rust, where Mutex allows one writer or one reader at any time, and Rwlock allows a number of readers or one writer at any time. However, when you want to have one writer and a number of readers to access the data at same time, none of them would work. The cases are quite common in radio processing or audio processing, where you have one input that continues writing data, and you have some readers to use the data continuously, and neither of them should block others unless there is no enough data from input. For such kinds of applications, lock-free data structure is essential when the data is shared among the writing and reading threads.

Here, `MulticastRingBuffer` is a ring buffer that has a fixed size and a head that records the position of latest written samples. `Arc<MulticastRingBuffer>` is used across threads, which gives you a shared reference to the ring buffer, so we need **Interior Mutability** in order to write samples to the `buffer`.  `UnsafeCell` makes it possible to mutate the buffer through a shared reference across threads via `Arc`, and the compiler will know that you are gonna mutate the buffer so that it would not make optimization by assuming this reference is immutable.

```rust
struct MulticastRingBuffer {
    buffer: Vec<UnsafeCell<Complex32>>,
    mask: usize,
    head: AtomicUsize,
}

impl MulticastRingBuffer {
	fn write_samples(&self, samples: &[Complex32]) {
		unsafe {
			let ptr = self.buffer.as_ptr() as *mut Complex32;
		}
	}
}
```
`head` is the position of the writer  that is a `Atomic`, which allows to be mutated across threads and the value increases up to `USIZE_MAX` then wraps back to 0. `mask` is used to calculate its absolute position within the buffer. When writing samples, because `self` is a reference, we need to convert `buffer` to mutable pointer in order to writing samples to it.

When one thread tries to read the samples if the samples are not enough, the thread needs to wait using some functions, like `spin_loop`. `spin_loop` takes CPU and provides low latency. However, if you don't want high CPU usage when it waits for data. `Condvar` could be used to notify the other threads that the samples are available, like `MulticastRingBufferCondvar`.
```rust
struct MulticastRingBufferCondvar {
    buffer: Vec<UnsafeCell<Complex32>>,
    mask: usize,
    head: AtomicUsize,
    notifier: Mutex<bool>,
    condvar: Condvar,
}
```
After the thread writes new samples to the buffer, `condvar` will notify all reading threads that there are new samples, you can read now. So when writing samples, this code follows after the new samples are written to the buffer:
```rust
let mut guard = self.notifier.lock()?;
*guard = true;
self.condvar.notify_all();
```
And in the reading thread, it checks the head and its tail for available data, if there is no, it waits until got notified by the writing thread:
```rust
let mut head_guard = multi_ring_buf.notifier.lock()?;
while my_read_ptr < target {
	head_guard = multi_ring_buf.condvar.wait(head_guard)?;
}
```
This design saves CPU usage, but have bigger latency comparing to `spin_loop` due to Condvar needs to wake up via OS scheduler, which might takes several `us`, even longer with OS scheduler jitter. Therefore, the buffer needs to be large so that new data will not overwrite old data before it is processed.
This repository has the code for these design, and also benchmarked the processing time: https://github.com/kewei/Rust_projects/tree/main/lock_free_buffer_bench.

