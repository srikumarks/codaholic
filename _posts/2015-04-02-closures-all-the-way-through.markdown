---
layout: post
title: "Closures all the way through"
date: 2015-04-02 08:58
comments: true
categories: 
- Javascript
- Programming
---

In [A Mental Model for Variables and Closures in Javascript][mm], I described
an allocation based model that I'd used with some success to teach and
understand the intricacies of closures, as implemented in Javascript engines.
While we have plenty of tutorials on closures that describe what results are
gotten under various conditions, with variables being magically "captured",
I've not seen many work through a mechanistic explanation of closures.  To that
end, [here is the slide deck][catwt] of a session I recently conducted which
walked the participants through a mechanistic understanding of closures and
objects, using the Chrome debugger. Read on to find out how to follow the deck
and the rationale behind it.

<!-- more -->

How to follow the slide deck
----------------------------

**Preparation**: Fire up your favourite text editor, a tab on the Chrome
browser, an open the debugger console by pressing "Ctrl-Shift-J" or
"Option-Shift-J" depending on the breed of your computer.

You're expected to type the code presented in the slide deck into the
text editor, copy-paste it into the debugger and run it. While I encourage
you to literally type in code that you see for the first time, you may
make the presented variations of the code by copy-paste (ex: adding
"debugger;" statements).

On closures, dogs, frisbees and Sir Isaac Newton
------------------------------------------------

A colleague of mine noted that it is amazing how people pick up the notion
of functions in JS and do crazy things with it, all without an awareness of
what exactly they're doing. They write functions that take functions as
arguments and return other functions without batting an eyelid. If one
studies the code, it would seem that some kind of a deep understanding
of the the concept of closures is at work, but this illusion is shattered
the moment one steps into the code and starts probing why the code works
the way it does. 

Here is an analogy - actually one of my favourite philosophical questions that
I enjoy leaving unanswered. 

Consider a dog that's adept at catching a frisbee thrown at it. This is an
amazing feat if you look at all that needs to be done to achieve it. The dog
has to track a flying coloured disc, anticipate where it is going based on its
ingrained understanding of gravity, basic physics and aerodynamics, figure out
how to coordinate its muscles with the right signals so that its mouth will
make contact with the colured disc in such a way that the disc gets trapped
between its teeth. All out of just experience. This is just wow!

Now consider Sir Isaac Newton, who thought through to the core of how all these
things happen in the world and formulated the now famous "laws of motion" and
"law of gravity". Brilliant work and none would dispute that.

Who "understands" physics better - the frisbee catching dog, or Sir Newton?

I leave that question unanswered always because I can argue equally in favour
of either side. For our purpose, I wish to highlight the advantages of each
kind of understanding.

While the dog performs a superbly fluent function that Sir Newton cannot even
dream of accomplishing to comparable fluency, the dog that doesn't ponder the
why and the mechanics of how it achieves what it does will never be able to
make the connect between gravity on earth and the gravity between the stars and
planets.

"ummm .. and what has this got to do with closures in Javascript" you ask?

Despite years and years of programming experience with closures and JS,
specifically by using libraries such as JQuery and AngularJS, without that
extra effort into inquiring how they work, we cannot possibly make the
transition from a "javascript programmer" to the level of the folks who create
these libraries using what looks like closure and object wizardry. This inquiry
involes seeing to the core of functional programming - how to model any given
domain using composition of functions - and what universal principles underly
them.

A mechanistic understanding of closures is a start and is comparable to
grokking Newton's laws of motion and gravity to predict what happens under
various circumstances. A higher level of understanding will involve deriving
beautiful overarching principles of physics such as conservation laws, the
principle of least action and such. To have a glimpse of that, dive into the
world of Haskell.

[mm]: /blog/2012/04/13/a-mental-model-for-variables-and-closures-in-javascript/
[catwt]: /talks/back2front-closures-27mar2015.pdf

