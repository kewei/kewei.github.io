---
title: Arc and Mutex
description: How to share data across threads using `Arc` and `Mutex`.
layout: post
author: "Kewei Zhang"
date: 2023-07-28 15:43:00 +0000
categories: [Rust, thread]
tags: [Rust, Arc, Mutex]
pin: true
math: true
mermaid: true
comments: true
image:
  path: /assets/img/2023-07-28/arc_mutex.png
  alt: Arc and Mutex
---

# Arc and Mutex
{: .mt-4 .mb-0 }

### The difference between Arc and Mutex

`Arc` (Atomic Reference Counting) and `Mutex` are both synchronization primitives in Rust, but they serve different purposes and have distinct use cases.

#### 1. Arc (Atomic Reference Counting)

- `Arc` is used for **shared ownership** of data across multiple threads.
    
- `Arc` allows you to create multiple immutable references to the same data. These references can be safely shared among threads. The data itself is immutable as long as there is at least one `Arc` reference to it.
    
- Use `Arc` when you need to share data between multiple threads and want to ensure the data remains immutable while being accessed concurrently.
    
- `Arc` is thread-safe; it implements atomic reference counting to ensure the count is correctly updated even with concurrent access.
    
- **Limitation:** Because `Arc` only provides shared **immutable** access, it is not suitable for scenarios where you need to mutate the shared data concurrently.
    

**Example:**

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3]);

    for _ in 0..3 {
        let data_clone = Arc::clone(&data);
        thread::spawn(move || {
            // Here, data_clone is an Arc reference to the shared data
            println!("{:?}", data_clone);
        });
    }
    // The Arc is dropped here, and data is deallocated when the last reference goes out of scope
}
```

#### 2. Mutex (Mutual Exclusion)

- `Mutex` is used for **exclusive ownership** of data across multiple threads.
    
- A `Mutex` protects data by ensuring that only one thread can access it at a time. When a thread acquires a "lock," it gains exclusive access; other threads attempting to access the data will be blocked until the lock is released.
    
- Use `Mutex` when you need to perform **mutable** operations on shared data while preventing data races.
    
- `Mutex` protects the "interior mutable" state of data structures to ensure thread safety.
    
- **Risk:** `Mutex` can introduce deadlocks if not used carefully (e.g., if a thread tries to acquire two locks in a different order than another thread).
    

**Example:**

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // We wrap the Mutex in an Arc so we can share the Mutex itself across threads
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));

    for _ in 0..3 {
        let data_clone = Arc::clone(&data);
        thread::spawn(move || {
            // Acquire the lock to mutate the data
            let mut data = data_clone.lock().unwrap();
            data.push(4);
            println!("{:?}", data);
        });
    }
}
```

---

### Summary Table

|**Feature**|**Arc**|**Mutex**|
|---|---|---|
|**Short for**|Atomic Reference Counting|Mutual Exclusion|
|**Primary Goal**|Shared ownership (Lifetime)|Exclusive access (Mutability)|
|**Access Type**|Immutable (Read-only)|Mutable (Read-Write)|
|**Thread Safety**|Atomic reference increments|Locking mechanism|

---

### Having Ownership vs. Mutability

Ownership in Rust applies to both mutable and immutable data. It ensures a single "manager" is responsible for deallocation. In the case of `Arc`, it’s not strictly about the data's mutability, but about **shared ownership**.

**Why ownership matters for `Arc`:**

1. **Shared Ownership:** Every `Arc` clone is an "owner." The data is only deallocated when the reference count reaches zero.
    
2. **Thread Safety:** `Arc` ensures the reference count is updated atomically (using CPU-level atomic operations) to prevent data races during the increment/decrement process.
    
3. **Immutability:** By default, `Arc` only allows read-only access. To change the data, you must nest a `Mutex` or `RwLock` inside the `Arc`.
    
4. **Lifetime Management:** Combined with Rust's borrowing rules, `Arc` ensures that no thread is left holding a reference to memory that has already been deallocated.