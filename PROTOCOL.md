# Ricoh GR IV — Protokoll (Reverse Engineering)

## >>> GEKNACKT: REZEPT SCHREIBEN (per BLE, via Android-HCI-Capture bestaetigt) <<<
Image Control ("Rezept") wird per **Bluetooth** geschrieben (NICHT WiFi/imgctrl!).
GR World macht genau ZWEI ATT-WriteRequests, sonst nichts (kein Commit/Trigger):

1. NAME  -> Characteristic **30adb439-1bc0-4b8e-9c8b-2bd1892ad6b0** (Service 9f00f387)
   = UTF-8-Name, variable Laenge, z.B. "Cinema (Yellow)"
2. PARAM -> Characteristic **3e0673e0-1c7b-4f97-8ca6-5c2c8bc56680** (Service 9f00f387)
   = 56-Byte-Block:

   Byte0: 0x01 (Version)
   Byte1: 0x00
   Byte2: Basis-Effekt-ID (effectList-Index; 0x0e=col_custom1 etc.)
   Byte3: Anzahl Parameter-Paare N
   dann N x (paramId:uint8, value:int8-signed)
   Rest: 0x00-Padding bis 56 Bytes

Beispiel "Cinema (Yellow)": 01 00 0e 0d | 0000 0103 02ff 0303 0400 05fe 06fd 0801 1300 1400 1500 1600 1700 | 00...
 -> Basis=0x0e, 13 Paare: p0=0 p1=3 p2=-1 p3=3 p4=0 p5=-2 p6=-3 p8=1 p19=0 p20=0 p21=0 p22=0 p23=0

Es gibt 3 Name-Chars (30adb439/e799198f/df77dd09 = "Negative Film"/"Custom 2"/"Custom 3")
und 3 Param-Chars (3e0673e0/0936b04c/cd879e7a) = die 3 Custom-Slots. GR World schrieb in
Slot 1 (30adb439/3e0673e0). Alle sind read+write -> lesen (bestehende Rezepte) + schreiben moeglich.

paramId->Name (aus ImageControlParameter-Enum, Blutter objs.txt):
  0=saturation 1=hue 2=highLowKey 3=contrast 4=contrastHighlight 5=contrastShadow
  6=sharpness 8=toning?(unbestaetigt) 12=filterEffect(mono) 19=shading 20=clarity
  21=grain 22=grainSize 23=grainStrength
Werte sind int8 (signed). Bereiche (an Kamera abgelesen):
  0,1,2,3,4,5,6,19,20 = -4..+4  |  8(toning)=1..3  |  21(grain)=0/1(off/on)
  22(grainSize)=1..3  |  23(grainStrength)=1..3  |  12(filterEffect)=nur Mono, offen
  ID 8 = toning BESTAETIGT. Bei grain=0 sind grainSize/Strength inaktiv (0).
VERIFIZIERT: Schreib-Test "REC-TEST" in Slot 3 -> Kamera zeigt es an. Lesen+Schreiben end-to-end OK.
Tool: recipe.py (Codec) + gr.py (BLE read/save/load/setname), laeuft auf Laptop via SSH.
HCI-Capture: btsnoop_hci.log (aus bugreport_gr.zip), Parser: btparse.py.



## Architektur (aus GR-World-APK, Flutter/Dart libapp.so)
- **BLE = Steuerkanal**, **WiFi = Datenkanal**. Image Control laeuft ueber **WiFi-HTTP**,
  NICHT ueber lesbare BLE-Werte (deshalb zeigte der BLE-Diff bei Saettigungs-Aenderung nichts).
- App ist Flutter; nutzt `flutter_blue_plus` (BLE) + `bluez.dart`. Dev-Projekt "grapp".

## Pairing (Voraussetzung fuer alles)
- Kamera verlangt **authentifiziertes Pairing (Numeric Comparison, kind=8)**, Code beidseitig bestaetigen.
- **Windows BLE-Stack kann die gebondete Verbindung NICHT halten** (connect->1s->Kamera trennt).
- **Linux/BlueZ funktioniert einwandfrei.** -> Wir arbeiten per SSH auf einem Linux-Laptop.

## WiFi-Zugangsdaten (per BLE lesbar, Service f37f568f)
- `9111cdd0` Network Type: 0=OFF, 1=AP-Mode  (schreiben=AP anschalten)
- `90638e5a` SSID (utf8), aktuell: `GR_H2491C2`
- `0f38279c` Passphrase (utf8), aktuell: `QV(n^bRL`
- `63bc8463` Channel

## HTTP-API (Kamera-AP, http://192.168.0.1)
- `GET /v1/photos/` — Fotos
- **`/v1/imgctrl?storage=in`** (oder `sd1`) — **IMAGE CONTROL (Rezepte!)**
  - Datenformat: `application/octet-stream` (binaere Image-Control-Datei)
  - Methode/genaue Form noch per Live-Test zu bestaetigen (GET zum Lesen, PUT/POST zum Schreiben)
- `ws://192.168.0.1/v1/changes` — WebSocket Aenderungen
- `/v1/configs/firmware...`, `/v1/logs/camera?type=monthly_capture&date=`
- storage-Werte: `in` (intern), `sd1` (SD-Karte)

## Image-Control-Datenmodell (Felder aus libapp.so)
- Rezept = **imageControlName + imageControlType + parameters[] + effect** (+ imageControlPath = Datei)
- Custom-Slots: `custom1ImageControlData`, `custom2ImageControlData`, `custom3ImageControlData`
- Parameter-Keys: `saturation`, `hue`, `contrast`, `contrastHigh`, `contrastLow`,
  `clarity`, `sharpness`, `toning`, `filterEffect`  (+ weitere)
- Preset/Effect-Namen (aus SQLite-Schema): effect_efc_nega_film, posi_film, retro,
  bleach_bypass, cross_process, hdr_tone, high_contrast, monochrome, mono_grainy,
  soft_monochrome, hard_monochrome, cine_green, cine_yellow, col_vivid, col_custom1..3

## LIVE-TEST ERGEBNISSE (WiFi-HTTP funktioniert!)
Laptop per `sudo nmcli` ins Kamera-WLAN (192.168.0.8), Kamera = 192.168.0.1, Server "Crow/0.1".
- `GET /v1/ping` -> 200 (JSON datetime)
- `GET /v1/props` -> 200, 5766B JSON. Enthaelt effect="efc_negaFilm", effectList
  (off,col_vivid,efc_monochrome,...,col_custom1/2/3), storages[in,sd1 beide writable].
  ABER: KEINE Image-Control-Detailparameter in props.
- `GET /v1/photos` -> 200 (dirs/files)
- **ALLE imgctrl-Varianten -> 404** (GET/PUT/POST/OPTIONS):
  /v1/imgctrl?storage=in, /imgctrl?storage=in, /v1/imageControl*, /v1/imageControlWrite ...
- `/imageControl`, `imageControlWrite`, `imageControlDetail` = **Flutter-App-Seiten**, KEINE Endpoints.
- Einzige echte /v1/-Endpoints: photos, changes(ws), configs/firmware, fwDownload, getInformation, logs/camera.
- Einziger imgctrl-String: `/imgctrl?storage=` (ohne /v1) -> aber 404.

## OFFENE FRAGE: Wie wird ein Rezept geschrieben?
Zwei Hypothesen, beide von der App referenziert:
1. **BLE-Blob**: write-only char `FE3A32F8` in Service `0F291746` (App referenziert genau diese
   UUIDs + writeCharacteristic + writeWithoutResponse). Ack evtl. via d8676c92 (notify).
2. **HTTP `/imgctrl?storage=in`**, aber Endpoint erst nach einem **Prepare/Session-Schritt** aktiv
   (analog zu firmware/prepare) oder nach Wechsel des operationMode.
Exaktes Byte-Format + Ausloeser NICHT aus statischen Strings ableitbar.

## FLUTTER-DECOMPILATION (Blutter) - ERFOLGREICH
Dart 3.11.1 App, obfusziert. Blutter-Build auf Laptop (ohne sudo) gebaut via:
- ICU 78 dev als .deb lokal entpackt -> ICU_ROOT=/home/mike/icu/prefix
- capstone 5.0.6 aus Quelle -> /home/mike/cap/prefix, .pc includedir gepatcht auf include/capstone
- PKG_CONFIG_PATH=/home/mike/cap/prefix/lib/pkgconfig, CC=gcc CXX=g++
- `venv/bin/python blutter/blutter.py apk/arm/lib/arm64-v8a out_blutter`
Ausgabe gesichert nach `blutter_out/` (Windows): pp.txt (Pool/Strings), objs.txt,
blutter_frida.js, blutter_asm.tar.gz (asm/ = dekompilierte Funktionen, obfusziert).

Erkenntnisse aus dem Pool (pp.txt):
- `/imgctrl\?storage=` @ pp+0x2d720, gehoert zu Klasse `xcb` (in asm/Juk.dart) / lib "Juk".
- HTTP-Methoden-Strings: GET@0x240, POST@0x1458, PUT@0x278f8, DELETE@0xb718.
- `http://192.168.0.1`@0xfdf0, `.../v1/photos/`@0x26168, `ws://.../v1/changes`@0x246a8.
- OFFEN: welche Methode + Ablauf fuer imgctrl -> muss aus asm/Juk.dart (Klasse xcb) traced werden.
  Obfuszierter asm, String-Konstanten nicht inline.

## ASM-MINING FAZIT (ausgereizt)
- VERIFIZIERT (grep-Methodik OK: pp+0x2ce80 -> Juk.dart gefunden): KEINER der
  Netzwerk-Pool-Strings wird von einer rekonstruierten Funktion geladen
  (http-base 0xfdf0, imgctrl 0x2d720, PUT 0x278f8, octet-stream 0x278a0, /v1/photos 0x26168).
  => Blutter hat die gesamte ASYNC-Netzwerk-Logik NICHT dekompiliert. HTTP-Schreib-Details
     statisch NICHT extrahierbar.
- Rezept-Format: Parameter nur als Anzeige-Labels (Saturation/Sharpness/Clarity/Toning) +
  Enum `ImageControlParameter.` + Liste `parameters`. `imageControlPath` + octet-stream
  => Rezept ist eine BINAERE DATEI (kein JSON mit Klartext-Keys). Klasse obfusziert.
  => Ohne BEISPIEL-Rezept (echte Bytes) kein sicheres Reverse des Byte-Layouts.
- SQLite-Schema = nur Nutzungs-Statistik (effect_*-Counter), nicht das Format.

## SCHLUSS: Android-Capture ist der Schluessel
PCAPdroid-Mitschnitt der echten App beim Rezept-Schreiben liefert BEIDES auf einmal:
(a) den echten /imgctrl-Request (Methode/Header/Ablauf) und (b) die Rezept-Bytes im Body.
Alternativ: Frida (blutter_frida.js vorhanden) am laufenden App-Prozess (braucht auch Android).
App-Storage (imageControlPath) enthaelt zudem gespeicherte Rezept-Dateien als Samples.

## MOEGLICHE NAECHSTE WEGE (Entscheidung noetig)
A) **Flutter-Decompilation** (Blutter/reFlutter) -> echten Image-Control-Write-Code lesen. Authoritativ, aufwaendiges Setup.
B) **Traffic-Capture** der echten GR-World-App beim Rezept-Schreiben (BLE-HCI + WiFi). Definitiv, braucht Android-Geraet.
C) **Live-Experimente**: /imgctrl-Aktivierung erraten (prepare/operationMode) + BLE fe3a32f8 testen. Trial&Error.
D) **MVP jetzt** mit dem was geht (BLE-Steuerung, WiFi-Creds, Fotos/props lesen), Write parallel weiter knacken.

## Was FUNKTIONIERT (Assets fuers Tool)
- BLE via Linux/BlueZ stabil (Pairing numeric comparison).
- WiFi-Creds per BLE lesbar, AP per BLE anschaltbar (network_type=1).
- WiFi-HTTP-API: ping/props/photos nutzbar.
- Laptop-WLAN-Umschaltung per `sudo nmcli` scriptbar (SSH bricht im AP-Fenster ab, Script laeuft im Hintergrund weiter).

## Referenzen
- github.com/dm-zharov/ricoh-gr-bluetooth-api (GR III GATT, gleiche UUIDs)
- lucas.io/grid/making-of-grid (Architektur BLE+WiFi)
- Play: com.ricohimaging.grworld

