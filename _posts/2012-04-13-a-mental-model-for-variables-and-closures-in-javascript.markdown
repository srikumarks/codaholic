---
layout: post
title: "A mental model for variables and closures in Javascript"
description: ""
sharing: true
footer: true
categories: Javascript
---

Closures and variables have a strained relationship in Javascript that causes
much confusion among newcomers and results in hard to spot bugs even for
experienced JS coders. It is good to have a clear and accurate "mental model"
of this relationship using which you can correctly predict what would happen
with any given piece of code.

I came up with such a mental model a while back and posted it on [Hacker News]
... which I reproduce here.

<!-- more -->

Pretend that variables refer to storage locations in an array and you
identify them using indices - `v = [..., v[100], v[101], v[102], v[103], ....]`

For example, take the function in [the stackoverflow answer] -

``` js
function foo(x) {
    var tmp = 3;
    return function (y) {
        alert(x + y + (++tmp));
    }
}
```

When you make a call to `foo(10)`, pretend that the bound variables in the body
of the function get assigned sequence numbers starting from 100 -

``` js
function foo() {
    v[100] = 10; // x = 10
    v[101] = 3;  // var tmp = 3;

    return function (y) {
      alert(v[100] + y + (++(v[101])));
    }
}
```

So you get as the result,

``` js
function (y) {
    alert(v[100] + y + (++(v[101])));
}
```

`y` remains as such because it hasn't been bound to a value just yet. That will
happen only when you call this result function.

Now, when you call foo another time like `foo(20)`, the sequence continues from 102 ...

``` js
function foo() {
    v[102] = 20; // x = 20
    v[103] = 3;  // var tmp = 3;

    return function (y) {
        alert(v[102] + y + (++(v[103])));
    }
}
```

So you get another function as the result -
    
``` js
function (y) {
    alert(v[102] + y + (++(v[103])));
}
```

The store now reads `v[100] = 10, v[101] = 3, v[102] = 20 and v[103] = 3`.  It
becomes clear what the two result functions do. Note that they do not share the
same storage locations and therefore the two `++` calls increment different
storage locations.  In this model, each `var` statement and each argument of a
function causes the index assigned to be incremented on a function call, and
unbound variables do not cause increments. The behaviour of closures created in
javascript is as though such an indefinitely increasing numbering scheme is
being used under the hood.


[Hacker News]: http://news.ycombinator.com/item?id=2688438
[the stackoverflow answer]: http://stackoverflow.com/questions/111102/how-do-javascript-closures-work
