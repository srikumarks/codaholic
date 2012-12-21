---
layout: post
title: "J-expressions"
description: ""
---

[JSON] has become a kind of de-facto standard for sharing data among services
on the web. The Lisp folks have enjoyed this luxury ever since ... well ever
since McCarthy made the language and his student implemented an interpreter for
it. What's more, they have also had the luxury of using the same syntax for
sharing *logic* .. and in fact take it for granted. This post is a proposal to
bring that "luxury" to the web programming world.

**Status:** Draft. Comments welcome. ({% include hnsubmit %})

## Why is it useful to be able to share logic? ##

If you've already bought into Lisp, you may not need much convincing here.  If
you do ask that question, I've written about [DSLs and J-expressions] in an
earlier post that may be of interest to you.

## Core syntax ##

1. A "J-expression" is an ordinary JSON-serializeable **object**, whose "first
   key" is taken to name an "operator", with the whole object serving as its
   "operand". 

2. A J-expression's key **must** be an identifier conforming to the following
   regular expression 

        (Letter|$|_)(Letter|DecimalDigit|$|_)*

3. We permit an extension to JSON whereby the key **need not** be a *quoted*
   string.  Nevertheless, a key **must** conform to the above regular expression
   even when quoted.

4. Everything else -- numbers, strings, dates and arrays -- is just as
   specified in [JSON].

5. We leave the question of how the "operator" must use the operands open in
   order to allow for eager and lazy computations, and meta-expressions.

6. Every J-expression **may** have a `comment` key whose value is for human
   consumption and is therefore expected to have nil impact on the meaning of
   the expression to a program.  It **may** therefore be ignored by parsers
   and interpreters. J-expression writers, however, **must** strip out these
   comment fields before serializing these expressions. This means `comment`
   is a "reserved key".

7. Every J-expression **may** have a `meta` key whose value is a [JSON] value
   and is considered of some relevance to the system processing the script.
   This could include additional documentation references, version information,
   source URL for a specification of the operator, etc.  ..  the structure of
   which is left unspecified. J-expression writers, therefore, **must not**
   strip out the `meta` entry. This means `meta` is a "reserved key".

 
#### An example ####

``` js
    {do: [
        {fetchURL: "http://some.data.source/",
          timeout: 30.0,
           onfail: {raiseError: "URL not reachable"}},
        {fork: [
            {displayInElement: "element_id"},
            {cacheInLocalStorage: "prefix"}
            ]}]}
```

The above script can be read as representing a "pipe" where the 
data is fetched from some URL and then (concurrently) displayed 
in an element and cached in local storage.

### Some considerations ###

1. J-expression parsers are expected to be support both quoted and
   unquoted keys in order to enable "logic" and "data" to take the
   same form.

2. J-expression writers are also allowed to use either form, but writing
   unquoted keys is preferred since it is more compact than quoted keys and
   closer in form to the notion of "programming" or "scripting".

#### The "first key" as the "operator" ####

Javascript implementations today preserve the order in which keys are inserted
into objects when traversing keys as long as the keys are not array indices.
Even when the specification does not require them to do so, Google considers it
a de-facto standard that insertion and enumeration order must match whenever
only string keys are involved - as noted in this [V8 bug]. This de-facto
standard permits easy identification of the "first key" in a JSON expression as
shown below --
    
``` js
    function jexpr_operator(jexpr) {
        for (var first_key in jexpr) {
            return first_key;
        }
    }
```

In languages such as [Python], [Ruby] and [Lua], we cannot assume that the
first key can be so easily identified, but since all these languages have
custom syntax checking parsers for scanning JSON, it is a trivial extension to
the parser to add a special key to an object to help identify the first key.

> The key `__jexpr_operator__` is reserved for parsers so that they can
> indicate what the "first key" is by giving its name as the value of this key.
> For the rest of the system, the presence of this key overrides any key order
> already obeyed by the system. J-expression writers must use this key to
> determine which key to write first, but **must not** output this key itself.

Parsers in languages with objects that do not preserve identity between key
insertion and enumeration order can add a small bit of logic to insert the
above special key.

``` js
    function jexpr_operator(jexpr) {
        if ("__jexpr_operator__" in jexpr) {
            return jexpr.__jexpr_operator__;
        } else {
            for (var first_key in jexpr) {
                return first_key;
            }
        }
    }
```

Beyond this, we do not need the order of the other keys to be preserved and
operator implementations should not make that assumption.

## Can we build programming languages using j-expressions? ##

It sure looks possible to do so. For example, here is a (toy) compiler for
[J a Scheme-like language] that is about as expressive as Scheme .. except for the 
lack of support for tail call elimination.

[J a Scheme-like language]: https://github.com/srikumarks/jexpr
[V8 bug]: http://code.google.com/p/v8/issues/detail?id=164
[DSLs and J-expressions]: /gyan/2012/04/14/creating-dsls-in-javascript-using-j-expressions
[JSON]: http://www.json.org
[Python]: http://www.python.org
[Ruby]: http://ruby-lang.org
[Lua]: http://www.lua.org
