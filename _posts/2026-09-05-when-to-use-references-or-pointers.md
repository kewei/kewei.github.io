---
title: When to Use References or Pointers in C++
description: If possibly, using references not pointers, unless it needs to be nullptr or it needs to point to a different object.
layout: post
author: kewei
date: 2026-09-05 11:55:00 +0200
categories:
  - C++
tags:
  - Reference
  - Pointer
pin: true
math: true
mermaid: true
comments: true
image:
  path: /assets/img/2026-09-05/reference_pointer.jpeg
  alt: Reference and Pointer
---

# Use References Instead of Pointers in C++
{: .mt-4 .mb-0 }

When working with objects in C++, you often have a choice between passing an object using a **pointer** or using a **reference**.

For example:

```cpp
void updateValue(int* value);
```

and:

```cpp
void updateValue(int& value);
```

Both can allow a function to modify the original object. However, they communicate different things and have different safety properties.

As a general rule:

> **Use a reference when the function requires a valid object. Use a pointer when `nullptr` is a meaningful possibility or when pointer semantics are actually needed.**


---

## 1. A Reference Must Refer to an Object

Consider a function that increments an integer.

Using a pointer:

```cpp
void increment(int* value)
{
    ++(*value);
}
```

The caller needs to explicitly pass an address:

```cpp
int x = 10;

increment(&x);
```

Inside the function, we have to dereference the pointer:

```cpp
++(*value);
```

There is also a potential problem:

```cpp
increment(nullptr);
```

The function cannot safely dereference `nullptr`.

With a reference, the function becomes:

```cpp
void increment(int& value)
{
    ++value;
}
```

The caller simply writes:

```cpp
int x = 10;

increment(x);
```

There is no explicit address-taking and no dereferencing.

This is one of the main advantages of references: **the function receives an object rather than a memory address.**

---

## 2. Pointers Can Be `nullptr`, References Cannot

A pointer can represent two states:

```cpp
int* p = &x;
```

or:

```cpp
int* p = nullptr;
```

This makes pointers useful when "no object" is a valid state.

For example:

```cpp
Employee* findEmployee(int id);
```

The function might return:

```cpp
return &employee;
```

if the employee exists, or:

```cpp
return nullptr;
```

if it doesn't.

A reference cannot represent this situation.

```cpp
Employee& findEmployee(int id);
```

There is no valid equivalent of:

```cpp
return nullptr;
```

Therefore, if a function requires an object to exist, a reference is often a better interface.

For example:

```cpp
void printEmployee(const Employee& employee);
```

The function can assume that `employee` refers to a valid object.

---

## 3. References Make Function Calls Cleaner

Compare these two functions:

```cpp
void update(int* value)
{
    *value += 1;
}
```

and:

```cpp
void update(int& value)
{
    value += 1;
}
```

The pointer version requires `&` at the call site:

```cpp
int x = 10;

update(&x);
```

The reference version looks like an ordinary variable:

```cpp
int x = 10;

update(x);
```

The reference version is generally easier to read.

It also avoids the extra `*` required when accessing the object through a pointer.

---

## 4. But `&` at the Call Site Does NOT Tell You Whether an Object Is Modified


You may assume that:

```cpp
foo(x);
```

means "pass by value", while:

```cpp
foo(&x);
```

means "the function will modify `x`."

That's not a reliable way to reason about C++ code.

Consider these four functions:

```cpp
void f1(const int* value);
void f2(int* value);
void f3(const int& value);
void f4(int& value);
```

All four can receive an existing object, but they have different semantics.

### `const int*`

```cpp
void f1(const int* value)
{
    std::cout << *value << '\n';
}
```

The function can read the integer, but cannot modify it through this pointer.

Call:

```cpp
int x = 10;

f1(&x);
```

### `int*`

```cpp
void f2(int* value)
{
    *value += 1;
}
```

The function can modify the object.

```cpp
int x = 10;

f2(&x);
```

### `const int&`

```cpp
void f3(const int& value)
{
    std::cout << value << '\n';
}
```

The function can read the object but cannot modify it.

```cpp
int x = 10;

f3(x);
```

### `int&`

```cpp
void f4(int& value)
{
    value += 1;
}
```

The function can modify the object.

```cpp
int x = 10;

f4(x);
```

So you cannot determine whether a function modifies an object simply by looking at the function call.

You need to look at the **function declaration**.

|Parameter|Can modify object?|Can represent no object?|
|---|--:|--:|
|`T*`|Yes|Yes (`nullptr`)|
|`const T*`|No|Yes (`nullptr`)|
|`T&`|Yes|No|
|`const T&`|No|No|

This is why the function prototype is so important.

---

## 5. `const T&` Is Especially Useful

One of the most common uses of references is passing a large object without copying it.

Suppose we have:

```cpp
void printName(std::string name)
{
    std::cout << name << '\n';
}
```

Passing by value potentially creates a copy of the string.

Instead, we can use a const reference:

```cpp
void printName(const std::string& name)
{
    std::cout << name << '\n';
}
```

Now the function receives a reference to the existing string.

The `const` tells us that the function cannot modify it.

For example:

```cpp
void printName(const std::string& name)
{
    // name += " Smith";   // Error
    std::cout << name << '\n';
}
```

This is a very common C++ pattern:

```cpp
const T&
```

means roughly:

> "I need to access an existing object, but I don't need to modify it."

---

## 6. Use `T&` When the Function Needs to Modify the Object

Suppose we want a function to normalize a vector:

```cpp
void normalize(std::vector<double>& values)
{
    // modify values
}
```

The caller can simply write:

```cpp
std::vector<double> values{1.0, 2.0, 3.0};

normalize(values);
```

The function operates on the original vector.

There is no need to explicitly take its address:

```cpp
normalize(&values);
```

and no need to dereference it inside the function.

This makes the intention relatively clear from the function declaration:

```cpp
void normalize(std::vector<double>& values);
```

The function may modify `values`.

---

## 7. References Also Clarify Ownership

Another important advantage of references is that they make ownership expectations clearer.

Consider:

```cpp
void process(Resource* resource);
```

You might wonder:

- Who owns `resource`?
    
- Should `process()` delete it?
    
- Is the pointer allowed to be `nullptr`?
    
- Does `process()` store the pointer?
    
- Is the object managed somewhere else?
    

The function declaration alone doesn't answer all of these questions.

Now consider:

```cpp
void process(Resource& resource);
```

This strongly communicates:

> "Give me an existing `Resource`. I will use it, and I am not being given ownership of it."

For example:

```cpp
Resource resource;

process(resource);
```

There is no implication that `process()` should destroy the object.

The caller owns the lifetime of the object.

---

## 8. A Reference Does Not Give Ownership

A reference is **not an ownership mechanism**.

Consider:

```cpp
void process(Resource& resource)
{
    // use resource
}
```

The function does not own `resource`.

It should not do something like:

```cpp
delete &resource;  // WRONG
```

The object may be a stack object:

```cpp
Resource resource;

process(resource);
```

It may also be owned by another object:

```cpp
std::unique_ptr<Resource> resource =
    std::make_unique<Resource>();

process(*resource);
```

In both cases, `process()` simply uses the resource.

This is one of the useful properties of references:

> **A reference normally expresses access without ownership.**

---

## 9. What About Pointers and Ownership?

Pointers are not necessarily ownership mechanisms either.

For example:

```cpp
void process(Resource* resource);
```

does **not** automatically mean that `process()` owns the resource.

However, the ownership relationship is less obvious.

Modern C++ provides smart pointers to express ownership explicitly.

For example:

```cpp
void process(std::unique_ptr<Resource> resource);
```

can communicate ownership transfer.

Or:

```cpp
void process(std::shared_ptr<Resource> resource);
```

can communicate shared ownership.

Therefore, modern C++ code generally prefers smart pointers when dynamic ownership is required.

For non-owning access, references are often preferable when `nullptr` is not needed.

---

## 10. When Should Use a Pointer?


Pointers are appropriate when a pointer's semantics are useful.

### Nullable objects

If the function legitimately accepts "no object":

```cpp
void process(Resource* resource)
{
    if (resource == nullptr)
    {
        return;
    }

    resource->process();
}
```

The caller can write:

```cpp
Resource* resource = nullptr;

process(resource);
```

A reference cannot express this state.

---

### Optional results

Pointers are also commonly used to represent an optional result:

```cpp
Resource* findResource(int id);
```

The result might be:

```cpp
Resource* resource = findResource(42);

if (resource != nullptr)
{
    resource->process();
}
```

Here, `nullptr` has a clear meaning:

> "The resource was not found."

---

### Pointer arithmetic and arrays

Pointers are also fundamental when working with contiguous memory or low-level APIs:

```cpp
void process(int* begin, int* end)
{
    for (int* p = begin; p != end; ++p)
    {
        // ...
    }
}
```

Although modern C++ often provides better abstractions such as `std::span`, pointers remain important for low-level programming.


---

## 11. A Practical Rule

When designing a function interface, you can often start with this decision process.

### Does the function require an object, not modify it?

If yes, consider a reference:

```cpp
void process(const Resource& resource);
```

### Does the function need to modify the object?

Use a non-const reference:

```cpp
void process(Resource& resource);
```

### Does "no object" have a meaningful interpretation?

Use a pointer:

```cpp
void process(Resource* resource);
```

### Does the function own the object?

Consider a smart pointer such as:

```cpp
std::unique_ptr<Resource>
```

or:

```cpp
std::shared_ptr<Resource>
```

depending on the ownership model.

---

## 12. A Complete Example

Consider a simple `Sensor` class:

```cpp
#include <iostream>
#include <string>

class Sensor
{
public:
    explicit Sensor(std::string name)
        : mName(std::move(name))
    {
    }

    void setValue(double value)
    {
        mValue = value;
    }

    double value() const
    {
        return mValue;
    }

    const std::string& name() const
    {
        return mName;
    }

private:
    std::string mName;
    double mValue{0.0};
};
```

We can then write:

```cpp
void printSensor(const Sensor& sensor)
{
    std::cout << sensor.name()
              << ": "
              << sensor.value()
              << '\n';
}
```

Because the parameter is:

```cpp
const Sensor&
```

the function can inspect the sensor but cannot modify it.

For modifying the sensor:

```cpp
void resetSensor(Sensor& sensor)
{
    sensor.setValue(0.0);
}
```

And if the sensor is optional:

```cpp
void resetSensor(Sensor* sensor)
{
    if (sensor != nullptr)
    {
        sensor->setValue(0.0);
    }
}
```

These three interfaces communicate different contracts:

```cpp
void printSensor(const Sensor& sensor);
void resetSensor(Sensor& sensor);
void resetSensor(Sensor* sensor);
```

The choice of `const T&`, `T&`, or `T*` tells the caller something important about what the function expects.

---

## Takeaway

References are often the better choice when a function needs to work with an existing object that must exist.

They provide several advantages:

- They cannot be `nullptr`.
    
- They avoid explicit address-taking at the call site.
    
- They avoid explicit dereferencing inside the function.
    
- They make non-ownership easier to understand.
    
- `const T&` provides an efficient way to read an existing object without copying it.
    
- `T&` clearly allows a function to modify an existing object.
    


Use a pointer when:

- `nullptr` is meaningful.
    
- You need pointer semantics.
    
- You are working with low-level memory.
    
- You are interfacing with APIs that use pointers.
    
- A pointer is part of the ownership or lifetime design.
    

And when ownership is involved, prefer modern C++ smart pointers rather than raw `new` and `delete`.