---
layout: post
title: "Handling Stdout/Stderr with JavaScript and WebAssembly"
date: 2020-06-20 21:59:00 +0100
categories:
- WebAssembly
tags:
- WebAssembly
- wasm
- strings
- JavaScript
- C
- libc
- wasm-libc
- clang
comments: true
---

In my
[previous post](https://rob-blackbourn.github.io/blog/webassembly/wasm/strings/javascript/c/libc/wasm-libc/clang/2020/06/20/wasm-string-passing.html)
I found out how to pass strings between JavaScript and WebAssembly. The problem
with that solution was that I had to export `console.log` to the WebAssembly
module. At some point I want to be able to take some library source code "off
the shelf", compile and use it without modification.
Sadly many libraries will use `stdin`/`stdout`
with `printf`, `puts`, or `perror`.

This is going to mean more work at the WASI layer!

You can find the source code for this post
[here](https://github.com/rob-blackbourn/example-wasm-stdout-stderr).

## The Problem

If a program issues a call like `printf` it's often sending its output to the
terminal, or possibly to a file. In any case this is certainly escaping the
WebAssembly sandbox. The
[wasm-libc](https://github.com/WebAssembly/wasi-libc)
library was built with the expectation of a WASI implementation. We can use this
to handle enough `stdio` to suit our needs.

## The C Code

Here is the C code I'd like to provide WASI support for.

```c
#define __wasi__

#include <stdlib.h>
#include <stdio.h>
#include <locale.h>

int is_locale_initialised = 0;

static void initLocale()
{
    // The locale must be initialised before using
    // multi byte characters.
    is_locale_initialised = 1;
    setlocale(LC_ALL, "");
}

__attribute__((used)) void callPerror(char* ptr)
{
    if (is_locale_initialised == 0)
        initLocale();

    perror("Help!");
}

__attribute__((used)) void writeToStdout(char* ptr)
{
    if (is_locale_initialised == 0)
        initLocale();

    fputs(ptr, stdout);
    fflush(stdout);
}

__attribute__((used)) void writeToStderr(char* ptr)
{
    if (is_locale_initialised == 0)
        initLocale();

    fputs(ptr, stderr);
    fflush(stderr);
}
```

Clearly the line at the start `#define __wasi__` needs some explanation. Without
this, compilation broke with the error
`#error <wasi/api.h> is only supported on WASI platforms`. When I checked the
include file the `__wasi__` was the culprit. Defining it enabled the
stdio functionaility.

## The JavaScript Example

This is basically a clone of the previous post. The example code is as follows.

```javascript
const setupWasi = require('./setup-wasi')

async function main() {
  // Setup the WASI instance.
  const wasi = await setupWasi('./stdio-example.wasm')

  // Get the functions exported from the WebAssembly
  const {
    writeToStdout,
    writeToStderr,
    callPerror
  } = wasi.instance.exports

  let ptr1 = null
  try {
    ptr1 = wasi.wasiMemoryManager.convertFromString('To stdout\n')
    writeToStdout(ptr1.byteOffset)
  } finally {
    wasi.wasiMemoryManager.free(ptr1)
  }

  try {
    ptr1 = wasi.wasiMemoryManager.convertFromString('To stderr\n')
    writeToStderr(ptr1.byteOffset)
  } finally {
    wasi.wasiMemoryManager.free(ptr1)
  }

  callPerror()
}

main().then(() => console.log('Done'))
```

I've used the helper function `convertFromString` to create the string in the
memory space of the WebAssembly instance.

## The WASI Code

Running the code exposes the calls we need to implement. It turned out we need:
`fd_close`, `fd_seek`, `fd_write`, and `fd_fdstat_get`.

My goal here is to implement `stdout` and `stderr` with `console.log` and
`console.error`. Here is my implementation.

```javascript
  fd_close = fd => {
    return WASI_ESUCCESS
  }

  fd_seek = (fd, offset_low, offset_high, whence, newOffset) => {
    return WASI_ESUCCESS
  }

  fd_write = (fd, iovs, iovsLen, nwritten) => {
    // We only care about stdout or stderr
    if (!(fd === STDOUT | fd === STDERR)) {
      return WASI_ERRNO_BADF
    }

    const view = new DataView(this.wasiMemoryManager.memory.buffer)

    // Create a UInt8Array for each buffer
    const buffers = Array.from({ length: iovsLen }, (_, i) => {
      const ptr = iovs + i * 8;
      const buf = view.getUint32(ptr, true);
      const bufLen = view.getUint32(ptr + 4, true);
      return new Uint8Array(this.wasiMemoryManager.memory.buffer, buf, bufLen);
    })

    const textDecoder = new TextDecoder()

    // Turn each buffer into a utf-8 string.
    let written = 0;
    let text = ''
    buffers.forEach(buf => {
      text += textDecoder.decode(buf)
      written += buf.byteLength
    });

    // Return the bytes written.
    view.setUint32(nwritten, written, true);

    // Send the output to the console.
    if (fd === STDOUT) {
      this.stdoutText = drainWriter(console.log, this.stdoutText, text)
    } else if (fd == STDERR) {
      this.stderrText = drainWriter(console.error, this.stderrText, text)
    }

    return WASI_ESUCCESS;
  }

  fd_fdstat_get = (fd, stat) => {
    // We only care about stdout or stderr
    if (!(fd === STDOUT | fd === STDERR)) {
      return WASI_ERRNO_BADF
    }

    const view = new DataView(this.wasiMemoryManager.memory.buffer)
    // filetype
    view.setUint8(stat + 0, WASI_FILETYPE_CHARACTER_DEVICE);
    // fdflags
    view.setUint32(stat + 2, WASI_FDFLAGS_APPEND, true);
    // rights base
    view.setBigUint64(stat + 8, WASI_RIGHTS_FD_WRITE, true);
    // rights inheriting
    view.setBigUint64(stat + 16, WASI_RIGHTS_FD_WRITE, true);        

    return WASI_ESUCCESS;
  }
```

For `fd_close` and `fd_seek` there's nothing to do. We Can't close `console.log`
and its not random access so we can't seek.
The `fd_stat_get` function was a bit of a guess with
the rights inherit, but it worked. The `fd_write` function needed to be sensitive to newlines as there's no way to suppress a newline in
the console functions. The text received gets appended until a newline is received, then we report it. The
`drainWriter` function is as follows.

```javascript
function drainWriter (write, prev, current) {
  let text = prev + current
  while (text.includes('\n')) {
    const [line, rest] = text.split('\n', 2)
    write(line)
    text = rest
  }
  return text
}
```

## Thoughts

I'm feeling quite pleased. We've added some complication with the WASI
stubs, but the code is still fairly small and understandable. My goal
is to get to a point where I can drop in a C library and compile it to
WebAssembly without modification. At the moment this seems like a
realistic possibility.
