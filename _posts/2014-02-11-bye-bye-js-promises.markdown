---
layout: post
title: "Bye Bye Javascript Promises!"
date: 2014-02-11 11:31
comments: true
categories: 
- Javascript
---

In my [previous post], I explored how programming with [promises] can 
be made close to programming with values. After some more work on it,
and some learning from [bluebird], I came to conclude that my brain 
doesn't think well with promises. So I wrote [a macro] for Javascript that
expands "tasks" into async state machines that communicate using channels
(i.e. CSP). I want to talk about the specific options for error management
implemented in the `task` macro.

[a macro]: https://github.com/srikumarks/cspjs

## Intro

My problem: I have async JS code on the server side and the client
written using callbacks, getting quite out of hand and I want to 
strap on some gear to be able to run faster without getting blisters.

1. I want my async code to be resilient to exceptions that may be thrown from
   other libraries I may use ... without me having to think about them *all*
   the time.

2. I want the flow of async logic to be clear in the code. I do not want syntax
   like '.then' distracting me and meddling with context (i.e. `this`). I want
   to be able to see the *process*.

3. I want to control the flow of data in my code instead of having the flow of
   data control when some code gets run.

4. I want to be able to think about error management up front and not leave it
   to, literally, the end of functions I write.

5. I want good performance with all this.

The [bluebird] promises library is the best in the category and it solves
problems (1) and (5). Regarding (2), I find promises can't seem to make up
their minds about whether they want to stand for values or the processes that
produce them. While `then` and `catch` seem to indicate they are about
processes, promises pretend to be values that have been promised in the future.
If you think of them as values, you can't do that everywhere (ex: in the `?:`
operator, for loops, etc.)

## sweet.js

When I asked myself "what would I do if JS was *really* a scheme?". The
answer was clear - I'd just write a macro to produce a state machine 
given async code and get on with life. After all, [ClojureScript] supports
[core.async], which is pretty much does this.

I present to you [cspjs], a compiler that transforms async "tasks" to
state machines that can be run in ES5 as well - i.e. in any browser.

That was easy! .. thanks to Mozilla's [sweet.js].

[sweet.js] is a neat implementation of hygeinic macros for Javascript.  So all
I did was to treat `;` as a sequencing operator and write a macro (called
`task`) that (in effect) reprogrammed the semicolon to produce an async state
machine instead of sync code.

Thank you sweet folks!

## cspjs

[cspjs] takes the "Communicating Sequential Processes" view to organizing
concurrent activities. For better or worse, my brain thinks well with 
this model and not so well with promises. 

The core interface is a `task` macro that expands to an asynchronous
state machine that can work with and call NodeJS style callbacks
of the form `function (err, result) { ... }`. A supporting class
called `Channel` provides communication between these state machines.

The body of the `task` macro is more or less normal Javascript, except
for some special constructs that clarify steps that call out to other
party async APIs, and to work with channels.

All that is straightforward. What I want to talk about in this post
is a change to the error management mechanism that `task` introduces.
Before that, below is the example [madeup-parallel] ported from the [bluebird]
benchmark suite, just to show what `task` code looks like.

``` js
task upload(stream, idOrPath, tag) {
    var queries = new Array(global.parallelQueries),
        tx = db.begin();

    catch (e) {
        tx.rollback();
    }

    var ch = new Channel();

    for (var i = 0, len = queries.length; i < len; ++i) {
        FileVersion.insert({index: i}).execWithin(tx, ch.receive(i));
    }

    await ch.takeN(queries.length);

    tx.commit();
}
```


## The traditional try-catch-finally model

I've never been a fan of the traditional `try {} catch (e) {} finally {}`
construct in various languages for a variety of reasons. 

1. I don't understand why there should be any code at all that should be
   outside of a `try` block. If it is outside, it means you're being
   sloppy/optimistic about some error conditions.

2. The catch occurs at the end of the try block, which means I tend to postpone
   thinking about error conditions until I reach the start of the catch block.
   It doesn't fit my code flow.

3. The code executed within `finally` blocks look like they have different
   scope than the code in the `try` block. The relationship between a finally
   block and the do-ing that it undoes is not usually clear, so it is not
   obvious what all operations need cleanup. C++ takes the sane route here
   by using destructor unwinding for cleanup instead of `finally` blocks.

4. With try-catch, I often find code like this -

``` js
try {
    ...some error prone code ...
} catch (e) {
    // Do nothing.
}
```

This is not "handling" an error condition. It is just sweeping it under the
carpet. "Handling" should mean "returning with a valid value".  Otherwise the
error must propagate up. If you don't know how to handle an error at some point
in the code, don't. Just let it bubble up.

## The error model in the `task` macro

The `task` macro supports `catch` and `finally` clauses, but no `try`, due
to reason (1) in the previous section. The scope of error handling is 
that of the task.

### throw 

Raising an error condition is done using `throw expr;` as usual. However,
within the context of a `task`, this translates to an error callback, as
does an error actually thrown by other library code. 

There is one difference between the way `throw` is interpreted by the `task`
macro and how it is usually interpreted in JS. In JS, anything can be thrown,
including `null`. However, the NodeJS style callback scheme uses `err === null`
tests to determine whether an error occurred, so throwing a `null` is 
semantically nonsensical. So `task`'s interpretation of `throw` is that
error propagation will begin only when the expression to be thrown is not
`null`.

### catch

A `catch (e) { ... }` clause traps any error that occurs in the code that
follows .. upto the end of the task's scope. One enhancement is that the
`catch` clause can take the form `catch (ErrorClass e) { ... }` in which case
only errors that match the class will be handled by the block, with everything
else bubbling up.

If you don't have a `return` in the body of a `catch` clause in a task,
the it will effectively rethrow the same error to the catch clause
further up the chain.

### finally

A `finally { ... }` block can be placed anywhere and is expected to specify the
code that is to be executed when control unwinds up the stack. The undo-ing is
expected to be about whatever happened "above" the `finally` block and never
about what happened "below" it.  Therefore there can be multiple `finally`
blocks and they will be run in the revers order in which they were encountered.

A `finally someFunc(args...);` statement is also supported. This form is a bit
different from the `finally` block. The value of `someFunc` and the `args` are
all computed when encountering this block during normal operation, but the
function call is scheduled to take place upon unwinding. Note that `someFunc`
can also be `someObj.method` and the context is kept correctly.  This is a
useful property in situations like this -

``` js
f <- fs.openFile('blah1');
finally f.close();
await doSomethingWith(f);
f <- fs.openFile('blah2');
finally f.close();
await doSomethingWith(f);
```

While unwinding, the second `finally` statement will close the "blah2"
file and the first one will close the "blah1" file.

Note that `finally` actions can occur even within loops and will be unwound
correctly with the correct state. The values of state variables that the
finally blocks will see will be the values they had at the time the finally
block was encountered, before an error happened that triggered the unwinding.
This is because these actions cannot possibly know what to do with the
values of the state variables at the time of the error condition.

WARNING: While technically you can `return` from within a `finally` block, you
really shouldn't. Also cleanup actions must not themselves throw errors.

## Channels and errors

[cspjs]'s channels are implemented such that they themselves will not raise any
error conditions, and will pass only non-error values to any callbacks passed
to them. Since channels are the glue between tasks, it is better to pass the
error to these tasks as values and let the logic in the tasks decide what to do
with them instead of assuming that if some error was passed on a channel, the
receiving task must bomb.

In particular, a channel's `.receive(id)` method gives a NodeJS style callback
that can be passed to a third party function that requires such a callback. The
result will then be placed into the channel and can be read within a `task`. If
any error occurred, the error will be passed as the `.err` property of the
object put into the channel. If the operation succeeded, the `.val` property
will give the result value. So a task is expected to look at the received
value and decide how to handle errors.

The behaviour of `throw` described in the previous section helps ease the task
of manual propagation of errors passed through channels. You can simply
`throw result.err;` and place the non-error code below. This is because
`throw` will only begin error propagation if an actual error value has been
given.

``` js
var ch = new Channel();
someTask(args ... , ch.receive(42));
r <- chan ch;
console.assert(r.id === 42);
throw r.err; // Will throw only if there is an error.
doSomething(r.val);
```

## Conclusion

For async code, the `task` macro in [cspjs] provides a more robust error
handling mechanism than ES5 javascript without garbling the syntax very much.
It does this in addition to providing a clean syntax for describing processes
and the channel based coordination between them. The result is code that can
used on the server side or on the browser side.

Resuming and retrying operations when a failure occurs is not directly
supported at the moment (as of 11 Feb 2014). This would require a different
compilation approach compared to the current switch-case state machine
implementation. However, since client code does not rely directly on the
`state_machine` API and sees only `Channel` and the `task` macro itself, if
this feature is desired, it would be possible at some point to change the
implementation to support it while maintaining backward compatibility of code.

In all, [sweet.js] is a pretty sweet deal indeed and permits a programmer to
bend the language while keeping compatibility with runtimes everywhere.

For my own JS code, for the foreseeable future, it will be [cspjs] and I will
stop debating which async management library to use, all thanks to [sweet.js]. 

To others out there selling promises ... well, good luck to you!

[promises]: http://promises-aplus.github.io/promises-spec/
[previous post]: /blog/2014/01/25/implementing-csp-channels-using-promises/
[bluebird]: https://github.com/petkaantonov/bluebird
[sweet.js]: http://sweetjs.org
[cspjs]: https://github.com/srikumarks/cspjs
[ClojureScript]: https://github.com/clojure/clojurescript
[core.async]: https://github.com/clojure/core.async
[madeup-parallel]: https://github.com/petkaantonov/bluebird/tree/master/benchmark/madeup-parallel
