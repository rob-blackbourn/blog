---
layout: post
title:  "WASI Marshalling with Finalizers"
date:   2020-07-07 08:19:00 +0100
categories:
- WebAssembly
tags:
- WebAssembly
- wasm
- wasi
- JavaScript
- C
- clang
- wasi-sdk
- marshalling
- finalizer
- FinalizationRegistry
comments: true
---

In my
[previous post](https://rob-blackbourn.github.io/blog/webassembly/wasm/javascript/c/clang/wasi-sdk/marshalling/2020/07/02/wasi-marshalling.html)
I described a marshalling library I'd written for WebAssembly (see
[here](https://github.com/rob-blackbourn/jetblack-wasi-marshalling) for the source). There's one
problem I have with it; too much copying!

But there's a good reason for the copying. In order to ensure all allocated memory
is freed, the marshaller goes through three steps: allocating memory and
copying data, calling the function, then copying results and freeing the memory.
Without this control memory will leak.

Enter finalizers ...

## FinalizationRegistry

Until recently there has been no way to hook into the end of a JavaScript
object's lifecycle. There is a "stage 3" proposal that is going into production:
[FinalizationRegistry](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/FinalizationRegistry).
This is currently avaliable in node (since 13.0.0), but must be enabled with the
flag "--harmony-weak-refs".
It should be available in chrome (84), and firefox (79) in a few days.

When an object registers with the finalization registry it gets called back when
the object gets garbage collected. If we register our WebAssembly objects then we
can free the memory we have allocated when the garbage gets collected.

## How does finalizing work?

I tried and tried to get finalizing to work. The following called the finalizer
cleanup function, but only when the program exited.

```javascript
// run with node --harmony-weak-refs

const unfinalizedTags = new Set()let counter = 0

const registry = new FinalizationRegistry(held => {
  console.log('cleanup')
  for (const tag of held) {
    console.log(tag)
    unfinalizedTags.delete(tag)
  }
})

// Nohting gets finalized here.
for (let counter = 0; counter < 1000; ++counter) {
  let array = new Array(1000000)
  const tag = "tag" + counter
  unfinalizedTags.add(tag)
  // Register with the finalizer.
  registry.register(array, tag)
  for (let i = 0; i < array.length; ++i) {
    array[i] = Math.random()
  }
  const sum = array.reduce((acc, value) => acc + value, 0)
  console.log(sum / array.length, counter)
  array = null
}

// The finalizer gets called with all the registered objects on exit.
```

The following *does* call the finalizer while the code is running.

```javascript
const unfinalizedTags = new Set()
let counter = 0

const registry = new FinalizationRegistry(held => {
  console.log('cleanup')
  for (const tag of held) {
    console.log(tag)
    unfinalizedTags.delete(tag)
  }
})

function makePointlessGarbage() {
  let array = new Array(1000000)
  const tag = "tag" + counter++
  unfinalizedTags.add(tag)
  registry.register(array, tag)
  for (let i = 0; i < array.length; ++i) {
    array[i] = Math.random()
  }
  const sum = array.reduce((acc, value) => acc + value, 0)
  console.log(sum / array.length, counter)
  array = null
  if (counter < 100) {
    setTimeout(makePointlessGarbage, 10)
  } else {
    console.log(`unfinalizedTags.size=${unfinalizedTags.size}`)
  }
}

setTimeout(makePointlessGarbage, 10)
```

This version calls the finalizer as the program is running but doesn't finalize
on exit, leaving some objects unfinalized. I'm not so concerned about the
unfinalized objects with respect to memory management, as the WebAssembly
instance will be destroyed on exit.

It seems that the program needs to
give up control (with `setTimeout`) to allow the garbage collection to run. It
also seems that the cleanup function gets passed an iterable of values to free,
which is not how I read the documentation.

## Implementation

### Arrays

Here is how we might unmarhsall an array using the copy then free pattern.

```javascript
unmarshall (memoryManager, address, array) {
  try {
    // Create a temporary typed array
    const typedArray = new this.type.TypedArrayType(
      memoryManager.memory.buffer,
      address,
      array.length)
    // Return a copy of the array.
    return Array.from(typedArray)
  } finally {
    // Free the memory
    memoryManager.free(address)
  }
}
```

We can change this to simply register the object for freeing. Note that creating the typed array is not really creating the array: it just provides a *view* into the
memory of the *element type* we want.

```javascript
unmarshall (memoryManager, address, array) {
  // Create a typed array
  const typedArray = new this.type.TypedArrayType(
    memoryManager.memory.buffer,
    address,
    array.length)
  // register for cleanup
  memoryManager.freeWhenFinalized(typedArray, address)
  // Return the typed array
  return typedArray
}
```

This implementation is much more lightweight. In the previous version there was
a point in time where both arrays existed in memory, which could be a problem
for big data. However the "copy then free" version could resolve arrays of pointers, as
the `Array` can hold references to other arrays, rather than just numbers.

### Strings

Strings present a minor problem. Once we have unmarshalled a byte array to a
string we have nowhere to keep the address from the WebAssembly instance. To
solve this I created a `StringBuffer` which is an extension of `Uint8Array`,
with a couple of class methods for marshalling and unmarshalling, and an
instance method to decode the string.

### Pointers

I implemented an `AddressType` which simply sets the contents of a `Pointer` to
the address. This works, but feels like a weak solution. I need some more use
cases before I know what to do here.

## Thoughts

We can now pass typed arrays, strings and pointers between JavaScript and a
WebAssembly instance with the memory being cleaned up through the garbage
collector and finalizers.

This is a huge win for big data applications, as only one copy of the data needs
to exist, reducing the memory footprint, and time spent copying between domains.
