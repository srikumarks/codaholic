---
layout: post
title: "Digging into ScriptProcessorNode"
date: 2013-01-25 21:20
comments: true
published: false
categories: 
---

The [Web Audio API] provides graph based API for audio generation and
processing primitives with a focus on high performance and low latency. For
custom processing that is not covered by the builtin native audio nodes, it
provides a [ScriptProcessorNode] whose processing is determined by a Javascript
function. Though the ScriptProcessorNode is presented like any other node type
by the API, its behaviour differs from the other native nodes in some
fundamental ways. In this post, I go through these differences with regard to a
few common audio processing scenarios.

<!-- more -->

## A custom source node

A "source node" is a node type that generates output audio without any inputs.
Since all script nodes have one input and one output, a source node needs to be
simulated by making a script node whose input is left unconnected. Here is a simple
sine wave generator.

``` js
var AC = new webkitAudioContext(); // We'll assume this here onwards.
var js = AC.createScriptProcessor(1024, 1, 1);
var sineGen = (function () {
    var phi = 0.0, dphi = 2 * Math.PI * 440.0 / AC.sampleRate;
    return function (event) {
        var i, N, out;
        out = event.outputBuffer.getChannelData(0);
        for (i = 0, N = out.length; i < N; ++i, phi += dphi) {
            out[i] = 0.25 * Math.sin(phi);
        }
    };
}());
js.onaudioprocess = sineGen;
js.connect(AC.destination);
```

The above code results in a 440Hz sine tone being generated. This is expected
to be identical in meaning to the following code using an [OscillatorNode]. 

``` js
var osc = AC.createOscillator();
osc.frequency.value = 440.0;
osc.connect(AC.destination);
osc.start(AC.currentTime);
```

The oscillator node will continue to run as long as its stop time is not
reached, which is set using `osc.stop(t)`. As long as its stop time is not
reached, `osc` is guaranteed to remain alive even if we lose all references to
it. So if we add `osc = null` to the above code, it would make no difference to
the result.

However, adding `js = null` will result in the script node stopping pretty
soon. It turns out that we need to keep around a reference to the script node
in order for it to run. In the WebKit implementation, this reference must be
a *global* reference, though in principle it isn't clear why a reference to 
the node through a closure is insufficient. In particular, it seems insufficient
to hold a reference to the script node through another node such as a gain
node that is connected to the script node's input.

A second inconsistency here is that the script node cannot be started/stopped
at an arbitrary sample accurate time like the oscillator node can.

## Dynamic lifetime

Consider the following trivial pipeline involving an oscillator node generating
a sawtooth wave which is filtered using a biquad filter, with the oscillator
set to run for 10 seconds.

``` js
var osc = AC.createOscillator();
osc.type = osc.SAWTOOTH;
osc.frequency.value = 440.0;
var filter = AC.createBiquadFilter();
filter.frequency.value = 440.0;
osc.connect(filter);
filter.connect(AC.destination);
osc.start(AC.currentTime);
osc.stop(AC.currentTime + 10.0);
osc = null;
filter = null;
```

Once the stop time arrives, the [dynamic lifetime] behaviour of source nodes
ensures that both the oscillator and the filter are destroyed. This feature of
the API permits us to treat such subgraphs as "voices" that we can trigger
and forget about, with the subsystem managing the cleanup process once a
voice is done.

Consider a similar situation with a script audio node filtering the
oscillator's output.

``` js
var osc = AC.createOscillator();
osc.type = osc.SAWTOOTH;
osc.frequency.value = 440.0;
var filter = AC.createScriptProcessor(512, 1, 1);
filter.onaudioprocess = (function () {
    var state = 0;
    return function (event) { 
        var inp = event.inputBuffer.getChannelData(0);
        var out = event.outputBuffer.getChannelData(0);
        var i, N;
        for (i = 0, N = outp.length; i++) {
            // Pardon the silly filter.
            state = 0.5 * (state + inp[i]);
            out[i] = state;
        }
    };
}());
osc.connect(filter);
filter.connect(AC.destination);
osc.start(AC.currentTime);
osc.stop(AC.currentTime + 10.0);
osc = null;
filter = null;
```

Again, we find that the script node stops processing audio well before the oscillator's
stop time is reached, unless a global reference to the `filter` is kept around.

This case involves two inconsistencies with the native nodes -

1. The oscillator source node is unable to keep its processing chain around
   in the absence of a global reference to the chain components, unlike the
   case with native components.

2. Since a global reference to the script node is needed for the oscillator
   to run, once the oscillator's stop time is reached, the processing chain
   will not be released unless explicit arrangement is made for it to be
   released at a later time.

## Impact on abstraction

If we want to be able to build sound and effects models that are presented as
stable nodes whose internals are hidden from the models' users, it looks like
the difference in behaviour between the [ScriptProcessorNode] and the other
native nodes creates a leak in the abstraction whereby one needs to be aware
of when a subgraph needs to be explicitly managed and when its management
can be left to the system. This is undesirable.

The [GraphNode] is one such abstraction which boxes an arbitrary subgraph
and makes it look like a node. This node can then be composed with other
such nodes, hoping that the internals of the node do not need to be
bothered about. If the internal part of a [GraphNode] contains a script
node, then explicit management is needed, unlike the case where the
internals are only native nodes.

## A well behaved script node

To mitigate the "leaky abstraction" that [GraphNode] is, I wrote a wrapper for
the script processor called [jsnode] that adds source-node-like behaviour to
script nodes while making it possible to write scripts that can self destruct
once they are done. Apart from support for source-node-like `start()` and
`stop()` methods, [jsnode] also adds support for multiple inputs, multiple
outputs (albeit single channel ones) and named `AudioParam` properties for the
node.

Using [jsnode], the first sine tone generator would read like this -

``` js
var AC = new webkitAudioContext();
var sh = new org.anclab.steller.Scheduler(AC);
    // Can be imported using <script src="http://sriku.org/lib/steller/steller.min.js"></script>
var js = sh.models.jsnode({
    numberOfInputs: 0,
    numerOfOutputs: 1,
    onaudioprocess: (function () {
        var phi = 0.0, dphi = 2 * Math.PI * 440.0 / AC.sampleRate;
        return function (event) {
            var i, N, out;
            out = event.outputs[0];
            for (i = 0, N = Math.min(out.length, event.samplesToStop); i < N; ++i, phi += dphi) {
                out[i] = 0.25 * Math.sin(phi);
            }
            return N;
        };
    }())
});
js.connect(AC.destination);
js.start(AC.currentTime);
js.stop(AC.currentTime + 10.0); // Die after 10 seconds.
js = null;
```

Losing the reference to the [jsnode] is not a problem as the required
referencing is managed internally. Also, after the stop time of 10 seconds
is reached, the node will self destruct.

Likewise, the script node can also operate as part of a subgraph that
is run in "fire and forget" mode.

``` js
var sh = new org.anclab.steller.Scheduler(AC);
var osc = AC.createOscillator();
osc.type = osc.SAWTOOTH;
osc.frequency.value = 440.0;
var filter = sh.models.jsnode({
    numberOfInputs: 1,
    numberOfOutputs: 1,
    onaudioprocess: (function () {
        var state = 0;
        var tailPeriod = 16;
        return function (event) { 
            var inp = event.inputs[0];
            var out = event.outputs[0];
            var i, N;
            for (i = 0, N = Math.min(outp.length, tailPeriod + event.samplesToStop); i < N; i++) {
                // Pardon the silly filter.
                state = 0.5 * (state + inp[i]);
                out[i] = state;
            }
            return N;
        };
    }())
});
osc.connect(filter);
filter.connect(AC.destination);
osc.start(AC.currentTime);
osc.stop(AC.currentTime + 10.0);
filter.stop(AC.currentTime + 10.0); // Automatically allows for a tail period of 16 samples.
osc = null;
filter = null;
```

Now, when the respective stop times occur, the oscillator and
the filter will be mopped up. This permits the "osc->filter"
combination to be hidden within a [GraphNode] without any
abstraction leakage.

## API suggestions

The implementation of [jsnode] suggests facilities that are best
provided out of the box for the [ScriptProcessorNode]. 

The primary features suggested are -

1. Support for source nodes with sample accurate scheduling,
2. Support for dynamic lifetime management for processor (filter) style nodes,
3. Support for "tail time" beyond indicated stop time,
4. Support for proper lifetime management even with a-rate AudioParams and multiple inputs.

[ScriptProcessorNode]: http://www.w3.org/TR/webaudio/#ScriptProcessorNode
[Web Audio API]: http://www.w3.org/TR/webaudio
[BiquadFilterNode]: http://www.w3.org/TR/webaudio/#BiquadFilterNode
[AudioBufferSourceNode]: http://www.w3.org/TR/webaudio/#AudioBufferSourceNode
[OscillatorNode]: http://www.w3.org/TR/webaudio/#OscillatorNode
[dynamic lifetime]: http://www.w3.org/TR/webaudio/#DynamicLifetime
[GraphNode]: https://github.com/srikumarks/steller/blob/experimental_buildsys/src/graphnode.js
[jsnode]: https://github.com/srikumarks/steller/blob/experimental_buildsys/src/models/jsnode.js

