# The face as a language

Why this is a continuous field and not a folder of PNGs.

## The problem with the obvious design

The obvious design is an atlas: draw 40 expressions, index them, cross-fade between them,
blink on a timer. It ships in a weekend and it dies in ninety seconds — because a viewer's
eye is extremely good at finding the frames. Once you have seen the seams, you cannot unsee
them, and the thing that was supposed to feel alive reads as a slideshow.

The deeper problem is that **the interesting expressions live between the ones you drew.**
A person who is 60 % relieved and 40 % embarrassed has a face. It is not on your atlas, and
it is not a cross-fade of two atlas entries either, because cross-fading two eye shapes gives
you two faint eye shapes, not one intermediate eye.

## The primitive

One shape does almost all of the work: a **stadium** ("lens") — a rectangle with semicircular
caps.

Each eye is a lens with independently controlled:

| Parameter | Meaning | Range (design units) |
|---|---|---|
| `hu` | upper cap height | 0 … 60 |
| `hw` | half-width | 10 … 46 |
| `lid` | upper lid coverage | 0 … 1 |
| `tilt` | mirror-capable rotation | −0.9 … 0.9 |
| `eyeY` | vertical placement | ±12 |

and each mouth is an **arc** — a quadratic Bézier stroke with live sag, width, openness and
vertical placement.

That is it. Everything expressive comes from moving those numbers.

## Two facts that save you a week

### A quadratic Bézier at t = 0.5 is not the midpoint

For control points `p0, p1, p2`, the curve at `t = 0.5` is

```
B(0.5) = (p0 + 2·p1 + p2) / 4
```

So if you want a visual sag of `s` below the chord, your control-point offset must be
**`2s`**, not `s`. Every "my smile is half as deep as it should be" bug is this line.

### Lens → arc must be geometric, not a fade

When a lid closes completely, an eye becomes a line. When a mouth smiles, an arc bows. If you
transition between them by fading opacity, you get:

- white at 50 % over black = **grey**
- grey over the white that is still there = **a double image**

The transition has to be geometric: the lens collapses to zero vertical extent *while* the
arc grows, and the vertical collapse must follow

```
extent ∝ (1 − arc)²
```

not `(1 − arc)`. Linear collapse leaves a visible fat line at the crossover point; squared
collapse is the curve that reads as "the eye folded into a line".

## The physiological layer is orthogonal

Blush, sweat, tear and pallor are **not expressions**. They are a separate layer painted
behind or over the face, driven by their own semantics:

| Channel | Driven by | Semantics |
|---|---|---|
| `blush` | event pulse, τ ≈ 2.6 s | *something just happened* |
| `sweat` | sustained arousal | *this is costing me* |
| `tear` | sustained negative valence | *this hurts* |
| `pale` | low power / startle | *I am not at full capacity* |

Keeping them orthogonal is what lets "happy but sweating" exist as a state without anyone
authoring it.

The blush in particular taught us a lesson worth writing down: it used to be driven by
*closeness*, i.e. a slow state. It looked like a skin condition. The fix was not a smaller
coefficient — it was recognising we had used a state where an event belonged. On a robot, a
flush has to mean *something just happened*. See [bug #7](02-the-seven-bugs.md).

## What 71 named expressions are for

They are **targets**, not assets. Each one is a set of shape parameters plus a PAD reading
inferred from it, so the system can:

- interpolate between any two of them without a transition animation
- let an emotional state bias the *current* target instead of replacing it
- report a PAD estimate upward, so the face is also an output device for internal state

Naming them matters for a different reason: a name is a contract with whoever writes the
behaviour layer. `relieved` has to look like relief to someone who did not design it, or the
whole thing is a private vocabulary that only its author can drive.

## The three-colour constraint

Pure black background, pure white for eyes and mouth, and exactly **one** accent — the blush.
That is the whole palette.

This is not minimalism for its own sake. A tiny, bright, high-contrast face reads from across
a room; a pastel gradient face reads as a phone screen. And a single accent colour means the
accent can carry meaning: if red appears, something happened.

The constraint is also why [bug #7](02-the-seven-bugs.md) was so painful. On pure black, in
RGB565, there are about seven distinguishable steps between black and a warm blush. You are
drawing a gradient with seven crayons.
