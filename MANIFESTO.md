# A robot's face is not a UI

### Or: why every "expressive" desktop robot reads as dead within ninety seconds

---

There is a genre of video that has been made about a thousand times. A small robot on a desk.
It opens its eyes. Someone waves. The eyes turn into two happy arcs. A heart floats up the
screen. The caption says *"it's alive 🥺"*.

It is not alive, and it is not even close. Here is the tell: **you can find the frames.**

Watch it for a minute and your visual system starts resolving the individual PNGs. The
expression does not change, it *swaps*. The blink is on a timer, so it blinks while it is
"listening intently", which no creature does. The emotion is a label chosen by the app, so
when you interrupt it mid-smile the smile does not break — it cross-fades into a different
smile. There is no *there* there.

We are not sneering at hobbyists. We are sneering at the industry, which has decided that
"emotional robot" means "a folder of images and a state machine", because that is what
shipped first and what is cheapest to commission from a design agency.

The problem is that this choice is **architecturally irreversible**. Once your face is an
atlas, every emotion you did not draw does not exist, every interaction that would need a
partially-formed expression is impossible, and no amount of adding frames fixes it. You have
built a slideshow and called it a nervous system.

---

## Five axes, because "aliveness" is not a feeling

This is the part that makes people uncomfortable: if you cannot measure it, you are not
building it, you are decorating. We hold that a companion's aliveness is a set of
**measurable** properties, and that each has a falsifiable test.

### 1. Continuity

**Claim.** Between any two of its expressions, the face passes through states that are
themselves meaningful.

**Test.** Sample the face at 10 ms intervals during an emotional transition. Every
intermediate frame must be independently classifiable by a naive observer. If any frame reads
as "two eyes faded together", you have an atlas.

### 2. Reactivity

**Claim.** Its expression is a function of events, with latency small enough to read as
reflex rather than response.

**Test.** Touch it. The change must begin within 100 ms and continue *past* the stimulus, on
an exponential decay. A robot that snaps back to neutral when the hand is removed is
rendering a button, not a feeling.

### 3. Expectation

**Claim.** It has states that are *about* something that has not happened yet.

**Test.** Wait for it to initiate. Do not respond. It must register a miss, change
observable internal state, and become more likely to initiate again sooner. This is the
single sharpest line between a companion and an appliance, and almost nothing on the market
crosses it.

### 4. Physiology over time

**Claim.** The same expression means different things at different times, because it is
modulated by a slow internal state.

**Test.** Reproduce the identical stimulus on day 1 and day 40. The response must differ in a
direction that is consistent with accumulated history. This is where memory stops being a
feature list and becomes a *body*.

### 5. Legible inner state

**Claim.** The face is not just an output; it is a readout of a continuous internal state
that can be queried and must be honest.

**Test.** Ask the system for its internal coordinates. The number should predict the face.
If the face is chosen by a lookup and the coordinates are computed separately, they will
disagree under stress — and disagreement is what observers unconsciously detect as "fake".

---

## The uncomfortable consequence

If you accept those five axes, then **the face is the least important part.** Geometry is
the cheapest, most copyable layer of the whole system. We are publishing ours precisely
because a competent graphics engineer reproduces it in two weeks.

What cannot be reproduced in two weeks is **continuity over time**: the same face meaning
something different on day 40 because it now knows your Tuesday. That lives in the mind, not
the body, and it is where the actual research problem is.

So: this repository is the body. It is open because the body is not the moat. The mind is
private because it is not finished, and because someone has to be first.

---

## What we are actually claiming

Not that we have built a conscious machine. We have not, and the word "conscious" should not
appear in a product page for anything on the market.

We are claiming something narrower and harder to dismiss:

> **A companion device that visibly wants something, and visibly fails to get it, is
> experienced as more alive than one that merely responds correctly.**

That is testable. It is falsifiable. And if it holds, then an enormous amount of current
product design is optimising the wrong variable — chasing expression *count* and animation
*smoothness*, when the thing that actually lands is a six-second window of hope followed by a
small, legible disappointment.

Forty lines of code. That is the whole trick.

---

## To the people building this

If you are working on the mind and you need a body that answers to it, the protocol is
documented and the field notes are complete. Open an issue.

If you are building a face and you take one thing from this repository, take the **dirty-band
blit** and take the **dead zone on the blush**. Then ask yourself, seriously, whether your
robot has ever wanted anything.

---

*MindChan is a research instrument from the MAC-Lab affective computing group. It is not a
product, it is not for sale, and it has no language model inside it — deliberately.*
