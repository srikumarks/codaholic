---
layout: post
title: "Taming the ScriptProcessorNode"
date: 2013-01-30 21:20
comments: true
published: true
categories: 
- Javascript
- Web Audio API
---

The [Web Audio API] provides graph based API for audio generation and
processing primitives with a focus on high performance and low latency. For
custom processing that is not covered by the builtin native audio nodes, it
provides a [ScriptProcessorNode] whose processing is determined by a Javascript
function. Though the [ScriptProcessorNode] is presented like any other node
type by the API, its behaviour differs from the other native nodes in some
fundamental ways. This post examines some of these differences using a simple
chime model as the use case, and derives some suggestions for the [Web Audio API] 
specification.

<!-- more -->

#### Contents
1. [A simple chime model](#simple-chime-model)
2. [Abstracting the chime model](#abstracting-chime-model)
3. [Replacing the gain node with a ScriptProcessorNode](#replacing-gain-node-with-scriptprocessornode)
    1. [Problem 1: The vanishing script node](#vanishing-script-node)
    2. [Problem 2: The eternal script node](#eternal-script-node)
4. [Replacing the oscillator with a ScriptProcessorNode](#replacing-oscillator-with-scriptprocessornode)
5. [Accounting for tail time](#accounting-for-tail-time)
6. [Reflections on the API](#reflections-on-api)
7. [Concluding suggestions](#concluding-suggestions)
8. [PS: Steller's jsnode model](#stellers-jsnode-model)


## <a name="simple-chime-model"></a>A simple chime model

Let's consider a simple kind of sound - a "chime", a decaying sinusoid
oscillating at a given frequency. This can be implemented using an
[OscillatorNode] and a [GainNode] as follows -

{% img /images/osc-gain.svg %}

``` js
var AC = new webkitAudioContext();
var osc = AC.createOscillator();
osc.frequency.value = 880.0;
var gain = AC.createGainNode();
osc.connect(gain);
gain.connect(AC.destination);
gain.gain.value = 0.25;
gain.gain.setTargetAtTime(0.0, AC.currentTime, 2.0);
osc.start(AC.currentTime + 5.0);
osc.stop(AC.currentTime + 15.0);
```

The above code produces a single chime that lasts for about 10 seconds, with a
decay time constant of 2 seconds. Though this is not very useful as it stands,
it helps to illustrate a couple of aspects of nodes in the [Web Audio API]. 

First, we see two types of nodes being used -- a "source" node (the oscillator)
which generates a signal without needing an input, and a "processor" node (the
gain) which takes an input signal, does something with it and sends a modified
signal to its output.

Second, we see a time limited triggering of the [OscillatorNode]. The node is
triggered 5 seconds into the future and is stopped 10 seconds after it starts.
The way the [Web Audio API] is designed, once the oscillator node has stopped,
it becomes rather useless and needs to be discarded. This is because start/stop
can be used only once on source nodes. Therefore when using an [OscillatorNode]
as in this example, the references to `osc` and `gain` are no longer necessary
and we can add the following lines at the end of the above code block.

``` js
osc = null;
gain = null;
```

With no references holding the oscillator and gain nodes from garbage collection,
one might think that they might be destroyed soon after the references are
given up, before the oscillator gets a chance to generate any sound. Fortunately,
the oscillator holds a reference to itself until its stop time is reached,
after which the reference is released and the subgraph between the source and the
context destination is destroyed. This behaviour of native source nodes is
documented in the [Dynamic Lifetime] section of the [Web Audio API] documentation.

## <a name="abstracting-chime-model"></a>Abstracting the chime model

The chime model described in the previous section only plays one chime. This is
practically useless in the real world. A more useful incarnation of the chime model
would permit us to trigger a chime at any time we want and at any frequency
we want as well. The model manages the nodes necessary for its function
purely internally. 

{% img /images/chime.svg %}

This is easily encapsulated as a function --

``` js
var AC = new webkitAudioContext(); // We'll assume this here onwards.

function chime(freq, output) {
    var stopTime = AC.currentTime + 10.0;
    var osc = AC.createOscillator();
    osc.frequency.value = freq || 880.0;
    var gain = AC.createGainNode();
    osc.connect(gain);
    gain.connect(output || AC.destination);
    gain.gain.value = 0.25;
    gain.gain.setTargetAtTime(0.0, AC.currentTime, 2.0);
    osc.start(AC.currentTime);
    osc.stop(AC.currentTime + 10.0);
    // References to osc and gain are given up
    // upon return from chime().
}
```

## <a name="replacing-gain-node-with-scriptprocessornode"></a>Replacing the gain node with a ScriptProcessorNode

Now let's consider what happens if we try to replace the gain node's behaviour
(limited to this example) using a [ScriptProcessorNode].

``` js <a name="chime_jsgain">chime_jsgain</a> A straightforward replacement of the gain node with a script node.
var AC = new webkitAudioContext();

function chime_jsgain(freq, output) {
    var osc = AC.createOscillator();
    var stopTime = AC.currentTime + 10.0;
    osc.frequency.value = freq || 880.0;
    var gain = AC.createScriptProcessor(1024, 1, 1);
    gain.onaudioprocess = (function () {
        var amplitude = 0.25;
        var decay = Math.exp(- 1.0 / (2.0 * AC.sampleRate));
        return function (event) {
            var i, N, inp, out;
            inp = event.inputBuffer.getChannelData(0);
            out = event.outputBuffer.getChannelData(0);
            for (i = 0, N = out.length; i < N; ++i, amplitude *= decay) {
                out[i] = amplitude * inp[i];
            }
        };
    }());
    osc.connect(gain);
    gain.connect(output || AC.destination);
    osc.start(AC.currentTime);
    osc.stop(stopTime);
}
```

### <a name="vaninshing-script-node"></a>Problem 1: The vanishing script node

If you try to make a chime using [chime_jsgain](#chime_jsgain), you'll find
that the sound stops abruptly well before the 10 seconds duration given.  This
is because the script node is garbage collected almost immediately.  This, I
believe, is a WebKit implementation bug since the oscillator node, which has a
persistent reference till stop time, holds a reference to the script node
through its output, and yet the script node is garbage collected. A known
workaround for this bug is to maintain a *global* reference to the script node.
For this, we can use the following simple scheme --

``` js <a name="keep">keep</a> Preserving script nodes.
var scriptNodes = {};
var keep = (function () {
    var nextNodeID = 1;
    return function (node) {
        node.id = node.id || (nextNodeID++);
        scriptNodes[node.id] = node;
        return node;
    };
}());
```

Using the `keep` utility, we can now rewrite `chime_jsnode` as --

``` js Keeping around a global reference to the script node.
function chime_jsgain(freq, output) {
    // ...
    var gain = keep(AC.createScriptProcessor(1024, 1, 1));
    // ...
}
```

This will result in the script node being preserved during the course of
the chime.

### <a name="eternal-script-node"></a>Problem 2: The eternal script node

With the modifications of the preceding section, the oscillator node will
disappear after 10 seconds but the script node will persist and continue to be
processed. To prevent this, we make arrangement for the script node to be
removed from the graph and for its global reference to be dropped once it has
processed enough samples.

``` js <a name="drop">drop</a> Dropping the global reference to a node.
function drop(node) {
    delete scriptNodes[node.id];
    return node;
}
```

We incorporate the new `drop` function into `chime_jsnode` as follows --

``` js A script implementation of gain node with proper end of life management.
function chime_jsgain(freq, output) {
    var osc = AC.createOscillator();
    var stopTime = AC.currentTime + 10.0;
    osc.frequency.value = freq || 880.0;
    var gain = keep(AC.createScriptProcessor(1024, 1, 1));
    gain.onaudioprocess = (function () {
        var amplitude = 0.25;
        var decay = Math.exp(- 1.0 / (2.0 * AC.sampleRate));
        var stopTime_samples = Math.ceil(AC.sampleRate * stopTime);
        var finished = false;
        return function (event) {
            var i, N, inp, out;

            if (finished) {
                return; // Don't do anything after stopTime has elapsed.
            }

            inp = event.inputBuffer.getChannelData(0);
            out = event.outputBuffer.getChannelData(0);

            // Limit the number of samples we compute.
            var now_samples = Math.floor(AC.sampleRate * AC.currentTime);
            N = Math.min(out.length, stopTime_samples - now_samples);

            for (i = 0; i < N; ++i, amplitude *= decay) {
                out[i] = amplitude * inp[i];
            }

            if (N < out.length) {
                // Reached end time before end of buffer.
                finished = true;
                setTimeout(function () {
                    drop(gain);
                    osc.disconnect();
                    gain.disconnect();
                }, 0);
            }
        };
    }());
    osc.connect(gain);
    gain.connect(output || AC.destination);
    osc.start(AC.currentTime);
    osc.stop(stopTime);
}
```

We now have a script node based implementation that works like the native
implementation in all respects. This kind of "stop time" behaviour can be
lifted into a separate function that transforms a `function (event, toSamp) {...}` 
into an `onaudioprocess` handler.

``` js <a name="scriptWithStopTime">scriptWithStopTime</a> End of life management of script nodes.
function scriptWithStopTime(stopTime, handler) {
    var node = keep(AC.createScriptProcessor(1024, 1, 1));
    var stopTime_samples = Math.ceil(AC.sampleRate * stopTime);
    var finished = false;
    node.onaudioprocess = function (event) {
        if (finished) {
            return;
        }

        var t = Math.floor(AC.sampleRate * AC.currentTime);
        var toSamp = Math.min(event.outputBuffer.length, stopTime_samples - t);

        handler(event, toSamp);

        if (toSamp < event.outputBuffer.length) {
            finished = true;
            setTimeout(function () {
                drop(event.node);
                event.node.disconnect();
            }, 0);
        }
    };
    return node;
}
```

Using [scriptWithStopTime](#scriptWithStopTime), we can now rewrite `chime_jsgain` as --

``` js
function chime_jsgain(freq, output) {
    var osc = AC.createOscillator();
    var stopTime = AC.currentTime + 10.0;
    osc.frequency.value = freq || 880.0;
    var gain = scriptWithStopTime(stopTime, (function () {
        var amplitude = 0.25;
        var decay = Math.exp(- 1.0 / (2.0 * AC.sampleRate));
        return function (event, N) {
            var i, inp, out;
            inp = event.inputBuffer.getChannelData(0);
            out = event.outputBuffer.getChannelData(0);
            for (i = 0; i < N; ++i, amplitude *= decay) {
                out[i] = amplitude * inp[i];
            }
        };
    }()));
    osc.connect(gain);
    gain.connect(output || AC.destination);
    osc.start(AC.currentTime);
    osc.stop(stopTime);
}
```

## <a name="replacing-oscillator-with-scriptprocessornode"></a>Replacing the oscillator with a ScriptProcessorNode

Now, let's consider what happens if we try to implement the [OscillatorNode]
(limited to this example) using a [ScriptProcessorNode]. We can use
[scriptWithStopTime](#scriptWithStopTime) to take a first shot at it.
While we're there, we'll also generalize the chime model to take a time
at which to trigger the chime.

``` js
function chimeJS(t, freq, output) {
    freq = freq || 880.0;
    t = Math.max(t || 0.0, AC.currentTime);
    var stopTime = t + 10.0;    
    var osc = scriptWithStopTime(stopTime, (function () {
        var phi = 0, dphi = 2.0 * Math.PI * freq / AC.sampleRate;
        return function (event, N) {
            var i, out;
            out = event.outputBuffer.getChannelData(0);
            for (i = 0; i < N; ++i, phi += dphi) {
                out[i] = Math.sin(phi);
            }
        };
    }()));
    var gain = AC.createGainNode();
    gain.gain.value = 0.0;
    gain.gain.setValueAtTime(0.25, t);
    gain.gain.setTargetAtTime(0.0, t, 2.0);
    osc.connect(gain);
    gain.connect(output || AC.destination);
}
```

Exposing the start time makes obvious a few design flaws --

1. If you schedule a chime into the future instead of "right now",
   the script node oscillator will keep running from "right now" until
   the end. The period from "now" to the scheduled start time is all
   wasted computation.

2. The node responsible for the chime starting on time is the `gain`
   node. Since the oscillator is continuously running, the phase at
   which the oscillator begins will depend on the time passed in,
   but ideally we want the oscillator to start with phase 0 when
   the chime starts .. and we want it to be sample accurate.

To solve the wasted computation problem, we can make the `onaudioprocess`
be a no-op until the start time. This is best done by writing a
`scriptWithStartStopTime` as we did with `scriptWithStopTime`.


``` js <a name="scriptWithStartStopTime2">scriptWithStartStopTime</a> Removing wasted computation.
function scriptWithStartStopTime(startTime, stopTime, handler) {
    console.assert(stopTime >= startTime);
    var node = keep(AC.createScriptProcessor(1024, 1, 1));
    var startTime_samples = Math.floor(AC.sampleRate * startTime);
    var stopTime_samples = Math.ceil(AC.sampleRate * stopTime);
    var finished = false;
    node.onaudioprocess = function (event) {
        if (finished) {
            return;
        }

        var t = Math.floor(AC.currentTime * AC.sampleRate);
        var fromSamp = Math.max(0, startTime_samples - t);

        if (fromSamp >= event.outputBuffer.length) {
            // Not started yet.
            return;
        }

        var toSamp = Math.min(event.outputBuffer.length, stopTime_samples - t);

        // Handler signature changed to include both start and stop indices.
        handler(event, fromSamp, toSamp);

        if (toSamp < event.outputBuffer.length) {
            finished = true;
            setTimeout(function () {
                drop(event.node);
                event.node.disconnect();
            }, 0);
        }
    };
    return node;
}
```

The above version saves *some* computation but not all. The JS node does not
need to receive any callbacks whatsoever until it is time to start generating
samples. To achieve this, we need to schedule the node to connect and start
just in time, say 0.1 seconds before the scheduled start time. To do this,
we need to generalize `scriptWithStartStopTime` to work with the nodes
to which the script node is expected to connect.


``` js <a name="scriptWithStartStopTime2">scriptWithStartStopTime</a> Delayed node setup.
function scriptWithStartStopTime(input, output, startTime, stopTime, handler) {
    startTime = Math.max(startTime, AC.currentTime);
    stopTime = Math.max(stopTime, AC.currentTime);
    console.assert(stopTime >= startTime);

    var kBufferLength = 512; // samples
    var prepareAheadTime = 0.1; // seconds

    var startTime_samples = Math.floor(AC.sampleRate * startTime);
    var stopTime_samples = Math.ceil(AC.sampleRate * stopTime);
    var finished = false;

    function onaudioprocess(event) {
        if (finished) { return; }

        var t = Math.floor(AC.currentTime * AC.sampleRate);
        var fromSamp = Math.max(0, startTime_samples - t);

        if (fromSamp >= event.outputBuffer.length) {
            return; // Not started yet.
        }

        var toSamp = Math.min(event.outputBuffer.length, stopTime_samples - t);

        // Handler signature changed to include both start and stop indices.
        handler(event, fromSamp, toSamp);

        if (toSamp < event.outputBuffer.length) {
            finished = true;
            setTimeout(function () {
                drop(event.node);
                input && input.disconnect();
                event.node.disconnect();
            }, 0);
        }
    }

    function prepareNode() {
        var node = keep(AC.createScriptProcessor(kBufferLength, 1, 1));
        node.onaudioprocess = onaudioprocess;

        // Setup the necessary connections.
        input && input.connect(node);
        output && node.connect(output);
    }

    var dt = startTime - AC.currentTime;
    if (dt <= prepareAheadTime) {
        prepareNode();
    } else {
        setTimeout(prepareNode, Math.floor(1000 * dt));
    }

    return node;
}
```

We can use `scriptWithStartStopTime` to include such dynamically determined
source or processing nodes within a subgraph ... except when two such nodes
need to be connected to each other and both remain unrealized for a while. In
such cases, we can use an intermediate unity gain node for simplicity. Here
then is our final chime model with a js node for the gain or the oscillator.

``` js
function chime_jsosc(t, freq, output) {
    freq = freq || 880.0;
    t = Math.max(t || 0.0, AC.currentTime);
    var stopTime = t + 10.0;   

    var gain = AC.createGainNode();
    gain.gain.value = 0.0;
    gain.gain.setValueAtTime(0.25, t);
    gain.gain.setTargetAtTime(0.0, t, 2.0);
    gain.connect(output || AC.destination);

    var osc = scriptWithStartStopTime(null, gain, t, stopTime, (function () {
        var phi = 0, dphi = 2.0 * Math.PI * freq / AC.sampleRate;
        return function (event, fromSamp, toSamp) {
            var i, out;
            out = event.outputBuffer.getChannelData(0);
            for (i = fromSamp; i < toSamp; ++i, phi += dphi) {
                out[i] = Math.sin(phi);
            }
        };
    }()));
}

function chime_jsgain(t, freq, output) {
    freq = freq || 880.0;
    t = Math.max(t || 0.0, AC.currentTime);
    output = output || AC.destination;
    var stopTime = t + 10.0;   

    var osc = AC.createOscillator();
    osc.frequency.value = freq;

    var gain = scriptWithStartStopTime(osc, output, t, stopTime, (function () {
        var amplitude = 0.25;
        var decay = Math.exp(- 1.0 / (2.0 * AC.sampleRate));
        return function (event, fromSamp, toSamp) {
            var i, inp, out;
            inp = event.inputBuffer.getChannelData(0);
            out = event.outputBuffer.getChannelData(0);
            for (i = fromSamp; i < toSamp; ++i, amplitude *= decay) {
                out[i] = amplitude * inp[i];
            }
        };
    }()));

    osc.start(t);
    osc.stop(stopTime);
}    
```

## <a name="accounting-for-tail-time"></a>Accounting for tail time

The gain node is simple in that it produces one output sample for each input
sample it receives on its input pin. Other node types, especially filter and
convolution nodes, may "tail off" well after the input to these nodes has
ceased. In other words, the true stop time of a node at which the node is
free to be garbage collected is at the end of such a tail time, which may
be only known when the node is running.

A simple way to account for this is to have the handler passed to
`scriptWithStartStopTime` use the `toSamp` argument only for information and
return the number of samples it actually produced. If the handler has produced
samples up to the buffer's capacity, then it cannot be stopped.  If it stopped
somewhere before the end of the buffer, then it can be taken to be finished and
cleanup can be triggered. This logic can be expressed in a modified
`scriptWithStartStopTime` as shown below --

``` js <a name="final-scriptWithStartStopTime">scriptWithStartStopTime</a> Supporting tail time.
function scriptWithStartStopTime(input, output, startTime, stopTime, handler) {
    startTime = Math.max(startTime, AC.currentTime);
    stopTime = Math.max(stopTime, AC.currentTime);
    console.assert(stopTime >= startTime);

    var kBufferLength = 512; // samples
    var prepareAheadTime = 0.1; // seconds

    var startTime_samples = Math.floor(AC.sampleRate * startTime);
    var stopTime_samples = Math.ceil(AC.sampleRate * stopTime);
    var finished = false;

    function onaudioprocess(event) {
        if (finished) { return; }

        var t = Math.floor(AC.currentTime * AC.sampleRate);
        var fromSamp = Math.max(0, startTime_samples - t);

        if (fromSamp >= event.outputBuffer.length) {
            return; // Not started yet.
        }

        var toSamp = Math.min(event.outputBuffer.length, stopTime_samples - t);

        // Return value of handler is used to decide when to stop. The handler is
        // required to produce as many samples as possible. If it still can't fill
        // up the buffer, it is deemed to have finished.
        var samplesProduced = handler(event, fromSamp, toSamp);

        if (fromSamp + samplesProduced < event.outputBuffer.length) {
            finished = true;
            setTimeout(function () {
                drop(event.node);
                input && input.disconnect();
                event.node.disconnect();
            }, 0);
        }
    }

    function prepareNode() {
        var node = keep(AC.createScriptProcessor(kBufferLength, 1, 1));
        node.onaudioprocess = onaudioprocess;

        // Setup the necessary connections.
        input && input.connect(node);
        output && node.connect(output);
    }

    var dt = startTime - AC.currentTime;
    if (dt <= prepareAheadTime) {
        prepareNode();
    } else {
        setTimeout(prepareNode, Math.floor(1000 * dt));
    }

    return node;
}
```

## <a name="reflections-on-api"></a>Reflections on the API

The [final form of scriptWithStartStopTime](#final-scriptWithStartStopTime)
expresses functionality that is essential to the use of script nodes as
algorithmic signal sources and signal processors that can be scheduled
to sample accuracy. Although this function adds the required functionality,
the [Web Audio API] would be better off with a script node type whose
lifetime can be managed just like the native nodes and suppor the
dynamic behaviour available with the native nodes.

Not having it builtin results in an inconsistency that is awkward to
work around - the fact that `start(t)` and `stop(t)` methods added in pure
Javascript are not only necessary for script nodes that are pure signal
generators, but also for signal processors. This is because a script node might
otherwise end up wasting compute cycles either idling or computing audio that
is going to be discarded by the rest of the chain. A `stop(t)` method is
necessary for such processor nodes because the [Web Audio API] does not
currently provide any notification mechanisms for monitoring connections to a
node's input and output so that processor nodes can self destruct when their
inputs die.

## <a name="concluding-suggestions"></a>Concluding suggestions

The use case worked through in this post demonstrates how abstractions built on
other nodes do not extend to script nodes without special considerations. The
two kinds of usages for script nodes discussed here -- sources and signal
processors -- are likely to be very common scenarios, which makes it worthwhile
to try to avoid the problems that arise with such usage of script nodes. Having
the following features built into the [ScriptProcessorNode] is therefore
important.

1. Permit creation of script nodes without inputs. This better models source
   nodes. Passing 0 for "number of input channels" may be enough at the API
   level.
2. Add `start(t)` and `stop(t)` methods to permit script nodes to be used as
   signal sources, with such nodes not taking up any system resources during
   their inactive periods. 
3. Add [dynamic lifetime] support similar to native nodes, whereby unreferenced
   "signal processor" script nodes driven by source nodes are automatically
   released once the source node finishes, even if the source node is itself a
   script node. To achieve this, the time at which the inputs cease must be
   available as part of the event structure passed to the `onaudioprocess`
   callback, so that the callback can begin any tail phase that it needs to
   complete before it commits suicide.
4. Specify a convention and/or API to support tail times beyond 
   the time indicated in a stop(t) call or after its inputs have
   been end-of-lifed.

### <a name="stellers-jsnode-model"></a>PS: Steller's jsnode model

The [Steller] library features a [jsnode] model that wraps the API's script
node into a node type that can be scheduled as discussed in this post. It also
adds some conveniences such as multiple inputs (albeit single channel), and
a-rate controllable and schedulable [AudioParam] parameters. You can use
Steller's jsnode model like this -

``` html
<script src="http://sriku.org/lib/steller/steller.min.js"></script>
<script>
    var sh = new org.anclab.steller.Scheduler(new webkitAudioContext);
    var jsn = sh.models.jsnode({
        numberOfInputs: 0,
        numberOfOutputs: 1,
        onaudioprocess: function (event) {
            // ...
            return toSamp - fromSamp;
        }});
    jsn.connect(AC.destination);
    sh.play(sh.fire(function (clock) {
        jsn.start(clock.t);
        jsn.stop(clock.t + 10.0);
    }));
</script>
```


[ScriptProcessorNode]:      http://www.w3.org/TR/webaudio/#ScriptProcessorNode
[Web Audio API]:            http://www.w3.org/TR/webaudio
[BiquadFilterNode]:         http://www.w3.org/TR/webaudio/#BiquadFilterNode
[AudioBufferSourceNode]:    http://www.w3.org/TR/webaudio/#AudioBufferSourceNode
[OscillatorNode]:           http://www.w3.org/TR/webaudio/#OscillatorNode
[GainNode]:                 http://www.w3.org/TR/webaudio/#GainNode
[dynamic lifetime]:         http://www.w3.org/TR/webaudio/#DynamicLifetime
[GraphNode]:                https://github.com/srikumarks/steller/blob/experimental_buildsys/src/graphnode.js
[jsnode]:                   https://github.com/srikumarks/steller/blob/experimental_buildsys/src/models/jsnode.js
[Steller]:                  https://github.com/srikumarks/steller
[jsnode]:                   https://github.com/srikumarks/steller/blob/experimental_buildsys/src/models/jsnode.js
[AudioParam]:               http://www.w3.org/TR/webaudio/#AudioParam
