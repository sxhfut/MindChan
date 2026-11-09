# The perception bugs

The [seven bugs](02-the-seven-bugs.md) came from getting the body to run. These four came
later, from getting it to *behave alive* — and they share one shape:

> **The sensor reports a number that looks plausible, so you debug your algorithm instead of
> the sensor.**

A display bug you can see. A link bug shows up as packet loss. A perception bug shows up as
*character*: "it seems uninterested today". That is the expensive part — a plausible number
produces a plausible behaviour, and plausible behaviour is not a symptom.

---

## 1. Static-field RMS sawtooth — the ADC has a DC offset

**Symptom**

With nobody moving and nothing making a sound, the mic level `er` climbs from ~1700 to
~10700 over about fifteen seconds, then collapses back, forever:

```
er: 1741 2260 3018 4102 6044 8110 9930 10740 9680 6120 2903 1877 1741 ...
```

Nothing in the room changes on that timescale. Our first two hypotheses were both wrong:
not mic gain drift, not speaker idle-state crosstalk on the shared I2S port.

**Cause**

The ADC's **DC bias drifts**. And the RMS of a signal with a DC component `D` is at least
`|D|` no matter what the audio does:

```
sqrt( E[(x + D)²] ) = sqrt( E[x²] + 2D·E[x] + D² ) = sqrt( E[x²] + D² )   for E[x] = 0
```

So a slowly drifting offset is *indistinguishable* from a slowly drifting sound level. Every
relative threshold ("louder than 1.6× the floor") misfires depending on where you are in the
sawtooth: sometimes the device hears voices in an empty room, sometimes it is deaf.

**Fix**

Remove the DC before taking the magnitude:

```c
double mean = sum / n;
float  rms  = (float)sqrt(fmax(0.0, acc / n - mean * mean));
```

Static-field level drops from a 1700–10700 sawtooth to a stable ~100. Thresholds stop
needing a floor hack.

**The part that actually hurt**

Every acoustic test that had passed before this was passing *on the offset*. Our
"18/18 acoustic checks" were largely measuring the DC component. Fixing the estimator
invalidated the whole suite and exposed a real defect that the offset had been hiding:
short sounds (`hmm`, `huh`, `oh`) only reach 50–70 % of their designed energy span.

**Lesson** — One wrong estimator can make an entire test suite green for the wrong reason.
When a metric has a floor you cannot explain, suspect the estimator before the world.

---

## 2. Skin-colour centroid thinks a wooden desk is a face

**Symptom**

The robot never takes initiative. It seeks attention once at boot, then waits forever,
patient, because its own state says *a human is sitting right here*:

```
fa: 0.233 0.241 0.236 0.251 0.245 ...     (49 consecutive static frames)
```

**Cause**

Skin-colour centroid. A warm wooden desk under a desk lamp is skin-coloured. There is no hue
window that separates "desk" from "face" — the *feature is insufficient*, not mistuned. We
spent a while moving thresholds before accepting that.

**Fix**

Three hard constraints layered on the centroid, all of which a real face satisfies and a
desk does not:

```c
if (area < 0.010f || area > 0.35f) return out;         // plausible face size
// bounding box must not touch the frame edge — a face in view is *inside* the frame
if (bx0 <= mw || by0 <= mh || bx1 >= W-1-mw || by1 >= H-1-mh) return out;
if (aspect < 0.45f || aspect > 2.20f) return out;      // not a wide slab
if ((float)n*(step*step)/(bw*bh) < 0.28f) return out;  // fill ratio: a desk fills its box,
                                                       // a face with eyes and hair does not
```

Result: 49/49 static frames correctly report "no one". We also added entering/exiting
hysteresis (3 frames in, 5 frames out) so a single bad frame cannot flip the state.

**Lesson** — When a false positive makes a *feature disappear* (here: autonomy vanishes
because the device believes you are always present), the bug hides behind the feature you
are proudest of. Test the negative case on the real desk, not on a blank wall.

---

## 3. The in-flight capture buffer makes the device scare itself

**Symptom**

Two failures that look unrelated:

1. Pet its head and it flinches — `startled` instead of `relieved`.
2. "How long did the interrupted sound play before it stopped?" measured as *nothing*.

**Cause**

An ordering bug in the capture loop:

```c
if (recOn) { ...analyze(cap)... return; }   // ← runs first
if (busy && gate) return;                   // ← the "deaf while speaking" guard
```

The guard only prevents the **next** capture. A buffer of 512 ms that *started before* the
device spoke and *finished after* it stopped is still analyzed — and it is full of the
device's own voice, because in-body pickup is 100–400× the noise floor:

```
{"ev":"ear_bang","rms":13983}
```

The chain, and this is the whole reason two symptoms appeared:

```
device sighs → in-flight block analyzed → "bang" → startled + a gasp
             → the gasp takes the vocal slot
             → voc_end of the just-interrupted sigh is never emitted
```

Neither symptom points at the microphone. One says "the emotion path is wrong", the other
says "the event is missing".

**Fix**

Three layers, because one is not enough:

```c
if (busy && gate) { if (recOn) discardRec = true; quietUntil = millis() + 260; return; }
...
if (recOn) { ...; if (discardRec) { discardRec = false; selfDiscards++; return; } ... }
if ((int32_t)(millis() - quietUntil) < 0) return;   // signed diff: wrap-safe
```

Plus a fourth, at the reaction site: do not emit a replacement gasp within 400 ms of any
sound starting — otherwise the gasp itself steals the event you were trying to measure.

Cost: 260 ms of deafness after the device speaks. It cannot hear you while it is talking
anyway, so this is free.

**Lesson** — "Skip while busy" is not the same as "this sample is invalid". Sampling happens
over a **window**, so the guard has to be checked against the window, not the instant.

---

## 4. A small model ignores a table of numbers

This one is on the host side, but identical in shape: the input looks informative and the
output looks responsive.

**Symptom**

The robot has persistent state — familiarity, loneliness, time since last seen — and we want
its *wording* to change with it. So the state goes into the system prompt:

```
we have met 1 time, familiarity 0.00 (0 = stranger, 1 = close)
loneliness 0.00 (higher = feels ignored, tone gets flatter and shorter)
time since we last met: unknown (clock not synced, do not guess)
```

Six turn dialogue reads fine. Then we tested it properly: same input, two extreme memory
states. **Two of three probes came back identical** (`"嗯？"` for both a stranger and a close
friend who had been ignored all night).

**Cause**

A 4B-class model does not do implicit conditioning on a table of floats. It follows
imperatives. A number is *context*; an instruction is a *constraint*.

**Fix**

Translate each reading into a behaviour directive, keep the number for auditability:

| Reading | Becomes |
|---|---|
| `bond < 0.25` | barely know them. Use plain "you", do not act familiar, do not bring up last time |
| `bond > 0.60` | know them well. Can be blunt, can tease, can pick up an old thread |
| `lon > 0.50` | ignored for a long time. Show a little displeasure first, keep it shorter, two or three characters is fine |
| `gap > 6 h` | overnight. Allowed one "it's been a while", do not complain |

Three of three probes now change with state, and the difference is legible:
`"嗯？来干嘛？"` (stranger) vs `"好久不见"` (close, and it noticed the gap).

**How to test this without fooling yourself**

At temperature 0.75 two runs *always* differ. That proves the sampler is stochastic, not
that memory reached the wording. The test must be:

```
pin temperature to 0
A1 = fresh instance, state A   ┐ must be equal  → determinism holds
A2 = fresh instance, state A   ┘
B  = fresh instance, state B     must differ from A1 → difference is attributable

(one instance reused twice differs only because its history grew — not because of state)
```

If `A1 != A2`, do not judge the causal claim — record that it could not be measured. A test
that cannot fail is worse than no test.

**Lesson** — Small models execute instructions and ignore context. And: if you cannot
separate "it responded to the state" from "it sampled differently", you have not tested
anything yet.

---

## If you hit one of these

Open an issue with the exact numbers. All four of these were found by noticing that a figure
was *stable when it should have moved*, or *moving when nothing moved* — a trace of the raw
value is worth more than a description of the behaviour.
