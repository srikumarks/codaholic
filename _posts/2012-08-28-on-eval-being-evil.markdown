---
layout: post
title: "On eval and evil."
description: ""
---

"eval is evil" has become a maxim repeated in the Javascript community.
Douglas Crockford, in [Javascript: The Good Parts], rightly advises against
hidden and explicit uses of eval for security and clarity reasons. Now, I find
`eval` useful to implement [DSLs in Javascript]. The in-browser [CoffeeScript]
compiler wouldn't be possible without `eval` (directly or indirectly). So, in
this post, I wish to explore what appears interesting about `eval` that is
relevant to building such DSLs.

For this post, I'll stick to the behaviour of `eval` in the **Chrome** browser
(i.e. the V8 engine, which also applies to [Node.js]). We'll go through a
number of contexts and examine how `eval` behaves in each of those. You can
copy paste the code shown here to Chrome's JS console and run them.

## What is `eval`?

A simplistic description is that you pass a Javascript string to `eval` and it
will "evaluate" it as Javascript code, whatever that means. The [ECMA-262]
specification (edition 5.1) has the following to say on `eval` -

> **10.4.2 Entering Eval Code**
> 
> The following steps are performed when control enters the execution context for
> eval code:
> 
> 1. If there is no calling context or if the eval code is not being evaluated by
>    a direct call (15.1.2.1.1) to the eval function then,
> 
>     a. Initialise the execution context as if it was a global execution context
>     using the eval code as C as described in 10.4.1.1.
> 
> 2. Else,
> 
>     a. Set the `ThisBinding` to the same value as the `ThisBinding` of the calling
>     execution context.
> 
>     b. Set the LexicalEnvironment to the same value as the `LexicalEnvironment`
>     of the calling execution context.
> 
>     c. Set the `VariableEnvironment` to the same value as the `VariableEnvironment`
>     of the calling execution context.
> 
> 3. If the eval code is strict code, then
>     
>     a. Let `strictVarEnv` be the result of calling `NewDeclarativeEnvironment`
>     passing the `LexicalEnvironment` as the argument.
>     
>     b. Set the `LexicalEnvironment` to `strictVarEnv`.
>     
>     c. Set the `VariableEnvironment` to `strictVarEnv`.
> 
> 4. Perform *Declaration Binding Instantiation* as described in 10.5 using the
>    eval code.
> 
> **10.4.2.1 Strict Mode Restrictions**
> 
> The eval code cannot instantiate variable or function bindings in the variable
> environment of the calling context that invoked the eval if either the code of
> the calling context or the eval code is strict code. Instead such bindings are
> instantiated in a new VariableEnvironment that is only accessible to the eval
> code.

## Introducing local variables

An expression of the form `eval("var x = 10;")` is capable of introducing a new
variable `x` in the context in which it is executed. However, as noted in the
ECMA specification, if the eval code is strict, then you cannot introduce a new
variable this way - i.e. `eval("var x = 10;")` will work, but 
`eval('"use strict"; var x = 10;')` will not work. No exception is thrown, but the
variable is simply not introduced into the enclosing environment, though it is
available to the rest of the evaled code.

Consider the following function -

``` js
function localVars(x, stmt) {
    eval(stmt);
    return x + y;
}
```

All of the following behave as one might expect -

1. `localVars(10, "var y = 5;")` returns `15`
2. `localVars(10, "var y = x + 5;")` returns `25`.
3. `localVars(10, "'use strict'; var y = 5;")` raises a `ReferenceError: y is not defined` exception.

## Capturing local variables in closures

Consider the following function -

``` js
function captureSecretValue(code) {
    var secret = 3.14159;
    return eval(code);
}
```

`captureSecretValue("secret")` returns `3.14159` as expected. You can also
create closures that capture the "secret" value -

``` js
var f = captureSecretValue("(function (x) { return secret * x; })");
console.log(f(1)); // prints "3.14159"
```

However, the following gives a `ReferenceError` -

``` js
function nest(code) {
    var nestedSecret = 2;
    return eval(code);
}

function captureNestedSecret(code, nest) {
    var secret = 3.14159;
    return nest(code);
}

console.log(captureNestedSecret("secret * nestedSecret", nest));
```

This illustrates that only the variables in the *lexical* context are available
to `eval` and not those in its *evaluation* context. The following will
therefore print `6.28318` as expected.

``` js
function captureNestedSecret(code) {
    var secret = 3.14159;
    function nest() {
        var nestedSecret = 2;
        return eval(code);
    }
    return nest();
}

console.log(captureNestedSecret("secret * nestedSecret"));
```

## Scope objects

If you have an object whose keys give variable names and whose values give their values,
you can use `eval` in conjunction with the `with` statement (beware: evil squared!) to evaluate
code in that scope. Here is what I mean -

``` js
function evalInScope(scope, code) {
    with (scope) {
        return eval(code);
    }
}

var scope = {a: 10, b: 20};
console.log(evalInScope(scope, "a + b")); // prints "30"
console.log(evalInScope(Math, "acos(0)")); // prints "1.5707963267948966"
```

What is more interesting is that you can capture the "variables" in a closure
that you create using eval as follows -

``` js
var f = evalInScope(scope, "(function (c) { return a + b * c; })");
console.log(f(0)); // prints "10"
console.log(f(2)); // prints "50"
```

Since it is not the *values* of the variables that are being captured, but
the *references*, you can now do -

``` js
scope.a = 100;
console.log(f(0)); // prints "100"
```

If you subsequently delete one of the variables in the `scope` object, you get
a `ReferenceError` as one might expect. The `scope` object therefore provides
access to the scope chain of the created closure. This interception is deep,
since you can introduce new variables into the scope by manipulating `scope`
as well.

``` js
var scope = {a: 10};
var f = evalInScope(scope, "(function (c) { return a + b * c; })");
console.log(f(0));  // ReferenceError
scope.b = 20;       // Introduce binding for "b".
console.log(f(0));  // prints "10"
console.log(f(2));  // prints "50"
```

### Named functions within `with` 

The following code doesn't work and throws a `ReferenceError` because
the `inner` closure is instantiated outside the `with` scope by the V8 
compiler, contrary to what it might look like. 

``` js
// (1) doesn't work
var scope = {message: "hello world"};
var greeting;
with (scope) {
    function inner() {
        greeting = message;
    }
    inner();
}
```

The following alternative works in V8 because the closure is *created* 
when executing a statement within the `with` clause.

``` js
// (2) works
var scope = {message: "hello world"};
var greeting;
with (scope) {
    var inner = function () {
        greeting = message;
    }
    inner();
}
```

This difference can be a WTF and points to the general recommendation of only
using the latter "name a function through assignment" approach. We know that
the definitions of named functions are lifted to the top of the surrounding
scope, but also know that they are lifted out of any surrounding `with` blocks
as well.

> **Update**: This inconsistency is a bug in V8 looks like. Firefox's VM 
> behaves consistently in both the cases above. I've submitted a [V8 bug report]
> for this problem.

[V8 bug report]: https://code.google.com/p/v8/issues/detail?id=2311


## Preventing access to global objects

In a browser environment, all global symbols are available as properties of the
`window` object. We can use this, in conjunction with the "scope object"
feature as discussed above, to evaluate code that is to be prevented from
touching any of these global objects or classes. This gives us a poor man's
sandbox.

``` js
function poorMansSandbox() {
    // arguments[0] is the code string to be evaluated in the sandbox.
    // "allowed" contains all the permitted symbols for the code. (This is
    // an incomplete list.)
    var allowed = { "eval":     true
                  , "Math" :    true
                  , "Object":   true
                  , "Array":    true
                  , "Function": true
                  , "String":   true
                  , "Date":     true
                  , "Number":   true
                  };

    // "inScope" contains all the visible definitions for the code.
    // Make sure the symbols "inScope" and "allowed" are themselves not visible.
    var inScope = { inScope: undefined
                  , allowed: undefined 
                  , poorMansSandbox: undefined
                  };

    // We need to run the eval in a separate function scope with
    // a properly bound "this" because otherwise "this" will be
    // the "window" object and all our efforts will have been in vain.
    // We make "this" the scope object itself (i.e. "inScope")
    return (function () {
        Object.getOwnPropertyNames(window).forEach(function (g) {
            if (!allowed[g]) {
                inScope[g] = undefined;
            }
        });
        with (inScope) {
            return eval(arguments[0]);
        }
    }).call(inScope, arguments[0]);
}
```

Though this prevents access to existing global properties, it doesn't prevent
access to properties that will be added to `window` *after* the `eval` happens.

## Conclusion

`eval` should be used with tons of caution. However, if you're interested in
making DSLs around Javascript, it helps to know its workings a bit deeper.
Remember - there is always something "good" in every "evil" ;)

[Javascript: The Good Parts]: http://shop.oreilly.com/product/9780596517748.do
[DSLs in Javascript]: /gyan/2012/04/14/creating-dsls-in-javascript-using-j-expressions/
[CoffeeScript]: http://coffeescript.org
[Node.js]: http://nodejs.org
[ECMA-262]: http://www.ecma-international.org/publications/standards/Ecma-262.htm

