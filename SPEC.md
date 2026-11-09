# SPEC — wire protocol

Version `m2.4`. Plain text over UDP. No framing, no handshake, no version negotiation.
One datagram = one message = one line.

## Ports

| | |
|---|---|
| Device | `20086/udp` |
| Host default | `20087/udp` |

The device replies to the **source** of the datagram for requests, and fans events out to
all live sinks (see [`docs/04-link-layer.md`](docs/04-link-layer.md)).

## Host → device

Every command is idempotent and may be sent 2–3× without side effects.

| Command | Effect |
|---|---|
| `q` | reply with a full state snapshot to the sender |
| `say:<emo>:<ms>` | begin speaking as `<emo>`, auto-end after `<ms>` |
| `v:<0..100>` | speech envelope sample, ~25 Hz, **never acknowledged** |
| `end` | end speaking, return to idle, arm a 6 s expectation |
| `listen:<ms>` | enter listening state for `<ms>` |
| `bargein` | interrupt immediately: close mouth, switch to a startle expression |
| `emo:<name>[:<ms>]` | set a named expression, default hold 3000 ms |
| `pad:<p>,<a>` | set internal PAD directly (debug) |
| `bench:<ms>` | bypass the render gate for `<ms>`, report true throughput |
| `bond:<0..1>` | set relationship strength (demo / reset) |
| `cue:<name>` | play a non-verbal cue |
| `sim:near\|leave\|stare\|touch` | inject a simulated social event |
| `gx,gy,pitch` | raw gaze / pitch override |
| `rst` | reset internal state to factory |

`<emo>` is one of 71 names. Unknown names fall back to the current expression — they never
throw and never wedge the state machine.

### Servo safety contract

`pitch` is clamped to **50 … 850** raw on *every* path: command handler, idle wanderer, gaze
controller, bench. There is no code path that can command outside this range. This is a
hardware-protection invariant, not a preference.

## Device → host

### State (reply to `q`, also emitted after state-changing commands)

A single JSON object, ~50 fields. Unspecified fields may be added at any time; hosts must
ignore unknown keys.

| Field | Meaning |
|---|---|
| `t` | device uptime, ms |
| `p`, `a`, `d` | PAD: pleasure, arousal, dominance in `[-1, 1]` |
| `e` | energy |
| `emo` | current expression name, or `auto` |
| `turn` | `0` idle · `1` listening · `2` speaking |
| `to` | current mouth openness from the envelope, `[0, 1]` |
| `tpk` | envelope packets accepted this turn |
| `exp` | expectation kind: `0` none · `1` seeking attention · `2` awaiting reply |
| `eh`, `em` | expectation hits / misses, cumulative |
| `lon` | loneliness `[0, 1]` |
| `bond` | relationship strength `[0, 1]`, persisted in NVS |
| `bl`, `bp` | blush level, blush pulse |
| `av` | gaze aversion `[0, 1]` |
| `face`, `fa`, `gx`, `gy` | face detected, face area, gaze x/y |
| `dwell` | ms since this person appeared |
| `alone` | ms since anyone was present |
| `sk` | count of self-initiated attention seeks |
| `fr` | frames rendered |
| `rm`, `rmx` | render time last / max, ms |
| `dms`, `bms` | eye-band draw, mouth-band blit, ms |
| `lp`, `lpm` | loop period last / max, **milliseconds** |
| `batt`, `wifi` | battery volts, link up |
| `k1`…`k6` | per-subsystem CPU time, µs |

> `lp` / `lpm` are the field names we regret. They are milliseconds, despite the internal
> variables being named `perfLoopUs`. `lpm: 4050` means a four-second stall.

### Events (unsolicited)

One line of JSON, `{"ev": "...", ...}`.

| Event | Fields | Emitted when |
|---|---|---|
| `boot` | `fw`, `cam`, `sprite`, `atlas`, `heap`, `psram` | startup |
| `disp` | `w`, `h`, `rot`, `sprite`, `bands`, `blen` | display configured |
| `wifi_up` | `ip`, `rssi` | link established |
| `host_bound` | `ip`, `port`, `sinks` | **new** endpoint registered |
| `host_lost` | — | no host packet for 10 s |
| `turn` | `state`, `emo`, `ms` | turn state change |
| `talk_end` | `pkts` | speaking finished, envelope packet count |
| `bargein_ack` | — | interrupt accepted |
| `expect_hit` | `src`, `waited`, `lon`, `bond` | expectation fulfilled |
| `expect_miss` | `waited`, `lon`, `miss` | expectation expired |
| `cmd` | `c` | discrete command received (never for `v:`) |
| `sim` | `what` | simulated social event injected |
| `emo_end` | `name`, `rp`, `ra`, `p`, `a`, `bond` | expression override expired |
| `bond_saved` | `bond` | relationship persisted to NVS |
| `cam_ok` / `cam_try_fail` | `port`, `err` | camera init |

## Example turn

```
host  → "q"
dev   → {"t":122925,"turn":0,"exp":0,...}

host  → "listen:8000"
dev   → {"ev":"cmd","c":"listen:8000"}
dev   → {"ev":"turn","state":"listening","ms":8000}

host  → "say:happy:6000"
dev   → {"ev":"turn","state":"speaking","emo":"happy","ms":6000}

host  → "v:43"  ×50 @25 Hz        (no reply, ever)

host  → "end"
dev   → {"ev":"talk_end","pkts":50}
dev   → {"ev":"turn","state":"idle"}

... 6 seconds of silence ...

dev   → {"ev":"expect_miss","waited":6019,"lon":0.16,"miss":2}
```

## Compatibility

The protocol has no version field. Instead:

- **Fields are append-only.** Never reuse a key for a different meaning.
- **Unknown keys must be ignored** by both sides.
- **Unknown commands must be ignored** by the device, not rejected.

This is crude and it has survived four rewrites without a single host/device version
mismatch. A version handshake would have been stricter and would have broken the link every
time the device rebooted before the host did.
