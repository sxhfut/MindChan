# Wiring an affect model to a body it was never written for

Our upstream "mind base" (MindOS) emits affect in a form designed for a humanoid: FACS action
units for the face, and action commands like `shoulders_drop`, `hand_fidget`, `chin_touch` for
the body. StackChan has a 2-DOF gimbal, two eyes, a mouth, a ring of white LEDs, and one touch
point. No shoulders. No hands. No chin.

This document is about that gap, because **the gap is where the honesty lives**. Anyone can
plot a PAD vector. The question is what you do when the model asks for a shoulder you don't have.

---

## 1. Three ways to land an action — and the one you must never take

| field | way | what actually happens |
|---|---|---|
| `gaze_down` | **direct** | pitch drops. The machine really has this. |
| `head_shake_slight` | **direct** | yaw sine, 520 ms. Shaking your head is not a metaphor here. |
| `shoulders_drop` | **degraded** | no shoulders → pitch sinks + breathing goes shallow and slow + LEDs dim to 0.55 |
| `hand_fidget` | **degraded** | no hands → low-frequency tremor, 0.18 amplitude. "Restless", not "shaking" |
| `weight_shift` | **degraded** | no centre of mass → gaze drifts + turn speed drops |
| `slow_exhale` | **degraded** | no lungs → one deep slow bob + slight sink + LEDs a bit darker |
| `chin_touch` | **impossible** | no hands, no chin |
| `open_gesture`, `hand_gesture_*` | **impossible** | no hands |

The fourth option — **silently dropping what you can't do** — is the one to avoid, and it is the
default outcome of any mapping layer you write quickly. The result is not a crash. The result is
"the emotion feels like it's missing something", which you will then chase in the emotion model,
the PAD calibration, the FACS weights, for a week.

So the five impossible actions are *registered* as impossible, with a reason, and printed at
runtime:

```
[做不到] chin_touch(无手无下巴)
```

Five actions it refuses to fake. That line is the most useful line in the log.

Coverage is checked by machine, not by eye — every `action_type` the upstream module can emit
must fall into exactly one of the three buckets, and a new upstream action that we haven't
handled shows up as `missing` instead of vanishing:

```
MindOS 动作类型 33 个
  已落点 32  不可表达 ['chin_touch','open_gesture','hand_gesture_stop','hand_gesture_wave','hand_gesture_open']
  未登记(会静默消失): []      ← this must be empty
```

That check caught `slow_exhale` on the first run. It had been in the upstream `lengthening`
group the whole time and we had mapped the rest of that group. Without the check it would have
been invisible forever.

---

## 2. The face is 20 continuous numbers, not a name

The device has 71 named expressions (`emo:<name>`) — the same shape as a sticker sheet. Feeding
FACS into that means compressing a continuous space into 71 points, and everything between the
points is lost. So the firmware got a direct channel:

```
face:<hold_ms>:hw=21.0,hu=46.0,hd=50.2,lid=0.00,tilt=0.00,arc=0.63,arcc=11.2,...
```

Twenty parameters, all continuous: eye width/height, lid, tilt, lid-arc and its curvature and
radius, inter-eye distance, eye height offset, asymmetry, mouth width/curve/open/asym/press, and
four physiological channels (blush / sweat / tear / pale). Two more (`grille`, `mshift`) exist in
the engine but are unused — they are zero across all 71 presets, so inventing coefficients for
them would be inventing a degree of freedom.

**The value ranges are measured, not guessed.** A script walked all 71 presets and produced
min/max/median per parameter. `hu` lives in 14–70, `mcurve` in −10–14, `arct` around 11°,
`edw` at 62 px. The bridge computes in *the engine's own units* and never normalises again —
two rulers stacked on top of each other is how you spend a week tuning.

Then intersect with what the firmware will actually accept:

```python
LIMIT = {"mw": (0.0, 1.0, 0.6), ...}   # measured across 71 presets
FW    = {"mw": (0.2, 1.4), ...}        # what faceSetField() clamps to
```

They are not the same. `mw` reaches 0 in the presets but the firmware floors it at 0.2; adding
15 % headroom to `arcc` reaches 31 against a firmware ceiling of 30. If you only clamp to
`LIMIT`, what you *sent* and what it *draws* differ, and the readback verification produces
false failures while your eyes report "looks slightly off".

### The 233-byte bug

The UDP receive buffer was `char buf[192]`. A full 20-parameter `face:` packet is **233 bytes**.
The command arrived, was truncated at the last field, was reported as `n=19`, and the face
changed — except one dimension, which stayed at its default forever.

Nothing about that looks like a bug. The eyes moved. The log said OK. In a 20-dimensional space
you cannot see one wrong dimension; you just think the face is "slightly not right" and go
re-tune the coefficients. It only surfaced because the readback check compares field by field:

```
回读逐字段一致（20 个参数错一维肉眼看不出来）   [OK]
```

Buffer is now 512, and an over-long datagram is reported as `truncated` and **dropped** — better
to not move than to move half-way.

---

## 3. Dominance was identically zero

The device reports a PAD triple. Valence moves. Arousal moves. Dominance is always exactly 0.00 —
the firmware has a decay term for it and no driver.

Upstream, dominance is what separates five emotions: fear needs `d < −0.2`, anger `d > 0`,
contempt `d > 0.4`, disgust `d > 0.1`. With `d ≡ 0`, **this machine can never be afraid, angry,
or contemptuous.** Not "rarely" — never. And it doesn't crash, doesn't warn; you just get a robot
with a permanently narrow emotional range and no idea why.

We gave `d` an auditable source instead of a magic number:

```
d ≈ 0.45·bond  − 0.55·loneliness  + 0.25·approach  − 0.18·avert
```

Familiar → has ground to stand on. Left alone → cowed. Someone leans in → forward. Someone looks
away → withdraws. It is a *definition*, not a measurement, and it is written down so it can be
argued with.

Result, four situations, four different faces:

| situation | P / A / D | reads as | mouth | eye arc | lid | skin |
|---|---|---|---|---|---|---|
| left alone | −0.45 / −0.06 / −0.10 | sadness | −3.3 | 0.00 | 0.16 | pale 0.27 |
| challenged | −0.55 / +0.72 / −0.21 | fear | −1.8 | 0.00 | 0.07 | pale 0.33 |
| frayed, low battery | −0.40 / +0.52 / −0.15 | anxiety | −0.9 | 0.00 | 0.06 | pale 0.24 |
| excited | +0.55 / +0.92 / +0.53 | joy | +9.4 | 0.82 | 0.00 | sweat 0.27 |

---

## 4. A pause on screen is not a pause in the air

The upstream verbal-realism module turns written text into speech with disfluencies, and marks
pauses inline: `这个[pause:500ms]我觉得`. If you print that to a terminal, you've added
characters. If you feed it to the TTS as-is, the TTS reads it out.

So `[pause:500ms]` becomes macOS `say`'s `[[slnc 500]]`: real silence in the waveform. Then the
mouth — which is driven by the real audio envelope at 25 Hz — closes by itself, because there
really is nothing to open for.

And then the timing of the body follows the *audio*, not an estimate. Upstream produces a
timeline assuming ~200 ms per Chinese character. We ignore that for anything timed and instead
find zero-runs in the measured envelope:

```python
def find_pauses(env, frame_s=0.04, min_ms=280):
    # runs of |x| ≈ 0 longer than 280 ms are "it's thinking"
    # shorter ones are stop-consonant closures — putting an action there looks like a twitch
```

Actions are then hung on the starts of those runs. The gaze only drifts away **while there is
actually no sound**. That is the difference between "thinking" and "looking away mid-sentence",
and you can hear it.

Budget note: this link does ~27 packets/second total, and the 25 Hz lip envelope eats most of it.
So a whole utterance gets **at most two face packets** — one opening expression, one optional
change during the first real pause — and the rest goes to the body channel.

---

## 5. Memory: events, not embeddings

"Memory" is easy to fake with a vector index and a similarity score. It is hard to fake the thing
that actually matters: **being able to say which time.** A cosine similarity cannot produce
"four hours ago you said the 532 nm laser dropped 30 %".

So the store is a JSONL file of events — who, when, the actual sentence, extracted keywords,
emotional salience, whether it was left unfinished. Retrieval is three explainable scores
(recency decay / salience / lexical overlap), and every recall carries its own justification:

```
对了，4 小时前你说激光器 532 纳米掉了 30%，后来弄好了吗。
  [why] 未完结 + 分 0.65（4 小时前，都提到「532/30%/激光器/纳米」，当时说得挺重，这事当时还没完）
```

The hard rule is a **gate, not a weight**: if lexical overlap is zero, the score is capped below
every threshold. Recency and emotional intensity can never rescue an unrelated memory. Without
that gate you get a machine that answers "what time is the meeting" with "you said your laser was
broken" — which is not remembering, it is pestering.

And an empty store returns `None`. Not a template. A template that says "I remember you said…"
when it doesn't is the single most damaging thing this class of system can do, because it is
indistinguishable from working.

---

## What this bought, concretely

- 20 continuous face parameters per utterance instead of one of 71 names
- 33/33 upstream action types accounted for; 5 declared impossible instead of faked
- hesitations/stutters that scale with cognitive load: 30 samples each, high-load state produced
  11 hesitations / 20 pauses / 12 stutters vs 5 / 7 / **0**
- four distinguishable emotional faces from four situations
- one real recall with a real timestamp and a real reason

None of that is a demo trick. It is a mapping layer that refuses to lie about what it cannot do.
