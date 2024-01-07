---
layout: post
title:  "Implementing DataFrames in JavaScript with Calculations in WebAssembly"
date:   2020-06-13 12:41:00 +0100
categories:
- WebAssembly
tags:
- WebAssembly
- wasm
- array
- arrays
- JavaScript
- C
- DataFrame
comments: true
---

This post describes a simple pandas style DataFrame implementation with the
array calculations performed in JavaScript.

The code for this post can be found [here](https://github.com/rob-blackbourn/example-wasm-dataframe-1).

## The DataFrame and Series Implementations

The series and data frame implementations are taken from my
[previous post](https://rob-blackbourn.github.io/blog/javascript/webassembly/dataframe/2020/06/10/example-js-dataframe.html)
with minor modifications.

### Type awareness

The series (and therefore the data frame) will need to be aware of the type of
data they contain. For simplicity I have limited the types to:

* `'int'`
* `'double'`
* `'object'`

A series is constructed as follows.

```javascript
const s1 = new Series('height', [1.82, 1.73, 1.69, 1.92], 'double')
```

The code for the series now looks like this.

```javascript
import arrayMethods from './array-methods'

export class Series {
  constructor (name, array, type) {
    this.name = name
    this.array = array
    this.type = type

    return new Proxy(this, {
      get: (obj, prop, receiver) => {
        if (prop in obj) {
          // This is a known property: i.e. name, arry or type.
          return Reflect.get(obj, prop, receiver)
        } else if (arrayMethods.has(prop)) {
          // The property is an operator.
          return (...args) => new Series('', ...arrayMethods.get(prop)(obj, ...args))
        } else {
          // The property requested is passed to the array.
          return Reflect.get(obj.array, prop, receiver.array)
        }
      },
      set: (obj, prop, value, receiver) => {
        if (prop in obj) {
          return Reflect.set(obj, prop, value, receiver)
        } else {
          return Reflect.set(obj.array, prop, value, receiver.array)
        }
      },
      apply: (target, thisArgument, argumentList) => {
        return Reflect.apply(target, thisArgument, argumentList)
      },
      // construct: Reflect.construct,
      defineProperty: Reflect.defineProperty,
      getOwnPropertyDescriptor: Reflect.getOwnPropertyDescriptor,
      deleteProperty: Reflect.deleteProperty,
      getPrototypeOf: Reflect.getPrototypeOf,
      setPrototypeOf: Reflect.setPrototypeOf,
      isExtensible: Reflect.isExtensible,
      preventExtensions: Reflect.preventExtensions,
      has: Reflect.has,
      ownKeys: Reflect.ownKeys
    })
  }

  toString () {
    return `(${this.name};${this.type}): ${this.array.join(', ')}`
  }
}
```

Comparing this to the
[previous implementation](https://github.com/rob-blackbourn/example-js-dataframe/blob/master/src/Series.js)
we can see three changes (plus a minor change to `toString`).

The constructor takes in an extra `type` parameter.

The operations are no longer defined in the class; they are now provided by the `array-methods` module.

```javascript
import arrayMethods from './array-methods'
```

The last change is to modify the way the proxy finds the operator methods with `arrayMethods.has(prop)`.

```javascript
    return new Proxy(this, {
      get: (obj, prop, receiver) => {
        if (prop in obj) {
          return Reflect.get(obj, prop, receiver)
        } else if (arrayMethods.has(prop)) {
          return (...args) => new Series('', ...arrayMethods.get(prop)(obj, ...args))
        } else {
          return Reflect.get(obj.array, prop, receiver.array)
        }
      },
      ...
    }
```

Here is the implementation of the DataFrame.

```javascript
import { Series } from './Series'

export class DataFrame {
  constructor (series) {
    this.series = {}
    for (const item of series) {
      this.series[item.name] = item
    }

    return new Proxy(this, {
      get: (obj, prop, receiver) => {
        return prop in obj ? Reflect.get(obj, prop, receiver) : Reflect.get(obj.series, prop, receiver.series)
      },
      set: (obj, prop, value, receiver) => {
        if (prop in obj) {
          Reflect.set(obj, prop, value, receiver)
        } else {
          value.name = prop
          return Reflect.set(obj.series, prop, value, receiver.series)
        }
      },
      apply: (target, thisArgument, argumentList) => {
        return target in thisArgument ? Reflect.apply(target, thisArgument, argumentList) : Reflect.apply(target, thisArgument.array, argumentList)
      },
      defineProperty: Reflect.defineProperty,
      getOwnPropertyDescriptor: Reflect.getOwnPropertyDescriptor,
      deleteProperty: Reflect.deleteProperty,
      getPrototypeOf: Reflect.getPrototypeOf,
      setPrototypeOf: Reflect.setPrototypeOf,
      isExtensible: Reflect.isExtensible,
      preventExtensions: Reflect.preventExtensions,
      has: Reflect.has,
      ownKeys: Reflect.ownKeys
    })
  }

  static fromObject (data, types) {
    const series = {}
    for (let i = 0; i < data.length; i++) {
      for (const column in data[i]) {
        if (!(column in series)) {
          series[column] = new Series(column, new Array(data.length), types[column])
        }
        series[column][i] = data[i][column]
      }
    }
    const seriesList = Object.values(series)
    return new DataFrame(seriesList)
  }

  toString () {
    const columns = Object.getOwnPropertyNames(this.series)
    let s = columns.join(', ') + '\n'
    const maxLength = Object.values(this.series)
      .map(x => x.length)
      .reduce((accumulator, currentValue) => Math.max(accumulator, currentValue), 0)
    for (let i = 0; i < maxLength; i++) {
      const row = []
      for (const column of columns) {
        if (i < this.series[column].length) {
          row.push(this.series[column][i])
        } else {
          row.push(null)
        }
      }
      s += row.join(', ') + '\n'
    }
    s += columns.map(column => this.series[column].type).join(', ') + '\n'
    return s
  }
}
```

The only changes are in the `fromObject` helper method which takes an extra
argument for the types and the `toString` to print the types. A DataFrame is now constructed as follows.

```javascript
const df = DataFrame.fromObject(
  [
    { col0: 'a', col1: 5, col2: 8.1 },
    { col0: 'b', col1: 6, col2: 3.2 }
  ],
  { col0: 'object', col1: 'int', col2: 'double'}
)
```

## Array Methods

A new file has been added to create an `ArrayMethods` singleton. This is used
as a central store of array methods.

```javascript
class ArrayMethods {
  constructor () {
    if (!ArrayMethods.instance) {
      this._methods = {}
      ArrayMethods.instance = this
    }

    return ArrayMethods.instance
  }

  set (name, method) {
    this._methods[name] = method
  }

  has (name) {
    return name in this._methods
  }

  get (name) {
    return this._methods[name]
  }
}

const instance = new ArrayMethods()
Object.freeze(instance)

export default instance
```

## The WebAssembly Calculations

The calculations have been written in C following the methods describe in
[this post](https://rob-blackbourn.github.io/blog/webassembly/wasm/array/arrays/javascript/c/2020/06/07/wasm-arrays.html),
and [this post](https://rob-blackbourn.github.io/blog/javascript/webassembly/clang/wasm/memory/malloc/2020/06/10/simplifyinf-memory-management.html).

Here is the code for addition.

```c
__attribute__((used)) int* addInt32Arrays (int *array1, int* array2, int length)
{
  int* result = (int*) allocateMemory(length * sizeof(int));
  if (result == 0)
    return 0;

  for (int i = 0; i < length; ++i) {
    result[i] = array1[i] + array2[i];
  }

  return result;
}

__attribute__((used)) double* addFloat64Arrays (double* array1, double* array2, int length)
{
  double* result = (double*) allocateMemory(length * sizeof(double));
  if (result == 0)
    return 0;

  for (int i = 0; i < length; ++i) {
    result[i] = array1[i] + array2[i];
  }

  return result;
}
```

Note how we need one function for integers and another for doubles.

There is a makefile to build the wasm.

## Marshalling JavaScript to WebAssembly

I have tidied up the marshalling between JavaScript and WebAssembly. I created
a class to manage the wasm functions.

```javascript
export class WasmFunctionManager {
  constructor (memory, allocateMemory, freeMemory) {
    this.memory = memory
    this.allocateMemory = allocateMemory
    this.freeMemory = freeMemory
  }

  createTypedArray (typedArrayType, length) {
    const typedArray = new typedArrayType(
      this.memory.buffer,
      this.allocateMemory(length * typedArrayType.BYTES_PER_ELEMENT),
      length)

    if (typedArray.byteOffset === 0) {
      throw new RangeError('Unable to allocate memory for typed array')
    }

    return typedArray
  }

  invokeUnaryFunction(func, array, typedArrayType) {
    let input = null
    let output = null

    try {
      input = this.createTypedArray(typedArrayType, array.length)
      input.set(array)

      output = new typedArrayType(
        this.memory.buffer,
        func(input.byteOffset, array.length),
        array.length
      )

      if (output.byteOffset === 0) {
        throw new RangeError('Failed to allocate memory')
      }

      const result = Array.from(output)

      return result
    } finally {
      // Ensure the memory gets freed.
      this.freeMemory(input.byteOffset)
      this.freeMemory(output.byteOffset)
    }
  }

  invokeBinaryFunction(func, lhs, rhs, typedArrayType) {
    if (lhs.length !== rhs.length) {
      throw new RangeError('Arrays must the the same length')
    }
    const length = lhs.length

    let input1 = null
    let input2 = null
    let output = null

    try {
      input1 = this.createTypedArray(typedArrayType, length)
      input2 = this.createTypedArray(typedArrayType, length)

      input1.set(lhs)
      input2.set(rhs)

      output = new typedArrayType(
        this.memory.buffer,
        func(input1.byteOffset, input2.byteOffset, length),
        length
      )

      if (output.byteOffset === 0) {
        throw new RangeError('Failed to allocate memory')
      }

      const result = Array.from(output)

      return result
    } finally {
      // Ensure the memory gets freed.
      this.freeMemory(input1.byteOffset)
      this.freeMemory(input2.byteOffset)
      this.freeMemory(output.byteOffset)
    }
  }
}
```

This follows the ideas presented in the previous posts, but adds some error
checking, and a `try ... finally` clause to prevent memory leaks.

## Setting up the operators

To set up the operators we first need to decide whether to use the `'int'`,
`'double'` or `'object'` functions.

```javascript
function chooseBestType(lhsType, rhsType) {
  if (lhsType === 'int' && rhsType == 'int') {
    return 'int'
  } else if (
    (lhsType === 'int' && rhsType === 'double') ||
    (lhsType === 'double' && rhsType === 'int')) {
    return 'double'
  } else {
    return 'object'
  }
}
```

Once we have chosen the type we need to build a wrapper function to invoke the
appropriate method. Here is the helper that makes a binary operation.

```javascript
function makeBinaryOperation(wasmFunctionManager, intFunc, doubleFunc, defaultFunc) {
  return (lhs, rhs) => {
    const bestType = chooseBestType(lhs.type, rhs.type)

    if (bestType === 'int') {
      const result =  wasmFunctionManager.invokeBinaryFunction(
        intFunc,
        lhs.array,
        rhs.array,
        Int32Array
      )
      return [result, bestType]
    } else if (bestType === 'double') {
      const result = wasmFunctionManager.invokeBinaryFunction(
        doubleFunc,
        lhs.array,
        rhs.array,
        Float64Array
      )
      return [result, bestType]
    } else {
      const result = defaultFunc(lhs, rhs)
      return [result, bestType]
    }
  }
}
```

Lastly we create the wasm instance and register the types. Here is an edited version which demonstrates registering the addition operator.

```javascript
export async function setupWasm () {
  // Read the wasm file.
  const buf = fs.readFileSync('./src-wasm/data-frame.wasm')

  // Instantiate the wasm module.
  const res = await WebAssembly.instantiate(buf, {})

  // Get the memory exports from the wasm instance.
  const {
    memory,
    allocateMemory,
    freeMemory,

    addInt32Arrays,
    addFloat64Arrays,
 
    // .. import the rest of the methods
  } = res.instance.exports

  const wasmFunctionManager = new WasmFunctionManager(memory, allocateMemory, freeMemory)

  // Register the add method.
  arrayMethods.set(
    Symbol.for('+'),
    makeBinaryOperation(
      wasmFunctionManager,
      addInt32Arrays,
      addFloat64Arrays,
      (lhs, rhs) => lhs.array.map((value, index) => value + rhs.array[index])
    )
  )

  // register the rest of the methods ...
}
```

## Running the code

We can run the code as follows.

```javascript
import { DataFrame } from './DataFrame'
import { setupWasm } from './setup-wasm'

function example () {
  'operator-overloading enabled'

  const df = DataFrame.fromObject(
    [
      { col0: 'a', col1: 5, col2: 8.1 },
      { col0: 'b', col1: 6, col2: 3.2 }
    ],
    { col0: 'object', col1: 'int', col2: 'double'}
  )
  console.log(df.toString())
  df['col3'] = df['col1'] + df['col2']
  console.log(df.toString())
}

async function main () {
  await setupWasm()

  example()
}

// Run the async main function.
main().then(() => console.log('Done')).catch(error => console.error(error))
```

## Thoughts

Clearly the DataFrame needs a lot of work. There is no concept of grouping,
indices, dates and times, etc. However we have a working proof on concept that provides a syntactically elegant and efficient implementation.
