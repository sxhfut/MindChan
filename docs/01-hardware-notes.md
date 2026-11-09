# Hardware notes — M5Stack StackChan (SKU:K151)

Everything here was read out of the BSP source (`StackChan-BSP`, `M5Unified`, `M5GFX`) and
confirmed on hardware. None of it is inferred from a product page.

## Board

| | |
|---|---|
| Product | M5Stack StackChan, `SKU:K151` |
| Base | CoreS3 |
| MCU | ESP32-S3 QFN56 v0.2, 2× Xtensa LX7 @ 240 MHz |
| Memory | 16 MB flash, 8 MB PSRAM, 512 KB internal SRAM |
| Display | 320×240 IPS |
| Toolchain | `esp32:esp32@3.3.12` + `M5Unified` + `M5GFX` + `M5StackChan` BSP |

## Display orientation

The panel is **natively landscape**, but the panel's own idea of "up" does not match yours.
Do not hardcode a rotation — probe it:

```c
if (M5.Display.width() < M5.Display.height()) {
    M5.Display.setRotation((M5.Display.getRotation() + 1) & 3);
}
```

If you hardcode, you get a face rotated 90° on exactly one batch of units and spend an
afternoon on it. (We did.)

## Servos

Two `SCS0009` on a serial bus, not PWM.

| | |
|---|---|
| Bus | UART1 @ 1 Mbps |
| TX / RX | GPIO 6 / GPIO 7 |
| Yaw | ID 1, zero 460, range ±1280 |
| Pitch | ID 2, zero 620 |
| Resolution | 1 step = 0.3125° |

**Safety contract:** pitch is clamped to `50…850` raw units on **every** code path — command
handler, idle wanderer, gaze controller, bench. No exceptions, no "just this once". A servo
that drives itself into its own housing is a hardware replacement, not a bug report.

Beware of the minimum inter-command gap: writing servos faster than ~40 ms apart causes the
bus to drop frames silently. The servo then does not move and the firmware believes it did.

## IO expander, LEDs, touch, power

| | |
|---|---|
| IO expander | `PY32L020` @ `0x6F` — pin 0 = servo VM_EN, pin 13 = RGB data |
| RGB LEDs | 12× `WS2812C` |
| Touch | `Si12T` @ `0x68`, three zones |
| IMU | `BMI270` |
| Power monitor | `INA226` @ `0x41` |
| NFC | `ST25R3916` @ `0x50` (via `M5Unit-NFC`) |

The touch controller reports **zones**, not coordinates. Zone identity is stable; position is
not. Design your interaction around "top / left / right" and you will be fine — design it
around a touch point and you will not.

## Camera — GC0308

| | |
|---|---|
| Sensor | GC0308 |
| Data | D0–D7 = 39, 40, 41, 42, 15, 16, 48, 47 |
| Sync | VSYNC 46, HREF 38, PCLK 45 |
| SCCB | must reuse the existing I2C bus — see [bug #1](02-the-seven-bugs.md) |

Full init that actually works:

```c
cfg.pin_sccb_sda  = -1;
cfg.pin_sccb_clk  = -1;
cfg.sccb_i2c_port = 1;
cfg.fb_count      = 1;
cfg.pixel_format  = PIXFORMAT_GRAYSCALE;   // we only need a face blob
cfg.frame_size    = FRAMESIZE_QQVGA;       // 160×120 is plenty
```

At `QQVGA` grayscale we get usable `faceArea` and gaze at a 100 ms cadence with ~2 ms of CPU
per frame. You do not need colour, and you do not need `QVGA`, for a face that just needs to
know *someone is there and roughly where*.

## Memory budget (measured at boot)

```
{"ev":"disp","w":320,"h":240,"rot":0,"sprite":1,
 "bands":[1,1,1],"blen":[31872,31872,7072],
 "heap":203416,"psram":8381300}
```

Three dirty-band sprites, 16-bit colour: 31,872 + 31,872 + 7,072 bytes ≈ **70 KB**, all in
internal RAM. That is the single most important number in this project — put those bands in
PSRAM and you lose two thirds of your frame rate (see [bug #3](02-the-seven-bugs.md)).
