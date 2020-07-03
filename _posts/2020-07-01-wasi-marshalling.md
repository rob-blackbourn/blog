---
layout: post
title:  "WASI Marshalling"
date:   2020-07-02 20:06:00 +0100
categories: WebAssembly wasm JavaScript C clang wasi-sdk marshalling
comments: true
---

# WASI Marshalling

In this series of posts I've done a lot of experimentation. An element which
I've had to implement a number of times is calling functions defined in a
WebAssembly module.

Calling a WebAssembly function involves transferring data to an instance,
calling a function, and retrieving the results. This is known as marshalling.

The source code for the examples can be found
[here](https://github.com/rob-blackbourn/example-wasi-marshalling)
and the marshalling package
[here](https://github.com/rob-blackbourn/jetblack-wasi-marshalling).

## Our First C Function

Here is an example function written in C and compiled to WebAssembly.

```C
__attribute__((used)) double* multipleFloat64ArraysReturningPtr (
    double* array1,
    double* array2, 
    int length)
{
  double* result = (double*) malloc(length * sizeof(double));
  if (result == 0)
    return 0;

  for (int i = 0; i < length; ++i) {
    result[i] = array1[i] + array2[i];
  }

  return result;
}
```

At the highest level the function takes two floating point arrays which it
multiplies and returns the result. At a lower level, two pointers to double
arrays are passed with a length. A result pointer is allocated with the required
length in which the result is stored. Finally the pointer to the result array
is returned.

To make this happen, first memory has to be allocated for the two input arrays
in the WebAssembly module. The data must then be copied into the arrays. When
the function is called it creates a third array by allocating space in the
WebAssembly. The pointer to this memory is then returned. The JavaScript must
copy the result array, then free the two input arrays and the result array.

## Our First JavaScript Prototype

I've written a package to simplify the marshalling of data between JavaScript
and WebAssembly. Here is how to create a binding for the function.

```javascript
const proto = new FunctionPrototype(
    [
        new In(new ArrayType(new Float64Type())),
        new In(new ArrayType(new Float64Type())),
        new In(new Int32Type())
    ],
    new ArrayType(new Float64Type(), 4))
```

This matches the `C` prototype.

```C
__attribute__((used)) double* multipleFloat64ArraysReturningPtr (
    double* array1,
    double* array2, 
    int length)
```

Note that the return type also declares the length of the returned array. The
`C` function will only return a pointer to the start of the array, so we need
the length. In the real world we'd probably make a function out of prototypes
that require length parameters.

We can invoke the function as follows.

```javascript
const result = proto.invoke(
  wasi.memoryManager,
  wasi.instance.exports.multipleFloat64ArraysReturningPtr,
  [1, 2, 3, 4],
  [5, 6, 7, 8],
  4)
```

As well as the function arguments we provide `wasi.memoryManager` to allow
access to the WebAssembly memory and `wasi.instance` to retirve the function.

When the function is invoked three things happen. First the input arguments are
*marshalled*. Memory is allocated for each array, and the array data is copied
into that memory. The integer value can be passed directly.

Second the function is called with the marshalled aruments and the return value
is received.

Third the memory for the input arguments is freed, the result value is copied
from the WebAssembly memory to a JavaScript array, and the memory that was
allocated inside the WebAssembly instance is freed.

## Memory

While not strictly part of WASI, memory management is the first problem to
solve. When a WebAssembly module is instatiated a memory buffer is either
passed, or created by the instance. When using the WASM standard library
provided by [wasi-libc](https://github.com/WebAssembly/wasi-libc) the memory
management functions `malloc` and `free` can be exported from the instance.

After instantiation they are held in the following class.

```javascript
class MemoryManager {
  constructor (memory, malloc, free) {
    this.memory = memory
    this.malloc = malloc
    this.free = free
    this.dataView = new DataView(this.memory.buffer)
  }
}
```

An array of floats might be marshalled as follows.

```javascript
function marshallFloat64Array(memoryManager, array) {
  const address = memoryManager.malloc(Float64Array.BYTES_PER_ELEMENT * array.length)
  const typedArray = new Float64Array(memoryManager.memory.buffer, address, array.length)
  typedArray.set(array)
  return address
}
```

An array that was allocated inside the WebAssembly instance might be
unmarshalled as follows.

```javascript
function unmarshallFloat64Array(memoryManager, address, length) {
  try {
    const typedArray = new Float64Array(memoryManager.memory.buffer, address, length)
    return Array.from(typedArray)
  } finally {
    memoryManager.free(address)
  }
}
```

## Our Second C Function

Here is our second `C` function.

```C
__attribute__((used)) void multipleFloat64ArraysWithOutputArray (double* array1, double* array2, double* result, int length)
{
  for (int i = 0; i < length; ++i) {
    result[i] = array1[i] + array2[i];
  }
}
```

This time the calling function allocates the memory for the output array and
passes it in.

## Our Second JavaScript Function Prototype

The JvaScript function prototype looks as follows.

```javascript
const proto2 = new FunctionPrototype(
  [
    new In(new ArrayType(new Float64Type())),
    new In(new ArrayType(new Float64Type())),
    new Out(new ArrayType(new Float64Type())),
    new In(new Int32Type())
  ]
)
```

We don't need to provide a return type as nothing is returned. The *output*
argument is wrapped in an `Out` class to inform the marshaller to unpack it
after the function is called.

The function is called as follows.

```javascript
const output = new Array(4)
proto2.invoke(
  wasi.memoryManager,
  wasi.instance.exports.multipleFloat64ArraysWithOutputArray,
  [1, 2, 3, 4],
  [5, 6, 7, 8],
  output,
  4)
```

Because we supplied the output array with the correct length, we didn't need
to supply a length argument to `ArrayType`.

## Where's WASI?

So far there's been no need for WASI. That ends with strings.

WASI enters the picture when we need to leave the WebAssembly sandbox and
interact with the system in which we're running. It's not obvious why this
should happen with strings. In JavaSCript strings use UTF-8. As soon
as we need to decode a string the C standard library will want to know about
our *locale* (our language environment). To do this it it checks the environment
variables, which exist outside the sandbox!

This introduces the first of our system call requirement: `environ_sizes_get`
and `environ_get`. The first call finds the amount of space required to store
the environment variables; the second fetches them.

The other need for WASI is when the code interacts with stdout/stderr. While
we might think this is unnecessary, it is common for C libraries to report
errors through the `perror` function which returns and error and reports the
cause to stderr.

There are a small bunch of functions we need to support here. The final outcode
is the following set of imports to the WebAssembly.

```javascript
const res = await WebAssembly.instantiate(buf, {
  wasi_snapshot_preview1: {
    environ_get: (environ, environBuf) => wasi.environ_get(environ, environBuf),
    environ_sizes_get: (environCount, environBufSize) => wasi.environ_sizes_getnvironCount, environBufSize),
    proc_exit: rval => wasi.proc_exit(rval),
    fd_close: fd => wasi.fd_close(fd),
    fd_seek: (fd, offset_low, offset_high, whence, newOffset) => wasi.fd_seek(fd, fset_low, offset_high, whence, newOffset),
    fd_write: (fd, iovs, iovsLen, nwritten) => wasi.fd_write(fd, iovs, iovsLen, ritten),
    fd_fdstat_get: (fd, stat) => wasi.fd_fdstat_get(fd, stat)
    }
})
```

With this relatively small set of imports we can import the vast majority of
publically available C libraries.

## Strings & Stdout/Stderr

You can check out the source code fore the implementations and use of these.
They are not typically use from JavaScript, and are only provided to allow
imported C libarries to run.

## Wiring It Up

The example project shows how this works with node and in the browser. Here's
the browser version.

```javascript
<script>
const {
    Wasi,
    Float64Type,
    ArrayType,
    Int32Type,
    StringType,
    FunctionPrototype,
    In,
    Out
} = wasiMarshalling

// Create the Wasi instance passing in environment variables.
const wasi = new Wasi({})

// Instantiate the wasm module.
WebAssembly.instantiateStreaming(
  fetch('example.wasm'), {
    wasi_snapshot_preview1: wasi.imports()
  })
  .then(res => {
    // Initialise the wasi instance
    wasi.init(res.instance)

    // The first example takes in two arrays on the same length and
    // multiplies them, returning a third array.
    const proto1 = new FunctionPrototype(
      [
        new In(new ArrayType(new Float64Type())),
        new In(new ArrayType(new Float64Type())),
        new In(new Int32Type())
      ],
      new ArrayType(new Float64Type(), 4)
    )

    const result1 = proto1.invoke(
      wasi.memoryManager,
      wasi.instance.exports.multipleFloat64ArraysReturningPtr,
      [1, 2, 3, 4],
      [5, 6, 7, 8],
      4)
    console.log(result1)
    ...
</script>
```

## Thoughts

We've got a ganeric marshalling layer between JavaScript and WebAssembly which
is pretty cool! We've also got enough WASI to drop in publically available
`C` libraries.

We've not yet addressed C++ libraries and we're waiting for the `flang` compiler
to be incorporated into the `llvm` toolchain, so there's a way to go before we
have access to everything we need to use JavaSCript to do real work with
native libraries.
