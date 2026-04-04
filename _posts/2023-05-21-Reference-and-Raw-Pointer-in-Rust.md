---
title: Reference and Raw Pointer in Rust
description: Safety about reference and row pointer in Rust.
layout: post
author: "Kewei Zhang"
date: 2023-05-21 21:46:00 +0000
categories: [Rust, Raw pointer]
tags: [Rust, Raw pointer, Reference]
pin: true
math: true
mermaid: true
comments: true
image:
  path: /assets/img/2023-05-21/raw_pointer.jpg
  alt: Reference and Raw Pointer
---

# Reference and Raw Pointer in Rust
{: .mt-4 .mb-0 }

Rust's ownership feature is one of the key points that give memory safety guarantee to Rust, without considering a garbage collector. When you assign one value, `str1` that has a pointer in stack pointing to value "I am a string" storing at heap, to another one, `str2`:

```rust
let str1 = String::from("I am a string");
let str2 = str1;
```

You "moved" `str1` to `str2`, so from now on, `str1` is not valid anymore. This guarantees that you do not need free `str1` when it goes out of scope; in other words, `str2` takes ownership of that string from `str1`.

But what about if we want to assign `str1` to `str2`, but at same time not taking ownership from `str1`? Reference, denoted as `&`, can help you with that. Reference refers to one value, without taking ownership of it. Rust has guaranteed that a reference will not live longer than the content it refers to, so it ensures that you will always get a valid value when you use a reference — memory safety.

A reference is like a pointer, in which there is an address pointing at a memory storing actual value; and this value is owned by some other variable. Then, here comes the question: **what is the difference between a reference and a pointer?**

### 1. Safety and Nullability

A first big difference is that a reference is always pointing to a valid value for the life of that reference, but a raw pointer can point to null. When you try to dereference a null pointer, you will get an error, undefined behavior, or crash.

The reason that Rust still has raw pointers (despite having smart pointers like `Box` and `RefCell`) is for interacting with foreign languages, for instance C or C++. Creating a raw pointer in Rust is as safe as a reference, but dereferencing a raw pointer is not a safe action and must be done within an `unsafe` block.

**Example: Reference vs Pointer**

```rust
let num = 10;
let num_ptr: *const i32 = &num;
println!("num_ptr: {:?}", unsafe{ *num_ptr });

let num_ref: &i32 = &num;
println!("num_ref: {:?}", *num_ref);
```

**Output:**

```
num_ptr: 10
num_ref: 10
```

We can dereference a reference `num_ref` safely, but dereferencing the raw pointer `num_ptr` requires the `unsafe` keyword. This signals that the programmer takes responsibility for ensuring the pointer is not null or invalid.

### 2. Automatic Dereferencing

Another difference is that you can get the referred value by directly using the reference `num_ref`, but you only get the memory address if you use the pointer `num_ptr` without an explicit dereference:

```rust
let num = 10;
let num_ptr: *const i32 = &num;
println!("num_ptr: {:?}", num_ptr);

let num_ref: &i32 = &num;
println!("num_ref: {}", num_ref);
```

**Output:**

```
num_ptr: 0x7fffaf6a1ab4
num_ref: 10
```

### 3. Tricky Fact: Null Pointers and Mutable References

One tricky fact is that you can modify a pointer through a mutable reference even if that pointer was initially null. However, attempting to dereference a null double-pointer directly will result in a segmentation fault.

```rust
let mut num = 15;
let mut null_pointer: *mut *mut i32 = ptr::null_mut();
let mut null_temp: *mut i32 = ptr::null_mut();
let mut null_ref = &mut null_temp;

*null_ref = &mut num as *mut _;
println!("null_ref: {}", unsafe { **null_ref });

// unsafe {*null_pointer = &mut num as *mut _;} // This would cause a crash/Not valid
```

**Output:**

```
null_ref: 15
```

In this case, we successfully modified the first-level pointer `null_temp` through the mutable reference `null_ref`. However, it remains the programmer's responsibility to ensure a pointer is valid before it is ever dereferenced.