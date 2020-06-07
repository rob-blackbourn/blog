---
layout: post
title:  "JavaScript Operator Overloading"
categories: JavaScript operator overloading
---

# Operator Overloading in JavaScript

Operator overloading is a topic which can bring language communities close to a
religious war. For one side it simplifies the expression of a solution, for the
other it obfuscates the solution by fundamentally re-writing the language
semantics.

I am looking at how vector arithmetic might be supported in JavaScript. Given
two arrays: `a1` and `a2` there are two approaches.

Without operator overloading:

```javascript
const a3 = multiplyArray(a1, a2)
```

With operator overloading:

```javascript
const a3 = a1 * a2
```

Now I've resolved that debate lets move to a potential solution.

## Babel

The [babel](https://babeljs.io/) transpiler transforms JavaScript into an
abstract syntax tree that can be modified by plugins before transforming it
back into JavaScript. This means we can intercept expressions like `a1 * a2`
and re-write the code to `multiply(a1, a2)`.

If these transpilations are popular they may be incorporated into the core
language, at which point the plugin can be retired.

## @jetblack/operator-overload-plugin

The [@jetblack/operator-overloading](https://github.com/rob-blackbourn/jetblack-operator-overloading)
plugin intercepts operator expressions allowing them to support new semantics:
including the vector operations described above.

## Usage

The project is described in more detail [here](https://github.com/rob-blackbourn/jetblack-operator-overloading).

The following code adds two integers and then two points.

The directive at the start is required to enable the transformation.

```javascript
'operator-overloading enabled'

class Point {

    constructor(x, y) {
        this.x = x
        this.y = y
    }
    
    [Symbol.for('+')](other) {
        const x = this.x + other.x
        const y = this.y + other.y
        return new Point(x, y)
    }
}

// Built in operators still work.
const x1 = 2
const x2 = 3
const x3 = x1 + x2
console.log(x3)

// Overriden operators work!
const p1 = new Point(5, 5)
const p2 = new Point(2, 3)
const p3 = p1 + p2
console.log(p3)
```
produces the following output:
```bash
5
Point { x: 7, y: 8 }
```

## Description

The plugin wraps expressions in arrow functions. The following is produced for `x + y`.

```javascript
(() => {
  'operator-overloading disabled';

  return x !== undefined && x !== null && x[Symbol.for("+")]
    ? x[Symbol.for("+")](y)
    : x + y;
})()
```

For example the following code:
```javascript
let x = 1, y = 2
let z = x + y
```
gets re-written as:
```javascript
let x = 1, y = 2
let z = (() => {
  return x !== undefined && x !== null && x[Symbol.for("+")]
    ? x[Symbol.for("+")](y)
    : x + y;
})();
```

This allows the creation of custom overrides such as:
```javascript
class Point {

    constructor(x, y) {
        this.x = x
        this.y = y
    }
    
    [Symbol.For('+')](other) {
        const x = this.x + other.x
        const y = this.y + other.y
        return new Point(x, y)
    }
}
```

