# Ricoh GR IV — Firmware API Reference (v1.11)

Reconstructed from the unpacked firmware `Gr4_v111.zip` (2026-07-14).
Source: `fw_local/` (webapid, camctld, bled, … plus string dumps). System = Yocto/Poky
kirkstone 4.0.20, ARM 32-bit. HTTP server = Crow (`webapid`), BLE = `bled`,
camera core = `camctld` (gRPC + protobuf). Camera AP: `http://192.168.0.1`.

Legend: **R** = readable, **W** = writable. "R via props" = the current value appears in
the `GET /v1/props` JSON; the matching `…List` field enumerates the allowed values.

---

## 1. HTTP endpoints

| Method | Path | Purpose | R/W |
|---|---|---|---|
| GET | `/v1/ping` | Reachability / handshake | R |
| GET | `/v1/props` | Full state + all `…List` value lists | R |
| PUT | `/v1/params/camera` | Set camera settings (form-urlencoded) | W |
| PUT | `/v1/params/device` | Set device settings | W |
| PUT | `/v1/params/lens` | Set focus | W |
| POST | `/v1/camera/shoot` | Trigger shutter | W |
| POST | `/v1/camera/shoot/cancel` | Cancel exposure (bulb/interval) | W |
| POST | `/v1/camera/shoot/compose` | Composed capture (multi-exp / composite) | W |
| POST | `/v1/lens/focus` | Drive focus (AF/MF control) | W |
| GET | `/v1/liveview` | **MJPEG live stream** (`multipart/x-mixed-replace`, `image/jpeg`, query `maxfps`) | R |
| GET/WS | `/v1/changes` | Websocket for state changes (pushes few/no setting changes) | R |
| GET | `/v1/photos` | Photo list | R |
| GET | `/v1/photos/infos` | Photo metadata (bulk) | R |
| GET | `/v1/photos/<dir>/<file>/info` | Metadata of one photo | R |
| GET | `/v1/photos/<dir>/<file>` | Download photo (JPEG or DNG) | R |
| GET | `/v1/photos/<dir>/<file>/imgctrl` | **Read a photo's Image Control recipe** | R |
| POST | `/v1/photos/<dir>/<file>/transfer` | Start transfer (full/thumb) | W |
| GET | `/v1/transfers` | Transfer status | R |
| GET | `/v1/logs/camera` | **Shutter count** (monthly/total) | R |
| GET | `/v1/debug/revisions` | Revision / debug info | R |
| POST | `/v1/configs/firmware/prepare` | Prepare firmware update | W |
| POST | `/v1/configs/firmware` | Execute firmware update | W |
| POST | `/v1/configs/firmware/cancel` | Cancel update | W |
| POST | `/v1/device/wlan/finish` | End WLAN session | W |
| POST | `/v1/device/finish` | End session / connection | W |

---

## 2. Camera parameters — `PUT /v1/params/camera`

All are **R** (value + `…List` in props) **and W**. Combine several with `&`.

| Parameter | Values | Note |
|---|---|---|
| `WBMode` | see §5 White balance | Live |
| `effect` | see §6 Image control | = image base (also via BLE recipe) |
| `exposureMode` | P/Av/Tv/M/… (numeric, `exposureModeList`) | Exposure program |
| `meteringMode` | `multi` / `center` / `spot` / `highlight` (`meteringModeList`) | Metering |
| `captureMode` | see §7 Drive modes | Drive / self-timer mode |
| `shootMode` | subset of §7 (bracket/interval/…) | |
| `stillSize` | `stillSizeList` (dynamic) | Photo resolution |
| `movieSize` | `4K30 4K24 FHD60p FHD30p FHD25p FHD24p HD60p HD30p HD25p HD24p` | Video format |
| `onePushBracket` | on/off | One-push bracket |
| `cameraOrientation` | `none positive reverse vertical_right vertical_left` | usually status only |
| `sv` | `svList` (ISO 100…204800, `auto`, `auto_hi`) | ISO |
| `av` | `avList` (dynamic) | Aperture |
| `tv` | `tvList` (dynamic) | Shutter speed |
| `xv` | `xvList` (±EV, 1/3 steps) | Exposure compensation |
| `reso` | `<x> <y>` | Liveview resolution |

Also readable in props: `crop_size` (**`35mm` / `50mm`** GR IV crop modes),
`aspect_ratio`, `focusMode`/`focusSetting` (see §4).

---

## 3. Device parameters — `PUT /v1/params/device`

| Parameter | Values / meaning | R/W |
|---|---|---|
| `channel` | WLAN channel (`channelList`) | R/W |
| `datetime` | Camera date/time | R/W |
| `operationMode` | Operation mode (`playback`, `bleStartup`, `other`, …) | R/W |
| `stillFormat` | `jpeg` / `rawdng` / `rawpef` | R/W |
| `geoTagging` | on/off — GPS tagging | R/W |
| `gpsInfo` | GPS data | R/W |
| `bdName` | Bluetooth device name | R/W |
| `bleEnableCondition` | `disable` / `enableAnytime` / `enablePowerOn` | R/W |
| `ssid` | WLAN SSID | R/W |
| `powerOffTransfer` | Transfer while camera is off | R/W |
| `autoResize` | Auto-downsize on transfer | R/W |
| `battery` | Battery level | **R only** |
| `manufacturer`,`model`,`serialNo`,`firmwareVersion`,`macAddress` | Device identity | **R only** |
| `storages` | Storage (equipped/writable/remain/numOfPhotos/numOfMovies) | **R only** |

---

## 4. Lens / focus — `PUT /v1/params/lens` & `POST /v1/lens/focus`

| Parameter | Values |
|---|---|
| `focusMode` | `manual snap multiauto multiauto_center spot pinpoint tracking continuous zone_select` |
| `focusSetting` | `focusSettingList` (snap distance, etc.) |
| (status) | `focused`, `focusEffectiveArea`, `AutoFocusStatus` — **R only** |

---

## 5. White balance — `WBMode` (all live-settable over HTTP)

```
auto  autoWarm  autoWhite  multiAuto  daylight  shade  cloud
daylightFluorescent  dayWhiteFluorescent  coolWhiteFluorescent  warmWhiteFluorescent
tungsten  incandescent  colorTemp1  colorTemp2  colorTemp3  manual1  manual2  manual3
```

> **WB compensation (A-B / G-M): does NOT exist in the API** — no handler/property in
> webapid, bled, or camctld. It is a camera-only setting. (Definitively confirmed.)

---

## 6. Image control / bases — `effect`

```
col_vivid            (Standard base)    efc_monochrome        efc_softMonochrome
efc_hardMonochrome   efc_highContrast   efc_posiFilm          efc_bleachBypass
efc_retro            efc_HDRTone        col_custom1/2/3       efc_negaFilm
efc_cineYellow       efc_cineGreen      efc_crossProcess
efc_monoStandard  efc_monoSoft  efc_monoHighContrast  efc_monoGrainy  efc_monoSolid  efc_monoHDRTone
```

### Image-control detail parameters — 56-byte BLE recipe block

Fine-tuning of a base (saturation, contrast, sharpness, grain, B/W filter, …) is NOT
the HTTP `effect` field — it goes exclusively through the BLE recipe block.

**Block layout (56 bytes):** `[0x01, 0x00, base_code, N, then N×(paramId:u8, value:int8), 0x00-padding]`.
`base_code` (byte 2) must exactly match the base's full parameter set or the camera
rejects it with app error `0x80`. The recipe name is a separate UTF-8 characteristic.

**Parameter IDs** (value = signed int8 unless noted):

| ID | Key | Meaning | Range | Notes |
|---:|---|---|---|---|
| 0 | `saturation` | Saturation | −4…+4 | |
| 1 | `hue` | Hue | −4…+4 | |
| 2 | `highLowKey` | High/Low Key | −4…+4 | |
| 3 | `contrast` | Contrast | −4…+4 | |
| 4 | `contrastHighlight` | Contrast (highlights) | −4…+4 | |
| 5 | `contrastShadow` | Contrast (shadows) | −4…+4 | |
| 6 | `sharpness` | **Sharpness** | −4…+4 | |
| 7 | `toningMono` | Mono toning | 0…5 | enum: `off S R G B P` |
| 8 | `toning` | Color toning | 1…3 | |
| 9 | `bleachBypass` | Bleach Bypass strength | −4…+4 | range estimated |
| 10 | `retro` | Retro effect | −4…+4 | range estimated |
| 11 | `hdrToning` | HDR toning | −4…+4 | range estimated |
| 12 | `filterEffect` | B/W filter effect | 0…4 | 0 = off; enables IDs 13–18 |
| 13 | `filterRHigh` | B/W filter Red — high byte | — | see note |
| 14 | `filterRLow` | B/W filter Red — low byte | — | see note |
| 15 | `filterGHigh` | B/W filter Green — high byte | — | see note |
| 16 | `filterGLow` | B/W filter Green — low byte | — | see note |
| 17 | `filterBHigh` | B/W filter Blue — high byte | — | see note |
| 18 | `filterBLow` | B/W filter Blue — low byte | — | see note |
| 19 | `shading` | Vignetting / shading | −4…+4 | |
| 20 | `clarity` | Clarity | −4…+4 | |
| 21 | `grain` | Grain on/off | 0…1 | 0 = off (then 22/23 inactive) |
| 22 | `grainSize` | Grain size | 1…3 | |
| 23 | `grainStrength` | Grain strength | 1…3 | |
| 24 | `hdrLevel` | HDR tone level | 1…3 | |
| 25 | `crossProcess` | Cross-process variant | −4…+4 | range estimated |

> **B/W filter (IDs 13–18):** each color is one signed 16-bit value split across two
> params, `value = int16(low | high<<8)`, range **−200…+200 %**.
> Red = (14 low, 13 high), Green = (16 low, 15 high), Blue = (18 low, 17 high).
> Only relevant when `filterEffect` (12) > 0.

**`base_code` (byte 2) values** — the base-type code, *not* the effectList index:

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

---

## 7. Drive / shoot modes — `captureMode` / `shootMode`

Each with self-timer (`Self2s/10s/12s`) and remote (`Remocon`, `Remocon3s`) variants:

```
single
continuousH / continuousM / continuousL           (burst fast/medium/slow)
interval / intervalComp / intervalMovie            (interval / interval-composite / timelapse video)
multiExp / multiExpContinuous                      (multiple exposure)
bracket / motionBracket / dofBracket               (exposure / motion / depth-of-field bracket)
mirrorUp                                            (mirror-up pre-release)
starstream                                          (star trail)
```
Plus GPS astrotracer: `ASTRO_TRACER_TYPE3`.

---

## 8. BLE — live-settable (service `9f00f387…`, via `bled`)

Get/set handlers in `bled` (independent of WLAN, usable on the home network):

| Function | R | W |
|---|---|---|
| WhiteBalance | ✔ | ✔ (12 base modes; not autoWarm/white/custom) |
| ISO | ✔ | ✔ |
| ShutterSpeed | ✔ | ✔ |
| ExposureCompensation | ✔ | ✔ |
| ShootingMode | ✔ | ✔ |
| FocusMode | — | ✔ |
| AutoFocusStatus | ✔ | — |
| **Recipe slots** (name + 56-byte block, 3 slots) | ✔ | ✔ |

`PropertyChangedNotify` pushes changes to connected clients.
What BLE cannot do: autoWarm/autoWhite/custom WB, WB compensation (→ HTTP, or not at all).

---

## 9. Read-only status (excerpt from props / logs)

- Shutter count: `GET /v1/logs/camera` → `monthly_capture`, `total_capture`
- `battery`, `storages` (remain, numOfPhotos, numOfMovies, recordableTime)
- Liveview status: `liveState`, `cameraModel`, `orientation`, `aspectRatio`, `resolution`
- Camera state: `capturing`, `stateStill`, `stateMovie`, `countDown`, `shotsTotal`, `shotsCurrent`

---

## 10. Internal layer (not directly remote-controllable)

`camctld` talks to the camera core over **gRPC/protobuf** via an IPCU channel:
`RemoteCameraCommand/CameraCommandSync`, `RemoteInterfaceCommand/InterfaceCommand`,
event dispatchers (Property/Status/ExpStatus), `RemoteSharedMemory`. There is a
`FactoryCmdset` (service/factory commands, `GPSetValue`/`GPSetData`) — local only,
not exposed over the WLAN/BLE API.

