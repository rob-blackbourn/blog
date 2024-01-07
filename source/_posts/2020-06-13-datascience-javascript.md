---
layout: post
title:  "The future of Data Science is JavaScript and WebAssembly"
date:   2020-06-13 14:57:00 +0100
categories:
- WebAssembly
tags:
- datascience
- data
- science
- WebAssembly
- wasm
- array
- arrays
- JavaScript
- C
- DataFrame
comments: true
---

In my previous posts
[[1]](https://rob-blackbourn.github.io/blog/webassembly/wasm/array/arrays/javascript/c/2020/06/07/wasm-arrays.html)
[[2]](https://rob-blackbourn.github.io/blog/javascript/webassembly/clang/wasm/memory/malloc/2020/06/10/simplifyinf-memory-management.html)
[[3]](https://rob-blackbourn.github.io/blog/javascript/webassembly/dataframe/2020/06/10/example-js-dataframe.html)
[[4]](https://rob-blackbourn.github.io/blog/webassembly/wasm/array/arrays/javascript/c/dataframe/2020/06/13/wasm-dataframes.html)
I demonstrated how to write a simple [pandas](https://pandas.pydata.org)-style
data frame in JavaScript using [WebAssembly](https://webassembly.org/) (a near
native speed virtual machine available in every modern browser or server-side
engine such as [nodejs](https://nodejs.org/en/)).
This demonstrated the following features:

* Elegant Syntax

  Expressions can be written in the following manner.

  ```javascript
  const df = DataFrame.fromObject(
    [
      { col0: 'a', col1: 5, col2: 8.1 },
      { col0: 'b', col1: 6, col2: 3.2 }
    ],
    { col0: 'object', col1: 'int', col2: 'double'}
  )

  df['col3'] = df['col1'] + df['col2']
  ```

* Near native speed.

  All operations are written in C and are performed in a compiled WebAssembly
  module.

* Portable.

  The code runs in **any** browser or server side JavaScript engine (e.g.
  chrome, nodejs, etc.).

* Uses standard tooling.

  The C code was compiled with the current standard
  [clang](https://clang.llvm.org/)
  compiler which supports WebAssembly
  [out of the box](https://lld.llvm.org/WebAssembly.html).

## The Future of Data Science?

Clearly this is a bold claim!

There are currently two clear platform leaders in the data science eco-system:
[R](https://cran.r-project.org/),
and [Python](https://www.python.org/)
using the [scipy](https://www.scipy.org/) toolkit,
and also a number of attractive outsiders. The choice of which platform to
choose typically depends on language preference, but more compellingly what
packages a particular language/version provides. It is not unusual for a project
to be constrained to a legacy version of a platform simply because a key package
has not been ported to a current version.

So how can JavaScript and WebAssembly help?

### Elegant Syntax

Surprisingly, yes! The posts referenced above clearly demonstrate how JavaScript
can be written in a manner which elegantly represents vector-arithmetic.

### Speed

With WebAssembly there is no significant compromise in speed over natively
compiled languages.

### Bindings

With other languages a *binding* must be written to integrate a package
into the language. In many cases this is **completely** unnecessary for
WebAssembly.

### Fortran

In my opinion this is the killer advantage. There are a huge number of key data
science libraries implemented in many different flavours of Fortran. Binding to
these libraries is an ongoing and fraught task.

Happily [LLVM](https://llvm.org) (a language compiler supported by the majority
of operating systems) is currently incorporating
[flang](https://github.com/flang-compiler/flang)
(a Fortran compiler) into it's toolchain, supporting **all** versions of Fortran
up to and including 2018. This provides the possibility of compiling **any**
Fortran package directly to WebAssembly.

## The Catch

The first catch is operator overloading. I'm not sure if "catch" is the right
word. My solution using the [babel](https://babeljs.io) transpiler works fine,
and transpiling is how [React](https://reactjs.org) (one of the most popular web
frameworks, which uses custom syntax in JavaScript) does and will always work.
There is a [proposal](https://github.com/tc39/proposal-operator-overloading)
to incorporate operator overloading into JavaScript (currently at stage 1 where
3 means acceptance), which leads me to believe it is a feature which my become
native to the language.

The second catch is the standard library. You'll note if you've read the
previous posts that I had to provide a memory allocator as this is not available
in the WebAssembly environment. WebAssembly is just a virtual machine, and at
present it doesn't come with the usual support code which most libraries depend
on. This is all being built out at the present in the form of
[wasi](https://wasi.dev/), but it's not here yet.

Lastly flang isn't in the current stable release of LLVM. It was 
[incorporated in the repo in November 2019](https://www.youtube.com/watch?v=yenZorebMOA),
and I would expect it to be in the next major release.

## Do I Want to Write in JavaScript?

Personally I don't care what language I write in, as long as it doesn't get in
the way of the problem I'm trying to solve. Because of the low cost of entry,
JavaScript has become ubiquitous, and so it's level of documentation and support
is huge, which I like.

If it provides seamless access to the packages I need I'm a buyer.
