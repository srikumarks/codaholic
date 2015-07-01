---
layout: post
title: "Wash dishes. Don't collect garbage."
date: 2014-07-06 12:58
comments: true
categories: 
- Computer Science
---

In computer science literature, "garbage collection" refers to the process by
which unused computer memory is reclaimed for use by a program. Such memory is
usually referred to as "garbage" and the "garbage collector" periodically runs
to do this job. Though I understand the process to some extent, I've never been
happy with the metaphor since it doesn't help at all with suggesting possible
techniques for doing the task and is just used to label this part of a
programming system with automatic memory management. In this post, I explore
"dish washing" as a metaphor for the same process and argue why it is a better
one to adopt for teaching purposes.

<!-- more -->

## What's the fuss about metaphors?

If you grok the concept, why bother coming up with different metaphors to explain
it? Why not just teach the known algorithms and leave innovation to the imagination
of the student/programmer? 

Computer science and programming are abstract activities. Much of the history
of development tools is about making these abstract activities accessible to
our senses, including recently popular talks and writing by [Bret
Victor][worrydream]. I think metaphors contribute to the same by bringing the
abstract activities into the fold of the physically familiar and hence
accessible to mental simulation. Even "garbage collection" triggers the
imagination of finding a garbage can full, taking it out and dumping it so that
the "collector" who pops by once in a while can take it away to be dumped into
a landfill or recycled as the case may be.

## What's wrong with "garbage collection" as a metaphor?

Wait a minute. Do I still have to explain this after I wrote about the image
that garbage collection invokes in the previous paragraph? If you do know about
GC in computer science, does that imagery bear any resemblance to the process
that scans memory for previously used memory that is no longer usable, and
makes it available to future code that needs it?

If you think that's a clear enough analogy for you, please stop here and move
on. ... But wait, before you actually move on, tell me a few possible
techniques for doing that which you can come up with by working with the
metaphor *alone*. Feel free to post in the comments section. I'm genuinely
interested to know.

## Washing dishes and memory management

I make something to eat in my kitchen. I use vessels to cook food and vessels
to put the cooked food in. At some point, I run out of free vessels. I can't
use the cooking vessels because they're not clean, and I can't use some of the
storage vessels either for the same reason. Some of the storage vessels still
contain food and I cannot use them. 

I need to wash me some dishes.

I run a program to do some useful work. It uses memory to compute information
I consume in some form. At some point, it runs out of free memory. It can't
use the memory it already touched because it doesn't know off hand whether
the information in them will be used or not. Some of the memory does contain
information that will be used.

My program needs to find it some memory.

## Cooking up metaphors

Let's imagine some scenarios that call for kitchen work.

1. When cooking dinner for two in a large well stocked kitchen, you can afford
   to just keep using vessels as you need without ever needing to wash
   anything.  A program that doesn't need much memory, running on a
   system with gobs of RAM, can afford to not deallocate memory at all.

2. When cooking for a party with a small kitchen, you'll need to wash dishes
   frequently so that you have vessels available to do the cooking. If you
   don't have enough vessels to serve the food, you don't have a party. A program
   that needs a lot of memory to run, when run on a system with constrained
   memory, will need to reclaim memory frequently to get the job done, and if
   there isn't enough memory to hold the program's output, the job crashes.

3. When kitchen size roughly matches the cooking task, different strategies may
   work depending on the kitchen structure. If it takes time to get new
   vessels, it may be faster to wash dishes frequently. If washing takes time,
   you may wish to schedule naps at long intervals. A program may likewise find
   it faster to reclaim unused memory frequently if allocation takes time, and
   would need to allow for fewer long pauses to minimize total time spent
   reclaiming memory.

4. An economical chef can maximize his kitchen's availability if vessels are
   cleaned up as soon as he's done with using them, at the cost of working
   slightly slower due to repeated cleaning of the same vessels. This strategy
   would work for small kitchens. A program may likewise reclaim memory as
   soon as it knows when such memory becomes unused.  It would need some way of
   tracking when memory becomes unused in order to do this, which would
   correspond to reference counting strategies, for ex. This strategy would
   work for embedded devices with constrained resources.

5. A chef may categorize vessels into "need this for most work" and "stores
   stuff for a long time". He'd make sure the first category gets washed
   frequently while not bothering to wash the second category as often. A
   program may likewise find that some kinds of allocations store stuff for a
   short term only and some other kinds store stuff for longer periods. It can
   then choose to focus on reclaiming the memory with high turn over with less
   frequent attempts to reclaim memory that's known to store information for
   a longer time. 

6. Well, you're the Chef. You can hire a hand to cleanup your vessels for you,
   so that the vessels that you need are available to you without delay. Of
   course, you're both working in the same space and would need to coordinate a
   bit so you don't bump into each other too often. A program can dedicate a
   processor to continuously reclaim memory so the main processing task isn't
   starved. Of course both programs are running on the same computer and they'd
   need to coordinate memory accesses and memory reclamation for consistency.
   
Heard of "generational garbage collection"? Now explain that with the original
metaphor for me please.

## Wrapping up

I believe I've shown dish washing to be a better way to explain various
strategies for what's traditionally called "garbage collection". Here's 
a rough legend for this analogy -

1. Kitchen = Computer
2. Chef = Processor
3. Cooking = Processing
4. Vessels = Memory
5. Food = Information

If the next time you're hanging around in your kitchen and come up with
some interesting way to manage your cooking, think what would happen if you
applied that to memory management in a computer system and have an aha
moment ...

... implement it, benchmark it against known approaches, write a paper about
it, publish some open source code that shows the technique ... and be sure to
write me a thank you note here for making that connection for you :)

Happy cooking!

[worrydream]: http://worrydream.com

<a href="https://www.flickr.com/photos/mumbaiphotographer/2173777325/"><img src="/images/deepakitchen.jpg" alt="Photo by Deepatheawesome"/></a>

