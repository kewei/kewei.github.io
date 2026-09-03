---
title: Lvalues, Rvalues, and Rvalue References in C++
description: Understand the distinction between lvalues, rvalues, lvalue references, and rvalue references.
layout: post
author: Kewei Zhang
date: 2026-09-03
categories:
  - C++
tags:
  - lvalue
  - rvalue
pin: true
math: true
mermaid: true
comments: true
image: 
	path: /assets/img/2026-09-03/lvalue_rvalue.jpeg   
	alt: lvalue and rvalue
---

# C++ Lvalues, Rvalues, and Rvalue References
{: .mt-4 .mb-0 }

The main reason C++ has rvalue references is **performance**: they allow expensive resources to be transferred instead of copied when the original object is no longer needed, i.e., ownership transfer.

---

## 1. The problem with copying

Copying is sometimes expensive.

Here:

```cpp
std::vector<int> v1(1'000'000);
std::vector<int> v2 = v1;
```

The vector contains one million integers.

A copy potentially requires:

```text
v1 ──> [1 2 3 4 ... 1,000,000]
              |
              | copy
              v
v2 ──> [1 2 3 4 ... 1,000,000]
```

But what if we don't need `v1` anymore?

For example:

```cpp
std::vector<int> createData()
{
    std::vector<int> data(1'000'000);

    return data;
}
```

When `createData()` returns, `data` is going away anyway.

It would be wasteful to copy one million integers just to create the return value.

That's where move semantics come in: **transfer the resources** from one object to another.

---

## 2. Copy versus move

Here:

```cpp
std::string str1 = "hello";

std::string str2 = str1;
```

This asks for a **copy**.

Conceptually:

```text
str1 ──> "hello"
       |
       | copy
       v
str2 ──> "hello"
```

But:

```cpp
std::string str2 = std::move(str1);
```

says:

> "I don't need the current contents of `str1` anymore. It is okay to transfer its resources to `str2`."

Conceptually:

```text
Before:

str1 ──> [resource]


After:

str1 ──> [moved-from state]

str2 ──> [resource]
```

For a `std::string`, the implementation can typically transfer ownership of an internal character buffer instead of copying every character.

This can be much cheaper.

---

## 3. What does `std::move()` actually do?

Actually:

> **`std::move()` does not actually move anything.**

For example:

```cpp
std::string str1 = "hello";

std::move(str1);
```

doesn't move the string.

Instead, `std::move(str1)` essentially tells the compiler:

> "Treat `str1` as an rvalue."

This allows an operation such as a move constructor or move assignment operator to be selected.

For example:

```cpp
std::string str2 = std::move(str1);
```

Here:

1. `std::move(str1)` turns the expression into an rvalue.
    
2. `std::string` sees an rvalue.
    
3. Its move constructor can be selected.
    
4. The string's resources can be transferred from `str1` to `str2`.
    

So `std::move()` is better thought of as a **permission/request to allow moving from an object**, not as the operation that performs the move.

---

## 4. So what is an lvalue?

A simple way to think about an **lvalue** is:

> An expression that refers to an object with an identifiable identity.

For example:

```cpp
std::string a = "hello";

a
```

`a` is an lvalue.

It is an object that has a persistent identity and can continue to exist after the current expression.

You can also write:

```cpp
a += " world";
```

because `a` refers to an actual object that you can modify.

---

## 5. What is an rvalue?

An rvalue is roughly a temporary value or an expression whose result is intended to be consumed rather than treated as a persistent object.

For example:

```cpp
std::string a = "hello";

std::string b = std::string("world");
```

The expression:

```cpp
std::string("world")
```

creates a temporary object.

That temporary is an rvalue.

Another example:

```cpp
A{}
```

creates a temporary `A` object.

This is why:

```cpp
A a{};
```

and:

```cpp
A&& a = A{};
```

mean very different things.

---

## 6. What is an lvalue reference?

An lvalue reference uses `&`:

```cpp
A a{};

A& b = a;
```

`b` is a reference to `a`.

There is only **one `A` object**:

```text
       ┌─────────────┐
a ─────┤             │
b ─────┤  A object   │
       └─────────────┘
```

`b` is essentially another name for the same object.

Therefore:

```cpp
b = ...;
```

modifies the same object referred to by `a`.

This is very different from:

```cpp
A b = a;
```

which creates a new object.


You might naturally try:

```cpp
A& b = A{};
```

But this is not allowed.

Because `A{}` is a temporary rvalue, while `A&` is an **lvalue reference**.

An ordinary lvalue reference expects an lvalue.

So:

```cpp
A a{};

A& b = a;      // OK
A& c = A{};    // ERROR
```

---

## 7. The rvalue reference: `&&`

C++ introduced **rvalue references**:

```cpp
A&& b = A{};
```

Now the reference can bind to an rvalue.

So:

```cpp
A&  b = a;      // lvalue reference
A&& c = A{};    // rvalue reference
```

The basic idea is:

```text
A&

"I am referencing an existing object."


A&&

"I am referencing something that can be treated as a temporary/movable value."
```

This enables move semantics.

---

## 8. A surprising example

Consider:

```cpp
std::string a = "hello";

std::string&& b = std::move(a);
```

It is tempting to think:

> "`b` is the moved version of `a`."

But **NO: Nothing has been moved here.**

`b` is simply a reference to `a`.

Conceptually:

```text
       ┌───────────────┐
a ─────┤               │
b ─────┤ "hello"       │
       └───────────────┘
```

There is still only one string object.

You can even do:

```cpp
b += " world";

std::cout << a;
```

and get:

```text
hello world
```

because `a` and `b` refer to the same object.

**Comparing to:**

```cpp
std::string b = std::move(a);
```

Creates a new object and allows it to take resources from `a`.

```text
a ──> moved-from object

b ──> original resources
```

Two objects. And afterwards, `a` is still valid, with empty string, something else, but not the original resource anymore.

---

## 9. The real purpose of `&&`

The syntax may look complicated:

```cpp
Buffer(Buffer&& other);
```

But the purpose is actually quite simple.

It allows a class to provide two different behaviors:

```cpp
Buffer(const Buffer& other);  // copy
Buffer(Buffer&& other);       // move
```

Then:

```cpp
Buffer a;

Buffer b = a;
```

selects the copy constructor.

While:

```cpp
Buffer c = std::move(a);
```

selects the move constructor.

So C++ can distinguish:

```text
"I need another independent copy."

            versus

"I don't need this object anymore;
you can take its resources."
```

---

## 10. One confusing rule

Here, we have:

```cpp
std::string&& b = std::move(a);
```

Even though `b` has type:

```cpp
std::string&&
```

the expression:

```cpp
b
```

is an **lvalue**.

For example:

```cpp
void foo(std::string&);
void foo(std::string&&);

std::string a = "hello";
std::string&& b = std::move(a);

foo(b);            // calls foo(std::string&)
foo(std::move(b)); // calls foo(std::string&&)
```

Because `b` is a **named variable**.

The fact that its type is `std::string&&` does not make the expression `b` an rvalue.

