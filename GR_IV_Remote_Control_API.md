# Ricoh GR IV — Remote Control API

A reverse-engineered reference for controlling the Ricoh GR IV (firmware **v1.11**)
over its two wireless interfaces: **Bluetooth Low Energy (BLE)** and **Wi-Fi (HTTP)**.
Compiled from the camera's firmware binaries and confirmed against a physical camera.

---

## 1. How it works — the big picture

The GR IV has two cooperating radios:

- **BLE = the control channel.** Always-on, low power. Used for pairing, waking the
  camera, reading status, switching the Wi-Fi access point on/off, and writing the
  three custom **Image Control recipe slots**. A handful of live shooting settings
  (ISO, white balance, shutter, exposure compensation) can also be set over BLE.
- **Wi-Fi = the data channel.** The camera runs an HTTP server (`http://192.168.0.1`,
  server banner "Crow"). Used for the full settings model, live view (MJPEG), photo
  browsing/download, and remote shutter/focus.

The usual flow: connect over BLE → optionally enable the camera's Wi-Fi AP over BLE →
join that Wi-Fi network → talk to the HTTP API. BLE alone is enough for recipes and a
few live settings; Wi-Fi is needed for live view, photos and the complete settings set.

Internally the camera firmware is embedded Linux (Yocto). The HTTP daemon and the BLE
daemon are thin front-ends that forward requests to a camera-core process over a local
gRPC/protobuf channel. Only what those two daemons expose is remotely reachable.

---

## 2. Bluetooth LE

### 2.1 Device identity & pairing

- Device Information service (`0x180a`) reports: Model `RICOH GR IV HDF`,
  Firmware `1.11`, Manufacturer `RICOH IMAGING COMPANY, LTD.`, plus serial number.
- BLE name follows the pattern `GR_Hxxxxxx`; advertised service UUID
  `9a5ed1c5-74cc-4c50-b5b6-66a48e7ccff1`; manufacturer ID `0x065F` (Ricoh).
- **All Ricoh vendor characteristics require pairing/bonding.** Without a bond you get
  "Insufficient Authentication". Standard services (GAP/GATT/Device Info) are readable
  unbonded.
- Pairing uses **numeric comparison** (`CONFIRM_PIN_MATCH`): a 6-digit code shown on
  both sides that must be confirmed on the camera and the client. Simple "just works"
  auto-pairing is not sufficient — the client must perform confirmed pairing and give
  the user time to accept on the camera.

### 2.2 Service map

| Service UUID (prefix) | Role |
|---|---|
| `9f00f387-…` | **Shooting / status service** (largest; status, live settings, recipe slots, shutter trigger) |
| `4b445988-…` | **Camera service** (power, battery, date/time, geo-tag, storage, file-transfer list) |
| `f37f568f-…` | **Wi-Fi control** (turn the camera AP on/off, read SSID & passphrase) |
| `0f291746-…` | Blob / data-transfer channel (write-only characteristic present) |
| `180a` | Standard Device Information (model/serial/firmware) |

**Wi-Fi control characteristics** (in `f37f568f-…`) — this is how a client enables the
camera's access point without touching the camera:

| Characteristic (prefix) | Meaning | Values |
|---|---|---|
| `9111cdd0-…` | Network Type | `0` = off, `1` = AP mode on |
| `90638e5a-…` | SSID | UTF-8, read/write |
| `0f38279c-…` | Passphrase | UTF-8, read/write |
| `63bc8463-…` | Wi-Fi channel | — |

Typical sequence to bring up Wi-Fi: read SSID + passphrase, write Network Type = `1`,
then join that network from the client and switch to the HTTP API.

### 2.3 Live shooting settings over BLE

The BLE daemon exposes get/set handlers for these (they take effect live; they are
**not** persisted as part of a recipe):

| Setting | Characteristic (in `9f00f387-…`) | Encoding | Read | Write |
|---|---|---|---|---|
| ISO | `206bd02c-78b2-42c4-820a-cf30e0963909` | sint32, little-endian (`0` = Auto) | ✔ | ✔ |
| White Balance | `2361f4ff-2c7e-4fc5-876b-f9b0efbc06fd` | sint8 (mode code) | ✔ | ✔ |
| Shutter speed | (shooting service) | — | ✔ | ✔ |
| Exposure compensation | (shooting service) | — | ✔ | ✔ |
| Shooting/drive mode | (shooting service) | — | ✔ | ✔ |
| Focus mode | (shooting service) | — | — | ✔ |
| Autofocus status | (shooting service) | — | ✔ | — |
| Shutter trigger | `559644b8-…` (Operation Request) | `1` = start, `2` = stop (param `1` = AF) | — | ✔ |

White-balance mode codes (sint8): `0` Auto, `1` Daylight, `2` Shade, `3` Cloudy,
`4` Tungsten, `6`–`9` Fluorescent (daylight/white/cool/warm), `10` Color Temperature,
`13` Manual, `16` CTE. The "auto warm / auto white" and "custom" WB modes are readable
but **cannot** be set over BLE — use the HTTP API for those.

### 2.4 Image Control recipe slots (the main BLE feature)

The camera stores **three custom Image Control profiles**. Over BLE each slot is two
characteristics in the `9f00f387-…` service: a **name** (UTF-8) and a **56-byte
parameter block**. Writing a recipe = write the name, then write the block.

| Slot | Name characteristic | Parameter characteristic |
|---|---|---|
| 1 | `30adb439-1bc0-4b8e-9c8b-2bd1892ad6b0` | `3e0673e0-1c7b-4f97-8ca6-5c2c8bc56680` |
| 2 | `e799198f-…` | `0936b04c-…` |
| 3 | `df77dd09-…` | `cd879e7a-…` |

The three slots represent the custom profiles of the **current mode dial position**.
In P/Av/Tv/M they are persistent; in the U1/U2/U3 user modes a BLE write is temporary
(the camera reloads the stored snapshot when the dial is turned) until the user mode is
re-saved on the camera itself.

#### Parameter block format (56 bytes)

```
Byte 0 : 0x01            version
Byte 1 : 0x00
Byte 2 : base_code       base image-control type (see table below)
Byte 3 : N               number of parameter pairs that follow
Byte 4.. : N × ( paramId:uint8 , value:int8 )
rest   : 0x00 padding up to 56 bytes
```

`base_code` must exactly match the complete parameter set expected for that base,
otherwise the camera rejects the write with application error `0x80`.

**`base_code` (byte 2) values:**

| Code | Base | Code | Base |
|---:|---|---:|---|
| 0 | Standard | 8 | Retro |
| 1 | Vivid | 9 | HDR Tone |
| 2 | Monochrome | 13 | Negative Film |
| 3 | Soft Monochrome | 14 | Cinema Yellow |
| 4 | Hard Monochrome | 15 | Cinema Green |
| 5 | Hi-Contrast B&W | 16 | Cross Process |
| 6 | Positive Film | | |
| 7 | Bleach Bypass | | |

**Parameter IDs** (value is signed int8 unless noted):

| ID | Name | Meaning | Range | Notes |
|---:|---|---|---|---|
| 0 | saturation | Saturation | −4…+4 | |
| 1 | hue | Hue | −4…+4 | |
| 2 | highLowKey | High/Low Key | −4…+4 | |
| 3 | contrast | Contrast | −4…+4 | |
| 4 | contrastHighlight | Contrast (highlights) | −4…+4 | |
| 5 | contrastShadow | Contrast (shadows) | −4…+4 | |
| 6 | sharpness | Sharpness | −4…+4 | |
| 7 | toningMono | Mono toning | 0…5 | enum: `off S R G B P` |
| 8 | toning | Color toning | 1…3 | |
| 9 | bleachBypass | Bleach Bypass strength | −4…+4 | range estimated |
| 10 | retro | Retro effect | −4…+4 | range estimated |
| 11 | hdrToning | HDR toning | −4…+4 | range estimated |
| 12 | filterEffect | B/W filter effect | 0…4 | 0 = off; enables IDs 13–18 |
| 13 | filterRHigh | B/W filter Red — high byte | — | see note |
| 14 | filterRLow | B/W filter Red — low byte | — | see note |
| 15 | filterGHigh | B/W filter Green — high byte | — | see note |
| 16 | filterGLow | B/W filter Green — low byte | — | see note |
| 17 | filterBHigh | B/W filter Blue — high byte | — | see note |
| 18 | filterBLow | B/W filter Blue — low byte | — | see note |
| 19 | shading | Vignetting / shading | −4…+4 | |
| 20 | clarity | Clarity | −4…+4 | |
| 21 | grain | Grain on/off | 0…1 | 0 = off (22/23 then inactive) |
| 22 | grainSize | Grain size | 1…3 | |
| 23 | grainStrength | Grain strength | 1…3 | |
| 24 | hdrLevel | HDR tone level | 1…3 | |
| 25 | crossProcess | Cross-process variant | −4…+4 | range estimated |

> **B/W filter (IDs 13–18):** each color channel is one signed 16-bit value split
> across two params, `value = int16(low | (high << 8))`, range **−200…+200 %**.
> Red = (14 low, 13 high), Green = (16 low, 15 high), Blue = (18 low, 17 high).
> Only relevant when `filterEffect` (12) > 0.

**Worked example** — the "Cinema Yellow" base with a few tweaks, as sent on the wire:

```
01 00 0e 0d 00 00 01 03 02 ff 03 03 04 00 05 fe 06 fd 08 01
13 00 14 00 15 00 16 00 17 00 00 … (padding to 56)
```

Decoded: version 1, base_code `0x0e` (Cinema Yellow), N = `0x0d` (13 pairs):
saturation 0, hue +1, highLowKey +3, contrast +2, contrastHighlight +3,
contrastShadow −2, sharpness −3, toning 1, and the B/W-filter bytes all 0.

---

## 3. Wi-Fi HTTP API

Base URL `http://192.168.0.1` once joined to the camera's access point. Responses are
JSON; errors carry `errCode` / `errMsg`.

### 3.1 Reading and writing settings

- **Read everything:** `GET /v1/props` returns one large JSON object with every current
  value and, for each enumerable setting, a matching `…List` field of allowed values.
- **Write settings:** `PUT /v1/params/{camera|device|lens}` with a
  **`application/x-www-form-urlencoded`** body. Rules that matter:
  - Method must be **PUT** (POST → 404), body must be **form-encoded** (JSON → 400).
  - Parameter names must match the property names exactly (a wrong name → 400).
  - Combine several settings with `&`.

```bash
# read state
curl http://192.168.0.1/v1/props

# set white balance to cloudy and ISO to 400 in one request
curl -X PUT http://192.168.0.1/v1/params/camera \
     --data 'WBMode=cloud&sv=400'
```

### 3.2 Endpoint list

| Method | Path | Purpose |
|---|---|---|
| GET | `/v1/ping` | Reachability / handshake |
| GET | `/v1/props` | Full state + all value lists |
| PUT | `/v1/params/camera` | Set camera settings |
| PUT | `/v1/params/device` | Set device settings |
| PUT | `/v1/params/lens` | Set focus |
| POST | `/v1/camera/shoot` | Trigger shutter |
| POST | `/v1/camera/shoot/cancel` | Cancel exposure (bulb / interval) |
| POST | `/v1/camera/shoot/compose` | Composed capture (multi-exposure / composite) |
| POST | `/v1/lens/focus` | Drive focus (AF/MF) |
| GET | `/v1/liveview` | MJPEG live stream (see §3.5) |
| GET/WS | `/v1/changes` | Websocket for state-change events |
| GET | `/v1/photos` | Photo list |
| GET | `/v1/photos/infos` | Photo metadata (bulk) |
| GET | `/v1/photos/<dir>/<file>/info` | Metadata of one photo |
| GET | `/v1/photos/<dir>/<file>` | Download a photo (JPEG or DNG) |
| GET | `/v1/photos/<dir>/<file>/imgctrl` | Read a photo's embedded Image Control recipe |
| POST | `/v1/photos/<dir>/<file>/transfer` | Start a transfer (full / thumb) |
| GET | `/v1/transfers` | Transfer status |
| GET | `/v1/logs/camera` | Shutter count (monthly / total) |
| GET | `/v1/debug/revisions` | Revision / debug info |
| POST | `/v1/configs/firmware/prepare` | Prepare a firmware update |
| POST | `/v1/configs/firmware` | Execute a firmware update |
| POST | `/v1/configs/firmware/cancel` | Cancel a firmware update |
| POST | `/v1/device/wlan/finish` | End the Wi-Fi session |
| POST | `/v1/device/finish` | End the session / connection |

### 3.3 Camera parameters (`PUT /v1/params/camera`)

Every parameter here is readable (value + `…List` in `props`) and writable.

| Parameter | Values |
|---|---|
| `WBMode` | see §3.6 |
| `effect` | see §3.7 (image-control base) |
| `exposureMode` | P/Av/Tv/M/… (numeric; `exposureModeList`) |
| `meteringMode` | `multi` / `center` / `spot` / `highlight` |
| `captureMode` / `shootMode` | see §3.8 (drive / self-timer / bracket / interval) |
| `stillSize` | from `stillSizeList` |
| `movieSize` | `4K30 4K24 FHD60p FHD30p FHD25p FHD24p HD60p HD30p HD25p HD24p` |
| `onePushBracket` | on / off |
| `sv` | ISO — `svList` (100…204800, `auto`, `auto_hi`) |
| `av` | Aperture — from `avList` |
| `tv` | Shutter speed — from `tvList` |
| `xv` | Exposure compensation — `xvList` (±EV, 1/3 steps) |
| `reso` | `<x> <y>` (live-view resolution) |

Also present in `props`: `crop_size` (`35mm` / `50mm` in-camera crop), `aspect_ratio`,
`cameraOrientation` (`none positive reverse vertical_right vertical_left`).

### 3.4 Device & lens parameters

`PUT /v1/params/device`: `channel`, `datetime`, `operationMode`, `stillFormat`
(`jpeg` / `rawdng` / `rawpef`), `geoTagging`, `gpsInfo`, `bdName`, `bleEnableCondition`
(`disable` / `enableAnytime` / `enablePowerOn`), `ssid`, `powerOffTransfer`,
`autoResize`. Read-only device fields: `battery`, `manufacturer`, `model`, `serialNo`,
`firmwareVersion`, `macAddress`, `storages` (equipped / writable / remain / numOfPhotos
/ numOfMovies / recordableTime).

`PUT /v1/params/lens`: `focusMode`
(`manual snap multiauto multiauto_center spot pinpoint tracking continuous zone_select`),
`focusSetting` (e.g. snap distance). Read-only focus status: `focused`,
`focusEffectiveArea`, autofocus status.

### 3.5 Live view

`GET /v1/liveview` returns a **motion-JPEG** stream:
`Content-Type: multipart/x-mixed-replace; boundary=--boundary`, each part
`Content-Type: image/jpeg`. An optional `maxfps` query parameter caps the frame rate.
Decode it as a standard MJPEG stream (split on the boundary, render each JPEG).

### 3.6 White-balance modes (`WBMode`)

All of these are settable live over HTTP (the HTTP API covers modes BLE cannot):

```
auto  autoWarm  autoWhite  multiAuto  daylight  shade  cloud
daylightFluorescent  dayWhiteFluorescent  coolWhiteFluorescent  warmWhiteFluorescent
tungsten  incandescent  colorTemp1  colorTemp2  colorTemp3  manual1  manual2  manual3
```

### 3.7 Image-control bases (`effect`)

```
col_vivid  efc_monochrome  efc_softMonochrome  efc_hardMonochrome  efc_highContrast
efc_posiFilm  efc_bleachBypass  efc_retro  efc_HDRTone  col_custom1  col_custom2
col_custom3  efc_negaFilm  efc_cineYellow  efc_cineGreen  efc_crossProcess
efc_monoStandard  efc_monoSoft  efc_monoHighContrast  efc_monoGrainy  efc_monoSolid
efc_monoHDRTone
```

`effect` selects only the base look. The fine parameters of a base (saturation,
contrast, sharpness, grain, B/W filter, …) are not HTTP fields — they live in the
56-byte BLE recipe block (§2.4).

### 3.8 Drive / shoot modes (`captureMode` / `shootMode`)

Base modes, each with self-timer (`Self2s` / `Self10s` / `Self12s`) and remote
(`Remocon` / `Remocon3s`) variants:

```
single
continuousH / continuousM / continuousL          (burst fast / medium / slow)
interval / intervalComp / intervalMovie          (interval / interval-composite / timelapse video)
multiExp / multiExpContinuous                     (multiple exposure)
bracket / motionBracket / dofBracket              (exposure / motion / depth-of-field bracket)
mirrorUp                                           (mirror-up pre-release)
starstream                                         (star trail)
```

GPS astrotracer mode: `ASTRO_TRACER_TYPE3`.

### 3.9 Other useful reads

- **Shutter count:** `GET /v1/logs/camera` → monthly and total capture counts.
- **Read a photo's recipe:** `GET /v1/photos/<dir>/<file>/imgctrl` returns the Image
  Control settings baked into an existing shot — useful for recovering "how was this
  taken".
- **Camera state (in `props`):** `capturing`, `stateStill`, `stateMovie`, `countDown`,
  `shotsTotal`, `shotsCurrent`, and live-view metadata (`liveState`, `orientation`,
  `aspectRatio`, `resolution`).

---

## 4. Known limitations

- **White-balance compensation (the A–B / G–M fine shift) is not exposed on any remote
  interface.** It has no HTTP property and no BLE handler; it exists only as an
  on-camera menu setting. (Verified across the HTTP daemon, the BLE daemon, and the
  camera-core binary.)
- Auto-warm / auto-white and the custom white-balance modes are **read-only over BLE**;
  set them over HTTP instead.
- Values are validated by the camera: an unknown parameter name and a valid name with
  an out-of-range value both return the same generic `400 Bad Request`, so you cannot
  probe for hidden parameters by response alone.

---

## 5. Quick reference — what you can do

| Goal | Interface | How |
|---|---|---|
| Write a custom Image Control recipe | BLE | Recipe slot: name char + 56-byte block (§2.4) |
| Set ISO / WB / shutter / exposure comp live | BLE or HTTP | BLE chars (§2.3) or `PUT /v1/params/camera` |
| Full settings model (all WB modes, drive modes, formats) | HTTP | `PUT /v1/params/camera` / `…/device` / `…/lens` |
| Live viewfinder | HTTP | `GET /v1/liveview` (MJPEG) |
| Remote shutter / focus | HTTP or BLE | `POST /v1/camera/shoot`, `/v1/lens/focus`; BLE Operation Request |
| Browse / download photos | HTTP | `/v1/photos…` |
| Recover a photo's recipe | HTTP | `GET /v1/photos/<dir>/<file>/imgctrl` |
| Shutter count | HTTP | `GET /v1/logs/camera` |
| Turn the camera Wi-Fi AP on | BLE | Wi-Fi control service (§2.2) |
| WB compensation remotely | — | Not possible (§4) |

---

## 6. Firmware internals (for the curious)

Provenance and internal layout, in case it helps someone dig further:

- Firmware is embedded Linux (Yocto/Poky **kirkstone 4.0.20**, ARM 32-bit).
- Three user-space daemons matter: **`webapid`** (the HTTP server, built on the "Crow"
  C++ framework), **`bled`** (the BLE GATT server), and **`camctld`** (the camera core).
- `webapid` and `bled` are thin front-ends. They forward requests to `camctld` over a
  local **gRPC/protobuf** channel (`RemoteCameraCommand/CameraCommandSync`,
  `RemoteInterfaceCommand`, Property/Status/ExpStatus event dispatchers,
  `RemoteSharedMemory`). Only what those two front-ends expose is remotely reachable.
- There is a `FactoryCmdset` (service/factory commands, `GPSetValue`/`GPSetData`) inside
  `camctld`, but it is **local only** — not exposed over Wi-Fi or BLE.

This is why probing for hidden features via HTTP/BLE is a dead end: if a setting has no
handler in `webapid`/`bled`, it is simply not reachable, regardless of what the core can
do. (This is how the "WB compensation is not remotely settable" conclusion in §4 was
reached — the handler does not exist in either front-end.)

