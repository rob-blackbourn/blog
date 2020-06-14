---
layout: post
title:  "Using libc with WebAssembly enabled DataFrames"
date:   2020-06-14 11:38:00 +0100
categories: WebAssembly wasm array arrays JavaScript C DataFrame libc wasm-libc clang
---

# Using libc with WebAssembly enabled DataFrames

This post describes how to use the C standard library with WebAssembly enabled
DataFrames. You can find the source code
[here](https://github.com/rob-blackbourn/example-wasm-dataframe-2).

## Introduction

In my previous
[post](https://rob-blackbourn.github.io/blog/webassembly/wasm/array/arrays/javascript/c/dataframe/2020/06/13/wasm-dataframes.html)
I avoided solving the problem of linking to standard
library to provide memory allocation for my WebAssembly project by writing my
own memory allocator. In order to use external C packages I will need a
standard library.

## wsi-libc

The [wasi-libc](https://github.com/WebAssembly/wasi-libc) project does what it
says on the tin by providing a standard library for wasm.

I have changed my setup as my laptop required reinstallation. I'm now running
Ubuntu 20.04 LTS.

I installed the latest (version 10) LLVM tool chain as a Debian package which
I got from [this page](https://apt.llvm.org/). I did the following.

```bash
wget https://apt.llvm.org/llvm.sh
sudo ./llvm.sh
```

That installed clang et al with the suffix "-10", such that clang could be
found as `/usr/bin/clang-10`.

I installed [wabt](https://github.com/WebAssembly/wabt) in `/opt/wabt`.

I cloned the wasi-libc library and built it as follows.

```bash
git clone git@github.com:WebAssembly/wasi-libc.git
cd wasi-libc
sudo WASM_CC=clang-10 WASM_NM=llvm-nm-10 WASM_AR=llvm-ar-10 INSTALL_DIR=/opt/wasi-libc make install
```

## The C Source Code

In my previous post I had two source files: `memory-allocation.c` and `array-methods.c`.
We can delete `memory-allocation.c` as we'll use `malloc` and `free` from the
standard library. I split he array methods into two files: `array-methods-int.c`
and `array-methods-double.c`, included `stdlib.h` and changed the memory allocation
to use the standard library.

Here is the start of `array-methods-int.c`.

```c
#include <stdlib.h>

__attribute__((used)) int* addInt32Arrays (int *array1, int* array2, int length)
{
  int* result = (int*) malloc(length * sizeof(int));
  if (result == 0)
    return 0;

  for (int i = 0; i < length; ++i) {
    result[i] = array1[i] + array2[i];
  }

  return result;
}
```

Then I changed the makefile as follows.

```makefile
SYSROOT=/opt/wasi-libc

LLVM_VERSION=10
CC=clang-$(LLVM_VERSION)
LD=wasm-ld-$(LLVM_VERSION)
CFLAGS=--target=wasm32-unknown-unknown-wasm --optimize=3 --sysroot=$(SYSROOT)
LDFLAGS=--export-all --no-entry --allow-undefined -L$(SYSROOT)/lib/wasm32-wasi -lc -lm

WASM2WAT=/opt/wabt/bin/wasm2wat

sources = array-methods-int.c array-methods-double.c
objects = $(sources:%.c=%.bc)
target = data-frame

.PHONY: all

all: $(target).wat

$(target).wat: $(target).wasm
	$(WASM2WAT) $(target).wasm -o $(target).wat

$(target).wasm: $(objects)
	$(LD) $(objects) $(LDFLAGS) -o $(target).wasm
	
%.bc: %.c
	$(CC) -c -emit-llvm $(CFLAGS) $< -o $@

clean:
	rm -f $(target).wasm
	rm -f $(target).wat
	rm -f $(objects)
```

The compilation stage node sets `--sysroot`. This appears to set up the include
paths correctly.

The link stage adds the path to the libraries `-L$(SYSROOT)/lib/wasm32-wasi` and
and includes libc and libm `-lc -lm`.

## The JavaScript Code

All I needed to do was to change the memory allocation imports to use `malloc` and `free`.

Everything just worked!

As a test I added a unary function `logFloat64Array`. This required sme minor
changes to the javascript to allow 'int' type series to fall through to the `double`
methods. Again it all just worked.

## Things To Do

I need to understand what the `crt1.o` file is all about. I tried to link, but
that gives me other errors.
