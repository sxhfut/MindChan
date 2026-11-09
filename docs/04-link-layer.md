# Link layer — surviving WiFi

The device is a body. The mind runs on a host. Between them is WiFi, which means the link
must be designed for loss from the first line, not hardened after it breaks.

## Two channels with deliberately opposite shapes

| | State | Commands / Events |
|---|---|---|
| Direction | device → host, on request | both ways, unsolicited |
| Rate | 10 Hz | bursty, 0–25 Hz |
| Loss tolerance | total — any packet may vanish | none — must be confirmed |
| Idempotent | yes | commands yes, events no |

Most designs merge these into one "message bus" and then discover they need different
retry policies, different ordering guarantees and different queue depths. Keeping them
separate from the start costs nothing and removes a whole class of bug.

## State is polled, never pushed

The intuitive design is: device pushes state at 10 Hz to a known host. It does not survive
contact with reality, because "a known host" is the entire problem — hosts restart, change
ports, and there are usually two of them (a console and a script) at the same time.

So the direction is inverted:

```
host  → "q"                     (whenever it wants a snapshot)
device → full state JSON         (to the requester, by construction)
```

The host gets a snapshot at exactly the rate it asks for, and there is no such thing as a
stale subscriber list for state. The device never has to guess who is listening.

This is the single decision that made the link stable.

## Events need subscribers, not a peer

Events are the opposite problem: the device has something to say and nobody asked.

The naive implementation — remember the last sender, reply to them — is correct for requests
and **wrong for events**. With two host sockets alternating, the device flip-flops its
destination and events go to whichever socket happened to speak most recently. Queries keep
working perfectly, which is what makes this so hard to diagnose.

See [bug #5](02-the-seven-bugs.md). The fix is a small subscriber ring:

```
kSinkMax = 3            // distinct (ip, port) endpoints
kSinkTtlMs = 60000      // drop endpoints silent for a minute
eviction = LRU
```

Every event is fanned out to all live sinks. Three tiny UDP sends; no state machine.

## Packet budget

This is the arithmetic that has to be done **before** choosing a protocol rate:

```
main loop period      ≈ 26 ms   (render gate)
packets per loop      =  1      → ceiling  ≈ 27 pkt/s
packets per loop      =  8      → ceiling  ≈ 300 pkt/s
```

A speech envelope is 25 Hz. A console poll is 5 Hz. A 1-packet-per-loop receiver is therefore
already over capacity on an idle system, and `end` / `bargein` queue behind the envelope.
Symptom: it finishes speaking and the mouth stays open. See [bug #6](02-the-seven-bugs.md).

Rule of thumb: **the receive path must be able to drain at least 4× your highest sustained
rate**, or every control message inherits the queueing delay of the data stream.

## Commands must be confirmed, not merely sent

UDP over WiFi drops. Three habits, all cheap:

1. **Prime the path.** ARP is not resolved for the first packet; send 5 throwaway `q`s at
   startup and ignore the replies. Without this, the first real command is simply lost.
2. **Repeat discrete commands** 2–3×. Every command in this protocol is idempotent by
   design, specifically so that repetition is free.
3. **Confirm state-changing commands.** `end` is not "send and hope" — the host sends it,
   polls, and retries up to 4× until the device reports `turn == idle`. A turn that silently
   fails to hand over is far more damaging than 200 ms of extra latency.

The same applies to interrupts: `bargein` must land within one batch-drain, or the illusion
of conversation collapses.

## Envelope streaming

`v:<0..100>` at 25 Hz, no reply, deliberately.

Two details that matter more than the rate:

- **The envelope must come from the audio actually playing**, sampled by wall-clock elapsed
  time against the player's start, not accumulated with `sleep()`. Drift over a
  three-second sentence is visible as a lip-sync error.
- **Do not acknowledge it.** We added a one-line receipt for every received command, which
  fanned the 25 Hz envelope out to three sinks — 75 extra outbound packets per second — and
  saturated the link so badly that `end` could no longer get in. High-rate telemetry must
  never generate low-rate-shaped side effects.

## Turn-taking is the actual product

The link exists to support three states on the device, and they must be visible in the face:

| State | Face | Meaning |
|---|---|---|
| `listening` | pupils forward, lids slightly raised, small forward lean | *I am receiving* |
| `speaking` | **mouth moving**, driven by the live envelope | *I am transmitting* |
| `idle` | — | *the turn is yours* |

The transition `speaking → idle` is where a companion is either born or lost. On `end` the
device does not go neutral; it **arms an expectation** with a 6-second window. If nothing
arrives, it emits `expect_miss`, raises its loneliness, and becomes more likely to initiate
next time. If something arrives — a touch, a face, a word — it emits `expect_hit` and visibly
relieves.

That is the difference between a device that waits for input and a device that *hopes*. It
costs about forty lines and it is the only part anyone remembers.
