---
title: References and Deref Coercion(Auto-dereferencing)
description: When and where to use or not use * operator for deref.
layout: post
author: "Kewei Zhang"
date: 2026-04-04 23:36:00 +0000
categories: [Rust, Reference]
tags: [Rust, Reference, Deref]
pin: true
math: true
mermaid: true
comments: true
image:
  path: /assets/img/2023-07-28/arc_mutex.png
  alt: Arc and Mutex
---

# References and Deref Coercion(Auto-dereferencing)
{: .mt-4 .mb-0 }

Deref coercion is a nice feature that allows you not have to write code like `***v` when you want to access data behind multiple layers of pointers. There are some data structures and some actions that trigger it.

### Data Structures
Basically any type implements the `Deref` or `DerefMut` trait can participate in auto-dereferencing. The most common ones include:

- **`String` → `&str`**: Allows you to pass a `&String` to a function expecting `&str`.
    
- **`Vec<T>` → `&[T]`**: Allows a reference to a Vector to be treated as a slice.
    
- **`Box<T>`, `Rc<T>`, `Arc<T>`**: These "transparent" pointers automatically defer to the inner type `T`.
    
- **`Ref<'a, T>` / `RefMut<'a, T>`**: Returned by `RefCell`, these allow access to the inner data while maintaining the borrow rules.

### Actions
#### A. Method Resolution (The Dot Operator)

This is the most common place you'll see it. When you call `value.method()`, Rust will:

1. Check if `value` has the method.
    
2. If not, try `&value` or `&mut value`.
    
3. If still not found, it **dereferences** `value` (using the `Deref` trait) and repeats the process.
    
4. It will "drill down" through as many layers as necessary (e.g., `&&&Box<String>` → `str`).
    

#### B. Special Function and Method Arguments

If a function signature asks for a specific reference type (like `&str`), and you provide a type that can dereference to it (like `&String`), the compiler performs **Deref Coercion** automatically.

```rust
fn hello(name: &str) { println!("Hello, {name}!"); }

let s = String::from("World");
hello(&s); // Works! &String is coerced to &str
```
These special argument types include aforementioned data structures, plus:
- `OsString` -> `&OsStr`, same like `String` and `&str`.
- `PathBuf` -> `&Path`

#### C. The `Index` Operator

When you use `container[index]`, Rust can auto-dereference the container. For example, if you have a `Box<[i32]>`, you can access elements with `boxed_slice[0]` because the `Box` dereferences to the slice.

### Not Always
#### Arithmetic
If you are trying to assign a value or compare two values, the types must match exactly. A `&i32` is not the same thing as an `i32`.

```rust
fn check_value(val: &i32) {
    // This fails: cannot compare &i32 to i32
    // if val == 10 { ... } 

    if *val == 10 { 
        println!("Match!"); 
    }
}
```


```rust
fn increment(num: &mut i32) {
    // You must dereference to reach the actual integer
    *num += 1; 
}
```

#### Assignment and Operators
Operators `+`, `-`, `==`, and `*=` usually require manual dereferencing (`*val`), except `.` and `[]`.
And for variable assignment like: `let x: i32 = &5;` will fail. You must write `let x: i32 = *&5`.