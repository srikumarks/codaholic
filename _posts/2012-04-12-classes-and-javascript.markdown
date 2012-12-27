---
layout: post
title: "Classes and Javascript"
description: ""
sharing: true
footer: true
categories: javascript
---

If you have been programming in Java and C++ for a while and are used to
thinking about problems in terms of classes and inheritance, you may find
yourself struggling with Javascript since it has only objects.  This post is to
-- a) provide you with a perspective on why programming with only objects is
powerful and b) show you how to translate the class-based concepts you're used
to thinking in into the Javascript world.

The Javascript object system comes from the language [Self] which pioneered
object oriented modeling using prototypes, along with many dynamic compilation
techniques that later went into the HotSpot JVM. The motivation for destroying
the class-object dichotomy is that working with classes forces you to think
about object categories well before you've really understood the problem
domain.  The claim by the Self folks is that your approximate and often
downright invalid classes get frozen in over time and the system becomes harder
to adapt to changes in understanding about the problem being modelled.

So, what *are* classes? ... really?

<!-- more -->

## Classes as machines ##

When you write a class in a language like Java or C++, you're in essence
writing a recipe for making objects with certain properties and behaviours. The
compiler then translates this recipe into a machine that can make such objects
whenever you need them at runtime. If you ignore the compilation step, you can
think of a "class" itself as the machine that produces objects, without loss of
accuracy.  It is this operational view of classes that offers the key to
reusing class-based modeling techniques in an objects-only language like
Javascript.

So, a class is a machine --- in goes some parameters available only at runtime
and out comes an object with the properties and behaviour as dictated by the
recipe. That thinking can be directly modelled as a function that takes some
parameters and produces an object with the right properties and behaviour.

``` js
var obj = MyClass(params...);
```

In what follows, I'll omit the parameters for simplicity of presentation and
you can always add them back later. So we have -

``` js
var obj = MyClass();
```

This gives a "first order" view of classes. Though a useful way of thinking
in itself, you soon hit a modelling wall the moment you want another class
that is only slightly different from one you already wrote. You cringe at 
having to copy-paste code to produce make objects of this slightly different
variety. What you want is some way to code up only the incremental difference
from `MyClass`.

## Combining classes ##

One way out is to write a function that can combine the properties and behaviour
of two given objects -- call it `blend`. So that will let you write -

``` js
var objVariant = blend(MyClass(), NewBehaviour());
```

.. where `NewBehaviour` is a function that models only what is different
from `MyClass`.

The above approach, though valid, is limited by the inability of `NewBehaviour`
to have any kind of say over the changes to be introduced to the original object.
It would be ideal if `NewBehaviour` could first take a look at the object produced
by `MyClass` before it makes any changes to it. Therefore, `NewBehaviour` is better
written as a function that takes in the object produced by `MyClass` and returns
a new object.

``` js
var objVariant = NewBehaviour(MyClass());
```

`NewBehaviour` can now be made arbitarily flexible. In retrospect, we could've
also written `MyClass` with an extra object parameter, so that the combination
of these two classes itself looks like an "object transformation machine".

``` js
var objVariant = NewBehaviour(MyClass(obj));
```

In other words, we model class inheritance in Javascript using function composition.
We can now write this notion of inheritance as a "higher order function" in Javascript
as -

``` js
function Inherit(Parent, Delta) {
    return function (obj) {
        return Delta(Parent(obj));
    };
}
```

## Multiple inheritance and "mixins" ##

The traditional notion of multiple inheritance froma number of classes `C1`, `C2`, and
so on in a specific order is then merely repeated application of `Inherit`. If we have
an array of classes `ClassArr` and we want to make a class that "inherits" from all of them,
all we need to do is --

``` js
function NoChange(obj) { return obj; }
var MultiObj = ClassArr.reduce(Inherit, NoChange);
```

.. which we can abstract again as a function that takes an array of classes
and produces a "multiply inherited class".

``` js
function MultiInherit(ClassArr) {
    return ClassArr.reduce(Inherit, NoChange);
}
```
            
#### Modify or copy? ####

Notice that we haven't placed any constraints on these functions so far
with respect to whether they destructively modify the object passed in,
or return another object with the necessary properties. Both these 
are acceptable and you may want to weigh which approach is suitable for
you. For instance, creating copies costs memory, but you trade off a one
time performance hit for repeated searches up an "inheritance tree".

## Javascript's "prototype" mechanism ##

Let us take a simple example of inheritance to make further digging easier to
follow. Say we have an `Image` class that can produce objects that know how
to `draw` themselves into a `context` in portrait mode.

``` js
function Image(obj, ...) {
    // ..

    obj.draw = function (context) {
        // (pseudo code)
        context.putPixels(obj.pixelData, 0, 0, obj.width, obj.height);
    };

    return obj;
}
```

That was easy enough, but say we now want a variant of `Image` which produces
objects that draw themselves in landscape mode. i.e., we want -
  
```
var landscapeImage = Landscape(Image(obj));
landscapeImage.draw(context);
```

How would we write the `Landscape` class?

``` js
function Landscape(obj) {
    function lsDraw(context) {
        context.save();
        context.rotate(90);
        obj.draw(context);
        context.restore();
    }

    obj.draw = lsDraw;
    return obj;
}
```

Simple enough? ... Oops! We've just introduced  an infinte draw loop because
the landscape object's draw keeps calling itself!

To solve this, we need to first store away the object's original
`draw` function and use *that* one within `lsDraw`.

``` js
function Landscape(obj) {
    var portraitDraw = obj.draw;

    function lsDraw(context) {
        context.save();
        context.rotate(90);
        portraitDraw.call(obj, context);
        context.restore();
    }

    obj.draw = lsDraw;
    return obj;
}
```

This is then the approach to use to model calling the "parent method" within a
"child class". 

If you're modifying a dozen different methods, it can get pretty tiring
to save every method away in a variable before calling it within a modified
method. This, then, motivates the "prototype chain" mechanism in Javascript.

## The "prototype chain" ##

Javascript provides a way for an object to delegate property and method
access to another object if they aren't part of itself. Though we can
set up arbitrary numbers of such delegation chains ourselves, the builtin
mechanism serves to ease single inheritance cases. Our `Landscape` class
can now be written as -

``` js
function Landscape(obj) {
    var extendedObj = Object.create(obj);

    function lsDraw(context) {
        context.save();
        context.rotate(90);
        obj.draw.call(extendedObj, context);
        context.save();
    }

    extendedObj.draw = lsDraw;
    return extendedObj;
}
```

`Object.create` makes a new object that looks and behaves *exactly* like `obj`,
but now when you add a new property to `extendedObj`, the original `obj`
remains unmofied. Now, we don't need to save away old method definitions
because the old object will always have them and so we can refer to the old
`draw` function as just `obj.draw`.

So, what's up with the weird `obj.draw.call(extendedObj, context)` and why
can't we just write `obj.draw(context)` instead?

To answer that, we need to look at what `obj.draw(..)` means to Javascript.
It means "draw `obj` making use of *its* properties". If the `draw` function
needs to know the `width` of the object to do the drawing, it will access
it will get `obj.width` within the draw function. If we later on modify the
width of the `extendedObj`, that modified value will not be accessible when
running `obj.draw()` because `obj` does not know anything about the existence
of `extendedObj`.

The solution then, is to tell the `obj.draw` function to run, but ask it to
make use of the properties of `extendedObj` instead. Now, since `extendedObj`
otherwise has all the properties of `obj`, there is no information loss to the
`obj.draw` function. The way we tell the `obj.draw` function to do that is to
use the `call` method of the `Function` object.

``` js
obj.draw.call(extendedObj);
```

The above code, btw, is equivalent to the following -

``` js
var oldDraw = obj.draw;
oldDraw.call(extendedObj, context);
```

## Adaptive methods ##

Let's take a second look at the implementation of `Image` and the `draw`
function in particular --

``` js
// ...
var obj = {};

obj.draw = function (context) {
    //...
    context.putPixels(obj.pixels, 0, 0, obj.width, obj.height);
    //...
};
```

If the draw function was originally implemented like above, then it would
be of no use to try to change the context using `Function.call` like in
`old.draw.call(extendedObj)` because the body of `old.draw` always explicitly
refers to the properties of `obj` itself. 

> Oops! It looks like the original `Image` implementation cannot be extended,
> although it would work fine on its own!

What we want in this case, is for the `draw` function to be more generic and
pull properties from an arbitrary object. That is done using the dynamically
scoped `this` argument, which, when used within the body of a function, refers
to the object that the caller passed as the first argument to the `call`
function.

Here is a modified `draw` -

``` js
obj.draw = function (context) {
    //...
    context.putPixels(this.pixels, 0, 0, this.width, this.height);
    //...
};
```

When this draw function is called with `obj.draw.call(extendedObj)`,
the given `extendedObj` will substitute `this` within the funciton body,
and we get the correct behaviour. When you simply call `obj.draw(context)`,
Javascript looks to the left of the `draw` name figures out that `obj`
must be the `this` that the caller intended. 

``` js
obj.draw(context) <==> obj.draw.call(obj, context)
```

> Therefore, in a prototype based object system, methods need to
> be *explicitly* designed to support extension through prototype
> chains.

## Implementation details ##

Every object `obj` in Javascript has an `obj.__proto__` property lurking in the
shadows, which refers to the object that will be looked up should some property
requested of  `obj` not be found in it. This is called the object's "prototype"
and Javascript provides some simple special syntax to support making objects
using this prototype chain - the `new` operator.

``` js
var obj = new MyClass();
```

What `new` does is exactly what the function `make` shown below does -

``` js
function make(ClassFn, args...) {
    var obj = {};
    obj.__proto__ = ClassFn.prototype;
    return ClassFn.call(obj, args...) || obj;
}
```

Since the newly created `obj` is passed as the context in `ClassFn.call`,
the body of `ClassFn` can refer to the object whose properties need to
be filled in simply using `this` and it need not bother creating a
new object as well, since one is already available.

Note: There is nothing special about `ClassFn.prototype` and if you're
writing your own `make` function, you can store a prototype object in 
any field of `ClassFn`. It just happens to be the property that `new`
refers to in the bulitin implementation.

## Wrapping it all up ##

1. Model classes as functions that produce objects.

2. Model inheritance and "mixins" using higher order functions
   and function composition.

3. When implementing methods, be aware of whether you want the method
   to be extensible via the prototype chain or not. If a method with
   a bound object in its body is called, it might end up modifying
   the behaviour of *all* the objects that it is a prototype of!    

4. Always be aware of what `this` value is being implicated in any 
   function or method call, by translating `fn(arg)` into `fn.call(??, arg)`.

[Self]: http://en.wikipedia.org/wiki/Self_(programming_language)
