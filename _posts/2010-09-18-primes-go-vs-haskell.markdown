---
layout: post
title: "Primes - Go vs. Haskell"
date: 2010-09-18
comments: true
categories: 
- Programming
---

Recently, a friend of mine pointed out a 
[concurrent implementation of Eratosthenes' prime sieve] 
expressed in google’s Go programming language. I
couldn’t resist trying a Haskell version of that same approach. 

<!-- more -->

What motivates me is that I find that Haskell code is generally more elegant
than what you can manage with most other languages. For a taste of that, here
is the simplest expression of a sieve in Haskell, the elegance of which I think
most other languages will have a tough time beating –

{% gist srikumarks/588433 %}

It says this in effect – “if you divide the infinite list of primes into two
parts, the latter part doesn’t contain any numbers divisible by the numbers in
the former.” … or rather it says that for one prime entry at a time.

So what does the concurrent code look like?

{% gist srikumarks/588430 %}

As expected, this is overall much more elegant looking than the equivalent Go
implementation. Much of the elegance is due to the use of lazy evaluation –
sending a lazily generated list of numbers through a channel using
writeList2Chan.

Now, this concurrent code runs like a … no it crawls like a … no I really can’t
think of a simile that expresses how slow this code is! It generates a handful
of primes every second on a 2GHz dual core mac! … while the straight forward
primes blazes through by comparison, despite its poor asymptotic behaviour.

Writing performant Haskell is a black art and this case amply demonstrates
that. With absolutely no clue about why this is such a drag, I started
transforming the code to see whether anything improved the performance.

Rather quickly though, I found that rewriting writeList2Chan to run more slowly
gave a huge boost. Here is what I replaced the writeList2Chan function with –

{% gist srikumarks/588439 %}

The original code above took about 23 seconds to print out up to 227! With this
change, it takes 9 seconds to print out up to 29573! The most crucial
ingredient of this change is … the yield !! Without it, the performance drags,
with it, at least you see thousands of primes fly by every second. Considering
that we’re chaining one sieve “process” for each prime found, I’d say … not
bad.

What’s more, you don’t need to replace all the mentions of writeList2Chan. You
only need to replace

    (writeList2Chan ch [2..])

with

    (slowlyWriteList2Chan ch [2..])

to get that kind of performance boost.  Black art indeed!

[concurrent implementation of Eratosthenes' prime sieve]: http://golang.org/doc/go_tutorial.html#tmp_353
[Go implementation]: http://golang.org/doc/go_tutorial.html#tmp_353
[despite its poor aasymptotic behaviour]: http://sites.google.com/site/haskell/notes/thegenuinesieveoferatosthenes
[performant Haskell]: http://www.haskell.org/haskellwiki/Performance
