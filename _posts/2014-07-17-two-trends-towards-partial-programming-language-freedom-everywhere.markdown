---
layout: post
title: "Two trends towards partial programming language freedom everywhere"
date: 2014-07-17 10:36
comments: true
categories: 
- Programming
---

Two common backends seem to be emerging that enable programmers to choose a
language and system suitable for their work irrespective of whether they're
developing server side code or client side code. Programmers who develop
services that sit behind communication protocols such as HTTP have always
enjoyed the freedom to choose the programming language and system that best
supports what they need to develop, because clients who use system requirements
placed by these languages do not get passed on to clients who make use of these
services. Client-side programmers have, however, had limited options when it
comes to programming language choice for various reasons, the most significant
of which is perhaps accessibility to APIs for doing various things on the
client device. Hence if you program for iOS, pick Objective-C.  Android? pick
Java. Windows? Perhaps C# .. or F# if you're adventurous. Web browser? You got
Javascript. For once, two common performant backends - Java byte code and LLVM
bit code - are emerging as the common ground enabling portability and hence
programming language diversity.

<!-- more -->

## Languages that target the JVM

Way back, Java took a leap by specifying a virtual machine and a portable byte
code format for developing portable code. While it can be endlessly debated
whether its motto "write once, run anywhere" actually came true or ended up
being "write once, debug everywhere", the underlying VM and byte code format
has become the compilation target of some modern functional programming
languages such as [Clojure][] and [Scala][]. While these languages leverage the
existing Java eco-system, they've nevertheless had to fight the impedance
mismatch between what their ideal runtime environment would require and what
they get with the JVM and Java byte code. To some extent the JVM/byte code
developers have responded by adding features such as dynamic invocation to the
system to better support late binding languages .. which Java itself isn't.

While Java was intended to be embedded into web browsers, it ended up with the
reputation of causing several sites to slow down because sites would have to
explicitly load up a limited JVM instance for running within a web page, with
the security limitations that come with it. I think it is safe to say that this
approach has fallen out of favour, especially with the stellar performance of
Javascript engines that are embedded in modern browsers. Since there is nothing
fundamentally different between embedded Javascript and embedded Java, what might've
killed Java in this context, I guess, is that it tried to be a complete syste,
instead of leveraging the capabilities of the browser and integrating with it.
Empirically, [you need 6x the amount of actual RAM needed to run your code in 
a garbage collected environment][6x] and trying to do everything will certainly
bloat this up.

Thus, the JVM and Java byte code provide a common robust platform and eco system
on which to build services, but not so much on the client side, despite experiments
such as Java Web Start.

## Javascript - the "assembly language of the web"

Ever since Google upped the ante with its performant V8 Javascript engine, 
just about every modern browser has kept pace with V8, or even outpaced it in
some ways, when it comes to performance. The performance and ubiquity of 
JS has led to it being called the [assembly language of the web][jasm].

In particular, [Emscripten][] has made it possible to port just about any
code in most system programming languages such as C and C++ to Javascript
for running within a browser. One interesting offshoot of the work on
[Emscripten][] is the [asm.js][] specification - a subset of Javascript 
which can be efficiently compiled into machine code with performance comparable
to native code. As of this writing (July 2014), only Firefox supports [asm.js][].


## Google salt

For a couple of years now, Google has had an initiative to have a secure
sandbox within its Chrome browser for running native x86 machine code, targeted
at client side applications that place high computational loads such as
audio/video/image processing. This is [NaCl][] - short for "Native Client".
While [NaCl][] being an x86 code based environment might suggest that any
language that compiles to x86 assembly can be used within [NaCl][], its tool
chain currently limits it to C/C++, though language support is growing.

## LLVM and what it's got to do with all this

[LLVM][] - short for "Low Level Virtual Machine" - is billed as a compiler
infrastructure project and provides a toolkit for working with a generic low
level assembly language and bit code format with pluggable optimization passes.
Apple funds as well as contributes to this open source project. LLVM saw
interesting applications in MacOSX such as optimizing the performance of OpenGL
driver calls on the fly.

The idea behind LLVM bitcode and assembly language has its parallels with 
Java byte code and the Java language, though its specification deals with
computation at a much lower level and does not, for example, mandate a
specific model of "objects" like Java byte code does.

Post the initial work on the compiler toolkit and optimization modules,
[LLVM][] has gained front ends for many languages including [C, C++ and
Objective-C++][clang] and is pretty much the default choice for new
experimental languages such as Mozilla's [Rust][] and Apple's [Swift][]. Chris
Latner, who started and heads the LLVM project, is the one behind Apple's
[Swift][]. [Haskell][], a language that's much older than the LLVM project,
has also grown [an LLVM backend][ghcllvm] for its flagship GHC compiler.

### Emscripten redux

I'd noted earlier that Emscripten can compile a variety of languages to a
subset of Javascript close to asm.js. The way it does this is to provide a
translator from LLVM bit code to Javascript! That way, code written in any
language that compiles to LLVM bitcode can be made available within a browser,
within limitations. So, really, Javascript is not the one being treated as
assembly language here, but there is a more generic target. If you're
developing an experimental system programming language, it now makes sense to
target LLVM bit code because not only can you leverage the optimization modules
within LLVM, but you can also run within a browser environment, thanks to
Emscripten.

### NaCl redux

Google's [NaCl][] x86 based sandbox now has a portable sibling called PNaCl -
or "Portable NaCl". PNaCl makes it possible for a browser application developer
to ship architecture independent code to the browser that can run at near
native speeds. 

How does PNaCl achieve its portability and performance? The "architecture
independent bitcode" is LLVM bitcode and the PNaCl system in Chrome compiles
this bitcode package to high performance native code ahead of execution time.
This way, you can distribute such code to x86 as well as ARM systems. While you
can ship code to the x86 NaCl only via the Chrome Web Store, PNaCl is available
today and enabled on latest desktop Chrome browsers on all platforms and you
can target it without going via the Chrome Web Store. This strategy is similar
to Java's byte code system, except that the compilation to native code is done
ahead of time instead of just in time.

### Safari

Apple recently [bumped up the performance][safari_llvm] of its Nitro Javascript
engine by converting JS into LLVM's "intermediate representation" (a.k.a. LLVM
assembly language) and then subjecting it to heavy optimization. This again
puts LLVM behind one of the most commonly available programming languages on
today's machines.

## About time for some unification on the client side

We've seen how LLVM is creeping under the hood of many client-side systems
today. LLVM bitcode isn't perfect for all languages - for ex, tail call
elimination is an optimization that can't be done on the LLVM IR,
disadvantaging Scheme implementations a bit, though tail recursion to loop
conversion can be done. Despite that LLVM has become the backend of choice for
high performance code across a wide variety of systems and is now shipping in
Safari, Chrome and (indirectly) Firefox.

Now, why did I put "indirectly" within parens when I mentioned Firefox?  As I
mentioned above, Emscripten takes LLVM bit code and converts it into a subset
of Javascript, which Firefox recognizes (as asm.js) and optimizes back into
native code. The compilation of asm.js code seems to therefore be a great
candidate to translate the JS back into LLVM IR for maximum execution speed.

## Conclusion - LLVM and the JVM

All this brings us to the concluding point - that today we have high
performance client-side and server-side code being backed by two systems - LLVM
and the JVM - with corresponding architecture independent code formats - LLVM
bitcode and Java byte code. With recent systems programming languages such as
[Rust][] and [Go][golang] capable of serving the construction of reliable
high performance services, I anticipate that LLVM-based services will grow
to gain a share comparable to what Java has achieved in the services world.

This bodes very well for accelerating the creation of tools optimized more for
programming productivity, safety, reliability, concurrency and distribution in
the near (and far) future since such systems need only target LLVM to gain
ubiquity. Furthermore, it may also become possible for these systems to share
libraries in a way that has not been possible before, except perhaps on the JVM
platform. 

Overall, being a polyglot who is always on the lookout for better tools to
think with, I'm pretty excited for the diverse programming ecosystem that's
emerging today.


[jasm]: http://www.hanselman.com/blog/JavaScriptisAssemblyLanguagefortheWebPart2MadnessorjustInsanity.aspx
[Emscripten]: https://github.com/kripken/emscripten
[Clojure]: http://clojure.org
[Scala]: http://www.scala-lang.org/
[asm.js]: http://asmjs.org
[NaCl]: https://developer.chrome.com/native-client
[LLVM]: http://llvm.org
[clang]: http://clang.llvm.org/
[Swift]: https://developer.apple.com/swift/
[safari_llvm]: http://www.infoq.com/news/2014/05/safari-webkit-javascript-llvm
[golang]: http://golang.org/
[Rust]: http://www.rust-lang.org
[Haskell]: http://haskell.org
[ghcllvm]: https://www.haskell.org/ghc/docs/7.6.3/html/users_guide/code-generators.html
[6x]: http://sealedabstract.com/rants/why-mobile-web-apps-are-slow/
