# Context-rs

[![Build Status](https://travis-ci.org/zonyitoo/context-rs.svg?branch=master)](https://travis-ci.org/zonyitoo/context-rs)
[![Build status](https://ci.appveyor.com/api/projects/status/ce622ulw4pil7vle?svg=true)](https://ci.appveyor.com/project/zonyitoo/context-rs)
[![License](https://img.shields.io/crates/l/context.svg)](https://github.com/zonyitoo/context-rs)

Context utilities in Rust

**WARN: `master` branch is deprecated, please migrate to `1.0` branch.**

```toml
[dependencies]
context = "*"
```

or use the dev version on master

```toml
[dependencies.context]
git = "https://github.com/zonyitoo/context-rs.git"
```

## Usage

```rust
#![feature(fnbox)]

extern crate context;
extern crate libc;

use std::mem;
use std::boxed::FnBox;

use context::{Context, Stack};

const STACK_SIZE: usize = 2 * 1024 * 1024; // 2MB

extern "C" fn init_fn(arg: usize, f: *mut libc::c_void) -> ! {
    // Transmute it back to the Box<Box<FnBox()>>
    {
        let func: Box<Box<FnBox()>> = unsafe {
            Box::from_raw(f as *mut Box<FnBox()>)
        };

        // Call it
        func();

        // The `func` must be destroyed here,
        // or it will cause memory leak.
    }

    // The argument is the context of the main function
    let ctx: &Context = unsafe { mem::transmute(arg) };

    // Switch back to the main function and will never comeback here
    Context::load(ctx);

    unreachable!("Should never comeback");
}

fn main() {
    // Initialize an empty context
    let mut cur = Context::empty();

    let callback: Box<FnBox()> = Box::new(move|| {
        println!("Inside your function!");
    });

    let mut stk = Stack::new(STACK_SIZE).unwrap();
    let ctx = Context::new(init_fn, unsafe { mem::transmute(&cur) },
                           Box::into_raw(Box::new(callback)) as *mut libc::c_void, &mut stk);

    println!("Before switch");

    // Switch!
    Context::swap(&mut cur, &ctx);

    println!("Back to main function");
}
```

Use `cargo run --example simple` to run this code snippet.

## Notices

* You **have to** drop the boxed function inside the initialize function!!

* The resources allocated inside the initialize function must be released before the last context switch.

* If you **context switch** inside your callback function, if you decided not to come back,
  you **must** release all your resources allocated inside your function.

* This crate supports platforms in

    - arm
    - i686
    - mips
    - mipsel
    - x86_64

* The assembly code is in AT&T-style, so currently it only supports `*-gnu` target on Windows.
