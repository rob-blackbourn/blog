---
layout: post
title:  "Using Typed Arrays and Finalizers with a WebAssembly DataFrame"
date:   2020-07-15 11:51:00 +0100
categories: WebAssembly wasm wasi JavaScript C clang wasi-sdk marshalling finalizer FinalizationRegistry data science dataframe series
comments: true
---

# Using Typed Arrays and Finalizers with a WebAssembly DataFrame

## Introduction

In my previous posts on data frames we looked at how to:

* [Create an expressive syntax using operator overloading and proxies](https://rob-blackbourn.github.io/blog/javascript/webassembly/dataframe/2020/06/10/example-js-dataframe.html)
* [Use WebAssembly for efficient array calculations](https://rob-blackbourn.github.io/blog/webassembly/wasm/array/arrays/javascript/c/dataframe/2020/06/13/wasm-dataframes.html)
* [Use libc with WebAssembly](https://rob-blackbourn.github.io/blog/webassembly/wasm/array/arrays/javascript/c/dataframe/libc/wasm-libc/clang/2020/06/14/wasm-dataframe-libc.html)

The main problem with that implementation was the marshalling layer which was
hand written. It required all arrays to be copied in and out of the WebAssembly
instance, with manual allocation and freeing of memory.

This implementation introduces a marshalling layer which makes use of the
[FinzalizationRegistry](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/FinalizationRegistry)
which provides a callback for managing the destruction of objects.


You can find the source code [here](https://github.com/rob-blackbourn/example-wasm-dataframe-3). To run the examples you will need a version of
node which supports the `--harmony-weak-refs` flag (I'm using 14.5.0), the
[wasi-sdk 11](https://github.com/WebAssembly/wasi-sdk)
(with it's bin directory on your path) and the
[wabt 1.0.16](https://github.com/WebAssembly/wabt)
toolkit (with it's bin directory on your path).

## Usage

You can find more information about the implementation of the marshalling layer in
[this post](https://rob-blackbourn.github.io/blog/webassembly/wasm/wasi/javascript/c/clang/wasi-sdk/marshalling/finalizer/finalizationregistry/2020/07/07/wasi-finalizers-1.html).
What I want to talk about here is *usability*.

Rather than bundle a fixed set of functions with the data frame I wanted it to be a structural object where the functions are
provided by an extensible repository. This way the user
isn't limited to the operations defined by the data frame implementation.

A `C` function for multiplying two arrays might be written as follows -

```c
__attribute__((used)) double* divideFloat64Arrays (double* array1, double* array2, unsigned int length)
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

The functions are registered in JavaScript as follows -

```javascript
DataFrame.registerFunction(
  // The function or operation name.
  Symbol.for('/'),
  new FunctionPrototype(
    // The arguments
    [
      new In(new TypedArrayType(new Float64Type(), null)),
      new In(new TypedArrayType(new Float64Type(), null)),
      new In(new Uint32Type())
    ],
    // The return type with automatic length discovery.
    new TypedArrayType(new Float64Type(), (i, args) => args[2])
  ),
  // The bound function
  wasi.instance.exports.divideFloat64Arrays
)
```

which allows the following -

```javascript
const df = DataFrame.fromObject(
  // The data
  [
    { name: 'mary', height: 175.2, weight: 65.1 },
    { name: 'fred', height: 183.7, weight: 82.2 }
  ],
  // The types
  { name: Array, height: Float32Array, weight: Float64Array}
)

// Calculations are performed by the bound function in the WebAssembly instance
df['approx_density'] = df['height'] / df['weight']
```

Functions can also be added for performing calculations directly in JavaScript.
For example -

```javascript
Series.registerFunction(
  // The function name
  Symbol.for('**'),
  // A null prototype is used for JavaScript functions
  null,
  // The bound function implemented in JavaScript
  (lhs, rhs, length) => lhs.map((value, index) => value ** rhs))

const height = Series.from('height', [1.82, 1.72, 1.64, 1.88], Float64Array)
const sqrHeight = height ** 2
```

There is a restriction on function prototypes. The first argument is always the
array from the first series, and the last argument is always the length of the
first array.

Although we have used `Symbol.for` to define the function for names that map to
an operation, we can use a string for arbitrary calculations -

```javascript
Series.registerFunction(
  // The function name
  'fillna',
  // A null prototype is used for JavaScript functions
  null,
  // The bound function implemented in JavaScript
  (lhs, fill, length) => lhs.map(value => Number.isNaN(value) ? fill : value))

const height = Series.from('height', [1.82, 1.72, Number.NaN, 1.88], Float64Array)
const maxHeight = height.fillna(0)
```

### The WASI Marshaller

In order for the data frame to be aware of the WebAssembly instance and the
associated marshalling support, it must be initialized.

```javascript
// Read the wasm file.
const buf = fs.readFileSync(fileName)

// Create the Wasi instance passing in environment variables.
const envVars = {}
const wasi = new Wasi(envVars)

// Instantiate the wasm module.
const res = await WebAssembly.instantiate(buf, {
  wasi_snapshot_preview1: wasi.imports()
})

// Initialize the wasi instance
wasi.init(res.instance)

// Register the functions
registerUnmarshalledFunctions(wasi)
registerInt32Functions(wasi)
registerFloat64Functions(wasi)

// Initialize the data frame
DataFrame.init(wasm)
```

## Thoughts

Efficiency has improved enormously. As typed arrays are visible both
within the JavaScript interpreter and the WebAssembly instance, we only need to
pass the *reference* to the arrays when performing WebAssembly calculations. The
finalizer support means we don't need to manually allocate, copy and free memory.

We've done all this without adding a much clutter beyond specifying the series
type. This could be improved by adding some heuristics to guess the types.

All this means it's easier to use the power of WebAssembly without adding to
much burden to the data scientist.

While there is still much left to do (indices, selecting, support for more types
such as timestamps), this feels like a good step forward.