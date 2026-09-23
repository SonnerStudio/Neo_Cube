# Paket 6 — Fremdsprachen-Ersatz durch SSPL

Ziel der Self-Hosting-Roadmap: Alle Implementierungsanteile in
fremden Sprachen durch SSPL-Substitute ersetzen, Host-ABI minimal
und explizit halten.

## Ersetzt (verifiziert)

| Fremdsprache | SSPL-Substitut | Status |
|---|---|---|
| `main.js` (Electron-Shell) | `start_app.sspl` | ✅ Self-Spawn via `sys_exe()`, Poll via `netz.hole`, Browser via `prozess.start` |
| `start_app.py` (pywebview) | `start_app.sspl` | ✅ identischer Zweig |
| `neocube-os/app.py` (HTTP-Server) | `neocube-os/app.sspl` | ✅ alle Endpunkte live getestet |
| `core/neocore_bridge.py` | `core/NeoCoreBridge.sspl` | ✅ Komponente, via `importiere core.NeoCoreBridge` |
| `neocube-os/check_ids.py` | `neocube-os/check_ids.sspl` | ✅ findet fehlende IDs |
| `tools/create_sd_image.py` | `tools/create_sd_image.sspl` | ✅ MBR/VBR-Bytes verifiziert |

`package.json` referenziert kein Electron mehr: `npm start` →
`sspl-run start_app.sspl`.

## Neue Host-ABI (minimal, Handle-Pattern wie `datei.oeffne`)

- `netz.hoere(port)` → Listener `{nimm_an()}` → Conn `{lese, sende, sende_bytes, schliesse}`
- `netz.hole(url)` — HTTP/1.0-GET → Body-Text
- `netz.offen(addr, timeout_ms)` → Bool (TCP-Probe, z.B. Port 445)
- `prozess.start(name, args[])` → `{ausgabe(), beende()}` — non-blocking Spawn
- `datei.liste(pfad)` → `[{name, ordner}]`
- `datei.schreibe_bytes(pfad, liste)`
- `datei.groesse(pfad)` → Int
- `utf8_bytes(s)` → List[Int] (korrekte Content-Length)
- `zeichen(code)` → 1-Zeichen-String
- `sys_exe()` → Pfad des laufenden sspl-Binaries

## Ehrliche Grenzen (nicht ersetzt — blockiert durch fehlende Backends)

| Datei/Verzeichnis | Grund |
|---|---|
| `neocube-os/script.js`, `lang.js`, `index.html`, `style.css` | Browser-Präsentationsschicht. Ersetzbar erst mit SERIL→JS/WASM-Backend oder nativer GUI-ABI. |
| `NeoCubeOS-Core/**/*.c/.h` (~75 Dateien) | Gast-OS-Code für die emulierte Gekko-CPU. Ersetzbar erst mit SERIL→Gekko-Backend (Toolchain-Schritt). |
| `tools/icon_to_gx.py` | Braucht PNG-Dekodierung + LANCZOS-Resample (PIL). Entweder `bild.*`-ABI oder reiner SSPL-DEFLATE-Decoder — noch nicht vorhanden. |

## Verifikation

- `app.sspl` live: `/api/drives` → `["D:","E:","F:"]`,
  `/api/list_dir` → sortiertes JSON, `/` → index.html (59431 B),
  `/api/launch_game` → URL-dekodierter Pfad, 404-Pfad korrekt.
- `start_app.sspl`-Flow: Spawn → Poll → Beenden eines echten
  Server-Subprozesses verifiziert.
- `create_sd_image.sspl`: MBR-Partitionseintrag (`80 01 01 00 0c fe ff ff`)
  und FAT32-VBR (`eb 58 90 MSDOS5.0`) byte-exakt.
