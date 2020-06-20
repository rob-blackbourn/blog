---
layout: post
title:  "How to Pass Strings Between JavaScript and WebAssembly"
date:   2020-06-20 11:04:00 +0100
categories: WebAssembly wasm strings JavaScript C libc wasm-libc clang
comments: true
---

# How to Pass Strings Between JavaScript and WebAssembly

In my
[first post](https://rob-blackbourn.github.io/blog/webassembly/wasm/array/arrays/javascript/c/2020/06/07/wasm-arrays.html)
on WebAssembly I dodged the classic "Hello, World!" example as I thought it might
be a bit tricky. Little did I realise how hard it was going to be!

In this post I'm going to look at how to solve it. The code for the project
can be found [here](https://github.com/rob-blackbourn/example-wasm-string-passing).

## The Problem

In JavaScript strings are encoded with utf-8. This format encodes characters
that cannot be represented in the ascii character set with more than one byte.

## The JavaScript Side

On the JavaScript side we can use
[TextEncoder](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder) and
[TextDecoder](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder/TextDecoder)
in the following manner.

```javascript
class WasiMemoryManager {
  constructor (memory, malloc, free) {
    this.memory = memory
    this.malloc = malloc
    this.free = free
  }

  // Convert a pointer from the wasm module to JavaScript string.
  convertToString (ptr, length) {
    try {
      // The pointer is a multi byte character array encoded with utf-8.
      const array = new Uint8Array(this.memory.buffer, ptr, length)
      const decoder = new TextDecoder()
      const string = decoder.decode(array)
      return string
    } finally {
      // Free the memory sent to use from the WebAssembly instance.
      this.free(ptr)
    }
  }

  // Convert a JavaScript string to a pointer to multi byte character array
  convertFromString(string) {
    // Encode the string in utf-8.
    const encoder = new TextEncoder()
    const bytes = encoder.encode(string)
    // Copy the string into memory allocated in the WebAssembly
    const ptr = this.malloc(bytes.byteLength)
    const buffer = new Uint8Array(this.memory.buffer, ptr, bytes.byteLength + 1)
    buffer.set(bytes)
    return buffer
  }
}
```

Note how `malloc` and `free` and used to manage the memory in the
WebAssembly module.

## The C Side

The C standard library provides support for multi byte characters
[[1]](https://en.cppreference.com/w/c/string/multibyte),
so I coded up a function to count the *letters* (not the bytes) of a string, and
a function to log "Hello, World!" in english and mandarin.

This was a complete failure!

Looking harder at the examples I noticed I was missing a called to `setlocale`,
so I added that to my C code. The final code looks as follows.

```c
#include <stdlib.h>
#include <string.h>
#include <wchar.h>
#include <locale.h>

// Import console.log from JavaScript.
extern void consoleLog(char* ptr, int length);

int is_locale_initialised = 0;

static void initLocale()
{
    // The locale must be initialised before using
    // multi byte characters.
    is_locale_initialised = 1;
    setlocale(LC_ALL, "");
}

// Count the letters in a utf-8 encoding string.
__attribute__((used)) size_t countLetters(char* ptr)
{
    if (is_locale_initialised == 0)
        initLocale();

    size_t letters = 0;
    const char* end = ptr + strlen(ptr);
    mblen(NULL, 0); // reset the conversion state
    while(ptr < end) {
        int next = mblen(ptr, end - ptr);
        if(next == -1) {
            return -1;
        }
        ptr += next;
        ++letters;
    }
    return letters;
}

// Say hello in english
__attribute__((used)) void sayHelloWorld()
{
    if (is_locale_initialised == 0)
        initLocale();

    const char* s1 = "Hello World";
    size_t len = strlen(s1);
    char* s2 = malloc(len + 1);
    consoleLog(strcpy(s2, s1), (int) len);
}

// Say hello in Mandarin
__attribute__((used)) void sayHelloWorldInMandarin()
{
    if (is_locale_initialised == 0)
        initLocale();

    const wchar_t* wstr = L"你好，世界！";
    mbstate_t state;
    memset(&state, 0, sizeof(state));
    size_t len =  wcsrtombs(NULL, &wstr, 0, &state);
    char* mbstr = malloc(len + 1);
    wcsrtombs(mbstr, &wstr, len + 1, &state);
    consoleLog(mbstr, (int) len);
}
```

Then I ran the program and BOOM I get an error
complaining that `wasi_snapshot_preview1` is not defined. I knew what this meant
(the libc implementation needed a [WASI](https://wasi.dev/) module to implement
system calls), but I didn't think I needed one, as I'm only using strings.

It turns out that the first thing `setlocale` does is check some environment
variables. To do that it needs some wasi functions to call out of the wasm
sandbox.

## Implementing a Minimal WASI

After a consulting the [WASI API documentation](https://github.com/WebAssembly/WASI/blob/master/phases/snapshot/docs.md)
I decided I needed to implement
[environ_get](https://github.com/WebAssembly/WASI/blob/master/phases/snapshot/docs.md#-environ_getenviron-pointerpointeru8-environ_buf-pointeru8---errno)
and it's companion
[environ_sizes_get](https://github.com/WebAssembly/WASI/blob/master/phases/snapshot/docs.md#-environ_sizes_get---errno-size-size). I put these in a class called `Wasi`
and passed them to the module on instantiation using the `wasi_snapshot_preview1`
key as follows:

```javascript
const res = await WebAssembly.instantiate(buf, {
  wasi_snapshot_preview1: wasi,
})
```

Now when I ran the program I got no errors about `wasi_snapshot_preview1`, but I
get a new error saying that `proc_exit` was missing. I found that in the docs,
and it gets called to terminate the process. As I've got nothing to terminate
I created a stub for it. My `Wasi` class looks as follows.

```javascript
const WasiMemoryManager = require('./wasi-memory-manager')

// An implementation of WASI which supports the minimum
// required to use multi byte characters.
class Wasi {
  constructor (env) {
    this.env = env
    this.instance = null
    this.wasiMemoryManager = null
  }

  // Initialise the instance from the WebAssembly.
  init = (instance) => {
    this.instance = instance
    this.wasiMemoryManager = new WasiMemoryManager(
      instance.exports.memory,
      instance.exports.malloc,
      instance.exports.free
    )
  }

  static WASI_ESUCCESS = 0

  // Get the environment variables.
  environ_get = (environ, environBuf) => {
    const encoder = new TextEncoder()
    const view = new DataView(this.wasiMemoryManager.memory.buffer)

    Object.entries(this.env).map(
      ([key, value]) => `${key}=${value}`
    ).forEach(envVar => {
      view.setUint32(environ, environBuf, true)
      environ += 4

      const bytes = encoder.encode(envVar)
      const buf = new Uint8Array(this.wasiMemoryManager.memory.buffer, environBuf, bytes.length + 1)
      environBuf += buf.byteLength
    });
    return this.WASI_ESUCCESS;
  }
  
  // Get the size required to store the environment variables.
  environ_sizes_get = (environCount, environBufSize) => {
    const encoder = new TextEncoder()
    const view = new DataView(this.wasiMemoryManager.memory.buffer)

    const envVars = Object.entries(this.env).map(
      ([key, value]) => `${key}=${value}`
    )
    const size = envVars.reduce(
      (acc, envVar) => acc + encoder.encode(envVar).byteLength + 1,
      0
    )
    view.setUint32(environCount, envVars.length, true)
    view.setUint32(environBufSize, size, true)

    return this.WASI_ESUCCESS
  }

  // This gets called on exit to stop the running program.
  // We don't have anything to stop!
  proc_exit = (rval) => {
    return this.WASI_ESUCCESS
  }
}

module.exports = Wasi
```

And to hook it all up I did the following.

```javascript
const fs = require('fs')
const Wasi = require('./wasi')

async function setupWasi(fileName) {
  // Read the wasm file.
  const buf = fs.readFileSync(fileName)

  // Create the Wasi instance passing in some environment variables.
  const wasi = new Wasi({
    "LANG": "en_GB.UTF-8",
    "TERM": "xterm"
  })

  // Instantiate the wasm module.
  const res = await WebAssembly.instantiate(buf, {
    wasi_snapshot_preview1: wasi,
    env: {
      // This function is exported to the web assembly.
      consoleLog: function(ptr, length) {
        // This converts the pointer to a string and frees he memory.
        const string = wasi.wasiMemoryManager.convertToString(ptr, length)
        console.log(string)
      }
    }
  })

  // Initialise the wasi instance
  wasi.init(res.instance)

  return wasi
}

module.exports = setupWasi
```

## The Example

The last thing to do was write the example. Here it is:

```javascript
const setupWasi = require('./setup-wasi')

async function main() {
  // Setup the WASI instance.
  const wasi = await setupWasi('./string-example.wasm')

  // Get the functions exported from the WebAssembly
  const {
    countLetters,
    sayHelloWorld,
    sayHelloWorldInMandarin
  } = wasi.instance.exports

  let buf1 = null
  try {
    const s1 = '犬 means dog'
    // Convert the JavaScript string into a pointer in the WebAssembly
    buf1 = wasi.wasiMemoryManager.convertFromString(s1)
    // The Chinese character will take up more than one byte.
    const l1 = countLetters(buf1.byteOffset)
    console.log(`expected length ${s1.length} got ${l1} byte length was ${buf1.byteLength}`)
  } finally {
    // Free the pointer we created in WebAssembly.
    wasi.wasiMemoryManager.free(buf1)
  }

  // Should log "Hello, World!" to the console
  sayHelloWorld()
  sayHelloWorldInMandarin()
}

main().then(() => console.log('Done'))
```

And it works!

## Thoughts

I realise node already has a WASI module, but I wanted to be able to
run the code in a browser. I would have preferred to use wasmer-js,
but I had problems getting it to work, and it felt like a bit of a
sledgehammer for the problem I was trying to solve.
