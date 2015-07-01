---
layout: post
title: "Make LLVM bitcode a ubiquitous web standard"
date: 2014-07-07 12:59
comments: true
published: false
categories: 
- Tech
---

Looking at evolution of the architecture of web browsers, the next logical step
appears to be to make LLVM bitcode ubiquitous and a standard feature of all 
web browsers. This would be A Good Thing for the web, methinks.

<!-- more -->

## Programming language plurality everywhere

Server side dev folks have always enjoyed choice of language/system when it
comes to building. You can simply choose the thing that works best for you,
that you're more comfortable with, is most performant for your app, etc. On the
client side, we have ... javascript. While JS is pretty fast these days
everywhere and we can cross compile with emscripten and all, I'm almost tempted
to say that the true web standard for programmability on both client-side and
server side ought to be some kind of a byte-code .. a concept the Java guys had
right way back.

If you consider that we have PNaCl compiling llvm-bitcode to native within the
browser, Emscripten cross compiling llvm-bitcode to asm.js (which Firefox's
engine converts to native), Safari's engine compiling js itself to
llvm-bitcode, why not make llvm-bitcode an open web standard? That
way, any language that compiles to llvm-bitcode would be equally usable on the
server side and on the client side.

## Sharing science

There's been something of a movement recently towards getting researchers to
publish all their source code and data openly. While in principle this is a
great idea, I think all such published code will rot over time as systems
evolve. Much of this code may also get locked into proprietary systems,
reflecting the same problem with publications being locked behind paywalls.

If a high performance portable runtime environment (like PNaCl) and a flexible 
display system (like a web browser) turn out to be ubiquitous, it would make
possible to publish much of scientific work so it is way more accessible to all
compared to proprietary approaches of today.

## Impact on education

The biggest potential downside to such a bitcode approach would probably be
educational. I've learnt a lot by looking at source code and the web was always
open in that way. With minified JS these days that's not so possible any more.
However, such a bitcode can make it easier to distribute source code in any
language, since a module can then compile and run such code right in the
browser.

The web browser is about the only program that, for me, brings back the
timeless joy of turning on a ZX Spectrum and instantly see a blinking cursor
telling me it is ready to obey my commands.

