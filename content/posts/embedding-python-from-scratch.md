---
title: "Embedding Python from Scratch"
date: 2021-11-10T22:39:24-08:00
draft: true
tags: [""]
author: "Hood Chatham"
showToc: false
TocOpen: false
draft: false
hidemeta: false
comments: false
description: ""
# canonicalURL: "https://canonical.url/to/page"
disableHLJS: false # to disable highlightjs
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
One of the key components of Pyodide is building CPython into a static wasm
library. Once we've built this library, we can write some code using the Python
apis, compile it, and link it together into an Emscripten binary.

One question that 

[https://github.com/dgym/cpython-emscripten]

```
emcc main.o -o python.asm.js -llibpython3.9 \
    # Some other options
    -s EXPORT_NAME="'createWasmModule'" \
    # have to make a file system and load the Python standard library into it
    -s FORCE_FILESYSTEM \ 
    --preload-file cpython/installs/python3.9/lib/python3.9@/lib/python3.9
```
this creates a

Suppose we have built CPython with Emscripten and we wish to embed the Python
runtime in the browser, but we don't have any of the Pyodide core code. The
built emscripten code automatically instantiates a web assembly module and
exports a convenient way to call C functions from Javascript. In particular, the
instantiation process creates an object named `Module` and a C function named
`some_name` is accessible as `Module._some_name`. We would like to start
executing Python code.

The first step of [embedding
Python](https://docs.python.org/3/extending/embedding.html) is to initialize the
Python interpreter. There are [many
parameters](https://docs.python.org/3/c-api/init_config.html) to configure
Python initialization and Pyodide uses a few of them, but for the simplest usage
we can just use the vanilla configuration. Calling
[`Py_Initialize()`](https://docs.python.org/3/c-api/init.html?highlight=py_initialize#c.Py_Initialize)
starts the Python interpreter in the default configuration. From Javascript, we
can do this with `Module._PyInitialize`.

After calling `Py_Initialize()`, the Python interpreter is up and running. We
want to ask it to run code. The simplest C api to run Python code is called
[`PyRun_SimpleString(code)`](https://docs.python.org/3/c-api/veryhigh.html?highlight=pyrun_simplestring#c.PyRun_SimpleString)
which takes a `char*` as an argument and returns a status code of `0` on success
and `-1` if a Python exception was raised. 

We want to accept the Python code as a Javascript string not a C string, so we
will need to convert the string before we can give it to `PyRun_SimpleString`.
Luckily, Emscripten has a convenient (but undocumented) API to convert
Javascript strings to C strings called `stringToNewUTF8`. This function measures
how much space the corresponding C string will take up, mallocs space for it,
then converts the Javascript string into the malloc'd memory. 

A complete `runPythonSimple` function
that runs a Javascript string as Python code looks as follows:
```js
function runPythonSimple(code){
    // Allocate space on wasm heap for code string and copy it into wasm heap.
    // code_c_string is the pointer to this string.
    const code_c_string = Module.stringToNewUTF8(code);
    // Run code as Python
    const errcode = Module._PyRun_SimpleString(code_c_string);
    // Free the C string
    Module._free(code_c_string);
    if(errcode){
        // If an error occurred, print the stack trace and clear the Python 
        // error flag.
        Module._PyErr_Print();
    }
}
```
Note that exported C functions have a leading underscore added to their name
like `PyRun_SimpleString` is accessed via `Module._PyRun_SimpleString`.
Functions with no leading underscore like `Module.stringToNewUTF8` are
Emscripten APIs. We can use this function to run Python code:
```js
runPythonSimple("x = [2*x+1 for x in range(10)]");
runPythonSimple("print('x:', x)"); // prints "x: [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]"
runPythonSimple("from statistics import mean; print(mean(x))"); // prints "10"
runPythonSimple("1+"); /* prints via console.warn:
    File "<string>", line 1
    1+
        ^
SyntaxError: invalid syntax
*/
```
[Here is a gist](https://gist.github.com/hoodmane/4a85fb0d961f8f014c472974a17f9c08) 
with a complete working example.

### What is missing

This is already fairly powerful, but note that there is no way to get results
from Python back into Javascript, to call a Python function with Javascript
arguments, or to call a Javascript function from Python. All of these
capabilities are provided by Pyodide core.

Pyodide core provides the following key features:
1. Conversions for various types between Javascript and Python.
2. Idiomatic access to Javascript objects from Python and vice versa.
3. Translation of errors between Javascript and Python.
4. The ability to implement Python modules in Javascript.
