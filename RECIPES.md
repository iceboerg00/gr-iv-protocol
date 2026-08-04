# Ricoh GR IV — Image Control Recipes: How Reading and Writing Works

Complete technical documentation of the recipe format and the transfer protocol, as
implemented in this project and verified against a real camera (RICOH GR IV HDF,
firmware 1.11).

This is the deep dive on the **recipe format** specifically. For the full remote-control
surface (all Wi-Fi HTTP endpoints, live BLE settings, focus, live view, etc.) see
[GR_IV_Remote_Control_API.md](GR_IV_Remote_Control_API.md).

---

## 1. What a recipe is

An "Image Control recipe" is what the camera calls a custom image profile. It consists
of exactly three things:

| Part | Type | Where it lives on the camera |
|---|---|---|
| **Name** | UTF-8 string, variable length | its own BLE characteristic |
| **Base effect** | one byte (`byte2`) | inside the 56-byte parameter block |
| **Parameters** | up to 26 × (id, value) pairs | inside the 56-byte parameter block |

The camera exposes **three custom slots**. Each slot is a pair of characteristics: one
for the name, one for the parameter block. Reading and writing a recipe means reading
or writing those two characteristics. There is no session, no commit, no trigger.

Everything else a photographer might consider part of a "look" — white balance,
ISO, exposure compensation, WB compensation, highlight/shadow correction — is **not**
part of the recipe block. See [§9](#9-what-is-not-in-the-recipe).

---

## 2. Transport: BLE only

Recipes travel over **Bluetooth Low Energy**, not Wi-Fi. This was established by
capturing the official GR World app's HCI traffic (`btsnoop_hci.log`) while it wrote a
recipe: the app performs exactly **two ATT Write Requests** and nothing else.

The early hypothesis that recipes go through the Wi-Fi HTTP endpoint `/v1/imgctrl` was
**wrong** — that path always returns 404 and does not exist in the firmware. Do not
pursue it.

### 2.1 Device

```
BLE name : GR_XXXXXXX
MAC      : XX:XX:XX:XX:XX:XX
Advertised service : 9a5ed1c5-74cc-4c50-b5b6-66a48e7ccff1
Manufacturer data  : ID 0x065f (RICOH)
```

### 2.2 Pairing is mandatory

Every Ricoh characteristic requires an authenticated bond. Without one, reads fail with
*Insufficient Authentication*. Only the standard services (`1800` GAP, `1801` GATT,
`180a` Device Information) are readable unpaired.

The camera demands **Numeric Comparison** pairing (`CONFIRM_PIN_MATCH`, kind=8): a
6-digit code must be confirmed on both the camera and the client.

Platform support, verified the hard way:

| Platform | Works? | Notes |
|---|---|---|
| Linux / BlueZ | ✅ | The reference implementation (`gr.py`, `server.py`) |
| Android | ✅ | Verified end-to-end with the GRade Flutter app on a tablet |
| iOS | ✅ (expected) | Same CoreBluetooth path as the official app; not yet tested here |
| **Windows** | ❌ | Bond succeeds, but the encrypted connection drops after ~1 s. WinRT stack limitation; not fixable from our side. |

### 2.3 Slot characteristics

All six live in service `9f00f387-8345-4bbc-8b92-b87b52e3091a`. All are **read + write**,
so both directions work on every slot.

| Slot | Name characteristic | Parameter characteristic |
|---|---|---|
| 1 | `30adb439-1bc0-4b8e-9c8b-2bd1892ad6b0` | `3e0673e0-1c7b-4f97-8ca6-5c2c8bc56680` |
| 2 | `e799198f-cf3f-4650-9373-b15dda1b618c` | `0936b04c-7269-4cef-9f34-07217d40cc55` |
| 3 | `df77dd09-0a48-44aa-9664-2116d03b6fd7` | `cd879e7a-ab9f-4c58-90ed-689bae67ef8e` |

Useful neighbour: battery level is a uint8 percentage at
`875fc41d-4980-434c-a653-fd4a4d4410c4`.

### 2.4 Which three slots you are talking to

**The three slots are the custom profiles of the mode currently selected on the mode
dial.** They are not global. If the dial is on P, you read and write P's three custom
profiles; turn the dial to U2 and the same three characteristics now address U2's
profiles. There is no way to address a different mode's slots remotely — turn the dial.

---

## 3. The 56-byte parameter block

```
Byte 0 : 0x01        version
Byte 1 : 0x00        constant / reserved
Byte 2 : base code   see §4  — NOT the effectList index
Byte 3 : N           number of parameter pairs that follow
Byte 4 .. 4+2N-1     N × ( paramId:uint8 , value:int8 signed )
rest   : 0x00        zero padding, total length always exactly 56
```

The block is **always written as a full 56 bytes**, padding included. Parameter pairs are
emitted in **ascending paramId order**.

The size is not arbitrary: 4 header bytes + 26 pairs × 2 bytes = 56. The block is
exactly large enough to hold every parameter the camera knows (IDs 0–25) and not one
byte more.

### 3.1 Value encoding

Values are **signed 8-bit**:

```python
def s8(b): return b - 256 if b >= 128 else b   # decode byte -> value
def u8(v): return v & 0xFF                     # encode value -> byte
```

So `-1` is stored as `0xFF`, `-4` as `0xFC`.

### 3.2 Worked example — "Cinema Yellow"

This is a real block captured from the camera. It is used as the round-trip test case in
both `recipe.py`'s self-test and `app/test/recipe_test.dart`.

```
01 00 0e 0d                                  header: v1, base 0x0e=14, N=13
00 00  01 03  02 ff  03 03  04 00  05 fe     p0=0  p1=+3 p2=-1 p3=+3 p4=0 p5=-2
06 fd  08 01  13 00  14 00  15 00  16 00     p6=-3 p8=+1 p19=0 p20=0 p21=0 p22=0
17 00                                        p23=0
00 00 ... 00                                 padding to 56 bytes
```

Note `13 00` is paramId `0x13` = 19 (shading), not "parameter 13". Parameter IDs in the
hex dump are hex; in this document and in the code they are decimal.

Decoding this block and re-encoding it must produce the identical 56 bytes. Both the
Python and Dart codecs pass this test.

---

## 4. Base codes (`byte2`) — the most important gotcha

`byte2` selects the base effect. **It is its own enum and it is NOT the index into the
camera's `effectList`.** The two enums agree for codes 0–5 and then diverge completely.

### 4.1 The real base codes (verified by capturing every base off the camera)

| `byte2` | Base (EN) | Base (DE) |
|---|---|---|
| 0 | Standard | Standard |
| 1 | Vivid | Kräftig |
| 2 | Monochrome | Monochrom |
| 3 | Soft Monochrome | Weiches Monochrom |
| 4 | Hard Monochrome | Hartes Monochrom |
| 5 | Hi-Contrast B&W | Hi-Kontrast S/W |
| 6 | Positive Film | Positivfilm |
| 7 | Bleach Bypass | Bleach Bypass |
| 8 | Retro | Retro |
| 9 | HDR Tone | HDR-Ton |
| 13 | Negative Film | Negativfilm |
| 14 | Cinema Yellow | Kino Gelb |
| 15 | Cinema Green | Kino Grün |
| 16 | Cross Process | Cross-Prozess |

That is 14 bases. Codes **10, 11, 12 are unknown** — no slot with those codes has been
captured. Codes above 16 have never been seen.

### 4.2 The other enum: `effectList` (Wi-Fi HTTP `/v1/props`)

```
0 off              1 col_vivid          2 efc_monochrome     3 efc_softMonochrome
4 efc_hardMonochrome  5 efc_highContrast  6 efc_negaFilm      7 efc_posiFilm
8 efc_cineYellow   9 efc_cineGreen      10 efc_crossProcess  11 efc_bleachBypass
12 efc_retro       13 efc_HDRTone       14 col_custom1       15 col_custom2
16 col_custom3
```

This list belongs to the **HTTP** API (`effect=` on `PUT /v1/params/camera`). Compare
code 14: in `byte2` it means *Cinema Yellow*; in `effectList` it means *col_custom1*.
They are different namespaces. Never convert one into the other.

### 4.3 Known quirk in the JSON output

`Recipe.to_dict()` / `Recipe.toMap()` emit a field `base_effect` computed as
`EFFECTS[base]` — i.e. it indexes the **effectList** with the **byte2** code. For codes
≥ 6 that string is therefore wrong: `base_kino_gelb.json` (Cinema Yellow, byte2=14)
carries `"base_effect": "col_custom1"`.

This is **deliberately preserved** in both the Python and the Dart port so that recipe
JSON stays compatible between the web app and the GRade app. The field is decorative —
nothing reads it back. The authoritative fields are `base_effect_id` (= byte2) and
`base_name` / `base_name_en`.

### 4.4 `byte2` must match the parameter set exactly

This is the rule that causes most write failures:

> **A base code implies an exact set of parameter IDs. If the block's parameter set does
> not match what the camera expects for that base, the camera rejects the write with
> app error 0x80.**

You cannot, for example, take the Monochrome parameter set and write it with `byte2=0`
(Standard), nor add a saturation parameter to a monochrome base. Each base has a fixed
roster.

This is why the project ships one captured template per base in `recipes/base_*.json`
(and mirrored as Flutter assets in `app/assets/bases/`). **The editor always starts from
a captured base template and only changes values, never the parameter roster.** That is
the single design decision that makes writing reliable.

Examples of how much the rosters differ:

```
base_standard.json    (byte2=0)   params 0,1,2,3,4,5,6,19,20                       (9)
base_kino_gelb.json   (byte2=14)  params 0,1,2,3,4,5,6,8,19,20,21,22,23           (13)
base_monochrom.json   (byte2=2)   params 2,3,4,5,6,7,12,13,14,15,16,17,18,19,20,21,22,23  (18)
```

Monochrome has no saturation (0) or hue (1) but does have toning (7) and the B&W colour
filters (12–18). Standard has neither toning nor grain.

---

## 5. Parameters

### 5.1 ID table

Internal names come from the `ImageControlParameter` enum recovered from the GR World
app via Blutter decompilation.

| ID | Internal name | Label (EN) | Range | Notes |
|---|---|---|---|---|
| 0 | saturation | Saturation | −4..+4 | |
| 1 | hue | Hue | −4..+4 | |
| 2 | highLowKey | High/Low Key | −4..+4 | |
| 3 | contrast | Contrast | −4..+4 | |
| 4 | contrastHighlight | Contrast Highlight | −4..+4 | |
| 5 | contrastShadow | Contrast Shadow | −4..+4 | |
| 6 | sharpness | Sharpness | −4..+4 | |
| 7 | toningMono | Toning (Mono) | 0..5 | **enum**, see §5.2 |
| 8 | toning | Toning | 1..3 | |
| 9 | bleachBypass | Bleach Bypass | −4..+4 | ⚠️ range estimated |
| 10 | retro | Retro Effect | −4..+4 | ⚠️ range estimated |
| 11 | hdrToning | HDR Toning | −4..+4 | ⚠️ range estimated |
| 12 | filterEffect | Filter Effect | 0..4 | 0 = off; gates 13–18 |
| 13 | filterRHigh | Filter Red (high byte) | — | part of 16-bit pair, see §5.3 |
| 14 | filterRLow | Filter Red (low byte) | — | part of 16-bit pair |
| 15 | filterGHigh | Filter Green (high byte) | — | part of 16-bit pair |
| 16 | filterGLow | Filter Green (low byte) | — | part of 16-bit pair |
| 17 | filterBHigh | Filter Blue (high byte) | — | part of 16-bit pair |
| 18 | filterBLow | Filter Blue (low byte) | — | part of 16-bit pair |
| 19 | shading | Shading (vignette) | −4..+4 | |
| 20 | clarity | Clarity | −4..+4 | |
| 21 | grain | Grain | 0..1 | off / on |
| 22 | grainSize | Grain Size | 1..3 | |
| 23 | grainStrength | Grain Strength | 1..3 | |
| 24 | hdrLevel | HDR Tone Level | 1..3 | |
| 25 | crossProcess | Cross Process | −4..+4 | ⚠️ range estimated |

The four estimated ranges (9, 10, 11, 25) have never been checked against the camera's
own UI limits. They are permissive enough not to block anything, but a value the camera
dislikes will be rejected at write time.

### 5.2 Enum parameter: Mono Toning (ID 7)

The byte value is an index, not a magnitude. Verified against the camera:

```
0 = off   1 = S   2 = R   3 = G   4 = B   5 = P
```

The UI renders this as a dropdown / chip selector rather than a slider.

### 5.3 The B&W colour filters are 16-bit (IDs 13–18)

Red, Green and Blue filter strength are each a **signed 16-bit value spread across two
int8 parameters** — a low byte and a high byte:

| Colour | low byte ID | high byte ID |
|---|---|---|
| Red | 14 | 13 |
| Green | 16 | 15 |
| Blue | 18 | 17 |

```dart
value = int16( low | (high << 8) )      // range -200 .. +200  (percent)
```

Both codecs expose this as a single logical control per colour:

```dart
int  filterValue(Map<int,int> params, int lo)              // combine
void setFilterValue(Map<int,int> params, int lo, int val)  // split back into both bytes
```

The UI shows **one slider per colour** (−200…+200 %), hides the high-byte IDs (13/15/17)
entirely, and only shows the RGB sliders when `filterEffect` (12) > 0.

Note the asymmetry between the ports: `recipe.py` declares ranges 13–18 as `-200..200`
each and validates the raw bytes independently, which is harmless (any byte of a valid
16-bit value falls inside −200..+200 after `s8`), but it means the Python `validate()`
does not actually enforce the 16-bit range. Only the Dart/JS UI does.

### 5.4 Validation rule for grain

If `grain` (21) is 0, then `grainSize` (22) and `grainStrength` (23) are inactive and
legitimately carry 0 — which is outside their nominal 1..3 range. Both `validate()`
implementations special-case this and do not report it as an error.

---

## 6. Reading a recipe

Per slot, two GATT reads. That is the whole operation.

```
1. read  <name characteristic>   -> raw bytes
2. strip trailing 0x00, decode UTF-8   -> name
3. read  <param characteristic>  -> 56 bytes
4. Recipe.from_block(block, name)
```

Decoding the block:

```python
version = block[0]          # 0x01
base    = block[2]          # byte2
n       = block[3]          # pair count
p = 4
for _ in range(n):
    params[block[p]] = s8(block[p+1])
    p += 2
```

Everything past `4 + 2N` is padding and is ignored. The decoder trusts `N` rather than
scanning for the padding, which matters because a value of `0x00` is a perfectly legal
parameter value and is indistinguishable from padding.

### Name-decoding detail

The name characteristic returns a fixed-size buffer with trailing NULs. Both ports strip
them, but differently:

- Python (`gr.py`, `server.py`): `.rstrip(b"\x00")` then `decode("utf-8", "replace")`
- Dart (`camera_ble.dart`): `takeWhile((b) => b != 0)` then `utf8.decode(allowMalformed: true)`

Dart cuts at the **first** NUL, Python strips only from the **end**. Both are safe for
real names; they would differ only on an embedded NUL, which the camera never produces.

### Reference implementations

| Where | Entry point |
|---|---|
| CLI (Linux) | `gr.py read` / `gr.py save <slot> <file>` |
| Web backend | `BLE.read_slots()` in `server.py` → `GET /api/slots` |
| Flutter app | `CameraService.readSlots()` in `app/lib/camera_ble.dart` |

All three read all three slots in one pass and return `Recipe` objects.

---

## 7. Writing a recipe

Also two GATT writes. **No commit, no trigger, no acknowledgement characteristic** — the
GATT table contains nothing of the sort, and the official app writes nothing else. The
write takes effect immediately.

```
1. write <param characteristic> <- 56-byte block   (Write Request, with response)
2. write <name characteristic>  <- UTF-8 name      (Write Request, with response)
```

Both writes use **Write Request** (`response: true` / `withoutResponse: false`), not
Write Command. This is what gives you the 0x80 rejection instead of silent failure.

### 7.1 Write order

The order is **not** enforced by the camera; both work. The implementations differ, on
purpose:

| Implementation | Order | Rationale |
|---|---|---|
| GR World app (HCI capture) | name → params | what the official app does |
| `gr.py` | name → params | mirrors the official app |
| `server.py`, `camera_ble.dart` | **params → name** | if the second write fails, the slot keeps a stale name rather than showing a new name over old settings |

⚠️ The comment in `app/lib/camera_ble.dart:197` says params-first is *"wie in der
Referenz-App"* (like the reference app). That is inaccurate — the reference app writes
the name first. The code is fine; only the comment misstates the reason. `server.py:85`
states the real reason correctly.

### 7.2 Encoding the block

```python
out = [version & 0xFF, 0x00, base & 0xFF, len(params) & 0xFF]
for pid in sorted(params):            # ascending ID order
    out += [pid & 0xFF, u8(params[pid])]
assert len(out) <= 56
out += [0x00] * (56 - len(out))
```

### 7.3 Empty name

The Flutter app substitutes `"GRade"` when the name is blank or whitespace, so a slot
never ends up nameless. The Python side writes whatever it is given.

### 7.4 Why a write gets rejected (app error 0x80)

In order of likelihood:

1. **`byte2` does not match the parameter roster.** The overwhelmingly common cause.
   Start from a `base_*.json` template; do not hand-assemble a parameter set.
2. **A value is outside the camera's accepted range** for that parameter.
3. **Not bonded**, or the bond was silently dropped — usually surfaces as
   *Insufficient Authentication* instead.

The camera gives no diagnostic detail. If a write is rejected, read the slot back and
compare.

### 7.5 Verifying a write

`gr.py load` reads the slot back after writing and compares both name and re-encoded
block, printing `OK` or `ABWEICHUNG!` (deviation). The Flutter app calls `_readSlots()`
again after a successful write so the UI reflects what is actually on the camera. This
read-back is the only trustworthy confirmation available.

---

## 8. Slot persistence — the U-mode trap

**Verified on the camera, and independently confirmed by a Ricoh forum thread. This is
not an app bug** — our writes are byte-identical to the GR World app's.

| Mode dial position | Behaviour |
|---|---|
| **P / Av / Tv / M** | The three custom slots persist. Written recipes survive. |
| **U1 / U2 / U3** | Written recipes are **temporary**. Turning the mode dial makes the camera reload the U snapshot from flash and your BLE-written profile is gone. |

To make a recipe stick in a U mode, it must be committed **on the camera** after every
change:

**Verified against the camera screen (photo, 2026-07-15.)** The menu tab is `C`, the
section is `1 User Mode`, and it holds exactly these items:

```
MENU → C → 1 User Mode
             Assign User Mode Dial     ← bind a box to a U position
             Save User Mode Box        ← commit current settings (this is the one)
             Load User Mode Box
             Rename User Mode
             Clear User Mode Assign.
```

Note the spelling: **"Assign User Mode Dial"** — earlier notes said "Usermode", which is
wrong.

The app surfaces this as a hint (`slotHint` in `index.html`, DE/EN, including the menu
path; the Flutter camera tab shows the equivalent U-mode note, targeted at the actual
dial position since §8.1).

> **A search caveat worth remembering.** These menu strings return *zero* web results, and
> that was briefly mistaken for evidence that the notes had invented them. Camera menu text
> lives only on the display — it is not indexed anywhere. **"Not findable online" says
> nothing about a camera menu.** The only authority is the camera screen. (The pairing path
> in `README.md` *was* genuinely wrong — see below — which is what prompted the doubt.)

### 8.1 Reading the mode dial over BLE (verified 2026-07-15)

The dial position is readable, which lets an app tell **which three slots it is looking
at** and warn only when the warning actually applies.

```
Characteristic : a3c51525-de3e-4777-a1c2-699e28736fcf   (2 bytes)
Byte 0         : exposure mode   0=P, 1=Av, 2=Tv, 3=M, 19=SN (Snap)
Byte 1         : user slot       0 = none, 1..3 = U1..U3
```

| Dial | Byte 0 | Byte 1 |
|---|---|---|
| P | 0 | 0 |
| Av | 1 | 0 |
| Tv | 2 | 0 |
| M | 3 | 0 |
| SN | 19 | 0 |
| U1 | *stored mode* | 1 |
| U2 | *stored mode* | 2 |
| U3 | *stored mode* | 3 |

**Byte 0 is not reliable for U positions**: it reports the exposure mode *saved inside
that user profile*, which differs per photographer (on the test camera U1/U2 held M and
U3 held P). **To answer "am I in a U mode?", read byte 1 only.**

The characteristic supports **notifications** — subscribe instead of polling, and the
camera pushes dial changes as they happen (firmware has
`ShootingServicePropertyChangedNotifyThread`). Turning the dial also swaps the slot
contents, so a client should re-read the slots on a dial change.

Method: read every readable characteristic before and after turning the dial and diff
(`app/lib/dial_scan.dart`). Confirmed by prediction — Av, U2 and U3 values were called
in advance and matched. Related characteristics found the same way:

| Characteristic | Meaning |
|---|---|
| `f662dcd8-ac6e-4e02-a4b2-ce92cd44c7c3` | list of available exposure modes (count byte + entries) |
| `0187979828-ee4d-9792-c9fd-249c88bbcc` | **exposure-compensation list** — count `0x1f`, then −15..+15 in 1/3 EV. Empty in M, populated in P/Av/Tv |
| `9c83df56-fd93-4639-8ca7-857bb7b3ca3d` | ISO list (count + uint32 LE values) |
| `b355330d-4adc-4434-a222-7b91404b4788` | shutter-speed list — populated only in Tv/M |
| `4866f4a9-2c83-457b-b393-b9535e1447e5` | aperture list — populated only in M |
| `3911f22d-9771-479d-b2b9-f729d9baf9dc` | focus mode (17 in SN, 10 otherwise) — probable |

> **Lesson.** `exposureMode` appears only in `webapid` (the HTTP daemon), not in the BLE
> daemon `bled` — yet the characteristic exists. **"Not in the bled strings" does not mean
> "not exposed over BLE."** The same was true of exposure compensation (§9). Do not treat
> a firmware symbol search as proof of absence on the wire.

---

## 9. What is NOT in the recipe

The recipe block covers Image Control only. The app models three separate tiers, and it
is important not to confuse them.

### 9.1 `params` — the block (BLE, transferable)

Everything in §5. Written to the camera.

### 9.2 `live` — camera settings (HTTP only, transferable over Wi-Fi)

White balance mode, ISO, aperture, shutter, exposure compensation. Stored in the recipe
JSON under `live` (`WBMode`, `sv`, `av`, `tv`, `xv`), **never** part of the 56-byte
block. Applied via:

```
PUT http://192.168.0.1/v1/params/camera
Content-Type: application/x-www-form-urlencoded

WBMode=cloud&sv=400
```

Rules discovered the hard way: PUT only (POST → 404), form-encoded only (JSON → 400),
`/camera` only, and the parameter name must exactly match a field from `/v1/props`
(`whiteBalance` → 400).

Partial BLE coverage also exists (ISO at `206bd02c-…` as sint32, WB mode at
`2361f4ff-…` as sint8, 12 of 17 WB modes settable), but HTTP covers all 17 modes and is
the preferred path.

### 9.3 `manual` — the memo pad (not transferable at all)

Highlight correction, shadow correction, peripheral illumination, high-ISO NR, WB
compensation (A–B / G–M grid, ±14), free-text notes. These are stored in the recipe JSON
so a recipe is self-documenting, and the UI clearly marks them as **not transferable** —
they must be dialled in by hand on the camera.

**WB compensation is definitively unreachable by any remote interface.** This was closed
out at the firmware level: the rootfs of firmware 1.11 was unpacked (Ricoh's custom LZ
container, cracked with the community tool `gr_unpack.py`), and the HTTP daemon
`/usr/bin/webapid` turned out to be an unstripped Crow C++ server. Its complete list of
settable properties is:

```
av, sv, tv, xv, reso, effect, channel, datetime, wb_mode, exposure_mode,
focus_setting, metering_mode, capture_mode, still_format, still_size,
movie_size, shoot_mode, operation_mode, auto_resize, geo_tagging,
power_off_transfer, ble_enable_condition
```

`wb_mode` is the only white-balance property. A rootfs-wide symbol search across
`camctld`, `webapid`, `bled`, `libcamera-controller.so` and `libcmfwk.so` found only
`WhiteBalance`, `WhiteBalanceList`, `WhiteBalanceSupported` — no compensation/shift/
offset property anywhere in the internal gRPC property system. **Custom firmware would
not help either.** Topic closed on all levels.

---

## 10. Recipe JSON file format

What `Recipe.to_dict()` / `toMap()` produce and `from_dict()` / `fromMap()` consume. This
is the interchange format for saved recipes, exports, QR codes and the base templates.

```json
{
  "name": "Kino Gelb",
  "version": 1,
  "base_effect_id": 14,
  "base_effect": "col_custom1",
  "base_name": "Kino Gelb",
  "base_name_en": "Cinema Yellow",
  "params": { "0": 0, "1": 3, "2": -1, "3": 3, "4": 0, "5": -2,
              "6": -3, "8": 1, "19": 0, "20": 0, "21": 0, "22": 0, "23": 0 },
  "live":   { "WBMode": "cloud", "sv": 400, "xv": 0.3 },
  "manual": { "wbAB": 0, "wbGM": 0, "highlight": "auto", "notes": "…" }
}
```

| Field | Authoritative? | Notes |
|---|---|---|
| `name` | ✅ | written to the name characteristic |
| `version` | ✅ | becomes byte 0; always 1 so far |
| `base_effect_id` | ✅ | **this is `byte2`** |
| `base_effect` | ❌ | derived, wrong for codes ≥ 6 — see §4.3 |
| `base_name` / `base_name_en` | ❌ | derived display labels |
| `params` | ✅ | keys are **strings** in JSON, ints in memory |
| `live` | ✅ | HTTP-only settings, never in the block |
| `manual` | ✅ | memo only, never transferred |

`params` keys are JSON strings (`"0"`, `"1"`) and are parsed back to ints on load. On
write, `to_block()` ignores `live` and `manual` entirely — that separation is what makes
the memo fields safe to store in the same file.

---

## 11. Base templates

`recipes/base_*.json` holds one captured template per base — 14 files, mirrored into
`app/assets/bases/` for the Flutter app. Each was captured off the real camera with
`capture.py <slot> <name> <file>`, so the parameter roster in each file is exactly what
the camera expects for that base code.

Loading:

- Web app: `server.py` reads them from `recipes/`
- Flutter: `app/lib/bases.dart` → `loadBases()` reads the assets and sorts by
  `baseDisplayOrder = [0, 1, 6, 13, 14, 15, 16, 8, 7, 9, 2, 3, 4, 5]` (colour bases
  first, monochrome last)

`recipe.py` additionally declares `SELECTABLE_EFFECTS = [14] + range(0,14)` and
`DEFAULT_BASE = 14` — base 14 (Cinema Yellow) is offered first because it carries the
fullest colour parameter set including toning and grain.

The user-facing flow is therefore always: **pick a base → the editor shows exactly that
base's parameters → adjust values → write.** The roster is never edited.

---

## 12. Implementation map

| File | Role |
|---|---|
| `recipe.py` | Codec, labels, ranges, enums, base names — the Python source of truth |
| `app/lib/recipe.dart` | 1:1 port of `recipe.py`. **Changes must be mirrored in both.** |
| `app/test/recipe_test.dart` | 11 tests incl. the exact Cinema Yellow round-trip |
| `gr.py` | Linux CLI: `read` / `save` / `load` / `setname` |
| `capture.py` | Captures a base template off a slot |
| `slotvals.py`, `slotinfo.py` | Slot inspection helpers |
| `server.py` | Flask backend (port 8770), BLE manager, `/api/slots` |
| `index.html` | Web UI (single file, serves both server.py and hybrid.py) |
| `hybrid.py` | BLE + HTTP mode (port 8772), serves the same `index.html` |
| `app/lib/camera_ble.dart` | Flutter BLE layer (flutter_blue_plus **1.x**, not 2.x) |
| `app/lib/camera_stub.dart` | Web stub — conditional export keeps `flutter build web` working |
| `app/lib/camera_screen.dart` | Camera tab: connect, battery, read slots, write slot |
| `app/lib/bases.dart` | Base template loader |

### Robustness notes worth keeping

BLE connections to this camera are transiently flaky, so `camera_ble.dart` wraps
everything:

- `_connectWithRetry(attempts: 5)` with backoff — the camera frequently refuses the
  first connect
- `_withRetry(op, attempts: 3)` around every read/write — reconnects if the link dropped
  between actions
- `requestConnectionPriority(high)` and a `createBond()` fallback if the device reports
  itself unbonded

`server.py` serialises all BLE operations behind a lock (`BLE.run`) because the camera
tolerates exactly one connection at a time — this is also why `grserver` must be stopped
before running the live/exploration tools.

---

## 13. Verification status

| Claim | Status |
|---|---|
| Recipes travel over BLE, two writes, no commit | ✅ proven by HCI capture of GR World |
| 56-byte block layout | ✅ verified, exact round-trip of a real block |
| Base codes 0–9, 13–16 | ✅ captured from the camera |
| Base codes 10, 11, 12 | ❓ never observed |
| `byte2` ≠ effectList index | ✅ confirmed (they agree only for 0–5) |
| Ranges −4..+4, 7 (0..5), 8 (1..3), 12 (0..4), 21–24 | ✅ read off the camera |
| Ranges for 9, 10, 11, 25 | ⚠️ estimated, not verified |
| Mono toning enum order | ✅ verified |
| 16-bit RGB filters | ✅ verified incl. round-trip test |
| Slot persistence P/Av/Tv/M vs U1–U3 | ✅ verified on camera + forum-confirmed |
| Error 0x80 on roster mismatch | ✅ observed repeatedly |
| WB compensation unreachable remotely | ✅ proven at firmware source level |
| End-to-end on Android (GRade app) | ✅ connect, read, write, load all confirmed on a real tablet |
| Windows BLE | ❌ known broken (stack limitation) |
| iOS | ❓ not yet tested |

---

## 14. Dead ends — do not re-investigate

- **`/v1/imgctrl` (Wi-Fi)** — always 404. Recipes are not written over Wi-Fi. The string
  exists in the app binary but the endpoint does not exist in the firmware.
- **BLE blob channel `fe3a32f8` in service `0f291746`** — looked like a file-transfer
  channel, is not used for recipes.
- **WB compensation over any remote interface** — see the caveat below; the empirical
  evidence stands, the firmware argument does not.
- **Firmware modding to expose WB compensation** — the property does not exist internally;
  a mod would have nothing to call. Also: no repack tool, likely signed, brick risk.

  > **Re-tested 2026-07-15** with the diff scanner (`app/lib/dial_scan.dart`), because §8.1
  > destroyed one of the three pillars this conclusion rested on: `exposureMode` is absent
  > from the `bled` strings yet **is** exposed over BLE. A firmware symbol search is
  > therefore **not** proof of absence on the wire.
  >
  > Result: changing the A-B/G-M shift on the camera (applied, not just cursor movement)
  > changed **none** of the 71–77 readable characteristics — reproducing the old Linux test,
  > which only covered 56. Together with `/v1/props` (a complete JSON dump with no such
  > field), WB compensation is not remotely *readable*.
  >
  > **Known limit of the method:** the scan reads only *readable* characteristics. A
  > **write-only** characteristic would be invisible to it — and setting, not reading, is
  > what an app would want. Ruling that out would mean blind-writing to unknown
  > characteristics with an unknown value encoding; the risk of corrupting camera state
  > outweighs the payoff. So: *not readable* is proven, *not settable* is merely very
  > likely.
- **Windows BLE** — bond works, connection cannot be held. Not fixable from our side.
- **WebSocket `/v1/changes`** — connects but never pushes setting changes.

---

## 15. References

- `github.com/dm-zharov/ricoh-gr-bluetooth-api` — GR III GATT documentation; the GR IV
  reuses the same UUIDs, which is how the shooting/camera/WLAN services were identified
- `github.com/yeahnope/gr_unpack` — unpacks Ricoh's custom firmware container (a GR III
  tool that works on GR IV firmware)
- `github.com/hhornbacher/gr3x-fw-hack` — firmware extraction pipeline
- `lucas.io/grid/making-of-grid` — the BLE-as-control-channel / Wi-Fi-as-data-channel
  architecture

