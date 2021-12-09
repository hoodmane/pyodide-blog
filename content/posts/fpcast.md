---
title: "Fpcast"
date: 2021-12-06T20:27:27-08:00
draft: true
tags: ["announcement"]
# author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Desc Text."
# canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---

# Function pointer cast emulation removal

## Some history

When I first started using Pyodide in summer 2020, my plan was to use it to
control the display of a niche type of mathematical charts. I hacked the Monaco
editor to make it behave like a repl, and then connected Jedi to Monaco
intellisense in order to get really good discoverability for my custom chart
API. This did not go as planned because of recursion errors.

The following is a minimal reproduction of the problem:
```py
import jedi
import numpy
jedi.Interpreter("numpy.in1d([1,2,3],[2]).", [globals()]).complete()
```

Run this in [Pyodide
0.16.1](https://pyodide-cdn2.iodide.io/v0.16.1/full/console.html) in Chrome and
you will see "RangeError: Maximum call stack size exceeded".

So as the users typed into my repl, some of the time the Jedi analysis would
cause these `RangeError`s. At first my response was to just throw them away,
reasoning that intellisense that works some of the time is better than
intellisense that works none of the time.

However, I gradually realized that the issue was more sinister. After one of
these `RangeError`s occurs, the Python interpreter is left in an invalid state.
They can happen part way through executing a Python opcode when some of the
interpreter state has been updated but other parts have not. Furthermore, the C
stack is unwound without taking the Python stack with it, leaving glitched
Python frame objects on the Python stack. After this, the situation may appear
normal, but subtle clues add up that something isn't right. Eventually, the
errors will snowball and the intepreter will crash. Or it might not, who knows.

## No protection

The analogous situation in a native Python would be if you set the recursion
limit very high and then did an infinite recurse:

```py
>>> import sys
>>> sys.setrecursionlimit(100000)
>>> def f(): f()
>>> f()
Segmentation fault (core dumped)
```

However, in this case the operating system kills the Python process and does not
give me an opportunity to continue using it. This is actually very helpful. The
wasm32-emscripten host environment that Pyodide runs in provides no such
protection. One of my first goals working on Pyodide was to fix this situation
by adding fatal error detection. Once this was added (in
[1131](https://github.com/pyodide/pyodide/pull/1131) and
[1151](https://github.com/pyodide/pyodide/pull/1151)) the situation was improved
because at least the interpreter was locked after the recursion error.

In fact, in web assembly it is even perfectly fine to dereference `NULL`. The
following code will not cause any sort of error:

```C
char *ptr = NULL;
*ptr = 'a'
```
Instead, a `96` is written into the first byte of the wasm memory. 

## Set the recursion limit lower?

Of course in order to trigger this bug in the native Python runtime, we had to
increase the recursion limit. The default recursion limit of 1000 is low enough
that in most cases Python won't segfault due to a stack overflow. Unfortunately,
in the version of Chrome on my desktop in Pyodide v0.16.1, to get
`jedi.Interpreter("numpy.in1d([1,2,3],[2]).", [globals()]).complete()` to throw
a Python `RecursionError` and not wreck the Python interpreter, instead of a
Javascript `RangeError`, we have to set the recursion limit to around 100. This
is unsatifactory for practically any use of Jedi, and indeed it's quite
restrictive for many applications.

Part of the difficulty determining a good way to set the recursion limit is that
different browsers on different systems come with a different amount of stack
space, different Python function calls can use very different amounts of stack
space, and the same call on different browsers can use different amounts of
stack space. We can't just set the recursion limit to a constant because some
browsers (firefox) can handle higher limits.

## How are we using up all this stack space?

Chromium sets the stack size to about 984 kilobytes. The source has the comment:
```C
// Slightly less than 1MB, since Windows' default stack size for
// the main execution thread is 1MB for both 32 and 64-bit.
```
Firefox manages to set the stack size several times higher, but for some reason a 
Python function call takes up about twice as much space on Firefox.

But 984 kilobytes is still a fair amount of space. Just what is happening that
Jedi manages to use that all in 120 Python call frames? That averages out to
over 8 kilobytes per call frame!

In my native Python,
```py
import sys; sys.setrecursionlimit(100_000)
def f(n): print(n); f(n+1)
f(0)
```

gets up to 20135 calls before the segmentation fault.

In Pyodide v0.17, looking at the stack trace when we cause an intentional stack
overflow, we see a bunch of units like:
```
    at pyodide.asm.wasm:wasm-function[767]:0x1cb77c
    at _PyFunction_Vectorcall (pyodide.asm.wasm:wasm-function[768]:0x1cb878)
    at byn$fpcast-emu$_PyFunction_Vectorcall (pyodide.asm.wasm:wasm-function[14749]:0x7a3736)
    at pyodide.asm.wasm:wasm-function[2764]:0x2ac6ae
    at _PyEval_EvalFrameDefault (pyodide.asm.wasm:wasm-function[2758]:0x2a980f)
    at byn$fpcast-emu$_PyEval_EvalFrameDefault (pyodide.asm.wasm:wasm-function[15508]:0x7a61f3)
    at PyEval_EvalFrameEx (pyodide.asm.wasm:wasm-function[2757]:0x2a4693)
```

This is actually repeated twice per Python function call. It turns out that the
calls `byn$fpcast-emu$_PyFunction_Vectorcall` and
`byn$fpcast-emu$_PyEval_EvalFrameDefault` are taking up over half the stack
space here.

### So what are these `byn$fpcast-emu$` calls?

`fpcast-emu` stands for "function pointer cast emulation". What is a function pointer cast?

```C
typedef int functype1(int);
typedef int functype2(int, int);
```




In most native architectures, a functions' arguments are stored


[
    ["x",      "0.16", "0.17", "0.18", "0.19"]
    ["chrome",    228,    279,   1659, 1946],
    ["firefox",   634,    920,   1991, 4358]

]
