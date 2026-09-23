# Paket 8 — NeoCubeOS-Core: C → SSPL-Portierung

Port des in C/libogc geschriebenen NeoCubeOS-Kerns
(`NeoCubeOS-Core/**/*.c`, ~3600 Zeilen) nach SSPL v10. Die SSPL-
Module liegen unter `NeoCubeOS-Core/sspl/neocube/os/` und sind über
`importiere neocube.os.<name>` ladbar (Singleton-Bindung:
kleingeschriebener letzter Pfadteil → Komponenten-Instanz).

## Etappe 1 — `core/` (abgeschlossen)

| C-Quelle | SSPL-Port | Inhalt |
|---|---|---|
| `core/mem_manager.c` | `mem_manager.sspl` (`MemManager`) | Linearer Heap-Allokator @ `0x80680000`, 4-Alignment, `allokiere`/`gib_frei`/`fuellstand` |
| `core/scheduler.c` | `scheduler.sspl` (`Scheduler`) | 16 Task-Slots; Task-Objekte mit `tick()` (statt `void(*)()`) — `registriere`/`laufe`/`entferne` |
| `core/interrupts.c/.h` | `interrupts.sspl` (`Interrupts`) | 32 IRQ-Callback-Slots + Enable-Maske; Handler mit `behandle(irq, kontext)`; `c_handler(ursache)`; IRQ_-Konstanten |
| `core/settings.c/.h` | `settings.sspl` (`Settings`) | `SystemSettings` als Komponentenfelder, alle 13 interaktiven Setter samt Clamp/Wraparound, `dirty`-Flag |
| `core/patch_engine.c/.h` | `patch_engine.sspl` (`PatchEngine`) | Byte-Muster-Patch auf Guest-RAM via `speicher.lese_u8/schreibe_u8`; PatchType/VideoForceMode-Konstanten |
| `core/ipc.h` + `bootstrap.c`-Registry | `ipc.sspl` (`Orchestrator` + `ipc_nachricht`) | Agent-IDs, Msg-Typen, Boot-Stages als `pub konstante`; Agent-Registry + `sende_nachricht`/`poll_alle` |
| `core/kernel.c/.h` | `kernel.sspl` (`Kernel`) | Init-Kette (mem→sched→spi→dma), `ein_tick`, `hauptschleife`, `panic` (flag statt Halt) |
| `dma/dma_exi.c` | `dma.sspl` (`Dma`) | Busy-Modell, `transfer`/`ist_besetzt`/`warte_fertig` |
| `spi/spi_hw.c` | `spi.sspl` (`Spi`) | EXI-Registerpokes via `speicher.*` @ `0xCC006800`: Init-Sequenz, `sd_cs_an/aus`, `transfer_byte`/`transfer_block`, `setze_takt` |

### Abbildungsregeln C → SSPL

- `static`-Modulzustand → Komponentenfelder; `importiere` liefert die
  Singleton-Instanz (entspricht den C-Globals 1:1).
- Funktionszeiger → Objekte mit Konventionsmethode (`tick()`,
  `behandle(irq, ctx)`, `bei_nachricht(msg)`).
- `void*`/`u8*` → Gast-Adressen; Pufferzugriffe via `speicher.*`
  (Big-Endian, Sparse-Store für nicht gemappte Bereiche).
- `struct` → Komponentenfelder bzw. Maps (`neocube_msg_t` →
  `{sender, empfaenger, typ, p1, p2, daten}`).
- `enum`/`#define` → `pub konstante`.
- `settings`-Sprachwechsel (`lang_init`/`lang_set`) → Ereignis
  `SETTINGS_SPRACHE` (GUI/lang.c ist Etappe 2+).

### Verifikation — `neocube_os_test.sspl` (grün, `=> 0`)

- MemManager: Alignment + Fortschritt (3→4B, 7→8B, Offsets)
- Scheduler: Registrierung, Dispatch-Reihenfolge, Entfernen
- Interrupts: Registrierung pro Maske, Dispatch, Global-Enable
- Settings: Defaults, alle Setter inkl. Wrap/Clamp, Dirty-Flag
- PatchEngine: 2 Treffer, korrekt ersetzte Bytes im Guest-RAM
- IPC: Agent-Registrierung (auto-`initialisiere`), Dispatch, Poll
- Kernel: Init-Kette, Ticks, Panic (`laeuft=0`, IRQs aus)

Aufruf:

```bash
cd "C:\Dev\Repos\SonnerStudio\Gamecube Emulator\NeoCubeOS-Core\sspl"
SSPL_MODULE_PATH="F:/Build SSPL Produktion" \
  sspl-run neocube_os_test.sspl
```

## Interpreter-Fix im Zuge der Portierung

`src/interpreter.rs`: Singleton-`initialisiere()` lief bisher **vor**
dem Export-Binding — Modul-Konstanten (`pub konstante`) waren in der
Init-Methode unsichtbar. Singleton-Erzeugung erfolgt jetzt nach dem
Binden der Exports.

### Gefundene SSPL-Fallen (für weitere Etappen)

- `diese.feld[i] = x` nicht erlaubt — Index-Assignment nur auf
  Bezeichner: Read-Modify-Write über lokale Kopie.
- `anhaenge(l, x)` gibt eine **neue** Liste zurück — Ergebnis zuweisen.
- `>>`/`<<` sind Funktionskomposition — Bit-Shifts heißen
  `shr(x,n)`/`shl(x,n)`.
- `&` bindet schwächer als `!=` — `(x & y) != 0` immer klammern.
- `warte` ist ein reserviertes Wort (await) — `warte_fertig`.
- `neu typ[N]` akzeptiert nur Literale, keine Konstanten.
- `importiere a.b.c` bindet die Singleton-Instanz `c` — Methoden
  laufen im Caller-Env: Modul-Konstanten müssen im aufrufenden
  Modul importiert sein.

## Etappe 2 — Agenten-Schicht + Bootstrap (abgeschlossen)

Alle Module unter `NeoCubeOS-Core/sspl/neocube/os/`:

| C-Quelle | SSPL | Inhalt |
|---|---|---|
| `event/event_agent.c` | `event_agent.sspl` | OS-State-Machine (BOOT/MENU/RUNNING/ERROR), MSG_INT_EVENT |
| `ssl/ssl_vm.c`+`bytecode.c` | `ssl_vm.sspl` | `SslVm` (pc/sp/bytecode als Guest-Adresse), `ssl_bytecode_aus_text`, `ausfuehren_kanal` (ephemer, pc/bytecode restauriert) |
| `ssl/ssl_agent.c` | `ssl_agent.sspl` | MSG_SSL_EXEC → `vm.lade`+`ausfuehren`, `laeuft`-Flag |
| `fs/fs_agent.c` | `fs_agent.sspl` | Stub-Level wie C |
| `audio/audio_agent.c` | `audio_agent.sspl` | `AUDIO_INIT`-Ereignis (48 kHz) ans Audio-Subsystem |
| `gui/gui_agent.c`+`gui.c`-Teil | `gui_agent.sspl` | `gui_subsystem_init`, `aktualisiere_boot_stufe`, `rendere_desktop`, MSG_GUI_RENDER |
| `dvd/dvd_agent.c` | `dvd_agent.sspl` | Disc-Scan ueber `DVD_READ_ABS`-Ereignis (Backend fuellt Guest-Puffer), Header-Parse: game_id/Region/Titel/Launcher-Flag, Poll alle 360 Frames |
| `storage/storage_agent.c` | `storage_agent.sspl` | Slot-B→SD2SP2→Slot-A-Proben via `sdcard.exi_id`+`fat32.init`, primaerer Mount |
| `storage/sdcard.c` | `sdcard.sspl` | SD-SPI-Protokoll ueber `spi`-Singleton: EXI-ID-Probe, SPI-Wakeup (128 Clocks), CMD0-Retries, CMD8 (SDHC), ACMD41, CMD17 `lese_sektor` → Guest-Puffer |
| `fs/fat32.c` | `fat32.sspl` | MBR+BPB-Parse, FAT-/Datenregion, 8.3-Namen, `.ISO`-Root-Scan, `liste_verzeichnis` |
| `hw/hw_agent.c` | `hw_agent.sspl` | SI-Port-Scan, RTC, ARAM-Audit (Guest-Roundtrip), GX-Readiness, EXI-Crowbar-Probe, `exi_geraet_name` |
| `dvd/iso_layer.c` | `iso_layer.sspl` | Fragment-Map-Struktur + Stub-API wie C |
| `games/game_scanner.c` | `game_scanner.sspl` | 3-Quellen-Scan (DVD/sdb:/sd2sp2:), `.dol/.iso/.gcm/.gcz`, GCM-Magic `0x5D1C9EA3`@0x1C, Regions-/Titel-Parse; Host-Verzeichnisse ueber `datei.liste`/`datei.oeffne().lese_bytes` |
| `core/launcher.c` | `launcher.sspl` | LauncherState als Komponente: Scan, Titel-Strip, Farb-Hash, Select-Clamps, Scroll-Lerp, `starte_gewaehltes` → `iso_layer.mount` |
| `core/bootstrap.c` | `bootstrap.sspl` | `boote()`: komplette Boot-Sequenz (VIDEO/PAD-Init → Kernel → PatchEngine → Agenten in C-Reihenfolge → 9 Boot-Stages → Scanner → Desktop); `hauptschleife(frames)` begrenzt den C-`while(1)`-Loop |

### Hardware-Anbindung (Host-sicher)

Physische libogc-Aufrufe werden als Ereignisse/Guest-IO modelliert:
- `VIDEO_*`, `PAD_*`, `AUDIO_*`, `DVD_Init` → `ereignis.sende(...)`.
- `DVD_ReadAbsAsyncPrio` → `ereignis.sende("DVD_READ_ABS",
  {puffer, laenge, offset})` — das Backend schreibt den Header via
  `speicher.*` in den Guest-Puffer (geteilter RAM; Map-Payloads
  werden kopiert und taugen nicht als Rueckkanal).
- EXI/SI/RTC/ARAM → `speicher.*`-MMIO bzw. deterministische Stubs.
- `datei.liste`/`oeffne`/`existiert` → Host-Dateisystem-Fallback
  fuer `scan_dir` (sdb:/sd2sp2: existieren hostseitig nicht →
  deterministisch leer).
- `sdcard.init` nutzt das EXI-Attach-Bit (CSR bit 12) als
  Early-Out — Abweichung zu C (dort nur kommentiert), noetig weil
  ungemappte MMIO-Writes in der Sparse-Map zurueckgelesen werden
  und TSTART sonst nie loescht (Safety-Loop pro Byte).

### Interpreter-Erweiterungen im Zuge der Portierung

`src/interpreter.rs`:
- Singleton-`initialisiere()` laeuft jetzt im **Modul-Env**
  (sieht eigene Imports + pub-Konstanten) — vorher Caller-Env,
  wodurch transitive Imports in Singleton-Init unsichtbar waren.
- **Transitiver Re-Export** (`Interpreter::reexported`): Namen,
  die ein Modul per `importiere` band, gehen in seine Exports ein —
  Modul-Konstanten wandern ueber Import-Ketten (Python-artig).
- `trait_registry` wird gecloned statt gemoved (Init braucht den
  Modul-Interpreter danach noch).

Weitere Fallen:
- Singleton-Bindung nur wenn `norm(Komponentenname)` ==
  `norm(Dateiname)` (lowercase, `_` entfernt) — `SdKarte` in
  `sdcard.sspl` band nicht → Komponente `Sdcard`.
- `Value::Map` wird beim `ereignis.sende` geklont — keine
  Rueckkanal-Mutation; Rueckgaben ueber Guest-RAM oder Komponenten.
- `datei.liste(pfad)`-Eintraege: `ordner` ist `Bool`, nicht Int.
- `teiltext(s, von, bis)` — `bis` ist Endindex, nicht Laenge.

### Verifikation — `neocube_os_agents_test.sspl` (gruen, `=> 0`)

- Alle 8 Agenten: IDs, Namen, Singleton-Init beim Import
- Event-Agent: MENU-Zustand, MSG_INT_EVENT-Dispatch
- SSL: `laeuft`/`bytecode` nach MSG_SSL_EXEC, pc fortschreitend,
  `ausfuehren_kanal` restauriert, `ssl_bytecode_aus_text`
- DVD: Header-Injection via DVD_READ_ABS-Listener → game_id
  "GM4P01", Region PAL, Titel, Launcher-Erkennung ("SDML01")
- Storage/FAT32 hostseitig sauber "nicht verfuegbar"
- HW: EXI-Namenstabelle, SI-Scan, RTC, ARAM-Roundtrip, GX-Check
- GUI: MSG_GUI_RENDER → frames+1
- Orchestrator: 8 Agenten registriert, poll_alle absturzfrei,
  Dispatch MSG_INT_EVENT
- Scanner: Disc-Eintrag aus DVD-Quelle, .dol ohne Magic, .txt
  verworfen, .gcm mit Magic + Region + Titel-Strip
- Launcher: leer ohne FS, injizierte Eintraege, Select-Clamps,
  Scroll-Lerp, `starte_gewaehltes` → iso_layer gemountet
- Bootstrap: komplette Boot-Sequenz, `boot_stufe==9`,
  320 gewartete Frames, 3 Hauptloop-Frames, alle Agenten aktiv

Regressionen: `neocube_os_test.sspl`, `gekko_exec_test`,
`gekko_fpu_test`, `gekko_sys_test`, `gekko_seril_test`,
`gekko_min_test`, `gekko_obj_test`, `neo_cube_smoke_test`,
`si_test` — alle gruen.

Aufruf:

```bash
cd "C:\Dev\Repos\SonnerStudio\Gamecube Emulator\NeoCubeOS-Core\sspl"
SSPL_MODULE_PATH="F:/Build SSPL Produktion;C:/Dev/Repos/SonnerStudio/Gamecube Emulator/NeoCubeOS-Core/sspl" \
  sspl-run neocube_os_agents_test.sspl
```

## Ausstehend (Etappe 3+)

- `gui/` restliche Module: `gx_renderer`, `coverflow`, `effects`,
  `font`, `icon_mesh`, `icon_system`, `lang` — brauchen das
  GX/VI-Subsystem des Emulators (aktuell Ereignis-Stubs)
- `audio/real_audio.c`, `audio/wav_loader.c` — DSP/ARAM-Backend
- `fat32`-Schreibpfad + Cluster-Chain-Follow (iso_layer-Fragmente)
- `ssl_vm` Opcode-Dispatch (aktuell Dummy-Loop wie C)
- `sdcard` Schreibpfad (CMD24), Multi-Block
- Echte Disc-Pfade: DI-Register-Backend fuer `DVD_READ_ABS`
- `datei.liste`-Ersatz fuer gemountete FAT-Volumes (virtuelle Pfade)
