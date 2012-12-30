---
layout: post
title: "Scratch pad for text with diacritics"
date: 2012-12-27 10:09
comments: true
sharing: true
footer: true
categories: javascript
---

Roman text with a few choice diacritics are a common need when writing about
Indian classical music. Creating unicode text with diacritics that can be
ported between applications is in general a pain. So, I made a small in-browser
app that serves as a [scratch pad for common diacritics].

### Usage

Straight forward. You just type your text in the provided box using normal
roman characters first - ex: "Sankarabharanam". Once you've typed that in,
you select each character to which you want to apply diacritical marks and
click on the mark you want. You can also press the number key corresponding
to the diacritical mark you want for the selected character. With this,
you can get "Śankarābharaṇam".

Your text is automatically saved to local storage, so you can visit the page
any time and retrieve your text. You'll *never* lose text you key in.

### Known issues

This works fine in Chrome and Safari on MacOSX, but not in Firefox. I haven't
yet figured out why it doesn't work in Firefox. If you find that it works in
Chrome/Safari on Windows/Linux, please let me know in the comments.

[scratch pad for common diacritics]: /demos/diacritics
