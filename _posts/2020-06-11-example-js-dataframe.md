---
layout: post
title:  "An example of a pandas style DataFrame in JavaScript"
date:   2020-06-10 15:12:00 +0100
categories: JavaScript WebAssembly dataframe
---

# A DataFrame in JavaScript

This post describes an example implementation of a pandas style DataFrame. See
[here](https://github.com/rob-blackbourn/example-js-dataframe) for the source
code.

The goal is to be able to do the following.

```javascript
const df = DataFrame.fromObject(
  [
    { 'col0': 'a', 'col1': 5, 'col2': 8.1 },
    { 'col0': 'b', 'col1': 6, 'col2': 3.2 }
  ]
)

df['col3'] = df['col1'] + df['col2']
```

In plain old JavaScript this is not possible. Let's see how we can implement it.

## The goal

So what is a *DataFrame*? To answer this we first need to look at a *Series*.

### What is a Series?

A *Series* is the building block of a *DataFrame*. I take a *Series* to be a
named vector (array), which will be initialised as follows.

```javascript
let s1 = new Series('height', [1.82, 1.76, 1.72, 1.89])
```

The series should be indexable.

```javascript
s1[0] === 1.82
```

The series should support vector arithmetic.

```javascript
let height = new Series('height', [1.82, 1.76, 1.72, 1.89])
let weight = new Series('weight', [81.3, 73.2, 68.9, 92.1])
let score = height / weight
```

### What is a DataFrame

I take a *DataFrame* to be a collection of *Series*.

```javascript
let df = DataFrame([
  height,
  weight
])
```

## The problems to solve

There are two problems to solve in order to implement this in JavaScript:

* Operator overloading
* Property accessing

### Operator Overloading

We want to support vector arithmetic (multiplying two arrays). Unfortunately
JavaScript does not support operator overloading so we will have to pre-process
the code. We can do this with the [babel](https://babeljs.io) transpiler
and a plugin. I'm using the
[@jetblack/operator-overloading](https://github.com/rob-blackbourn/jetblack-operator-overloading)
plugin, which is a bag-of-bolts, but I wrote it so I know how it works! 

### Property accessing

In order for a series to have both a name and be indexable we need control over
the property accessing. We can do that with a
[Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
object. The `Proxy` object provides a layer of indirection between requests on
the object, and the actions performed.

## Setting up your environment

Lets write some code!

First install the node modules. I'm using babel, the operator overloading
plugin, and
[standardjs](https://standardjs.com) as a linter and formatter.

```bash
# Initialise the package.json
npm init -y
# Install the babel tool chain.
npm install --save-dev @babel/core@7.10.1 @babel/preset-env@7.10.2 @babel/cli@7.10.1 @babel/node@7.10.1
# Install the operator overloading plugin.
npm install --save-dev git+https://github.com/rob-blackbourn/jetblack-operator-overloading.git#0.1.0
# Install standardjs for linting and formatting
npm install --save-dev babel-eslint@10.1.0 standard@14.3.4
```

We configure standardjs by editing the package.json and adding the following.

```json
{
  ...
  "standard": {
    "parser": "babel-eslint"
  }
}
```

If you are using using vscode create the `.vscode/settings.json` as follows 
then restart vscode to start the standardjs server.

```json
{
  "javascript.validate.enable": false,
  "standard.enable": true,
  "standard.autoFixOnSave": true,
  "[javascript]": {
    "editor.formatOnSave": false
  }
}
```

Configure babel by creating the `.bablerc` file with the usual preset and the
operator overloading plugin. The operator overloading plugin requires arrow
functions. Targeting `node` (or any modern browser) achieves this.

```json
{
  "presets": [
      ["@babel/preset-env", {
        "targets": {
          "node": "current"
        }
      }]
  ],
  "plugins": ["module:@jetblack/operator-overloading"]
}
```

Edit the `package.json` and add some scripts for building the transpiled code
and running it.

```json
{
  ...
  "scripts": {
    "build": "babel src --out-dir=dist --source-maps",
    "start": "node dist/main.js"
  },
  ...
}
```

## The Implementation

### Series Code

In the `src` directory create a file called `Series.js` with the following
content.

```javascript
export class Series {
  constructor (name, array) {
    this.name = name
    this.array = array

    return new Proxy(this, {
      get: (obj, prop, receiver) => {
        if (prop in obj) {
          return Reflect.get(obj, prop, receiver)
        } else {
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

  [Symbol.for('+')] (other) {
    return new Series(
      `${this.name}+${other.name}`,
      this.array.map((value, index) => value + other.array[index])
    )
  }

  [Symbol.for('-')] (other) {
    return new Series(
      `${this.name}-${other.name}`,
      this.array.map((value, index) => value - other.array[index])
    )
  }

  [Symbol.for('*')] (other) {
    return new Series(
      `${this.name}*${other.name}`,
      this.array.map((value, index) => value * other.array[index])
    )
  }

  [Symbol.for('/')] (other) {
    return new Series(
      `${this.name}/${other.name}`,
      this.array.map((value, index) => value / other.array[index])
    )
  }

  [Symbol.for('/')] (other) {
    return new Series(
      `${this.name}/${other.name}`,
      this.array.map((value, index) => value / other.array[index])
    )
  }

  toString () {
    return `(${this.name}): ${this.array.join(', ')}`
  }
}
```

The code can be split into two parts.

The constructor returns a `Proxy`
object, which  intercepts calls to the Series. It first checks if the property
or function is provided by the Series itself. If not it delegates the action
to the array. Note how we the `Proxy` is returned from the constructor.

The operators are provided by the `[Symbol.for('+')]` methods.

### DataFrame Code

In the `src` directory create a file called `DataFrame.js` with the following
content.

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
        return prop in obj ? Reflect.set(obj, prop, value, receiver) : Reflect.set(obj.series, prop, value, receiver.series)
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

  static fromObject (data) {
    const series = {}
    for (let i = 0; i < data.length; i++) {
      for (const column in data[i]) {
        if (!(column in series)) {
          series[column] = new Series(column, new Array(data.length))
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
    return s
  }
}
```

As with the `Series` class we use a `Proxy` object to control property
accessing.

I decided to keep the constructor semantically clean; it just takes an array
of `Series`. However, in the real world we want a variety of constructors. The
convenience class method `DataFrame.fromObject` provides a way of building the series from
a list of objects.

## Try it out

Let's start with a series.

```javascript
'operator-overloading enabled'

import { Series } from './Series'

const height = new Series('height', [1.82, 1.72, 1.64, 1.88])
const weight = new Series('weight', [81.4, 72.3, 69.9, 79.5])
const ratio = weight / height
console.log(ratio.toString())

> (weight/height): 44.72527472527473, 42.03488372093023, 42.6219512195122, 42.287234042553195
```

And lets test using array methods to see if the rquest gets forwarded by the
proxy to the array.

```javascript
import { Series } from './Series'

const s1 = new Series('numbers', [1, 2, 3, 4])
s1.push(5)
console.log(s1.toString())

> (numbers): 1, 2, 3, 4, 5
```

Finally let's test the DataFrame.

```javascript
'operator-overloading enabled'

import { DataFrame } from './DataFrame'

const df = DataFrame.fromObject(
  [
    { col0: 'a', col1: 5, col2: 8.1 },
    { col0: 'b', col1: 6, col2: 3.2 }
  ]
)
console.log(df.toString())
> col0, col1, col2
> a, 5, 8.1
> b, 6, 3.2

df['col3'] = df['col1'] + df['col2']
console.log(df.toString())
> col0, col1, col2, col3
> a, 5, 8.1, 13.1
> b, 6, 3.2, 9.2
```

## Thoughts

Obviously there's a lot  in a pandas DataFrame not covered here, but I think this
demonstrates how the basic syntax could be achieved.
