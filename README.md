# Ricoh GR IV — Bluetooth & Wi-Fi Remote Control Documentation

Reverse-engineered documentation of the Ricoh GR IV's remote-control interfaces
(firmware **v1.11**), captured while building [GRade](https://apps.apple.com/app/id6792491646),
a free recipe app for the GR IV. Everything here was verified against a real camera —
via BLE HCI captures of the official GR World app, the camera's Wi-Fi HTTP API, and
firmware analysis. Shared so the open-source gear community can build on it.

## The two references

- **[GR_IV_Remote_Control_API.md](GR_IV_Remote_Control_API.md)** — the complete surface.
  BLE services and pairing, live shooting settings (ISO, aperture, shutter, EV, WB,
  focus), the Wi-Fi HTTP API (all endpoints, `PUT /v1/params/…`, live view, photo
  browsing), drive modes, and known limitations. **Start here.**
- **[RECIPES.md](RECIPES.md)** — the deep dive on the Image Control "recipe" format:
  the exact 56-byte parameter block, every parameter ID and its encoding, the base
  codes, and how reading/writing the three custom slots works over BLE.

## The 60-second picture

The GR IV has two cooperating radios:

- **BLE = control channel.** Pairing, status, waking the camera, switching the Wi-Fi AP
  on/off, a handful of live settings (ISO/WB/shutter/EV/focus), and writing the three
  custom Image Control recipe slots.
- **Wi-Fi = data channel.** An HTTP server at `http://192.168.0.1` for the full settings
  model, live view (MJPEG), and photo browsing/download.

Typical flow: pair over BLE → optionally enable the Wi-Fi AP over BLE → join it → use
the HTTP API. The camera uses the **same BLE UUIDs as the GR II/III**, so
[dm-zharov's GR III API](https://github.com/dm-zharov/ricoh-gr-bluetooth-api) is a useful
companion for the shared parts.

## Gotchas that cost real time

- The HTTP API returns **HTTP 200 with an `errCode` inside the JSON body** — always check
  the body, not just the status code.
- `PUT /v1/params/…` needs **form-urlencoded** bodies (JSON → 400) and the method must be
  **PUT** (POST → 404). Parameter names must match `props` exactly.
- The camera accepts **only one BLE connection at a time**. A backgrounded official app
  will block yours.
- Recipe writing is **BLE, not Wi-Fi** — the `/v1/imgctrl` HTTP path is a dead end (404).
- Grain/toning are **0-based on the wire**, 1-based in the camera UI; writing the
  displayed value gets rejected with ATT app error `0x80`. `filterEffect` only takes 0/1
  over BLE.
- In U1–U3 dial modes, written recipe slots are **temporary** until "Save User Mode Box"
  is run on the camera.
- Pairing is **numeric-comparison** (6-digit code on both sides); plain auto-pairing is
  not enough. The Windows BLE stack cannot hold the encrypted connection — use
  Linux/BlueZ, macOS, iOS or Android.
- Waking: write `0` to the OperationMode characteristic to wake from sleep; write `0` to
  CameraPower to power off. A fully powered-off camera has BLE off — nothing can wake it.

## Disclaimer

Independent community effort, not affiliated with or endorsed by Ricoh Imaging Company,
Ltd. "RICOH" and "GR" are trademarks of their respective owners. Use at your own risk.

## License

CC BY 4.0 — use it freely, attribution appreciated.
