---
title: How to Use C Libraries with Bindgen in Rust?
description: Instructions on how to use Bindgen in Rust.
layout: post
author: "Kewei Zhang"
date: 2023-05-21 16:33:00 +0000
categories: [Rust, Bindgen]
tags: [Rust, Bindgen, C]
pin: true
math: true
mermaid: true
comments: true
image:
  path: /assets/img/2023-05-21/rust_c.png
  alt: Rust and C Binding
---

# How to Use C Libraries with Bindgen in Rust?
{: .mt-4 .mb-0 }


Although Rust is a decent language when considering security and speed (that's why I am learning it.), it is still a new language with not big community. There are many stable C libraries that we can use through bindings when there are no or unstable corresponding Rust libraries. In this blog, I will describe how to use [Bindgen](https://docs.rs/bindgen/latest/bindgen/) to link C libraries, the commands are for Ubuntu, should be working for Debain-based Linux. Bindgen is stable for C libraries, but not very stable for C++. [This](https://rust-lang.github.io/rust-bindgen/) is an official tutorial, but it is not really clear.

While providing C/C++ headers, Bindgen can generate Rust bindings to C/C++ libraries, and call C/C++ functions with Foreign Function Interface (FFI) code. One important thing is that Rust needs to know where to find the libraries.

In order for bindgen to work, we need `libclang` to preprocess, parse, and type check C/C++ header files, the official tutorial gives good guidance [here](https://rust-lang.github.io/rust-bindgen/requirements.html). In particular, in Ubuntu, we install the packages with:

```bash
sudo apt install llvm-dev libclang-dev clang
```

Then, we create a Rust project, in `Cargo.toml`, we add bindgen to [build-dependencies]:

```rust
[build-dependencies]
bindgen = "0.65.1" # This is the latest version now
```

build-dependencies declares other Cargo-based crates for the build script, so when you build your Rust project, it will generate the bindings for your own target/platform.

Next step is to create a file called: `wrapper.h` (could be at the project root directory), and add the header files you want to generate bindings for. In this example, we use libusb-1.0 as an illustration. If you do not have libusb-1.0 in your system. You can install it with:

```bash
sudo apt-get install libusb-1.0-0-dev
```

After installation, you can check where the library is installed with:

```bash
dpkg -L libusb-1.0-0 | grep \.so
```
You probably get something similar to this depending on your system:

```bash
/usr/lib/x86_64-linux-gnu/libusb-1.0.so.0
```

This path to the library .so will be needed later when we want Rust to link it.

In `wrapper.h`, we have this content, because in Ubuntu libusb.h is in `/usr/include/libusb-1.0/`:

```c++
#include <libusb-1.0/libusb.h>
```
So next step is that we need to tell Rust how to use build script to generate bindings for this above header. We create a `build.rs` at the project root directory. As the [official](https://doc.rust-lang.org/cargo/reference/build-scripts.html) page says, build scripts communicate with Cargo by printing to stdout. Cargo will interpret each line that starts with `cargo:` as an instruction that will influence compilation of the package. In `build.rs`, we need a `main()` function, inside this function, we first add a path for Cargo to search the libusb-1.0, which is the path we talked before: `/usr/lib/x86_64-linux-gnu`:

```c++
println!("cargo:rustc-link-search=/usr/lib/x86_64-linux-gnu");
```

This line of code tells Cargo to also search in this folder for libraries, it acts like `-L` to the compiler. Then, we add another line of code for the library we want to link:

```rust
println!("cargo:rustc-link-lib=usb-1.0");
```

This code passes `-l` to the complier, and tells Cargo the library name. This will link to dynamic library first if available, otherwise links to static library: [official explanation](https://doc.rust-lang.org/rustc/command-line-arguments.html#option-l-link-lib). One can also specify library type with one of dylib, static, framwork (macOS).

Next code we want to add to the build script is

```rust
println!("cargo:rerun-if-changed=wrapper.h");
```

This tells Cargo to monitor wrapper.h, when the files is modified, it rerun the build script. If this is a directory, it will scan the whole directory. If you don't want the build script to rerun, the following setting should be set:

```rust
println!("cargo:rerun-if-changed=build.rs")
```

The next step is to create a Builder for bindgen and configure settings for the needed bindings:

```rust
let bindings = bindgen::Builder::default()
    .header("wrapper.h")
    .parse_callbacks(Box::new(bindgen::CargoCallbacks))
    .generate()
    .expect("Unable to generate bindings");
```

Here we configure `wrapper.h` as the header file that bindgen would generate bindings for.

The last step is to write the generated bindings to one file, and later the package can include this file to use the bindings:

```rust
let output_path = PathBuf::from(env::var("OUT_DIR").unwrap());
bindings.write_to_file(output_path.join("bindings.rs"))
    .expect("Could not write bindings");
```

So in this cargo package, we can use this following code to include above generated bindings, and use the library that it links to.

```rust
include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

One can also use bindgen to generate bindings for a local/self-compiled library with header files, the steps will be:

- Include your header files in `wrapper.h`, remember to use correct path
    
- Set the `cargo:rustc-link-search` to the path that your library is located
    
- Set `cargo:rustc-link-lib` to your own library name
    

Until now, we have successfully generated bindings for the header, and linked to the library that defines functions for declarations in the header. And we could use the library in the package with above `include!`.

[Here is the code in Github for this tutorial](https://github.com/kewei/Rust_projects/tree/main/tutorial-bindgen), and I also added the usage of some functions from libusb library in main.rs, just to illustrate how the library is used after generating the bindings.

