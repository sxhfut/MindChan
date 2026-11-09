# Hearing, and the difference between a face and a head

We shipped a claim for two days: *when you speak to it, it looks up at you.*

It was in the code review. It was in the firmware comment. It was in the test
suite, green. And it had never once happened.

---

## 1. Three layers, all wrong in the same direction

The firmware said:

```cpp
/* ── 听觉反应 ─────────────────────────────────────────────
 *   有人开口   → 抬头看过去 + 进 Listening
 */
```

The implementation said:

```cpp
if (v && !earVoicePrev) {
    if (gTurn == Turn::Idle) {
        gTurn     = Turn::Listening;
        g.gazeX   = 0.0f;
        g.gazeY   = 0.0f;
        setEmotion("attentive", 2200);
    }
}
```

The test said:

```python
check("表情转为 attentive（听得见→看得出来）", saw_attn is not None)
```

Head pitch is computed from a separate accumulator:

```cpp
int upOff = (int)(g.lookUp * 150.0f);   // g.lookUp is driven by the CAMERA
```

So the face changed, the eyes recentred, and the gimbal did not move a
millimetre. The comment described an intent, the code implemented a subset, and
the test asserted a different subset than the one the comment described. Each
layer was internally consistent. The union of the three was a lie.

**This is the most dangerous shape of bug we have hit on this project**, more
than any of the hardware ones, because there is no failing signal anywhere. The
robot looks *attentive*. It just doesn't look *at you*.

The fix was to make the test assert the physical claim instead of the proxy:

```python
check("听见后头抬起来了 (pitch 均值 +20 以上)", up - pitch0 >= 20, ...)
check("抬头不是相机造成的 (fa 全程 0)", face_during == 0, ...)
check("声停后头放回原位 (pitch 回落到静场 ±25)", abs(pitch_end - pitch0) <= 25, ...)
```

That middle check matters. A raised head is ambiguous — the camera's proximity
term raises it too. Forbidding any face detection for the duration of the test is
what makes the result attributable to *hearing* and nothing else. **If your
assertion cannot be attributed to the mechanism you claim, it is not testing the
mechanism.**

---

## 2. What "looking toward a sound" honestly means on a 2-DOF gimbal

We have no direction of arrival. The two microphones turned out to be one
microphone read twice (see `docs/05-perception-bugs.md`), and DOA has not been
re-measured since. So we cannot turn toward the source.

What we do instead is an **orienting response**, not a localisation:

| phase | motion | duration |
|---|---|---|
| rise | pitch +130 raw (~40°) | 240 ms |
| scan | yaw sweeping ±105 raw at 0.62 Hz | 2.4 s |
| fall | pitch back to base | 1.6 s |

That is what a person does when a sound arrives from an unknown direction: head
up, eyes searching, then settle. A fabricated precise turn would be worse than
nothing — it would be a claim the hardware cannot back, visible to anyone who
walks around the desk and watches it track nothing.

The cost, stated plainly: **the head moves 0.5–0.7 s after the sound starts.**
The microphone records in 512 ms blocks, because the alternative — re-arming a
short buffer every 150 ms — produces a noise floor that sawtooths between 3500
and 11000 in a silent room (see `ear.h` for that whole story). The buffering is
the price of a trustworthy level estimate, and it shows up as latency.

---

## 3. The longer you talk, the earlier it stops listening

Found while fixing #1, and worse than #1.

```cpp
baseRms += (rms - baseRms) * 0.06f;   // noise floor, once per 128 ms sub-block
float thr = fmaxf(baseRms * 1.6f, 180.f);
last.voice = (rms > thr) && ...;
```

The noise floor is a running average — and it was updating *while you were
speaking*. At 6% per 128 ms the baseline reaches 75% of the speech level in two
seconds. After two seconds of continuous talking, `rms > 1.6 × base` becomes
false, the block is classified as quiet, six quiet blocks accumulate, and the
device fires `ear_end` and hands the turn back **while you are mid-sentence**.

Measured consequence: a 3-second utterance would be "heard" for 2 seconds. A
10-second explanation would be heard for 2 seconds and then ignored.

Worse, it corrupts the test that was supposed to catch it: `peak / quiet_median
≥ 1.6×` shrinks as the denominator climbs.

Fix — freeze the floor while the signal is hot, keep a slow leak so a genuinely
noisy room still adapts:

```cpp
float hotThr = fmaxf(baseRms * 1.6f, 180.f);
baseRms += (rms - baseRms) * (rms > hotThr ? 0.004f : 0.06f);
```

Zero point zero zero four is a 30-second time constant. Long enough that a
sentence cannot drag the floor up behind it; short enough that a fan switching on
is absorbed within half a minute.

**Generalise this one.** Any "adaptive" threshold that updates during the event
it is trying to detect will eventually learn to ignore the event. The slower the
adaptation, the longer the event can be before it disappears.

---

## 4. `{ }` is valid JSON

A different kind of silence, on the language side.

We constrained the local model's output with `"format": "json"`. It obeyed. It
also, roughly once every fifteen turns, produced:

```
{ }
```

Legal JSON. Zero meaning. The turn falls straight through to the fallback line
*"I glitched for a second, say that again"* — which, in a demo, is the single
worst sentence the machine can say.

The error is in thinking `json` is a *content* constraint. It is a *syntax*
constraint, and empty is syntactically perfect. The fix is a schema:

```python
OUT_SCHEMA = {
    "type": "object",
    "properties": {
        "say": {"type": "string", "minLength": 2},
        "emo": {"type": "string", "enum": EMO_OK},
    },
    "required": ["say", "emo"],
}
```

Now the decoder cannot emit an empty sentence, and cannot emit an emotion the
firmware does not have — both are grammar violations, not probabilities. Same
model, same temperature: five consecutive clean runs, where before it failed one
run in three.

Two lessons we are keeping:

- **Push every claim you can from probability into grammar.** Each one you move
  is a class of failure that stops existing rather than getting rarer.
- **A retry is error correction, not a second draft.** Ours ran at temperature
  0.65 — the same creativity that makes the first answer good makes the
  correction unreliable. Retries now run at 0.2.

---

## 5. The number it invented

Asked *"what was that project number I told you about?"* — with nothing ever
having been said — the 4B replied:

> `是P-319，你忘了？` — *"It's P-319, did you forget?"*

Not a hallucination in the usual sense. It is being *helpful*: the shape of the
question implies an answer exists, so it supplies one, in exactly the format the
real thing would have. A prompt rule ("do not invent") did not stop it, because
rules about truthfulness do not compete well against the instinct to answer.

So it is a gate, not an instruction. Anything in the reply shaped like an
identifier — letters plus digits, or five or more digits — that appears nowhere in
the conversation history or the memory store triggers one re-ask that names the
offending token. If it does it again, the machine says *"you never told me that
one; say it and I'll keep it"* rather than a plausible number.

The test for this is deliberately shaped to be *falsifiable*: ask for something
that was never given, then assert both that no identifier-shaped token appeared
and that the reply either declines or asks. The first version of that test also
failed a perfectly good answer — *"which time? point me at the passage"* — because
it only accepted the words "I don't". **A check that punishes correct behaviour is
worse than a missing check**, so a question mark now counts as passing.

---

## 6. What this device still cannot do

Being precise, because the demos are recorded:

- **No direction of arrival.** It cannot turn toward you. It looks up and scans.
- **No speech recognition.** It hears *presence*, not *words*: level, structure
  (zero-crossing rate), onsets, ends, sudden transients. Everything above that
  happens on the host, and the host currently takes text.
- **Latency floor of ~0.5 s** on the hearing side, from the 512 ms capture block.
- **It does not speak words.** The host's TTS plays through the Mac's speaker; the
  device lip-syncs a 25 Hz envelope. The six things it can actually voice itself
  are a sigh, "mm", "huh?", "oh", "mmm" and a gasp.
