---
layout: post
title: "A critique of Tuna"
description: ""
sharing: true
footer: true
categories: javascript
---

Google has open sourced the [Tuna] set of effects used in their [Jam with Chrome]
project. Here, I collect some thoughts about the code design decisions for their
effects framework, since I myself have written [Steller].

<!-- more -->

## Parameters

In Tuna, the effects module's parameters are presented to the API user as
object properties and implemented using getters and setters. This is obviously
simple from the perspective of a module user. However, it has several disadvantages -

If a module contains another module and you want to expose that parameter,
you're forced to write another getter/setter to handle the indirection. Such
parameters cannot be passed around "by reference". The only thing you can
do with them is to set/get their values. "By reference" passing is a simple
way to expose an internal module's parameters to the module user which is an
important kind of composability.

Owing to the getter/setter approach, meta information about parameters such as 
their range, defaults, automation capabilities, etc. need to be stored elsewhere 
in a "defaults" object, instead of being directly associated with them.

In some of the effects, some internal values and other parameters need to be 
recomputed when a couple of parameters change. In this case, the code for
updating the dependent parameters is duplicated since it is not particularly 
convenient to write a shared function local when defining parameters using
`Object.create` like this -

``` js
    //...
    baseFrequency: {
        enumerable: true,
        get: function () {return this._baseFrequency},
        set: function (value) {
            this._baseFrequency = 50 * Math.pow(10, value * 2);
            this._excursionFrequency = Math.min(this.sampleRate / 2, this.baseFrequency * Math.pow(2, this._excursionOctaves));
            this.filterBp.frequency.value = this._baseFrequency + this._excursionFrequency * this._sweep;
            this.filterPeaking.frequency.value = this._baseFrequency + this._excursionFrequency * this._sweep;
        }
    }, 
    excursionOctaves: {
        enumerable: true,
        get: function () {return this._excursionOctaves},
        set: function (value) {
            this._excursionOctaves = value;
            this._excursionFrequency = Math.min(this.sampleRate / 2, this.baseFrequency * Math.pow(2, this._excursionOctaves));
            this.filterBp.frequency.value = this._baseFrequency + this._excursionFrequency * this._sweep;
            this.filterPeaking.frequency.value = this._baseFrequency + this._excursionFrequency * this._sweep;
        }
    }, 
    //...
```
     
A simpler way to solve this would be to have parameters be first class objects
that can be inspected, instead of just accessing and setting their values. Meta
data about them can travel along with them. In [Steller], watcher functions can
be installed on parameters run code when the parameter's value changes.

## Inheritance using the `prototype` mechanism

One of the code design choices that vastly complicates the code in this case
is the choice of using `prototype` based inheritance to define the properties
of instantiated effects modules. While this is common in the JS world,
in this case it deprives the programmer of a local context in which to hide
internal objects from the effects module's user and to write shared private
code like in the case above. Any potential loss of efficiency in not using
the prototype mechanism for this application is insignificant since object
creation is not the thing we should be optimising for.

The "define all methods and properties within the constructor function"
approach is much more suitable in this situation.

[Tuna]: http://github.com/Dinahmoe/tuna
[Jam with Chrome]: http://www.google.com/?q=jam+with+chrome
[Steller]: http://github.com/srikumarks/steller
