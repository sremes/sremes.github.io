---
layout: post
title:  "Tuning a MikroTik 5 GHz AP: frequency, C/e, and the control half"
date:   2026-10-18 16:00:00 +0300
categories: wifi mikrotik
---

MikroTik's wireless documentation lists the parameters but not what they do to the spectrum. Two details decide where your signal actually lands, and neither is obvious: `frequency` is the centre of the *control* 20 MHz channel, and the `C`/`e` letters in `channel-width` say which 20 MHz half that is. Get those wrong and everything else — channel number, width, interference — is guesswork.

## `frequency` is the control channel centre

RouterOS wants the centre of the control 20 MHz channel, not the centre of the bonded block.

Set `channel-width=20/40mhz-eC` and `frequency=5200`, and you get the 40 MHz block spanning 5170–5210 MHz — channels 36 and 40 — with the primary on channel 40. Scanners and clients will label the block channel 38 / 5190 MHz, because that is the composite centre. Do not type 5190.

Same for 80 MHz: `channel-width=20/40/80mhz-Ceee` with `frequency=5180` produces the block over 5170–5250 MHz (channels 36–48), which clients show as channel 42 / 5210 MHz. Again, 5210 is a display artefact, not an input.

## One letter per 20 MHz

The width string has one letter per 20 MHz sub-channel, in ascending frequency order:

- `C` marks the control (primary) channel.
- `e` marks an extension channel.
- `XX` or `XXXX` instead of letters lets the AP pick the least crowded control channel.

So 40 MHz has two layouts — `Ce` (control low) and `eC` (control high) — and 80 MHz has four: `Ceee`, `eCee`, `eeCe`, `eeeC`. Several (width, frequency) pairs describe the identical block; they differ only in which 20 MHz half carries the beacons.

## Valid layouts in the 36–48 block

This is the Finnish/EU 5 GHz range where every client works, so it is the interesting one.

| Block | Span | Setting |
|---|---|---|
| ch 36 | 5170–5190 | `20mhz` @ 5180 |
| ch 40 | 5190–5210 | `20mhz` @ 5200 |
| ch 44 | 5210–5230 | `20mhz` @ 5220 |
| ch 48 | 5230–5250 | `20mhz` @ 5240 |
| 36+40 | 5170–5210 | `20/40mhz-Ce` @ 5180 (primary 36) · `20/40mhz-eC` @ 5200 (primary 40) |
| 44+48 | 5210–5250 | `20/40mhz-Ce` @ 5220 (primary 44) · `20/40mhz-eC` @ 5240 (primary 48) |
| 36–48, 80 MHz | 5170–5250 | `20/40/80mhz-Ceee` @ 5180 · `eCee` @ 5200 · `eeCe` @ 5220 · `eeeC` @ 5240 |

The four 80 MHz layouts are the same spectrum. There is no hidden better one — any difference you measure between them is noise. All rows are legal in Finland at ≤200 mW, indoor.

## The control half is not cosmetic

Bonded channels use asymmetric clear-channel assessment:

- The primary half uses preamble detection at roughly −82 dBm (−62 dBm for non-802.11 energy).
- The extension half uses a relaxed detect threshold around −72 dBm.

So mid-level energy that sits between those two thresholds does not force a deferral on the extension half, but does on the primary. Beacons and management frames also ride the primary.

Practical rule: put the noisier 20 MHz half in the extension role.

I measured the swap on my own AP from the far corner of the apartment, with a weak client signal: `eC` @ 5200 (primary on ch 40) beat the previous `Ce` @ 5180 (primary on ch 36) on upload, with download unchanged. The direction matches the theory; at low RSSI a single run is noisy, so trust the sign, not the size.

## The band plan decides the rest

Finland follows the EU plan:

| Range | Channels | Notes |
|---|---|---|
| 5150–5250 | 36–48 | Non-DFS, ≤200 mW, indoor. The only universally compatible range. |
| 5250–5350 | 52–64 | DFS + TPC. |
| 5350–5470 | — | No Wi-Fi channels exist here in any regulatory domain. |
| 5470–5725 | 100–140 | DFS, ≤1 W. Channels 120–128 share the weather-radar band (10-minute listening at startup); a radar hit means vacating for 30 minutes. |
| 5725–5850 | 149–165 | Not available in the EU (US/Japan only). |

The practical consequence is client support. A Nintendo Switch 2 only handles the non-DFS ranges — 36–48, plus 149–165 where that band exists — and never scans DFS. In Europe that leaves 36–48 or nothing. Plenty of IoT devices behave the same way. The "free" channels a site survey shows up in the DFS range are free because clients and APs avoid them, or because a neighbour got radar-moved off them; they are not spare capacity you can design around.

## What I run

Daily driver: `country=finland`, `band=5ghz-a/n/ac`, `channel-width=20/40mhz-eC`, `frequency=5200`.

I also tried 80 MHz (`Ceee` @ 5180): upload got clearly worse, download stayed comparable. That is expected — every doubling of width needs about 3 dB more transmit power to hold per-subcarrier SNR, and the phone is the power-limited side, not the AP. Wider channels also share airtime with more neighbours. 40 MHz is the daily driver; 80 MHz is a fallback for a clean band or a big transfer.

Measurement notes: Wi-Fi numbers swing more than 20% run to run. Measure three times from the same spot against the same server, and do not chase differences under 10%.

## Pitfalls

- Do not enter the composite centre (5190, 5210) as `frequency`; RouterOS wants the control 20 MHz centre.
- The channel-width dropdown only lists what the radio supports.
- Set `country`; the regulatory domain caps both the frequency list and the power. Without it the frequency list is fiction.
- Re-scan when neighbours move. Non-DFS 5 GHz is a shared resource.

## Links

- [MikroTik — Wireless Interface](https://help.mikrotik.com/docs/spaces/ROS/pages/8978446/Wireless+Interface)
- [MikroTik forum — control frequency and channel layout](https://forum.mikrotik.com/t/cap-ac-suck-with-20mhz-channel/269989)
- [MikroTik forum — C/e letter semantics](https://forum.mikrotik.com/t/sxt-ac-bandwidth/81038)
- IEEE 802.11-15/0516r2 — CCA threshold summary (primary −82 dBm, secondary −72 dBm)
