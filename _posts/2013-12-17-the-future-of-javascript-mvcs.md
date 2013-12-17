---
layout: post
title: "The Future of JavaScript MVC Frameworks"
description: ""
category: 
tags: []
---
{% include JB/setup %}

## Introducing Om

<div style="padding: 10px 0px 10px 45px; border-bottom: 1px solid
#ccc;">
<blockquote class="twitter-tweet" lang="en"><p>often devs still approach performance of JS code as if they are riding a horse cart but the horse had long been replaced with fusion reactor</p>&mdash; Vyacheslav Egorov (@mraleph) <a href="https://twitter.com/mraleph/statuses/411549064787152896">December 13, 2013</a></blockquote>
<script async src="//platform.twitter.com/widgets.js"
charset="utf-8"></script>
</div>

We've known this for some time over here in the
[ClojureScript](http://github.com/clojure/clojurescript) corner of the
world - all of our collections are immutable and modeled directly on
the original Clojure versions written in Java and modern JavaScript
engines have now been tuned to the point that it's no longer uncommon
to see collection performance within 2.5X of the Java Virtual Machine.

Wait, wait, wait. What does the performance of persistent data structures have
to do with future of JavaScript MVCs?

A whole lot.

We'll see how (unintuitively) immutable data allows a new library,
[Om](http://github.com/swannodette/om), to outperform nearly every
existing JavaScript MVC. Om itself is built upon the absolutely
wonderful [React](http://facebook.github.io/react/) library from
Facebook, however our approach allows Om to deliver even better
results than using React out of the box.

Before we proceed it's important to understand that the following
benchmarks are not the silly uninformed things you find on
[jsperf.com](http://jsperf.com). These benchmarks are designed
demonstrate *fundamentally flawed design decisions in JavaScript MVCs
that defy global optimization*. As far as using
[TodoMVC](http://todomvc.com), it's only because it suitably
demonstrates anti-patterns that will exist in any typical JavaScript
MVC application. But it's also quite familiar to many JavaScript
developers such that I can easily point out bad ideas that you're
bound to recognize in your own code base if you leverage an MVC
framework.

Of course you can correct these issues on a tedious case by case
basis, but the whole point of Om is make an entire class of hand
optimizations techniques *obsolete*. That's the power of starting with
the right design to begin with.

## Game of Benchmarks

<div style="padding: 10px 0px 10px 45px; border-bottom: 1px solid
#ccc;">
<blockquote class="twitter-tweet" lang="en"><p>ClojureScript om based TodoMVC looks 30-40X faster than Backbone.js TodoMVC, which means other JS frameworks left completely in the dust</p>&mdash; David Nolen (@swannodette) <a href="https://twitter.com/swannodette/statuses/412033352699744256">December 15, 2013</a></blockquote>
<script async src="//platform.twitter.com/widgets.js"
charset="utf-8"></script>
</div>

Open the [Om TodoMVC in a tab]() and run the first benchmark - it creates
200 todos. On my 11 inch Macbook it takes Safari 7 around 100ms to render.

Open up the [Backbone.js TodoMVC in a tab]() and run the same
 benchmark.  On my machine this takes around 700ms to render.

Under Chrome and Firefox, Om on my machine is consistently 2-3X
faster. 2-7X faster across 3 major browser should be enough to get
anyone interested. If you try toggling all of the todos you'll notice
Om feels natural, while Backbone.js will feel a bit janky. This is
probably because Om always re-renders on
[requestAnimationFrame](http://www.paulirish.com/2011/requestanimationframe-for-smart-animating/).

Ok, excellent work! But, uh, that's nowhere near the 30X-40X claim!

Try the following.

Open the [Om TodoMVC in a tab]() and run the first benchmark - it creates
200 todos, toggles them all 5 times, and then deletes them. On my 11
inch Macbook it takes around 50ms to render.

Open up the [Backbone.js TodoMVC in a tab]() and run the second benchmark
which does the exact same series of operations. On my machine this
takes around 2500ms seconds to render.

*How is this possible?*

Simple.

The first one doesn't do any work that it doesn't have to! Om is a
thoroughly *decoupled* design: data, views and control logic are not
inextricably tied together. This counter to MVC approaches that
directly link together changes in the model, the view, and to
fundamentally orthogonal concerns like serializing app state into
localStorage. Every time you update a model you have to pay for a
model change, a view update, and a localStorage write.

WAT.

Now it may appear you could adopt this architecture with a traditional
JS MVC. But actually you couldn't, at least not without discarding
much of the APIs they provide. You would need to ditch event
communication between models and views. The View layer would need to
replaced with React or something like it. And to get Om level
rendering performance you need to switch all your data to immutable
values. At which point you would probably just React, use
[mori](http://swannodette.github.io/mori/) for your data, and you
would be left with about 6 functions.

Just like Om.

Hopefully the former gave you some food for thought. A compile to
JavaScript language that uses data structures slower than the native
ones provided by JavaScriopt is globally faster for rich user
interfaces.

Chew on that.

What follows is a pretty technical description.

## How it works

My recent in interest React is thanks to my friend Brandom Bloom has
been bugging me for month to give React a closer look, however I was
turned off by the OOP-y documentation and the inline HTML. I didn't
give it a proper chance until I saw the
[JSConf EU 2013 presentation by Peter Hunt](http://2013.jsconf.eu/speakers/pete-hunt-react-rethinking-best-practices.html)
that explained the architecture.

Modifying and querying the DOM is a huge performance bottleneck and
React adopts an approach that avoids that without sacrificing
expressivity. While it present a friendly well designed Object
Oriented interface, everything underneath the hood is has been crafted
with the eye of a pragmatic functional programmer.

The React Devs have been ridiculously friendly and responsive in
answering many questions so I could determine an passable interface to
React from ClojureScript.

When React does a diff on the virtual DOM specified by your
components there is a very critical function -
`shouldComponentUpdate`. If this returns false, React will never
compute the children of the component. This is aboslute critical to
understand - React builds the virtual DOM tree lazily for diffing.

As it turns out `shouldComponentUpdate` is extremely conservative
because JavaScript devs tend to mutate objects - so in order to
determine if some properties of a component has changed they have to
manually walk the JavaScript object / array to figure this out.

Instead of using JavaScript objects Om uses ClojureScript data
structures which we cannot be changed. Because of this we can provide
a component that implements `shouldComponentUpdate` - all it needs to
do is a reference equality check. This simply change means we can
always determine the path changed starting from the root in
logarithmic time.

Because of this fact we don't need React convenience like `setState`
which support efficient subtree updating. Subtree updating for Om is
always lightning fast even if your UI graph is massive.

Because we always re-render from the root, batched updates are trivial
to implement. We don't even bother with the batched update support in
React, we just rolled our own 6 lines change.

Finally because we always have the entire state of the UI in a single
piece of data we can trivially serialize all of the important app
state - we don't need to bother with serialization protocols, or
making sure that everyone implements them correctly. Om UIs are always
serializable.

But this also means that Om UIs get undo for free. You can simply
snapshot any state in memory and reinstate it whenever you like. It's
memory efficient as ClojureScript data structures work by sharing
structure anyway.

## Closing Thoughts

In short I don't think there is a future in JavaScript MVCs. I think
if you sit down and think about in the end only something akin to Om
(even it's tucked away under a traditional hood) will deliver an
optimal balance between simplicity, performance, and expressivity.