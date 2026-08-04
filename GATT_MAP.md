# Ricoh GR IV — BLE / GATT Map

Device: **RICOH GR IV HDF**, serial `REDACTED`, firmware `1.11`
BLE name: `GR_XXXXXXX`, address: `XX:XX:XX:XX:XX:XX`
Advertised service: `9a5ed1c5-74cc-4c50-b5b6-66a48e7ccff1`
Manufacturer data: ID `0x065f` (RICOH) = `da010802b12c00000301`

> All Ricoh characteristics require **pairing/bonding** (otherwise
> "Insufficient Authentication"). Standard services (GAP/GATT/DevInfo)
> are readable without pairing.

## Standard services (readable)
- `1800` Generic Access — Name = "GR_XXXXXXX"
- `1801` Generic Attribute
- `180a` Device Information
  - `2a24` Model = "RICOH GR IV HDF"
  - `2a25` Serial = "REDACTED"
  - `2a26` Firmware = "1.11"
  - `2a28` Software = "1.11"
  - `2a29` Manufacturer = "RICOH IMAGING COMPANY, LTD."

## Ricoh services (require pairing)

### 9a5ed1c5-74cc-4c50-b5b6-66a48e7ccff1  (advertised; 6x read-only)
- f5666a48 / 35fe6272 / 0d2fc4d5 / b4eb8905 / 6fe9d605 / 97e34da2  (read)

### 9f00f387-8345-4bbc-8b92-b87b52e3091a  (LARGE: ~34 chars, many notify/write/read)
Likely the main control/status service. Contains among others:
- many notify,write,read chars (handles 116..204)
- write-only: 559644b8 (h152), 009a8e70 (h154)

### 4b445988-caa0-4dd3-941d-37b4f52aca86  (LARGE: ~16 chars)
- write-only: e450ed9b (h262), 5f0a7ba9 (h264)

### 84a0dd62-e8aa-4d0f-91db-819b6724c69e
- 28f59d60 (write,read)

### f37f568f-9071-445d-a938-5441f2e82399
- 9111cdd0 (notify,write,read), 90638e5a, 0f38279c, 63bc8463, 460828ac (read), c4b7dfc0

### 0f291746-0c80-4726-87a7-3c501fd3b4b6  (candidate for blob/file transfer)
- d8676c92 (notify,write,read)
- fe3a32f8 (write-only)  <-- possible data write channel

## CONFIRMED MAPPING (cross-referenced with dm-zharov GR III API)
The GR IV uses the same UUIDs as GR II/III!

- `9f00f387...` = **Shooting service**
  - `559644b8` (h152, write) = Operation Request (1=Start, 2=Stop shooting; Param 1=AF)
  - many others = Shutter/Aperture/ISO/WB/... (read/write/notify)
- `4b445988...` = **Camera service**
  - `b58ce84c` (h234) = Camera Power (0=Off, 1=On, 2=Sleep)
  - contains Battery, DateTime, GeoTag, Storage, FileTransferList ...
- `f37f568f...` = **Wi-Fi control service**  << key to the Wi-Fi data channel!
  - `9111cdd0` (h308) = Network Type (0=OFF, 1=AP mode)
  - `90638e5a` (h311) = SSID (utf8, read/write)
  - `0f38279c` (h313) = Passphrase (utf8, read/write)
  - `63bc8463` (h315) = probably Channel
- Standard `180a` = Camera Information (Firmware/Serial/Model)

## STILL UNKNOWN (not in the GR III docs -> candidates for Image Control!)
- `9a5ed1c5...` (advertised; 6x read) - GR IV specific?
- `84a0dd62...` - `28f59d60` (write,read)
- `0f291746...` - `d8676c92` (notify/write/read) + `fe3a32f8` (WRITE-ONLY)
  -> write-only blob channel = classic for file/recipe transfer!

## Architecture insight (from the lucas.io/grid making-of)
BLE = control channel, Wi-Fi = data channel. Flow:
1. read SSID+passphrase over BLE, write Network Type = 1 (AP)
2. join the camera's Wi-Fi, then hit the HTTP API at http://192.168.0.1
   - GET /v1/photos, /v1/props ... (GR IV: ?storage= for internal/SD)
3. Image Control could be a BLE blob (0f291746) OR a newer HTTP endpoint
No auth beyond the standard BLE pairing.

## Pairing findings
- The camera requires authenticated pairing: **CONFIRM_PIN_MATCH** (kind=8),
  a 6-digit numeric code comparison that must be confirmed on BOTH sides.
- Windows/bleak auto-pairing (ConfirmOnly) is NOT enough -> broken bond.
- Pairing solution: WinRT custom pairing with a Deferral and time for the
  camera to confirm (see pair_custom.py) -> Status 0 (Paired, protection=3).

## Windows blocker (open)
After a successful bond, Windows cannot hold the ENCRYPTED connection:
connect -> ~1s -> camera disconnects. MTU stays 23, GATT "Unreachable".
Tested and unsuccessful: bleak read, WinRT MaintainConnection/GattSession,
BT radio restart. Everything worked while unpaired (but then "Insufficient
Auth" on Ricoh chars). => Windows BLE stack limitation.
Works on iOS / Android / Linux (BlueZ) according to references.

## Next steps
1. Establish pairing/bonding -> make characteristics readable
2. Read values, map names/meanings
3. Decompile the GR World APK to cross-check UUIDs/meaning
4. Identify the Image Control write sequence
