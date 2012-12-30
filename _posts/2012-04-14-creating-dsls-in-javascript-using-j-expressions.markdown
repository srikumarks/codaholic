---
layout: post
title: "Creating DSLs in Javascript using J-expressions"
description: ""
sharing: true
footer: true
categories: Javascript
---

Scheme and Lisp have for long had powerful meta-programming abilities due to
the syntax of their language being the same as the syntax for the main data
structure supported by the language - the humble list. These languages are
therefore well suited for inventing smaller special purpose "domain specific
languages.

Javascript, on the other hand, has a "full blown syntax" that makes
meta-programming not for the faint of heart. One consequence of the lack of
such ability is that developers have not had the benefit of the abstraction
possible through small special purpose DSLs.

Here, I outline an approach for creating DSLs in Javascript using the now
prevalent [JSON] format that is native to the language. The initial part tries
to explain the kinds of scenarios in which one might consider building a DSL,
which is important to have an idea about. Later, I get into the actual
representation using JSON.

<!-- more -->

## When to make a DSL? .. or "Hunting of the Closure". ##

Under normal circumstances, you should consider developing your abstractions as
"modules". Modules (or "libraries") offer basic encapsulation and hiding
facilities that JS developers are familiar with. You can help them benefit from
your work by leveraging that familiarity. 

On the rare occasion though, you might find yourself writing code that starts
looking like a small "compiler"  or "runtime". In such circumstances, creating
a DSL for that part of the system might be worthwhile to consider -- the purpose,
again, being that it might ease the job of those that your package targets.

A tipping point can happen when you discover that you are working with a
category of objects that are "closed over" a suite of operations.  In
mathematics, a set of objects is said to be "closed" with respect to a function
if the function maps objects of the set to other objects in the same set and we
say the set "has the closure property". But before we get to why that's
significant, let's take a quick look at a simple example.
  

### The "closure property" - an example ###

Say we want a framework to create and render visuals that vary in time -- like,
maybe, umm ... a video? Here is a simple representation of such a visual, which
we'll call a "show"  -

``` js
function show(t) {
    // calculate...
    return image;
}
```

Given such a "show", it is trivial to write a renderer for it -

``` js
function render(show, duration_ms) {
    var startTime_ms = Date.now();
    requestAnimationFrame(function () {
        var t = Date.now() - startTime_ms;
        context.draw(show(t));
        if (t < duration_ms) {
            requestAnimationFrame(arguments.callee);
        }
    });
}
```

Now, I want an operator which will delay a given show by a specified amount
of time. Simple -

``` js
function delay(show, dt_ms) {
    return function (t) {
        return show(t - dt_ms);
    };
}
```

How about a "cut" from one show to another at a given time. Also simple -

``` js
function cut(when, show1, show2) {
    return function (t) {
        return (t < when ? show1 : show2)(t);
    };
}
```

Now, if we represent video clips using such `show`s, those two functions under
which the set of `show`s is closed give us all we need to make an edited
production .... because we can combine them.  Note that `delay` takes a show
and produces a show, and so does `cut` ... which takes *two* shows, yes, but
that's the same idea. This kind of combinatorial power is valuable when
modelling a domain.
  

> ##### ShowTime #####
> The interface to [muvee Reveal]'s rendering engine "ShowTime" is built
> around this kind of an abstraction. The details of [ShowTime] and the DSL for
> [authoring muvee's "styles"] built on top of this API are available publicly.
> In this case, the DSL abstracts away implementation details such as the deep
> nesting of functions that might result in a naive implementation.

[muvee Reveal]: http://www.muvee.com
[authoring muvee's "styles"]: http://muvee-style-authoring.googlecode.com
[ShowTime]: http://muvee-style-authoring.googlecode.com/svn/doc/main/ShowTime_-_the_muvee_timeline.html

## Functions are closed under composition ##

Here is what makes the closure property worth hunting for --

> When you have several functions under which your set is closed, it is also
> closed under *arbitrary compositions* of these functions.

So, well, why bother with a DSL? Why not just use function composition?  One
reason is that, in the real world, your set is closed with respect to each of
these functions *only under certain constraints*. These constraints might make
it difficult to predict whether a given composition of these functions is a
valid operation in your domain. You will need to validate a composition
*before* you begin to act on it, which becomes difficult if you only have
access to the composition and not the parts. A DSL that abstracts over these
constraints by exposing only that subset of compositions that are valid in the
domain would therefore be of great help to people reusing your work.
  

> The DSL's purpose, then, is to hide the constraints under which closure is
> possible and composition operations are valid - i.e. where composition and
> use are context sensitive.
  

## A DSL for asynchronous operations ##

Let's consider an example where having a DSL seems beneficial - composing
asynchronous operations. 

It is a common convention for a function that does some asynchronous action to
accept "callback" and "errback" functions to call when it succeeds or fails
respectively.  Composing such actions into sequences soon turns into what is
usually referred to as "callback hell" and the person reading the source soon
looses sight of the whole flow of control.

So, do we satisfy our "DSL criteria" here?

1. Closure property? CHECK. A sequence of operations is itself an operation. 
2. Constraints? CHECK. A sequence is allowed to proceed normally only when an
   operation doesn't fail. There are many error handling strategies when an
   operation fails and we may not have seen all of them yet.

Let's then make a small DSL for sequencing asynchronous operations.
  

## Representing asynchronous operations ##

First we're going to need a way to define new asynchonous operations
in plain Javascript for our "primitives". A simple representation is
to model an async operation (let's call them "asop"s for convenience)
as a function that takes a specific set of arguments as follows --

``` js
function asop(M, input, success, failure) {
    // ...
    // we're good.
    M.start(success, input, M.end, M.failure);

    // Something bad happened
    M.start(failure, M.error("bad", input, success, failure), M.end, M.end);
}
```

The function takes four parameters -

1. `M` is a module that provides facilities for starting an asop --
   `M.start` and a terminal operation `M.end` that does nothing and stops all
   processing.
2. `input` is the data being passed to the asop to do its job.
3. `success` is an asop to start once this asop finishes
   successfully.
4. `failure` is an asop to start if this asop cannot be completed for
   some reason.
  

> Note that an asop is recursively defined in terms of and makes use of two
> other asops `success` and `failure`, which themselves have the *same*
> function signature.


We take `M.error` to be a utility that produces an error object that captures
the current error state to pass to the failure operation. Here is a simple
definition for `M` --

``` js
M.start = function (op, input, succ, fail) {
    setTimeout(function () {
        op(M, input, succ, fail);
    }, 0);
};
M.error = function (descr, input, succ, fail) {
    return new Error({descr: descr, input: input, success: succ, failure: fail});
},
M.end = function () {};
```
  

## Sequencing two operations ##

Since our operations are ordinary functions, we can write a "sequencing operator"
for combining two operations as a higher order function as follows --

``` js
function seq(first, next) {
    return function (M, input, success, failure) {
        M.start(first, input, seq(next, success), failure);
    };
}
```
  

*Note*: This looks like infinite recursion, but it will terminate automatically
when `M.end` is passed for `next` because, we define `M.end` to be the operation
that doesn't continue further.
  

## Making a chain of operations ##

The humble `seq` operator can now be put to use to chain an arbitrary
number of operations given as an array ---

``` js
function chain(operations) {
    var revOps = operations.slice(0).reverse();
    return function (M, input, success, failure) {
        M.start(revOps.reduce(seq, success), input, M.end, failure);
    };
}
```
  

## Umm ... why do we need a DSL again? ##

You'd be right in asking that question. So far, it looks like the language is
expressive enough to let us code up all these compositions using plain higher
order functions. So, let me throw something into the mix that might cause you
to stumble if you think such functions alone are adequate.

Consider the exchanges that happen between a web server and a client (browser).
A client instructs a web server to perform some operation, the server gets back
when done, or with progress notifications, then the client does a few async things
like store some data locally, and then makes the next request and so on. The ability
to represent a *sequence* of actions directly is a boon to take care of all these 
exchanges happening between these two dance partners.
  

> ### So here is your task - ### 
> 
> Abstract away the transaction to survive a termination of connection with the
> server. I'm not talking packet loss here.  I'm talking about the user
> initiating a series of operations and in the middle of things shutting down
> the browser. What I want to happen is for the operations to continue exactly
> from wherever they left off -- because I cannot "throw an exception" and undo
> a rocket launch. If the client was waiting for a response from the server, it
> should again start waiting for that response; and by implication if the
> server was doing something and found out that the connection had terminated
> at the "next" step, it should've stored away whatever it needed to do to
> continue, and must resume that when the user connects again, maybe after some
> mild handshaking. In particular, it is *not* acceptable to terminate the
> operation sequence just because the browser was closed.
  

Trying to do something like this is where reifying processes as functions
breaks down. To a lisp/scheme programmer, a solution would already be in sight
-- use s-expressions to represent operations and store away instructions in
that form. What might we do in the Javascript world?
  

## J expressions ##

The image I have in my mind now is Douglas Crockford standing with legs apart,
arms folded, cape flying with a speech bubble pointing to his head saying
"JSON to the rescue!".

Since we have our framework taking care of coordinating the success and failure
of operations, all it takes to *specify* an operation, given a known set of
named "primitive" operations, are the following --

1. The *name* of the operation to start.
2. An *input* object to customize any details of the operation.

So, there you go, we can change our representation of an operation from being
a function to something that looks like -

``` js
{opname: input_object}
```

(Technically, that'll be `{"opname": input_object}` in JSON, but let's ignore
that difference since it is *serializable* using JSON, which is what's
important.)

The input object is also a literal or computed *value* which we restrict to
JSON-serializable values. The named primitive operation can itself be provided
in a module as a function that takes the whole spec object and does what it is
supposed to do.

We can now specify a series of operations as an array in this form -

``` js
{do: [
    {ask: "Enter file name: ", 
        type: "file"},
    {fetchFile: {showProgress: "progress_bar"},
        reportInterval: 1.0},
    {spawn: {withRetry: {uploadToDropbox: {user: "cat", password: "meow"}},
                maxTries: 5,
                  onfail: {do: [
                            {deleteTempFiles: null},
                            {ask: "Dropbox upload failed. Try again?", 
                                type: "yes/no",
                                 yes: {retry: true},
                                  no: {raise: "Give up"}}
                            ]}}},
    {cacheInLocalStore: "prefix"},
    {showInElement: "element_id"}
]}
```

If we are given the primitive operations as members of a module `M`, we can run
a primitive operation as follows -

``` js
function runPrimitiveOperation(opspec) {
    return function (M, input, success, failure) {
        for (var opname in opspec) {
            M[opname](M, opspec, input, success, failure);
            return;
        }
    };
}
```
  

> #### Note ####
> 
> Here I'm making use of the fact that JS engines today such as V8 always
> enumerate the keys of an object in the order in which they were added to it.
> This is what lets us identify the *first* key as determining the operation to
> start with the operation implementation knowing what to make of the other
> keys in the expression, the parameters and so on.
> 
> Also note that we've put in a pipeline here -- i.e. `ask` gets a file name
> that it passes as input to `fetchFile` which passes the fetched file data to
> `spawn`, and so on. `spawn` is a higher order operation, since it takes an
> operation as the argument to perform concurrently, while continuing by
> passing its input through to `cacheInLocalStore`.

Our framework, then, serves as an interpreter for this mini language, providing
the necessary abstractions on *how* we expect these operations to be executed
behind the scenes. What is important is that *users* of our framework are not
forced to think about browser shutdown/restart cases and can simply assume
that the infrastructure we provide will "do the right thing".

Now, I'm not going to go into details of what one actually needs to do to
*implement* the shutdown/restart behaviour, but at this point, it ought to be
clear enough that given a serializable representation of "operations still left
to do" and a serializable representation of the input to give to this sequence,
we can dump both at any point to disk (plus some book keeping info) and resume
after a browser/server restart. We'll need to do that for *each task sequence*
that's active at the point the shutdown is initiated.

The important thing is that the one writing the script can focus on writing the
script instead of the operational details.

## Recap ##

The main takeaways from this post are the following -

1.  If you find categories of objects in your problem domain that are closed
    over a number of functions and you want to abstract away the constraints
    under which those functions can be combined, you may find it useful to
    create a DSL for your domain.

2.  The consistent implementation of key enumeration order of objects in
    Javascript engines allows us to write **specifications of composeable
    computations** using generic JSON-serializable notation that we can call
    "J-expressions".

        {opname: main_parameter,
            keyword1: value1,
            keyword2: value2,
            ...}

3.  J-expressions, in combination with functions that implement what the
    operation named "opname" is supposed to do, are then a systematic way to
    approach DSL construction within Javascript.  The notation is as expressive
    as s-expressions. For instance, here is a way to write a "higher order
    asop" :)
       
        {lambda_the_ultimate: ["arg"],
            do: [
                {step1: ..},
                {step2: ..}
            ]}

[JSON]: http://www.json.org
  
