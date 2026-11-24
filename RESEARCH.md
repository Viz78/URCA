# Research: existing open-source work

Inspected on 2026-11-23 by cloning each repository and reading the code (not just READMEs).

## Repository status

| Project | Available | License | Platform / language | Relevant to |
|---|---|---|---|---|
| [OpenRazer](https://github.com/openrazer/openrazer) | Yes (active, last commit 2026-11-18) | GPL-2.0-or-later (docs CC-BY-SA-4.0) | Linux kernel driver (C) + Python daemon | **Viper 8KHz (PID 0091): merged support** |
| [OpenRazer PR #2862](https://github.com/openrazer/openrazer/pull/2862) | Yes, **open / not merged** | GPL-2.0-or-later | Linux kernel driver | **BlackShark V2 Pro 2.4 (PID 0555)** |
| [blackshark-control](https://github.com/AbdullahSAhmad/blackshark-control) | Yes (2026-10-13) | GPL-2.0-or-later | Linux, Python, user-space hidraw | **BlackShark V2 Pro 2023 (PID 0555)**, tested on firmware 2.5 |
| [OpenSynapse](https://github.com/JohnsonRan/OpenSynapse) | Yes (v0.10.0, 2026-11-14) | Source GPL-2.0-or-later; release binaries GPL-3.0 (embedded VIA data) | **Windows**, Go + WebView2 | Viper 8K via an OpenRazer-generated catalog; Windows HID transport; macros |
| Non-Synapse-Mouse (edward-p) | **No.** The URL returns 404 and the owner's 36 public repos contain nothing Razer-related | — | — | Nothing could be evaluated |

## Viper 8KHz (1532:0091)

**Source of truth:** OpenRazer `driver/razermouse_driver.c`, every `case USB_DEVICE_ID_RAZER_VIPER_8K` block,
plus `daemon/openrazer_daemon/hardware/mouse.py` class `RazerViper8KHz`.

Features the driver exposes for this PID: firmware, serial, device mode, DPI X/Y, DPI stages (max 5),
polling rate (125/500/1000/2000/4000/8000), logo LED brightness, and logo effects (static, spectrum, reactive,
breath, none). Max DPI 20000.

| Function | Class:Cmd | Size | Txn ID | Notes |
|---|---|---|---|---|
| Firmware (get) | 00:81 | 02 | FF | |
| Serial (get) | 00:82 | 16 | FF | |
| Device mode (get / set) | 00:84 / 00:04 | 02 | FF | We do **not** switch to driver mode (0x03); it changes how the mouse's own buttons behave |
| DPI (get) | 04:85 | 07 | FF | arg0 = NOSTORE |
| DPI (set) | 04:05 | 07 | FF | builder always writes VARSTORE |
| DPI stages (get / set) | 04:86 / 04:06 | 26 | FF | VARSTORE, stride 7 |
| Polling rate (get) | 00:C0 | 01 | 1F | "polling_rate2" codes: 01=8000 02=4000 04=2000 08=1000 10=500 40=125 |
| Polling rate (set) | 00:40 | 02 | 1F, then FF | **Sent twice**: arg0=00 with txn 1F, then arg0=01 with txn FF (driver comment: "Razer sends each request once with 0x00 and once with 0x01") |
| Logo brightness (get/set) | 0F:84 / 0F:04 | 03 | 1F | led 0x04 |
| Logo static | 0F:02 | 09 | 1F | [VARSTORE, 04, 01, 00, 00, 01, R, G, B] |

Transport: USB control SET_REPORT/GET_REPORT, report type Feature, wValue 0x0300, **wIndex 0 (interface 0)**,
600 µs wait. The PID isn't in any of the "longer wait / index 3" special cases.
OpenSynapse's `devices.json` (generated from OpenRazer commit 477f4d6) lists byte-identical transaction IDs
and the same two-step polling sequence for "0091". The two sources agree.

**Windows mapping** (from OpenSynapse `internal/razer/hid_windows.go`): open the HID collection whose
FeatureReportByteLength = 91 with `CreateFile(access = 0)`. Windows allows that even though it owns the mouse
collection. Then use `HidD_SetFeature` + `HidD_GetFeature` with byte 0 = report ID 0. CRC = XOR of buffer bytes 3..88.

**Button remapping: no verified source for PID 0091.** OpenRazer has no button remapping for any mouse.
OpenSynapse has onboard remapping (class 02, cmd 84/8C/0C, taken from Synapse 4), but its own comment says
"only been tried on the DeathAdder V4 Pro; a device without them does not answer". It only enables extras for
that model family. So: onboard remapping on the Viper 8K is **unknown**. `urc-diag viper --probe-buttons`
sends the read-only button-list query once to find out. Until it is verified, remapping is done in software (see ARCHITECTURE.md).

**Not verified by any source:** whether OpenRazer maintainers tested every function on real 0091 hardware.
Support is merged, but Milestone 1 confirms each read on your unit.

## BlackShark V2 Pro 2023 (1532:0555)

**Sources:** blackshark-control `blackshark/protocol.py` (tested on Linux Mint, headset firmware 2.5) and
OpenRazer PR #2862 `driver/razerblackshark_driver.{c,h}` (open; derived from Synapse USB captures of this device).
blackshark-control credits PR #2862 for the protocol. These are not independent confirmations; they come from one capture effort.

Transport: the dongle's vendor HID interface (usage page 0xFF00), **report ID 2, 64-byte reports**. Requests are
**output** reports on the interrupt OUT endpoint, replies are **input** reports (PR #2862 header). The interface
number isn't stated (the PR's event regex suggests if03). The Windows module therefore selects the collection by
its descriptor (vendor page, 64/64 input/output, output report ID 2), not by a hard-coded MI number.

Frame: `[0]=02 [1]=80 [2]=8+len [5]='P' [6]='A' [7]=08 [9]=type [10]=cmd [11]=0 [12]=len [13..]=payload`.
Reply: `[12]=cmd [13]=01 (ACK) [14]=len [15..]`. "Remote" frame `[7]=0E [9]=02 [10]=E1 [11]=on/off` wraps every exchange.

| Feature | Read | Write | Status |
|---|---|---|---|
| EQ preset Game 07 / Music 08 / Movie 09 / Custom FF / Esports FA–FE | 03:13 | 04:93 + 04:9D (+ prep 03:1E) | Supported. Verify by reading back |
| 10-band custom EQ (−12..+12 dB, 31 Hz–16 kHz) | 03:15 | 0D:95 | Supported. **Danger:** a 0x95 frame sent during a cross-family transition overwrites another preset's curve (observed by PR author). Custom must be confirmed active first |
| Game/Music/Movie curves | — | — | Built into firmware; the curves **cannot be read back** |
| Physical EQ button | unsolicited 0x13 frame | — | This is the hook for **EQ locking** |
| Battery %, charging | 21, 2A | — | Supported |
| Mic mute state | 55 | — | Read only |
| Mic monitoring | 18/19 | 98/99 | Supported |
| Auto power-off | 2C | AC | 15–60 min or never |
| Bluetooth do-not-disturb | 27 | A7 | Exists. Not exposed (Bluetooth-related, not needed) |
| **Volume** | — | — | **No HID volume command is known.** The volume wheel works inside the headset and isn't reported over the dongle. PC-side volume can use the standard Windows Core Audio endpoint API (not a device protocol) |
| THX Spatial | — | — | Proprietary Synapse software; not available |

Behavioral warnings from PR #2862: the 2.4 GHz link dozes after about 0.3 s and drops the first frame (so a sacrificial
remote frame is sent first). "An ACK from this device does not on its own mean the setting took." Command 0x9D
"neither stores nor acts on it". It is still part of Synapse's apply sequence and is mirrored as captured.

**Bluetooth vs 2.4 GHz:** everything above goes through the USB dongle only. When the headset is used over
Bluetooth (to a phone or the PC's BT stack), the dongle can't reach it and no setting can be read or written.
This project never talks to the Bluetooth side.

## Verified on the user's hardware (2026-11-23, Windows 11 23H2, `urc-diag` read-only)

Synapse (RazerAppEngine) was running during these reads.

**Viper 8KHz:** firmware v1.3. The control collection is **MI_00 (top-level, UsagePage 01 / Usage 02), 91-byte feature
report, report ID 0**, as expected. All 7 GETs (00:81, 00:82, 00:84, 04:85, 04:86, 00:C0, 0F:84) answered with
status 02 (Successful) on the first try, with the documented transaction IDs. Decoders confirmed against the device state:
DPI 4000x4000; stages 2000/3200/4000 with stage 3 active (stride-7 layout correct); polling code 01 = 8000 Hz;
logo brightness 178. The device mode read **0x03 (driver mode)**, probably set by Synapse.
Other collections: MI_00 Col03 (consumer), MI_01 Col01–05 (keyboard/consumer/system/vendor 16-byte inputs), MI_02 (keyboard).
**Button-list probe (02:84, tx 1F): ANSWERED** with status 02 and 10 button IDs `01 02 03 04 05 06 07 60 09 0A`
(raw args `0A 01 02 03 04 05 06 07 60 09 0A`). So the Viper 8K speaks at least the read side of the Synapse-4 button
protocol.

**Per-button read (02:8C, size 0x50, tx 1F, args `[00 NoStore, button, layer 0]`, layout from OpenSynapse
buttons.go): ANSWERED for all 10 IDs** (`urc-diag viper --read-buttons`). The reply has data size 0x0A, args `[profile, button, layer, fn, n, data…]`:

| ID | fn | data | Meaning (fn constants from buttons.go) |
|---|---|---|---|
| 01 | 01 | 01 | mouse left |
| 02 | 01 | 02 | mouse right |
| 03 | 01 | 03 | mouse middle |
| 04 | 01 | 04 | mouse back (MB4) |
| 05 | 01 | 05 | mouse forward (MB5) |
| 06 | **05** | 1E BD | **unknown function 0x05** (not in buttons.go) |
| 07 | **05** | 2E ED | **unknown function 0x05** |
| 60 | 06 | 06 | DPI cycle up, so 0x60 is the DPI button |
| 09 | 01 | 09 | wheel up |
| 0A | 01 | 0A | wheel down |

**User-confirmed (2026-11-23):** IDs 06/07 are the second pair of side buttons, and they're bound to **macros** in Synapse.
So fn 05 = macro (inferred from that fact, not from a protocol source). The 2-byte data (`1EBD`, `2EED`) is probably a
macro ID or handle. Its meaning and where the macro body is stored are unknown. Never overwrite 06/07: we
can't rebuild those macros.
Unlike OpenSynapse's DeathAdder note ("layer byte reads 1 whatever was asked"), the reply's layer byte here varies
(00 for 01/02/04/05, 01 for 03/60/09/0A, 05 for 06/07). Its meaning is unclear, so it isn't used.
**Write (02:0C) still unverified.** OpenSynapse's own warning applies: only profile 0 (NoStore) took effect on the DeathAdder,
and an ACK alone doesn't prove the change took. A first write test needs the user's OK, Synapse closed, and a 02:8C read-back.

**BlackShark V2 Pro dongle:** the control collection is **MI_03 Col02, UsagePage FF00 / Usage 01, 64-byte in/out,
report ID 02**. The interface number is now confirmed. MI_03 Col01 is consumer control (3-byte input). Every query was answered
on the first attempt after the sacrificial remote frame. Results: serial 15 ASCII chars; battery 33 %; charging 0; preset FF (Custom);
EQ curve 10 × 0; mic mute 0; sidetone 0; idle 0x1E = 30 min; DND 0.
Firmware reply: the payload is `02 06 0B 00` repeated six times (the length byte is larger than 4), parsed as "2.6". It could be
2.6.11, **unconfirmed**. The raw frame isn't in the trace yet.
**Unsolicited 0x13 on the EQ button: not tested.** The user's headset button is physically unreliable, so the
EQ-lock hook can't be verified on this unit. Plan for it, but don't depend on it. Another way to detect the change is
periodically re-reading 03:13.

### Milestone 2 write tests (with the user's OK)

| Write | Frame | Result |
|---|---|---|
| Logo brightness 178 → 128 → 178 | 0F:04 tx 1F `[01 VARSTORE, 04 LOGO, level]` | **VERIFIED** both ways with a 0F:84 read-back. Other settings unchanged. Synapse running at the time |
| DPI 4000 → 3200 → 4000 | 04:05 tx FF `[01 VARSTORE, X BE, Y BE, 0, 0]` | **VERIFIED** both ways with a 04:85 read-back. The stage table and active stage (3) **did not change**: the current DPI is independent of the stages |

| MB4 toggle clicker (software, WH_MOUSE_LL + SendInput) | — | **VERIFIED twice** (16:49 and 16:53): MB4 caught and swallowed, left clicks injected (user-confirmed), 5 s auto-stop fired on time. Synapse running; MB4 still arrives as XBUTTON1 in driver mode |

| DPI stages: stage 1 2000 → 1800 → 2000, active 3 | 04:06 tx FF `[01 VARSTORE, active, count, {i 0-based, X BE, Y BE, 0, 0}×n]` | **VERIFIED** both ways with a 04:86 read-back. The device accepts OpenRazer's **0-based** write index and reports stages **1-based** on read. The current DPI (4000) and other settings were unchanged |

| Polling rate 8000 → 1000 → 8000 Hz | 00:40 sent twice: tx 1F `[00, code]` then tx FF `[01, code]` (codes 01=8000, 08=1000) | **VERIFIED** both ways with a 00:C0 read-back. Other settings unchanged |

| MB4 hold-left toggle | — | **VERIFIED** (17:00): held for 3 s, released on the second press |
| MB5 repeat right at 1 ms / hold 0 | — | **FAILED in practice**: the engine stopped on each press (log shows `off`), but the target app (Notepad context menus) couldn't keep up, and its backlog kept "clicking" afterwards, so the user saw it as unstoppable. Fix: interval floor `ClickerBinding.MinIntervalMs` = 5 ms; the user's MB5 is set to 20 ms. **Retest at 20 ms VERIFIED** (17:04, user-confirmed): it started and stopped cleanly on single presses |

All four Milestone 2 mouse writes are verified on this unit. Not yet tested on hardware: MB5 (right-click toggle),
stopping a burst with a second MB4 press.

### Milestone 3 headset write tests (with the user's OK)

Sequences from blackshark-control protocol.py and OpenRazer PR #2862 (they agree): remote on ×2, prep 03:1E, remote,
04:93 [preset], remote, 04:9D [flag 01 classic / 02 esports], (remote, 0D:95 [10 signed bands]), remote ON; 35 ms
after each remote frame; up to 4 applies with a 100 ms settle and a 0x13 read-back.

| Write | Result |
|---|---|
| Preset Custom → Game → Custom → Game | **VERIFIED** each step with a 0x13 read-back, on the first apply (same family). The custom curve (0x15) was unchanged by the preset switches. Synapse running at the time. **The headset was left on Game**, which the user prefers |

| EQ lock (target Game, 5 s poll), preset forced to Music by a second `urc` process | **VERIFIED** (17:23): restored Game about 1 s after the change, with a verified read-back. The headset ended on Game |

**Observation:** every open handle on the dongle's vendor collection receives every input report. The lock process
saw the *other* process's 0x13 replies as unsolicited frames and re-checked immediately. So "unsolicited 0x13"
doesn't prove an EQ-button press. It can be Synapse's or another program's read. That's harmless for the lock (it only re-checks).

| Custom EQ `2,1,0,0,0,0,0,1,1,2` then flat | **VERIFIED** (0x15 read-back). Preset Game → Custom was confirmed before the bands were sent; the curve was then restored to flat and the preset to Game |

| Switches into and between Esports 1–5 and back to Game (via the agent) | **VERIFIED**, each step with a 0x13 read-back, including the cross-family switches |

**EQ curves (2026-11-23).** 0x15 on an Esports preset returns that preset's own curve. On Game/Music/Movie it returns the
Custom slot's curve (as PR #2862 says), so those three can't be read from the device. Curves below are in dB, 31 Hz → 16 kHz:

| Preset | Curve | Source |
|---|---|---|
| Game (07) | -3 -3 -4 0 5 5 4 1 0 -1 | read off Synapse's EQ graph (user screenshots) |
| Movie (09) | 4 4 3 0 -3 -1 3 5 2 1 | Synapse screenshot |
| Music (08) | 2 2 1 1 2 3 3 3 1 0 | Synapse screenshot |
| Esports 1 (FA) | -1 0 0 0 -1 0 2 3 2 3 | **device read**; equals Synapse "Apex Legends" |
| Esports 2 (FE) | -4 5 5 3 -2 3 4 4 -1 4 | **device read**; equals Synapse "Fortnite" |
| Esports 3 (FB) | -5 -4 3 5 3 -2 1 5 4 -4 | **device read**; equals Synapse "CS:GO" |
| Esports 4 (FD) | 0 0 0 0 0 1 4 4 4 -3 | **device read**; equals Synapse "Valorant" |
| Esports 5 (FC) | 0 0 0 0 0 1 4 4 4 -3 | **device read** (re-read from a clean switch); same as Esports 4 |

Synapse's "Call of Duty" curve (-2 0 3 3 3 3 0 0 0 0) is in none of FA–FE. Its preset ID is unknown, and no other IDs were probed.
The app reads Esports/Custom curves live and uses the table only for Game/Music/Movie.

## Reuse decision

- License: **GPL-2.0-or-later**, compatible with all three usable sources.
- No source files were copied (the sources are C, Python and Go). The protocol constants, frame layouts and
  command sequences were reimplemented in C# from the code above, with file/line attribution in comments.
- OpenSynapse's Windows approach was adopted as a design: zero-access feature-report handles, serialized set/get,
  and Busy handling.
