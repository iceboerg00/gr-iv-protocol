# Ricoh GR IV — BLE / GATT Landkarte

Geraet: **RICOH GR IV HDF**, Serie `REDACTED`, Firmware `1.11`
BLE-Name: `GR_XXXXXXX`, Adresse: `XX:XX:XX:XX:XX:XX`
Beworbener Service (Advertising): `9a5ed1c5-74cc-4c50-b5b6-66a48e7ccff1`
Hersteller-Daten: ID `0x065f` (RICOH) = `da010802b12c00000301`

> Alle Ricoh-Characteristics verlangen **Pairing/Bonding** (sonst
> "Insufficient Authentication"). Standard-Services (GAP/GATT/DevInfo)
> sind ohne Pairing lesbar.

## Standard-Services (lesbar)
- `1800` Generic Access — Name = "GR_XXXXXXX"
- `1801` Generic Attribute
- `180a` Device Information
  - `2a24` Model = "RICOH GR IV HDF"
  - `2a25` Serial = "REDACTED"
  - `2a26` Firmware = "1.11"
  - `2a28` Software = "1.11"
  - `2a29` Manufacturer = "RICOH IMAGING COMPANY, LTD."

## Ricoh-Services (brauchen Pairing)

### 9a5ed1c5-74cc-4c50-b5b6-66a48e7ccff1  (beworben; 6x read-only)
- f5666a48 / 35fe6272 / 0d2fc4d5 / b4eb8905 / 6fe9d605 / 97e34da2  (read)

### 9f00f387-8345-4bbc-8b92-b87b52e3091a  (GROSS: ~34 chars, viele notify/write/read)
Vermutlich Haupt-Steuer-/Statusservice. Enthaelt u.a.:
- viele notify,write,read chars (handles 116..204)
- write-only: 559644b8 (h152), 009a8e70 (h154)

### 4b445988-caa0-4dd3-941d-37b4f52aca86  (GROSS: ~16 chars)
- write-only: e450ed9b (h262), 5f0a7ba9 (h264)

### 84a0dd62-e8aa-4d0f-91db-819b6724c69e
- 28f59d60 (write,read)

### f37f568f-9071-445d-a938-5441f2e82399
- 9111cdd0 (notify,write,read), 90638e5a, 0f38279c, 63bc8463, 460828ac (read), c4b7dfc0

### 0f291746-0c80-4726-87a7-3c501fd3b4b6  (Kandidat fuer Blob-/Datei-Transfer)
- d8676c92 (notify,write,read)
- fe3a32f8 (write-only)  <-- moeglicher Daten-Schreibkanal

## BESTAETIGTE ZUORDNUNG (Abgleich mit dm-zharov GR III API)
Die GR IV nutzt dieselben UUIDs wie GR II/III!

- `9f00f387...` = **Shooting-Service**
  - `559644b8` (h152, write) = Operation Request (1=Start,2=Stop Shooting; Param 1=AF)
  - viele weitere = Shutter/Aperture/ISO/WB/... (read/write/notify)
- `4b445988...` = **Camera-Service**
  - `b58ce84c` (h234) = Camera Power (0=Off,1=On,2=Sleep)
  - enthaelt Battery, DateTime, GeoTag, Storage, FileTransferList ...
- `f37f568f...` = **WLAN-Control-Service**  << Schluessel zum WiFi-Datenkanal!
  - `9111cdd0` (h308) = Network Type (0=OFF, 1=AP-Mode)
  - `90638e5a` (h311) = SSID (utf8, read/write)
  - `0f38279c` (h313) = Passphrase (utf8, read/write)
  - `63bc8463` (h315) = vmtl. Channel
- Standard `180a` = Camera Information (Firmware/Serial/Model)

## NOCH UNBEKANNT (nicht in GR-III-Doku -> Kandidaten fuer Image Control!)
- `9a5ed1c5...` (beworben; 6x read) - GR-IV-spezifisch?
- `84a0dd62...` - `28f59d60` (write,read)
- `0f291746...` - `d8676c92` (notify/write/read) + `fe3a32f8` (WRITE-ONLY)
  -> WRITE-ONLY-Blob-Kanal = klassisch fuer Datei-/Rezept-Transfer!

## Architektur-Erkenntnis (aus lucas.io/grid Making-of)
BLE = Steuerkanal, WiFi = Datenkanal. Ablauf:
1. per BLE SSID+Passphrase auslesen, Network Type = 1 (AP) schreiben
2. PC ins Kamera-WLAN, dann HTTP-API auf http://192.168.0.1
   - GET /v1/photos, /v1/props ... (GR IV: ?storage= fuer intern/SD)
3. Image Control koennte BLE-Blob (0f291746) ODER neuer HTTP-Endpoint sein
Keine Auth ausser Standard-BLE-Pairing.

## PAIRING-ERKENNTNISSE
- Kamera verlangt authentifiziertes Pairing: **CONFIRM_PIN_MATCH** (kind=8),
  6-stelliger Zahlencode-Abgleich, muss auf BEIDEN Seiten bestaetigt werden.
- Windows/bleak Auto-Pairing (ConfirmOnly) reicht NICHT -> kaputter Bond.
- Loesung fuers Pairing: WinRT Custom Pairing mit Deferral + Zeit fuer
  Kamera-Bestaetigung (siehe pair_custom.py) -> Status 0 (Paired, protection=3).

## WINDOWS-BLOCKER (offen)
Nach erfolgreichem Bond kann Windows die VERSCHLUESSELTE Verbindung nicht
halten: connect -> ~1s -> Kamera trennt. MTU bleibt 23, GATT "Unreachable".
Getestet & erfolglos: bleak read, WinRT MaintainConnection/GattSession,
BT-Funkmodul Neustart. Unpaired ging alles (aber dann "Insufficient Auth"
auf Ricoh-Chars). => Windows-BLE-Stack-Limitation.
Funktioniert laut Referenzen auf iOS / Android / Linux(BlueZ).

## Naechste Schritte
1. Pairing/Bonding herstellen -> Characteristics lesbar machen
2. Werte auslesen, Namen/Bedeutungen zuordnen
3. GR World APK dekompilieren zum Abgleich der UUIDs/Bedeutung
4. Image-Control-Schreibsequenz identifizieren

