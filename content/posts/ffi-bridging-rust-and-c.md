---
title: "FFI: Bridging Rust and C"
date: 2024-01-20
description: "Exploring Foreign Function Interface to call C code from Rust and vice versa, with practical examples."
tags: ["Rust", "C", "Systems", "FFI"]
---

When working on systems programming, you often need to interface between different languages. Rust's Foreign Function Interface (FFI) makes it straightforward to call C code and expose Rust functions to C.

## Calling C from Rust

Let's start with a simple C library that we want to use from Rust.

### The C Code

First, here's a simple C header and implementation:

```c
// mathlib.h
#ifndef MATHLIB_H
#define MATHLIB_H

#include <stdint.h>

// Simple arithmetic functions
int32_t add(int32_t a, int32_t b);
int32_t multiply(int32_t a, int32_t b);

// More complex: working with pointers
void fill_array(int32_t* arr, size_t len, int32_t value);
int32_t sum_array(const int32_t* arr, size_t len);

// String operations
size_t string_length(const char* s);

#endif
```

And the implementation:

```c
// mathlib.c
#include "mathlib.h"
#include <string.h>

int32_t add(int32_t a, int32_t b) {
    return a + b;
}

int32_t multiply(int32_t a, int32_t b) {
    return a * b;
}

void fill_array(int32_t* arr, size_t len, int32_t value) {
    for (size_t i = 0; i < len; i++) {
        arr[i] = value;
    }
}

int32_t sum_array(const int32_t* arr, size_t len) {
    int32_t sum = 0;
    for (size_t i = 0; i < len; i++) {
        sum += arr[i];
    }
    return sum;
}

size_t string_length(const char* s) {
    return strlen(s);
}
```

### The Rust Bindings

Now let's create Rust bindings to call this C code:

```rust
// src/lib.rs
use std::ffi::CString;
use std::os::raw::{c_char, c_int};

// Declare the external C functions
#[link(name = "mathlib")]
extern "C" {
    fn add(a: c_int, b: c_int) -> c_int;
    fn multiply(a: c_int, b: c_int) -> c_int;
    fn fill_array(arr: *mut c_int, len: usize, value: c_int);
    fn sum_array(arr: *const c_int, len: usize) -> c_int;
    fn string_length(s: *const c_char) -> usize;
}

// Safe Rust wrappers
pub fn safe_add(a: i32, b: i32) -> i32 {
    unsafe { add(a, b) }
}

pub fn safe_multiply(a: i32, b: i32) -> i32 {
    unsafe { multiply(a, b) }
}

pub fn safe_fill_array(arr: &mut [i32], value: i32) {
    unsafe {
        fill_array(arr.as_mut_ptr(), arr.len(), value);
    }
}

pub fn safe_sum_array(arr: &[i32]) -> i32 {
    unsafe { sum_array(arr.as_ptr(), arr.len()) }
}

pub fn safe_string_length(s: &str) -> usize {
    let c_string = CString::new(s).expect("CString::new failed");
    unsafe { string_length(c_string.as_ptr()) }
}
```

## Exposing Rust to C

Now let's go the other direction—exposing Rust functions to C.

### Rust Library

```rust
// src/lib.rs
use std::ffi::{CStr, CString};
use std::os::raw::c_char;
use std::ptr;

/// A simple struct that we'll expose to C
#[repr(C)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}

/// Calculate distance between two points
#[no_mangle]
pub extern "C" fn point_distance(p1: *const Point, p2: *const Point) -> f64 {
    if p1.is_null() || p2.is_null() {
        return -1.0;
    }

    unsafe {
        let p1 = &*p1;
        let p2 = &*p2;
        let dx = p2.x - p1.x;
        let dy = p2.y - p1.y;
        (dx * dx + dy * dy).sqrt()
    }
}

/// Create a new point (caller must free with point_free)
#[no_mangle]
pub extern "C" fn point_new(x: f64, y: f64) -> *mut Point {
    Box::into_raw(Box::new(Point { x, y }))
}

/// Free a point allocated by point_new
#[no_mangle]
pub extern "C" fn point_free(p: *mut Point) {
    if !p.is_null() {
        unsafe {
            drop(Box::from_raw(p));
        }
    }
}

/// String processing: reverse a string
#[no_mangle]
pub extern "C" fn reverse_string(s: *const c_char) -> *mut c_char {
    if s.is_null() {
        return ptr::null_mut();
    }

    let c_str = unsafe { CStr::from_ptr(s) };
    let rust_str = match c_str.to_str() {
        Ok(s) => s,
        Err(_) => return ptr::null_mut(),
    };

    let reversed: String = rust_str.chars().rev().collect();

    match CString::new(reversed) {
        Ok(c_string) => c_string.into_raw(),
        Err(_) => ptr::null_mut(),
    }
}

/// Free a string allocated by Rust
#[no_mangle]
pub extern "C" fn free_string(s: *mut c_char) {
    if !s.is_null() {
        unsafe {
            drop(CString::from_raw(s));
        }
    }
}
```

### Using from C

```c
// main.c
#include <stdio.h>
#include <stdlib.h>

// Declare the Rust functions
typedef struct {
    double x;
    double y;
} Point;

extern Point* point_new(double x, double y);
extern void point_free(Point* p);
extern double point_distance(const Point* p1, const Point* p2);
extern char* reverse_string(const char* s);
extern void free_string(char* s);

int main() {
    // Test point functions
    Point* p1 = point_new(0.0, 0.0);
    Point* p2 = point_new(3.0, 4.0);

    double dist = point_distance(p1, p2);
    printf("Distance: %.2f\n", dist);  // Should print 5.00

    point_free(p1);
    point_free(p2);

    // Test string reversal
    char* reversed = reverse_string("Hello, World!");
    if (reversed) {
        printf("Reversed: %s\n", reversed);  // Should print "!dlroW ,olleH"
        free_string(reversed);
    }

    return 0;
}
```

## Build Configuration

To build this, you'll need a `build.rs` for the Rust side:

```rust
// build.rs
fn main() {
    // Tell Cargo to link against our C library
    println!("cargo:rustc-link-lib=mathlib");
    println!("cargo:rustc-link-search=native=./lib");

    // Rebuild if C files change
    println!("cargo:rerun-if-changed=src/mathlib.c");
    println!("cargo:rerun-if-changed=src/mathlib.h");
}
```

And `Cargo.toml`:

```toml
[package]
name = "rust-c-ffi"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "staticlib"]
```

## Key Takeaways

- Use `#[repr(C)]` for structs shared between Rust and C
- Use `#[no_mangle]` and `extern "C"` for functions callable from C
- Always handle null pointers defensively
- Memory allocated in Rust must be freed by Rust (and vice versa)
- Use `CString` and `CStr` for string interop

FFI is powerful but requires careful attention to memory management and type compatibility. The Rust compiler can't protect you across the FFI boundary, so thorough testing is essential.
