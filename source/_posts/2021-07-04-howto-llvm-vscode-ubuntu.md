---
layout: post
title:  "How To Use clang With vscode On Ubuntu 20.04"
date:   2021-07-04 14:36:00 +0100
categories:
- Admin
tags:
- vscode
- clang
- llvm
- ubuntu
- ubuntu-20.04
comments: true
---

## Introduction

I wanted to use the clang compiler for a C++ project, using vscode as the IDE;
simples? No!!!

I'm running Ubuntu 20.04, and it seemed the obvious approach was to install the
default llvm tool chain (`sudo app install llvm`), write the obligatory "hello
world" program, and go to the vscode debug page and click "create a launch.json
file". So far so good. Now I hit the *play* button and I get the error "unable
to find /usr/local/bin/lldb-mi". WTF?

Now we go down the rabbit hole. You may choose to take the blue pill and move
one, or skip to the *solution*.

## lldb-mi

Googling the error provided some useful facts.

* lldb is the debugger for the llvm tool-chain,
* lldb-mi is an executable which is not part of the base llvm project,
* Ubuntu or llvm have stopped shipping this executable with the llvm package.

Given we are C++ programmers and therefore not lightweight, we choose to build
[lldb-mi](https://github.com/lldb-tools/lldb-mi). After cloning the project
we get an error creating the cmake build file with the default gcc compiler
which complains about `lib_lldb`. Google has little to offer here. I bite the
bullet and decide to build the entire llvm tool-chain.

## llvm

After cloning llvm and following the build instructions the build starts using
`gcc` as the compiler. I notice that 10% of the build has occurred in 10
minutes, so I go for a coffee. At some point the build crashes on my 12 core
64GB machine, as it runs out of memory!

After a little more googling I install the `lld` linker and set the build type
to "release" to reduce memory usage. I kick off the build again, and now I get
a compilation error complaining about `_Atomic` being undefined. Again googling
doesn't help much, so I decide to use the llvm tool chain to build itself. After
installing llvm and setting the C and C++ compiler to clang/clang++ I try again
and it works!

## lldb-mi revisited

Using my compiled llvm tool chain I can successfully compile `lldb-mi`, and
vscode now allows me to debug my program. Superb!

## Solution

Install the llvm tool chain with the `lld` linker. Also use `ninja` instead of
`make` (`cmake` should be installed already).

```bash
sudo apt install lld
sudo apt install llvm
sudo apt install ninja-build
```

Clone llvm and build it. I used version 12.

```bash
git clone --depth 1 --branch release/12.x git@github.com:llvm/llvm-project.git
cd llvm-project
mkdir build
cd build
LLVM_PROJECTS="clang;clang-tools-extra;compiler-rt;debuginfo-tests;libc;libclc;libcxx;libcxxabi;libunwind;lld;lldb;mlir;openmp;parallel-libs;polly;pstl"
LLVM_TARGETS="X86"
INSTALL_PREFIX=$HOME/local/llvm-12.0.0
C_COMPILER=/usr/bin/clang
CXX_COMPILER=/usr/bin/clang++
LINKER=lld
cmake \
    -G Ninja \
    -DCMAKE_INSTALL_PREFIX=$INSTALL_PREFIX \
    -DCMAKE_BUILD_TYPE=Release \
    -DLLVM_USE_LINKER=$LINKER \
    -DCMAKE_C_COMPILER=$C_COMPILER \
    -DCMAKE_CXX_COMPILER=$CXX_COMPILER \
    -DLLVM_ENABLE_PROJECTS="$LLVM_PROJECTS" \
    -DLLVM_TARGETS_TO_BUILD="$LLVM_TARGETS" \
    ../llvm
cmake --build .
cmake --build . --target install
```

Build lldb-mi:

```bash
git clone git@github.com:lldb-tools/lldb-mi.git
cd lldb-mi
mkdir build
cd build
cmake -G Ninja -DCMAKE_INSTALL_PREFIX=$INSTALL_PREFIX ..
cmake --build .
cmake --build . --target install
```

Assuming you have the tool-chain on your path:

```bash
export PATH=$HOME/local/lvmm-12.0.0/bin:$PATH
```

The vscode generated files looked like this:

`launch.json`

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "clang++ - Build and debug active file",
            "type": "cppdbg",
            "request": "launch",
            "program": "${fileDirname}/${fileBasenameNoExtension}",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${fileDirname}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "lldb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "C/C++: clang++ build active file",
            "miDebuggerPath": "/home/rob/local/llvm-12.0.0/bin/lldb-mi"
        }
    ]
}
```

`tasks.json`

```json
{
    "tasks": [
        {
            "type": "cppbuild",
            "label": "C/C++: clang++ build active file",
            "command": "/home/rob/local/llvm-12.0.0/bin/clang++",
            "args": [
                "-g",
                "${file}",
                "-o",
                "${fileDirname}/${fileBasenameNoExtension}"
            ],
            "options": {
                "cwd": "${fileDirname}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "detail": "Task generated by Debugger."
        }
    ],
    "version": "2.0.0"
}
```

## Epilogue

It's a day of my life that I won't get back, but I'm loving the speed of the
compiler and the more sensible error messages.
