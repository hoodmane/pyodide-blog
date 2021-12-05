---
title: "Invoking Javascript From C"
date: 2021-11-11T20:26:18-08:00
draft: true
author: "Hood Chatham"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: ""
# canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: true
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

To create our bridge between Python and Javascript, we connect them via C. So
first we need good foreign function interface capabilities between C and Python
and between C and Javascript. 

Interoperation between C and Python comes with CPython. From C, we can call
Python functions using APIs like
[`PyObject_Call`](https://docs.python.org/3/c-api/call.html?highlight=pyobject_#object-calling-api).
We can also write [C extension
modules](https://docs.python.org/3/extending/extending.html?highlight=extending)
and this gives us a way to invoke C code from Python.

The interoperation between C and Javascript comes from Emscripten. We saw last
time how we can use `Module._some_c_function` to invoke C functions from
Javascript. There is one last direction we need which is to invoke Javascript
code from C. This last direction is somewhat different than the others because
the C code runs in side the web assembly virtual machine and is considered to be
less privileged than the host Javascript code. The Javascript browser tab could
be thought of as an operating system and calling Javascript functions from C is
like making a system call.

Javascript is the language of the host system in Emscripten. Luckily, we can
write both "privileged" Javascript code to make our "operating system" and
unprivileged C code to make our application. Emscripten provides a C macro
called
[`EM_JS`](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/Interacting-with-code.html#calling-javascript-from-c-c)
to declare a new Javascript function inside of C. It works as follows:
```c
#include <emscripten.h>

EM_JS(int /* return type */, my_func /* function name */, (int a, int b) /* arguments */, {
    // function body
    console.log("Was passed arguments a:", a, " and b:", b);
    return a+b;
});

int main(int argc, char** argsv){
    int res = my_func(5, 7);
    printf("res: %d"); // prints "res: 12"
}
```
The return type and input types are only used by the C type system. Everything
in C is just a number, and no matter what types are given to the arguments, all
the variables of `my_func` will just contain numbers. Similarly, the return
value had better be a boolean or a number. If we try to return something else, I
believe that it just gets replaced with a 0.

What is most interesting to me about `EM_JS` is that it needs some side channel
for getting the body of the function to the linker. When we invoke a C compiler,
all it produces is an object file. So the Javascript source code needs to go
into the object file somehow. The way this happens is that the Javascript code
is turned into a string which is placed into a special section of the object
code:
```C
__attribute__((section("em_js")))
char* __em_js__my_func = "{ console.log(\"Was passed arguments a:\", a, \" and b:\", b); return a+b; }"
```
The linker reads the `em_js` section and puts the data from this section into a
Javascript source file. Then it deletes the entire `em_js` section.

It is interesting to think about what it would take to make a `PY_CODE` macro
that does a similar thing with Python code. Unfortunately, we can see above that
whitespace was brutally modified when `EM_JS` converted the body of `my_func` to
a string. Indeed per the [GCC
documentation](https://gcc.gnu.org/onlinedocs/gcc-3.4.3/cpp/Stringification.html):

> Any sequence of whitespace in the middle of the text is converted to a single
> space in the stringified result.

So I guess a `PY_CODE` macro is a non-starter. Sometimes whitespace insensitive languages
come in handy.
