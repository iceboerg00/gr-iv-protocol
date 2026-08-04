# Ricoh GR IV — Bluetooth & Wi-Fi Protocol Documentation

Reverse-engineered documentation of the Ricoh GR IV's remote control interfaces,
created while building [GRade](https://apps.apple.com/app/id6792491646), a free
recipe app for the GR IV. Shared so the open-source gear community can build on it.

Everything here was captured from BLE HCI snoops of the official GR World app,
the camera's Wi-Fi HTTP API, and firmware analysis — verified against a real GR IV.

## Contents

| File | What's inside |
|---|---|
| [GATT_MAP.md](GATT_MAP.md) | Full BLE GATT table: services, characteristics, flags — pairing, camera status, battery, mode dial, recipe slots, Wi-Fi provisioning, wake from sleep / power off |
| [PROTOCOL.md](PROTOCOL.md) | The Image Control recipe protocol: the 56-byte parameter block, slot name characteristics, parameter IDs and value encodings |
| [GR_IV_Remote_Control_API.md](GR_IV_Remote_Control_API.md) | The Wi-Fi HTTP API: endpoints for shooting (`/v1/camera/shoot`, `/v1/lens/focus`), live params (`sv` = ISO, `av` = aperture, `tv` = shutter, `xv` = EV, WB modes), props, image browsing/download |
| [FIRMWARE_API.md](FIRMWARE_API.md) | Parameter tables and constraints extracted from firmware analysis — what is readable vs. writable |
| [RECIPES.md](RECIPES.md) | Deep dive into the Image Control recipe space: bases, parameter ranges, display vs. wire encodings, U-mode quirks |

## Hard-won gotchas

- The HTTP API returns **HTTP 200 with an `errCode` inside the JSON body** — always
  check the body, not just the status code.
- The camera accepts **only one BLE connection at a time**. A backgrounded official
  app will block yours.
- Grain/toning values are **0-based on the wire** while the camera UI displays them
  1-based. Writing the displayed value gets rejected with ATT error 0x80.
- In U1–U3 dial modes, written recipe slots are **temporary** until "Save User Mode
  Box" is run on the camera.
- Wake from sleep: write `0` to the OperationMode characteristic; power off: write
  `0` to CameraPower (see GATT_MAP.md). A fully powered-off camera has BLE off —
  nothing can wake it.

## Disclaimer

This is an independent community effort, not affiliated with or endorsed by Ricoh
Imaging Company, Ltd. "RICOH" and "GR" are trademarks of their respective owners.
Use at your own risk.

## License

CC BY 4.0 — use it freely, attribution appreciated.
