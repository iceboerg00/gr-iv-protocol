# Ricoh GR IV — Protocol (Reverse Engineering)

## >>> CRACKED: WRITING A RECIPE (over BLE, confirmed via Android HCI capture) <<<
Image Control ("recipe") is written over **Bluetooth** (NOT Wi-Fi/imgctrl!).
GR World makes exactly TWO ATT write requests, nothing else (no commit/trigger):

1. NAME  -> characteristic **30adb439-1bc0-4b8e-9c8b-2bd1892ad6b0** (service 9f00f387)
   = UTF-8 name, variable length, e.g. "Cinema (Yellow)"
2. PARAM -> characteristic **3e0673e0-1c7b-4f97-8ca6-5c2c8bc56680** (service 9f00f387)
   = 56-byte block:

   Byte0: 0x01 (version)
   Byte1: 0x00
   Byte2: base effect ID (effectList index; 0x0e=col_custom1 etc.)
   Byte3: number of parameter pairs N
   then N x (paramId:uint8, value:int8-signed)
   Rest: 0x00 padding up to 56 bytes

Example "Cinema (Yellow)": 01 00 0e 0d | 0000 0103 02ff 0303 0400 05fe 06fd 0801 1300 1400 1500 1600 1700 | 00...
 -> base=0x0e, 13 pairs: p0=0 p1=3 p2=-1 p3=3 p4=0 p5=-2 p6=-3 p8=1 p19=0 p20=0 p21=0 p22=0 p23=0

There are 3 name chars (30adb439/e799198f/df77dd09 = "Negative Film"/"Custom 2"/"Custom 3")
and 3 param chars (3e0673e0/0936b04c/cd879e7a) = the 3 custom slots. GR World wrote to
slot 1 (30adb439/3e0673e0). All are read+write -> reading (existing recipes) + writing possible.

paramId->name (from the ImageControlParameter enum, Blutter objs.txt):
  0=saturation 1=hue 2=highLowKey 3=contrast 4=contrastHighlight 5=contrastShadow
  6=sharpness 8=toning?(unconfirmed) 12=filterEffect(mono) 19=shading 20=clarity
  21=grain 22=grainSize 23=grainStrength
Values are int8 (signed). Ranges (read off the camera):
  0,1,2,3,4,5,6,19,20 = -4..+4  |  8(toning)=1..3  |  21(grain)=0/1(off/on)
  22(grainSize)=1..3  |  23(grainStrength)=1..3  |  12(filterEffect)=mono only, open
  ID 8 = toning CONFIRMED. When grain=0, grainSize/Strength are inactive (0).
VERIFIED: write test "REC-TEST" to slot 3 -> the camera displays it. Read+write end to end OK.
Tools: recipe.py (codec) + gr.py (BLE read/save/load/setname), running on a laptop over SSH.
HCI capture: btsnoop_hci.log (from bugreport_gr.zip), parser: btparse.py.

> Note on wire encoding: grain size/strength (and toning) are 0-based on the wire while the
> camera UI shows them 1-based. Writing a value one higher than the max gets rejected by the
> camera with ATT app error 0x80. filterEffect (12) only accepts 0/1 over BLE; the camera's
> named color filters are presets for the RGB filter parameters, not raw values.

## Architecture (from the GR World APK, Flutter/Dart libapp.so)
- **BLE = control channel**, **Wi-Fi = data channel**. Note: recipe writing turned out to be
  BLE (above), not Wi-Fi — an earlier hypothesis that Image Control ran over Wi-Fi HTTP was
  wrong (which is why a BLE diff on a saturation change showed nothing readable).
- The app is Flutter; uses `flutter_blue_plus` (BLE) + `bluez.dart`. Dev project "grapp".

## Pairing (prerequisite for everything)
- The camera requires **authenticated pairing (Numeric Comparison, kind=8)**, confirm the code on both sides.
- **The Windows BLE stack CANNOT hold the bonded connection** (connect->1s->camera disconnects).
- **Linux/BlueZ works flawlessly.** -> We work over SSH on a Linux laptop.

## Wi-Fi credentials (readable over BLE, service f37f568f)
- `9111cdd0` Network Type: 0=OFF, 1=AP mode  (write=turn AP on)
- `90638e5a` SSID (utf8), e.g.: `GR_XXXXXXX`
- `0f38279c` Passphrase (utf8): `[REDACTED — read from your own camera]`
- `63bc8463` Channel

## HTTP API (camera AP, http://192.168.0.1)
- `GET /v1/photos/` — photos
- **`/v1/imgctrl?storage=in`** (or `sd1`) — investigated for IMAGE CONTROL (recipes), but see below
  - all imgctrl variants returned 404; recipe writing is done over BLE (see top), not this endpoint
- `ws://192.168.0.1/v1/changes` — WebSocket changes
- `/v1/configs/firmware...`, `/v1/logs/camera?type=monthly_capture&date=`
- storage values: `in` (internal), `sd1` (SD card)

## Image Control data model (fields from libapp.so)
- recipe = **imageControlName + imageControlType + parameters[] + effect** (+ imageControlPath = file)
- custom slots: `custom1ImageControlData`, `custom2ImageControlData`, `custom3ImageControlData`
- parameter keys: `saturation`, `hue`, `contrast`, `contrastHigh`, `contrastLow`,
  `clarity`, `sharpness`, `toning`, `filterEffect`  (+ more)
- preset/effect names (from the SQLite schema): effect_efc_nega_film, posi_film, retro,
  bleach_bypass, cross_process, hdr_tone, high_contrast, monochrome, mono_grainy,
  soft_monochrome, hard_monochrome, cine_green, cine_yellow, col_vivid, col_custom1..3

## LIVE TEST RESULTS (Wi-Fi HTTP works!)
Laptop joined the camera Wi-Fi via `sudo nmcli` (192.168.0.8), camera = 192.168.0.1, server "Crow/0.1".
- `GET /v1/ping` -> 200 (JSON datetime)
- `GET /v1/props` -> 200, 5766B JSON. Contains effect="efc_negaFilm", effectList
  (off,col_vivid,efc_monochrome,...,col_custom1/2/3), storages[in,sd1 both writable].
  BUT: NO Image Control detail parameters in props.
- `GET /v1/photos` -> 200 (dirs/files)
- **all imgctrl variants -> 404** (GET/PUT/POST/OPTIONS):
  /v1/imgctrl?storage=in, /imgctrl?storage=in, /v1/imageControl*, /v1/imageControlWrite ...
- `/imageControl`, `imageControlWrite`, `imageControlDetail` = **Flutter app screens**, NOT endpoints.
- The only real /v1/ endpoints: photos, changes(ws), configs/firmware, fwDownload, getInformation, logs/camera.
- The only imgctrl string: `/imgctrl?storage=` (without /v1) -> but 404.

## RESOLVED: how a recipe is written
The 404s above ruled out the HTTP path. The Android HCI capture (see top) showed the answer:
recipes are written over BLE as two writes (name + 56-byte param block) to the custom slot
characteristics in service 9f00f387. The write-only blob channel `FE3A32F8` in service
`0F291746` was a red herring for this purpose.

## FLUTTER DECOMPILATION (Blutter) - SUCCESSFUL
Dart 3.11.1 app, obfuscated. Blutter build on the laptop (without sudo):
- ICU 78 dev extracted locally as .deb -> ICU_ROOT=/home/user/icu/prefix
- capstone 5.0.6 from source -> /home/user/cap/prefix, .pc includedir patched to include/capstone
- PKG_CONFIG_PATH=/home/user/cap/prefix/lib/pkgconfig, CC=gcc CXX=g++
- `venv/bin/python blutter/blutter.py apk/arm/lib/arm64-v8a out_blutter`
Output: pp.txt (pool/strings), objs.txt, blutter_frida.js, asm/ (decompiled functions, obfuscated).

Insights from the pool (pp.txt):
- `/imgctrl\?storage=` @ pp+0x2d720, belongs to class `xcb` (in asm/Juk.dart) / lib "Juk".
- HTTP method strings: GET@0x240, POST@0x1458, PUT@0x278f8, DELETE@0xb718.
- `http://192.168.0.1`@0xfdf0, `.../v1/photos/`@0x26168, `ws://.../v1/changes`@0x246a8.

## ASM MINING CONCLUSION (exhausted)
- Blutter did NOT decompile the async network logic; HTTP write details are not statically
  extractable. This is why the Android traffic capture (not static analysis) was the key that
  cracked recipe writing.
- recipe format: parameters appear only as display labels (Saturation/Sharpness/Clarity/Toning)
  plus the `ImageControlParameter.` enum and a `parameters` list. Confirmed via the HCI capture
  to be the 56-byte binary block documented at the top.
- The SQLite schema = usage statistics only (effect_* counters), not the format.

## References
- github.com/dm-zharov/ricoh-gr-bluetooth-api (GR III GATT, same UUIDs)
- lucas.io/grid/making-of-grid (BLE+Wi-Fi architecture)
- Play: com.ricohimaging.grworld
