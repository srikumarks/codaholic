---
layout: post
title: "Fun with nine nadais"
date: 2013-05-22 22:31
categories: 
- Carnatic music
- Tala Keeper
---

Here's something fun done in [Tala Keeper](http://talakeeper.org) - a walk
through of [*Nine Nadais* in one big 36 beat cycle][1]. If you're viewing this
on your iOS device, tapping that link will open it in Tala Keeper. If not, the
pattern will play in another browser window in the free HTML5 simulator.

<!-- more -->

## Recite along

Try reciting the following sols along with Tala Keeper when playing this sequence -

    /1 taam taam taam taam
    /2 taka taka taka taka
    /3 takiṭa takiṭa takiṭa takiṭa
    /4 takadimi takadimi takadimi takadimi
    /5 takatakiṭa takatakiṭa takatakiṭa takatakiṭa
    /6 takiṭatakiṭa takiṭatakiṭa takiṭatakiṭa takiṭatakiṭa
    /7 takadimitakiṭa takadimitakiṭa takadimitakiṭa takadimitakiṭa
    /8 takadimitakadimi takadimitakadimi takadimitakadimi takadimitakadimi
    /9 takadimitakatakiṭa takadimitakatakiṭa takadimitakatakiṭa takadimitakatakiṭa

Alternative sols with kaarvais - 

    /1 taam taam taam taam
    /2 taka taka taka taka
    /3 takiṭa takiṭa takiṭa takiṭa
    /4 takadimi takadimi takadimi takadimi
    /5 ta-dim-ta ta-dim-ta ta-dim-ta ta-dim-ta
    /6 taamtataamta taamtataamta taamtataamta taamtataamta
    /7 ta-dim-takiṭa ta-dim-takiṭa ta-dim-takiṭa ta-dim-takiṭa
    /8 takadimitakadimi takadimitakadimi takadimitakadimi takadimitakadimi
    /9 ta-dim-ta-dim-ta ta-dim-ta-dim-ta ta-dim-ta-dim-ta ta-dim-ta-dim-ta

## How to make this pattern

1. The [base of the pattern][p1] is a simple 4-beat cycle with the ball bouncing
   "left-right-right-right" at 70bpm. So tap out that pattern and store it in
   the first memory location. Make sure that the chime is off and the nadai and
   kalai are neutral before storing. You're now going to make 8 other variations
   of this with different nadai pulses and stitch them into a sequence.

2. Now subdivide each beat into two by tapping the "×÷", then tapping the pulse
   pad twice and finishing off with "÷". Store [this pattern][p2] in the second
   memory location.

3. Do similar subdivisions for each of the nine nadais and store each in nine
   different memory locations.

Now you need to sequence all these nine pieces into a single pattern. 

1. "Recall" the first memory location - [the base pattern][p1].

2. Tap "+". Then recall the second memory location where you stored
   [subdivisions of 2][p2]. Now the [combined pattern][p12] will be 8 beats
   long, with the first 4 beats being the base pattern and the next 4 beats
   having subdivisions of 2.

3. Tap "+" again, then recall the third memory location. Repeat this until
   you've covered all the nine nadais you'd stored away.

4. Optionally turn on the "chime" now.

*Note:* I made the nadai patterns a bit more interesting than a plain "/N"
subdivision by using pauses. To introduce a pause in the nadai pulse, just
tap in the blank area (say, just above the "floor" line) instead of the pulse
pad.

PS: Comments on [Google+](https://plus.google.com/102694714835839603248/posts/JmqRJWsdtWD).

<script>
if (/(iPhone)|(iPod)|(iPad)|(iOS)/.test(window.navigator.userAgent)) {
    // On iOS Safari. Turn the tala keeper links into tala:// so
    // that it opens directly in Tala Keeper instead of visiting
    // http://talakeeper.org/tk3 first.
    (function () {
        var links = Array.prototype.slice.call(document.querySelectorAll('a')).filter(function (link) {
            var href = link.getAttribute('href');
            if (/^https?:\/\/talakeeper\.org\/tk3\?/.test(href)) {
                // To remove all event handlers on the link,
                // I need to clone the node and replace the node with
                // the clone.
                var new_link = link.cloneNode(true);
                new_link.setAttribute('href', href.replace(/^https?:/, "tala:"));
                new_link.removeAttribute('target');
                link.parentNode.replaceChild(new_link, link);
            }
        });
    })();
}
</script>

[1]: http://talakeeper.org/tk3?bpm=70&pat=lrrr%7Clrrr%7Clrrr%7Clrrr%7Clrrr%7Clrrr%7Clrrr%7Clrrr%7Clrrr&kal=_%7C_%7C_%7C_%7C_%7C_%7C_%7C_%7C_&nad=_%7C__%7C___%7C____%7C_,_,_%7C_,__,_%7C_,_,_,_%7C___,___,%7C_,_,_,_,_&gat=_._%7C_._%7C_._%7C_._%7C_._%7C_._%7C_._%7C_._%7C_._&name=Nine%20Nadais
[p1]: http://talakeeper.org/tk3?bpm=70&pat=lrrr&c=n
[p2]: http://talakeeper.org/tk3?bpm=70&pat=lrrr&nad=__&c=n
[p12]: http://talakeeper.org/tk3?bpm=70&pat=lrrr%7Clrrr&nad=_%7C__&c=n


