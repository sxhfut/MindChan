# The seven bugs

Seven things that cost us hours, none of which were written down anywhere. Every one is
reproducible on a stock StackChan (CoreS3) with the official BSP.

Written down because the failure modes were **misleading**: in five of the seven, the
symptom pointed at a completely different subsystem than the cause.

---

## 1. The camera will not init on I2C port 0

**Symptom**

```
{"ev":"cam_try_fail","port":0,"err":262}
sccb init err
ESP_ERR_INVALID_STATE (0x103)
```

Port 1 then works on the same board with the same pins, which sends you looking for a
hardware fault that does not exist.

**Cause**

The GC0308 driver will create a *second* I2C bus for SCCB if you hand it an SDA pin. That
bus collides with the one `M5Unified` already owns. Passing `pin_sccb_sda = 12` does not
fix it — it just makes it fail more slowly, which reads as "almost working".

**Fix**

```c
cfg.pin_sccb_sda  = -1;     // reuse the bus M5Unified already opened
cfg.pin_sccb_clk  = -1;
cfg.sccb_i2c_port = 1;      // 0 fails, 1 works
cfg.fb_count      = 1;
```

**Lesson** — When a driver offers to "help" you by creating a bus, check whether one
already exists. Two buses on the same pins fail in a way that looks like a wiring problem.

---

## 2. `Serial.write()` blocks for 2.5 seconds

This one is the worst of the seven, because it silently destroys the *whole* main loop, not
just the serial port.

**Symptom**

Command latency of 10–15 seconds. Events arriving out of order, or long after they
happened. And a telemetry field that reads `lpm: 4050` — which we misread for a week as
4 milliseconds because the firmware names the variable `perfLoopMaxUs` while actually
storing milliseconds.

**Cause**

USB-CDC `Serial.write()` blocks when the TX buffer is full. If nothing is draining the port,
that block lasts seconds. During the block, nothing else runs: no UDP receive, no render, no
turn state machine.

Two aggravating factors:

- **Testing with `cat` on the port masks it completely** — which is exactly what we were
  doing while debugging. The bug only appears when nobody is reading.
- **`Serial.availableForWrite()` over-reports free space.** Guarding with
  `if (availableForWrite() >= n)` is not sufficient; bytes still get written into a buffer
  that will back up.

**Fix**

```c
// in setup(), before anything else
Serial.setTxTimeoutMs(0);   // drop on overflow, never block
```

Keep the size guard as a courtesy, but the timeout is what actually saves you.

**Measured**

| Serial port state | Sent | Replies | First reply |
|---|---|---|---|
| unread | 78 | **3** | 2.02 s |
| being read | 97 | **97** | 0.05 s |

**Lesson** — On an embedded target, any blocking write in the main loop is a whole-system
bug. The debug channel must never be able to stall the product.

---

## 3. PSRAM is 8× slower than internal SRAM for pixel writes

**Symptom**

20 fps with sprite buffers allocated from PSRAM; 54 fps with identical code and buffers in
internal SRAM. `dms 11.3 → 1.7 ms`, `bms 13.4 → 9.0 ms`.

**Cause**

Per-pixel 16-bit writes to PSRAM on the ESP32-S3 go through a cache and are the bottleneck.
Shrinking the sprite barely helps, because the cost is per write, not per byte.

**Fix**

`setBuffer()` onto static internal RAM, bypassing the allocator entirely.

**The trap inside the trap**

`setPsram(false)` does **not** mean "use internal RAM". It selects
`AllocationSource::Dma`, which needs one large contiguous block — and after WiFi + camera
init there isn't one. `createSprite()` returns `nullptr` and nothing logs an error. You
then render into a null sprite and wonder why the screen is black.

Always check the return value of every `createSprite()`, and log the band sizes at boot:

```
{"ev":"disp","w":320,"h":240,"rot":0,"sprite":1,
 "bands":[1,1,1],"blen":[31872,31872,7072],
 "heap":203416,"psram":8381300}
```

**Lesson** — "disable PSRAM" is not the same instruction as "use fast memory".

---

## 4. Flicker is a structural problem, not a timing problem

**Symptom**

The face visibly tears. Every instinct says frame rate, so you raise it, and the tearing
gets *faster* without going away.

**Cause**

Clear-then-draw. The panel is shown the state after `fillRect()` and before the shape is
drawn. No amount of frames-per-second fixes a two-step update of a one-step surface.

**Fix**

Per-region off-screen buffers, blitted **once per region per frame**. Our three bands are
left eye, right eye, mouth — 31,872 / 31,872 / 7,072 pixels. The panel never sees an
intermediate state, so there is nothing to tear.

**Consequence worth designing around**

Once you have dirty bands, the blush cannot be a separate overlay rectangle: it would
re-cut the bottom of the eye. It has to be painted **into** the eye band, *before* the eye
is drawn, so the eye occludes it naturally. That is a design decision forced by the
rendering architecture, and it is the correct one — blush belongs to the cheek, the cheek is
behind the eye.

**Lesson** — If you can see intermediate states, your architecture is wrong, not your clock.

---

## 5. The device answered to whoever spoke last

**Symptom**

"The device is online but its events are missing." State queries worked perfectly. Only
unsolicited events vanished — and only sometimes.

**Cause**

The device stored the **last** packet source as its reply address. During testing there were
two host sockets: a control console (its own ephemeral port, e.g. 53533) and a script bound
to 20087. They alternated, so `host_bound` flip-flopped:

```
{"ev":"host_bound","port":53533}
{"ev":"host_bound","port":20087}
{"ev":"host_bound","port":53533}
```

Query replies were unaffected — they answer the requester, by construction. So the only
thing that broke was the thing that has no requester: the asynchronous event.

**Fix**

Keep a small ring of recent endpoints (`kSinkMax = 3`, 60 s TTL, LRU eviction) and fan every
event out to all active sinks. Cost is three tiny UDP sends per event.

**Lesson** — "Reply to the last sender" is fine for request/response and wrong for
everything else. Once a device has events of its own, it needs **subscribers**, not a peer.

---

## 6. One packet per loop is a throughput ceiling of 27 pkt/s

**Symptom**

After it finished speaking, its mouth stayed open. Interrupts were ignored. The device was
responsive to queries but slow to act.

**Cause**

```c
int sz = udp.parsePacket();
if (sz <= 0) return;        // exactly one datagram per loop iteration
```

The main loop is gated at 26 ms for rendering, so the entire device could receive
**~27 datagrams per second**. A speech-envelope stream is 25 Hz by itself. Add a 5 Hz
console poll and you are over capacity, permanently: `end` and `bargein` sit in the queue
behind dozens of `v:` packets.

A second contributing bug, self-inflicted: we added a one-line "command receipt" event for
every command received. That included the 25 Hz envelope, fanning it out to 3 sinks —
**75 extra outbound packets per second**, saturating the link so `end` could not get in.

**Fix**

```c
for (int i = 0; i < 8; i++) {          // batch-drain
    pollIncoming();
    if (!pollHadPacket) break;         // queue empty, stop calling parsePacket
}
```

And: never echo high-rate telemetry. Receipts are for discrete commands only.

**Lesson** — A per-loop read of 1 is a rate limit you didn't know you had. Compute your
packet budget before you design a 25 Hz protocol.

---

## 7. Quantising alpha gave us concentric rings

**Symptom**

A blush that should fade smoothly rendered as **seven hard concentric rings**. It looked
like someone had drawn circles with a compass.

**Cause**

Three quantisations stacked:

1. `alpha >> 3` collapsed 256 levels into 32.
2. RGB565 then collapsed those into ~7 distinguishable steps between black `(0,0,0)` and
   the blush colour `(216, 92, 62)`.
3. On a pure black background there is almost no red headroom to begin with, so the top of
   the range is where it hurts most.

**Fix**

Keep alpha continuous (0–255), then dither with a 4×4 Bayer matrix:

```c
const uint8_t kBayer[16] = {0,8,2,10, 12,4,14,6, 3,11,1,9, 15,7,13,5};
float f    = powf(1.0f - d, 1.8f);                       // radial falloff
float jit  = (kBayer[(y & 3) * 4 + (x & 3)] / 16.0f - 0.5f)
             * 0.085f * f;                                // proportional to intensity
uint8_t a  = clampf(f + jit, 0.0f, 1.0f) * 255.0f;
```

The proportionality matters: **fixed-amplitude dither turns a soft edge into grey sand.**
Noise must scale with the thing it is hiding.

**The related semantic bug**

The blush also used to be on all the time, which looked like a skin condition. The real
mistake was deeper than a coefficient: we had treated a **state** ("closeness") as if it could
drive an **event** ("a flush"). A blush on a robot has to mean *something just happened*.
Now it is a pulse with τ ≈ 2.6 s, with only 15 % residual from the current expression and a
hard dead zone below 0.14 — below that, it is zero, not faint.

**Lesson** — When you quantise a continuous quantity for display, count your remaining
levels *at the colour you are quantising to*. And keep events and states in separate lanes.

---

## If you are reading this because you hit one of them

Open an issue with the exact symptom. Six of these seven were found by noticing that the
symptom and the cause lived in different subsystems, and a symptom log is the only way that
pattern becomes visible to anyone else.
