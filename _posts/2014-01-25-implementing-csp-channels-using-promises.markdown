---
layout: post
title: "Implementing CSP channels using Promises"
date: 2014-01-25 16:18
comments: true
categories: 
- Javascript
---

[Promises] are the new old thing in the land of Javascript async abstractions,
though they aren't as good as CSP-style channels for async programming. In this
post, I describe a channel implementation based on promises that attempts to
bring some CSP-style programming abilities into the JS world ... now, instead
of in ES6.

<!-- more -->

Synchronous function and method calls return values immediately. If the
return value is not available immediately, one can restore sequential
programming style to some extent by returning a "promise" for the value that will
be "fulfilled" at some point in the future. [Promises] enable programming
with such "values available in the future".

However, as [Rich Hickey argues], promises invert the kind of control you want
when programming, and the CSP ("Communicating Sequential Processes") style of
programming by reading values from channels, doing some processing and writing
results into channels offers a better approach to system building in terms of
parts.

While Clojure's [core.async] and [go] use the CSP model, with [Haskell] leading the
pack in elegance (imho), Javascript is sort of trailing behind with (again, imho)
inferior abstractions such as generators planned in ES6.

As I looked into this, I found that a simple implementation for CSP-style
channels was possible in Javascript if we think of these channels as producing
*promises* for values at the reading end, and taking in values or promises
at the writing end. This leads to the same kind of code modularity enabled
by the approach taken in [core.async].

Here is a quick hello-world to illustrate the API. (I use [CoffeeScript] below
since it is briefer than JS.)

``` coffeescript
ch = new Channel()
ch.take().then console.log
setTimeout (-> ch.put('Hello world!')), 1000
```

`ch.take()` produces a promise for a value, that will get fulfilled when
`ch.put()` is finally called 1 second later with one.

So far, this is rather unimpressive relative to a pure promise-based approach -

``` coffeescript
p = new Promise()
p.then console.log
setTimeout (-> p.resolve('Hello world!')), 1000
```

Suppose I want to implement a "process" that reads strings from a channel
`a`, splits them into words according to a regular expression `r` and sends
the word array into a channel `b`, promises begin to look alien to this
problem. It is, however, rather easy in terms of such a promise-producing
channel.

``` coffeescript
splitter = (a, r, b) ->
    a.take().then (str) ->
        b.put str.split(r)
        splitter a, r, b
    return b
```

Now, calling `splitter(a, r, b)` would setup such a process running and we
can pump strings into `a` and get split words from `b`.

Suppose again that the regular expression to use for the splitting is
(for toy reasons), coming from another *channel* `r` instead of being
fixed, then the code becomes a bit more complicated -

``` coffeescript
splitter = (a, r, b) ->
    a.take().then (str) ->
        r.take().then (re) ->
            b.put str.split(re)
            splitter a, r, b
    return b
```

The pattern here is that we want to invoke some method on an object that's
been promised, but isn't yet available. If we encode this operation as a
method on a `Promise` object, the above code can be simplified as would
many such cases be. For this, we add a `send` method to `Promise.prototype`
that would forward a method call to the value when the value becomes
available, but returning a promise for the result of the method invocation
immediately. This would permit regular sequential style programming
with promises.

``` coffeescript 
Promise.prototype.send = (methodName, dynArgs...) ->
    Promise.all(dynArgs).then (args) =>
        @then (obj) -> obj[methodName] args...
```

With the new `send`, our `splitter` becomes -

``` coffeescript
splitter = (a, r, b) ->
    a.take().send('split', r.take()).then (words) ->
        b.put words
        splitter(a, r, b)
```

Notice that `a.take()` and `r.take()` are both called before `send()`.
`Promise.all` makes sure that a value is available from `r` before
proceeding to the method invocation part. Also, the positioning
of the recursive call to `splitter` is such that the process will
work on the next string before a consumer reading channel `b` takes
the previous result. 

Also note that if the method invocation happens to return a promise,
then the resolved value of the promise is what will be passed
into the final function. This is handy when working with async
APIs.

Below is the code for a full implementation of `Channel`, assuming
a `Promise` implementation as provided by [Bluebird].

``` coffeescript
Promise = require 'bluebird'

Channel = ->
    queue = []
    pending = []

    # Produces a promise for a value that will be placed into 
    # the channel at some point in the future.
    @take = ->
        if queue.length is 0
            p = Promise.defer()
            pending.push p
        else
            p = queue.shift()
            p.resolve p.$value
        return p.promise

    # Puts the given value (which can be a promise) into the channel, 
    # that will get passed to takers. The result value is itself a 
    # promise that will be fulfilled with the given dynValue at the 
    # point a takers gets it.
    @put = (dynValue) ->
        if pending.length is 0
            p = Promise.defer()
            p.$value = dynValue
            queue.push p
        else
            p = pending.shift()
            p.resolve dynValue
        return p.promise

    # Empty the channel by rejecting all the put and take operations.
    @drain = (err) ->
        while queue.length > 0
            queue.pop().reject err
        while pending.length > 0
            pending.pop().reject err
        return this

    return this
 
# Forward a method invocation on the value promised
# by a Promise instance to the value itself, resolving
# all argument promises along the way.
Promise.prototype.send = (methodName, dynArgs...) ->
    Promise.all(dynArgs).then (args) =>
        @then (obj) -> obj[methodName] args...
 
if typeof module isnt "undefined"
    module.exports = Channel
```

The above `channel.put` implementation itself returns a promise.
This promise is fulfilled with the value passed to `put` once
a consumer reads the value from the channel. Therefore if in the
above `splitter` example, if you'd wanted to make the recursive call
wait until the value placed on channel `b` is read by the consumer
before processing the next request from channel `a`, you can
place the recursive call in a `then` on the promise returned by
`b.put()`.

Thoughts?

[Bluebird]: https://github.com/petkaantonov/bluebird
[Promises]: http://promises-aplus.github.io/promises-spec/
[Rich Hickey argues]: http://www.infoq.com/presentations/clojure-core-async
[core.async]: https://github.com/clojure/core.async
[go]: http://golang.org/
[Haskell]: http://www.haskell.org
[CoffeeScript]: http://coffeescript.org/
