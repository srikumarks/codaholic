---
layout: post
title: "Hello Javascript Promises!"
date: 2014-10-14 23:48
comments: true
published: false
categories: 
- Javascript
- Promises
- cspjs
---

Wierd that I'd write a "hello" post on promises after I wrote [Bye Bye
Javascript Promises!][bbjsp]. This post has a two fold agenda. One is to point
out that I'm not against promises per se, and the other is to introduce
something new in the JS world - well, the same old new - data flow variables,
which was what promises used to be called.

[cspjs][] has an experimental branch [dfvars][] branch that introduces a
syntactically simpler way to work with promises. [cspjs][] could interop with
callbacks, but ignored APIs being written using promises. Now it can also
interop with promises in the [dfvars][] branch and I'd like some feedback
from the JS community on this.

[cspjs]: https://github.com/srikumarks/cspjs
[dfvars]: https://github.com/srikumarks/cspjs/tree/dfvars

<!-- more -->

## What are data flow variables - dfvars?

[Mozart/Oz][] takes a different stance to what happens when an unbound variable
is encountered when evaluating an expression. When most languages will raise an
error or assume some default value silently, Oz blocks the current thread until
the variable gets bound somwhere else. This is in essence a "promise", where the
variable's value is promised to be set elsewhere.

## Declaring dfvars in cspjs

In [cspjs][], `var` declarations must have initializers. This makes it
convenient to assume that a `var` declaration with no initializers for the
variable is declaring such a dfvar. Here is a silly example that waits 
forever -

```
task {
    var x;
    console.log('going to wait ... ', x);
    await x;
    console.log('x is now bound to ', x); // This won't happen.
}
```

## Binding a dfvar 

The `:=` assignment operator binds a dfvar to a ground value. Here is an even
sillier example -

```
task {
    var msg;
    var launch = task { msg := "Launched Mangalyaan!"; };
    launch(); // Spawn off the task.
    await msg;
    console.log(msg);
}
```

We use the "uninitialized var" declaration to create a dfvar and the `:=`
operator to bind a dfvar to a ground value. The RHS of a `:=` bind statement
can evaluate to a normal value or turn out to be a "thenable", in which case
its outcome will be bound to the outcome of the dfvar.

## dfvars within data structures

You can store dfvars within arrays and objects. No special declaration is necessary
in this case. You only need to use `x[y] := expr;` kind of statements to perform the
bind, and cspjs will take care of the test. Keeping to the tradition of silly examples -

```
task {
    var x = {};
    var t = task { x['planet'] := 'earth'; }
    t();
    await x;
    console.log('Hello planet', x['planet']);
}
```

## A more involved example

When I started this section, I started with the [doxbee sequential][doxseq]
example. Soon, I realized that the regular `await` based code is more succinct
and clearer than the promise or dfvar based code, especially given that we have
to await on dfvars explicitly. [Doxbee parallel][doxpar] is more elegant
though.

The key is that if you're immediately going to wait on a promise returned from
an operation before doing anything else, `await` based code is way more
elegant. If you're going to make many dfvars before waiting on a subset of
them, then dfvar based code can be elegant. In either case, dfvar-based code
looks more elegant than promise based code in JS.


[Mozart/Oz]: http://mozart.github.io/
[doxseq]: https://github.com/petkaantonov/bluebird/blob/master/benchmark/doxbee-sequential/promises-bluebird.js

