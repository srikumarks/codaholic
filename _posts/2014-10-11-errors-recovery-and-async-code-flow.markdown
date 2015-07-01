---
layout: post
title: "Errors, recovery and async code flow"
date: 2014-10-11 15:56
comments: true
categories: 
- Javascript
- cspjs
- Asynchronous
---

`try-catch-finally` style error management is common in many programming
languages. Though the underlying mechanism of propagating errors up a "call
stack" is alright from a development perspective, the common syntax ends up
invariably mangling the code flow. In [cspjs][], a macro library presenting an
easy-to-use syntax for working with async Javascript code, I attempted what
felt to me to be a better way to fit error handling and recovery code into the
statement-by-statement sequential flow of activity.

In [Bye Bye Javascript Promises][bbjsp], I intended to present this key reason
why I wrote [cspjs][] - error handling and recovery specification that respects
code flow - but I didn't do a good job of presenting it. I attempt that in this
post.

<!-- more -->

## Code flow

Async code flow in Javascript is conceptually similar to conventional
sequential code in C/C++/Java and such languages. There is a correspondence
between the textual ordering of "statements" in code and the time ordering of
the activities performed when those statements are executed.

<pre style="font-family:monospace">
statement 1;
statement 2;
...
statement n;
</pre>

Given code like above, `statement k` is executed only *after* `statement k-1`.
(We'll ignore conditional branches and looping for now.)

This lets us draw the following picture, where the `NOW` line steps through the
statements one at a time. 

<pre style="font-family:monospace">
statement 1;                       PAST
statement 2;
---------------------------------- NOW -------
statement 3;
...                                FUTURE
statement n;
</pre>

The rough mental model of such "statement execution" used by programmers also
has the same temporal order. We think about what "statement 1" should do, then
about what "statement 2" should do, and so on as we actually write code. We
plan ahead before writing such code (hopefully!), but right in the middle of
the act itself, at least for me, the time order of the activities dominates.

## The dangling catch

I'd argued in [my previous post][bbjsp] that it doesn't make sense to have most
code outside of a try block. If we assume that, then we can simplify the
error handler writing as follows -

<pre style="font-family:monospace">
statement 1;                         PAST
statement 2;                    protected actions
--------------------------------------------------
catch (e) {
    handler ..                      GUARD
}
--------------------------------------------------
statement 3;                        FUTURE
...                            unreliable actions
statement n;
</pre>

This composes well with multiple such guard handlers since the code flow
reflects the temporal order of the impact of the code.

<pre style="font-family:monospace">
statement 1;                         PAST
statement 2;                    protected actions
--------------------------------------------------
catch (e) {
    outer handler ..                GUARD
}
--------------------------------------------------
statement 3;                        FUTURE 1
statement 4;                   unreliable actions
statement 5;
--------------------------------------------------
catch (e) {
    inner handler ..                GUARD
}
--------------------------------------------------
statement 6;                        FUTURE 2
...                            unreliable actions
statement n;
</pre>

The job of a `catch` guard block is to improve the reliability of the actions
that temporally (and textually) follow it, from the perspective of those
actions that precede it. 

In [cspjs][], the scope of such a `catch` is the inner-most block scope -
including `if-then-else`, `switch`, `while` and `for`.

## Action replay

When dealing with unreliable code, one way to improve reliability is to retry
actions until they succeed, or until you've "tried enough". For example, if
these actions are failing due to an unreliable network connection, an
exponential backoff may be used to improve its reliability and recoverability.

Say the "handler" in the first example wishes to try the following actions
again. This is expressed in [cspjs][] as -


<pre style="font-family:monospace">
statement 1;                         PAST
statement 2;                    protected actions
--------------------------------------------------
catch (e) {
    handler ..                      GUARD
    if (some_condition) {
        retry;
    }
}
--------------------------------------------------
statement 3;                        FUTURE
...                            unreliable actions
statement n;                    may be retried
</pre>

An unconditional `retry;` statement would result in an infinite loop,
but other than that there isn't any restriction on where within a `catch`
block the `retry;` statement can be placed. The retry will return control
to `statement 3;` and onwards if ther retry condition is met. The purpose
of such a catch guard is to not let some significant work done by statements
1 and 2 go to waste as far as possible.

## Mopping up

In order for a `catch` block to be able to improve the reliability of the
activity that follows it, especially if it wants a `retry`, then the handler
block needs to assume that the state of the system is pretty much how it was
when `statement 3;` is about to be entered. If it cannot make that assumption,
then the code for such a handler would have to deal with way too many cases
before it can make a retry.

In order for this assumption to be valid, any resources grabbed by the previous
invocation of the sequence must be released before the handler takes effect.
This is done in [cspjs][] using "dangling finally" clauses, which are also
block scoped, like the `catch` clauses. These dangling finally clauses are
similar to `defer` in golang. 

<pre style="font-family:monospace">
statement 1;                         PAST
statement 2;                    protected actions
--------------------------------------------------
catch (e) {
    handler ..                      GUARD
}
--------------------------------------------------
statement 3;                        FUTURE
statement 4;                    unreliable actions
--------------------------------------------------
finally {
    release resource grabbed by (4)
}
--------------------------------------------------
statement 5;                        more
...                             unreliable actions
statement n;
</pre>

The "release resource .." statements will get to run once the scope in which
it is embedded closes - i.e. after `statement n;` completes either successfully,
or some error occurs between statements 5-n.

When an error occurs in `statement 6;`, say, it is easy to trace which finally
clauses execute before the `catch` handler is hit - just trace back the steps 
textually.

Though `catch` clauses can also be used to release resources, they will be
called only under error circumstances. Therefore when it comes to resource
release, `finally` is almost always the right choice.

## Finally some subtleties

If you treat such a `finally` clause itself as a statement, then it cannot
make any assumptions about how the statements following it (in time and in
code) modify the variables constituting the system state as these steps proceed.
Here is an example (using [cspjs][] syntax)  -

<pre style="font-family:monospace">
f <- fs.open("file1.txt");
finally { f.close(); }
await doSomething(f);
f <- fs.open("file2.txt");
finally { f.close(); }
await doSomethingElse(f);
</pre>

On cursory reading of the above code, one might suspect that the finally
clauses will close "file2.txt" twice, since the state variable `f` is being
reused to open the second file. That's clearly undesirable.

Therefore when the first `finally` block executes, it must make sure that the
`f` it refers to is the first file - i.e.  the value of the `f` at the time the
`finally` block is encountered, rather than the value "now", whenever "now"
happens to be. To ensure this, the state machine to which [cspjs][] compiles
async tasks saves the state variables and restores them before processing the
cleanup steps. The above code expands to something like this in principle -

<pre style="font-family:monospace">
f <- fs.open("file1.txt");
$s1 = f;
finally {
    f = $s1;
    f.close();
}
await doSomething(f);
f <- fs.open("file2.txt");
$s2 = f;
finally {
    f = $s2;
    f.close();
}
await doSomethingElse(f);
</pre>

The end result is that you don't have to worry about these details and it "does
the right thing".

## Finally, finally statements

The code in the previous section can also be written like this -

<pre style="font-family:monospace">
f <- fs.open("file1.txt");
finally f.close();
await doSomething(f);
f <- fs.open("file2.txt");
finally f.close();
await doSomethingElse(f);
</pre>

i.e. a statement form of `finally` is also supported, with slightly different
behaviour than the block form. The statement form evaluates all variables (including
arguments to the invocation), but postpones the actual call until later. If a 
resource release is a single statement, this statement form can be used. 

## Conclusion

I hope the above helped clarify the motivation behind changing the syntax of
`catch` and `finally` so that they respect the temporal nature of code flow
that we're used to in sequential programming.

## References

1. The statement form of `finally` looks and behaves somewhat like `defer` in
   [Go][golang] except for being block scoped, but the origin is older than
   that.  Though I've forgotten the original inspiration, it features in [a
   scheme dialect][muse] I built a few years before [Go][golang] was announced.

2. I borrowed the word `await` in [cspjs][] from C#'s `async` and `await`
   keywords.

[bbjsp]: /blog/2014/02/11/bye-bye-js-promises/
[cspjs]: https://github.com/srikumarks/cspjs
[golang]: http://golang.org
[muse]: https://code.google.com/p/muvee-symbolic-expressions/wiki/ExceptionHandling
