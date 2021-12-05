---
categories: ["Development"]
title: "Hiwire"
date: 2021-11-11T21:23:12-08:00
# author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: ""
tags: ["internals"]
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

Using `EM_JS` we can define Javascript functions that we call from C. However,
we can only pass numbers from Javascript into C. For instance, suppose we want
to invoke a Javascript function from C. We want to be able to say something like:
```C
EM_JS(JsRef, get_js_function, (), {
    let x = 0;
    let result = function f(){
        x++;
        return x;
    };
    return result;
})

EM_JS(int, call_js_func, (JsRef func), {
    return func();
})

int main(){
    JsRef func = get_js_function();
    // Maybe store func for a while, then later call
    int v1 = call_js_func(func); // should return 1
    int v2 = call_js_func(func); // should return 2
}
```
However, this doesn't work. When we try to return a Javascript object from
`get_js_function`, it just gets replaced with a 0. The [Wasm reference
types](https://github.com/WebAssembly/reference-types/blob/master/proposals/reference-types/Overview.md)
feature is designed for exactly this purpose. Unfortunately, Emscripten support
for reference types is still work in progress. 

Instead we use a side table. We define a map 
`let _hiwire = new Map();`

We define a `JsRef` to be a pointer to an empty struct:
```C
struct _JsRefStruct {};
typedef struct _JsRefStruct* JsRef;
```
To prevent C from implicitly casting `JsRef` to a different type, we compile
with  `-Werror=int-conversion` and `-Werror=incompatible-pointer-types`. Then we
can cast our hiwire keys to `JsRef` and not worry that they will ever be
confused for normal integers.

Now using `hiwire`, the original code works:
```C
EM_JS(JsRef, get_js_function, (), {
    let x = 0;
    let result = function f(){
        x++;
        return x;
    };
    return Module.hiwire.new_value(result);
})

EM_JS(int, call_js_func, (JsRef func), {
    func = Module.hiwire.get_value(func);
    return func();
})

int main(){
    JsRef func = get_js_function();
    // Maybe store func for a while, then later call
    int v1 = call_js_func(func); // should return 1
    int v2 = call_js_func(func); // should return 2
}
```