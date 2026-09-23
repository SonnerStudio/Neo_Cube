# SSPL-Fähigkeitsanalyse für Emulator-Entwicklung
## Neo-Cube / SonnerStudio — Ganzheitliche Bewertung (Stand: 21.09.2026)

> Ziel dieses Dokuments: Feststellen, was an SSPL noch entwickelt werden muss,
> damit ein erstklassiger GameCube-Emulator (und später andere Systeme) damit
> gebaut werden kann. Bewertet wird der **Gesamtfall** — Sprache, Runtime,
> Toolchain, Ökosystem — nicht nur einzelne Teilaspekte.

---

## 1. Ist-Zustand: Was ist real, was ist deklariert?

### 1.1 SSPL v9.5 (systemweit installiert)

| Behauptung | Befund (verifiziert) |
|---|---|
| Compiler `sspl schmiede` | `sspl.exe` ist eine **62 KB .NET/Mono-Assembly** (C# Host, `sspl_host.cs`). Es ist ein Bootstrap-Interpreter, kein nativer Compiler. |
| `.ssplx` = kompiliertes Paket | `.ssplx` ist **Quelltext** mit MAGIC-Header und HASH-Footer — kein Maschinencode, kein Bytecode-Binary. |
| SERIL-Bytecode-VM | Existiert als Text-Opcodes (`[SERIL:OP_PRINT]` etc.). Die VM in `system/ausfuehrung.sspl` ist ein **Stub**: Sie matched Strings und loggt — sie führt keine Semantik aus. |
| Native Aufrufe (`vrf.aufruf`, `rufe_nativ_auf`) | Delegieren an `host_dispatch()` in `betriebssystem.sspl` → **liefert konstant `0`**. Kein echter FFI. |
| VRF C++-Engine (`vrf-api.exe`, 923 KB) | Existiert real als REST-API-Server (Port 8780): Event-Pipeline, Security-Analyse, Telemetrie. Es ist ein **Analyse-/Visualisierungsdienst**, keine JIT-/GPU-/Hardware-Engine. |
| Sprachdialekt des Emulators (`paket`, `komponente`, `funktion`, kleingeschrieben) | `sspl run neo_cube.sspl` → **„0 SSPL-Keywords erkannt"**. Der Neo-Cube-Dialekt wird von v9.5 gar nicht geparst; die Ausgabe ist eine leere SERIL-Hülle. |
| SSGE (Grafikengine) | In v9.5 nur Deklarationsverzeichnisse (`ssge_ship_*`). Kein rendernder Code. |

**Fazit v9.5:** Eine ausgeklügelte *Spezifikations- und Orchestrierungs-Hülle*.
Als Ausführungsplattform für Emulator-Hotpaths (CPU-Kern, GPU-Pipeline, DSP)
ist v9.5 aktuell **nicht geeignet** — die Ausführung ist nominell interpretiert
und die Systembrücke ist eine Stubsammlung.

### 1.2 SSPL v10 (in Entwicklung, Rust-Codebase unter `C:\Dev\SSPL-v10`)

Hier liegt der **echte** Sprachkern:

| Komponente | Stand |
|---|---|
| Lexer/Parser | Real (`lexer.rs` 335, `parser.rs` 2026 Zeilen), Tests grün (Core 203/203) |
| Interpreter | **Tree-Walk-Interpreter** (`interpreter.rs`, ~3350 Zeilen). `Value` ist ein getaggtes Enum (Int/Float/List/Map/Channel/Future/Pointer…). |
| Threads | `spawn` existiert (`std::thread`), `ChannelSender/Receiver` via `std::sync::mpsc` vorhanden. Auf wasm32 deaktiviert. |
| Wasm-AOT | Backend existiert (`src/wasm/`), „Host→Wasm AOT" als eigenes Binary (`sspl-wasm-aot`). |
| Pointer/Raw-Memory | `Value::Pointer(usize)` existiert, aber **`alloc` liefert eine Fake-Adresse (`size*1000`), `ptr_read` liefert `0`, `ptr_write` ist No-Op**. Kein echtes Raw-Memory. |
| JIT | `JitCompiler` in `env.rs` ist ein **Stub** (`should_compile`/`compile_function` ohne Implementierung). |
| FFI/Plugins | `libloading`-Feature vorhanden (`src/plugin`) — native Dylibs ladbar. Gute Basis. |
| Grafik (SSGE) | Umfangreicher, echter Renderer auf **wgpu** (Vulkan/Metal/DX12): Frame-Graph, Meshlets, ReSTIR, HWRT, Virtual Texturing, Vis-Buffer, Audio (cpal/rodio). Produktionsreife unklar (Debug-Lauf auf M4: ~29–35 ms/Frame ≈ 30 FPS, keine Release-Abnahme). |
| Numerik | Breitengetaggte Ints/Floats, Soft-Float (128/256), `soft_limbs` — solide Basis für bitgenaue Emulation. |
| Status | „Verifizierter Entwicklungskandidat" — **keine Produktionsfreigabe**; Windows-Abnahme offen. |

---

## 2. Anforderungsprofil: Was ein Spitzen-GameCube-Emulator braucht

Abgeleitet aus der GameCube-Hardware (Gekko PPC750CL @ 486 MHz, Flipper-GPU,
Macaron-DSP, MEM1 24 MB + ARAM 16 MB, HW-Register @ `0xCC000000`):

| # | Anforderung | Begründung |
|---|---|---|
| R1 | **Echtes Raw-Memory** (mmap/VirtualAlloc, typisierte Views u8–u64, f32/f64, Big-Endian-Accessor) | MEM1/ARAM-Abbildung, fastMEM-Ansatz, MMU-Translation pro Zugriff. Millionen Zugriffe/Sekunde. |
| R2 | **Dynamische Rekompilierung (Dynarec/JIT)** PPC→x86_64/ARM64 | Interpreter allein reicht für Cycle-Accuracy nicht (~100–1000× zu langsam). Gekko: Integer + Paired-Single (SIMD-artig). |
| R3 | **Deterministischer Scheduler + Cycle-Zähler** (Sub-Instruction-Timing, Event-Queue) | CPU/GPU/DSP/DVD/EXI müssen taktgenau synchronisiert werden; sonst Audio-Crackle/Desyncs wie bei älteren Emulatoren. |
| R4 | **Echte Parallelität**: getrennte Threads für CPU, GPU-FIFO, DSP, DVD; lock-free SPSC-Rings, Affinity/Priorität | Entkoppelte Hardware-Domains, VBLANK-Sync, „Power-Mode"-Übertaktung. |
| R5 | **Native GPU-Anbindung** auf Kommando-Ebene (Command-Buffer, Compute, Texturen, Framebuffer-Readback) | Flipper: CP-FIFO → XF-Transform → TEV (16 Stages) → EFB. Ubershader/Async-Compile gegen Stutter. |
| R6 | **Audio-Ausgabe mit niedriger Latenz** + Sample-Accurate-DMA | Macaron-DSP (LLE später; HLE-Ucode zuerst), ARAM-Streaming. |
| R7 | **Stabiles FFI / C-ABI** zu OS-APIs (Datei-Raw-I/O, Sockets, Input-Devices, Timer) | SD-Passthrough, BBA/Ethernet, Controller (HID/XInput), Hochauflösende Timer. |
| R8 | **Serialisierbarer Maschinenzustand** (Savestates) + Determinismus | Savestates, TAS, Chrono-Debugging, Netplay-Sync (Lockstep/Rollback). |
| R9 | **Endianness- und Bitpräzision** als First-Class-Feature | PPC ist Big-Endian; alle MMIO/Strukturen byte-exakt. |
| R10 | **Tooling**: Disassembler, Breakpoints, Memory-Viewer, Profiler, Trace | Entwicklung und Verifikation der Emulationskorrektheit (Testsuite gegen reale Hardware). |

---

## 3. Gap-Analyse: SSPL v10 vs. Anforderungsprofil

| Req | Status in v10 | Bewertung |
|---|---|---|
| R1 Raw-Memory | `Pointer(usize)` vorhanden, aber **Fake-Semantik** (alloc→Fake-Adresse, read→0, write→No-Op) | 🔴 Fehlt vollständig |
| R2 JIT/Dynarec | Nur Stub (`JitCompiler`), Wasm-AOT als separater Pfad | 🔴 Fehlt (Wasm-AOT ist vielversprechendster Ansatzpunkt) |
| R3 Scheduler/Cycles | `zeit`-Modul rudimentär; kein deterministischer Event-Scheduler | 🔴 Fehlt |
| R4 Parallelität | `spawn` + mpsc-Channels vorhanden; keine SPSC-Rings, keine Affinity, kein RT-Budget | 🟡 Teilweise |
| R5 GPU-Backend | **SSGE/wgpu ist die stärkste vorhandene Komponente** — echte Render-Infrastruktur | 🟢 Vorhanden (aber für Emulation muss ein Low-Level-Pfad her, nicht die High-Level-SSGE-Pipeline) |
| R6 Audio | cpal/rodio im SSGE-Feature-Set | 🟡 Vorhanden, Latenz-/DMA-Pfad zu bauen |
| R7 FFI | `register_native_function` + libloading-Plugin-System | 🟡 Basis vorhanden, C-ABI-Stabilität offen |
| R8 Savestates | `Value` ist serde-fähig (mit `skip` auf Handles) — Snapshot-Konzept (omni_state) existiert als Idee | 🟡 Ansatz vorhanden |
| R9 Bitpräzision | IntTagged/FloatTagged/SoftFloat — **ungewöhnlich stark** | 🟢 Vorhanden |
| R10 Tooling | LSP, Debugger-Verzeichnis, Chrono-Debugger-Konzept | 🟡 Gerüst vorhanden |

### Gesamtbewertung (gewichtete Matrix)

| Dimension | Gewicht | Score (0–10) | Kommentar |
|---|---:|---:|---|
| Sprachkern (Lexer/Parser/Interpreter) | 15 % | 7 | Real, getestet; Tree-Walk ist korrekt, aber langsam |
| Speicher-/Pointer-Modell | 15 % | 1 | Nur deklariert, keine echte Semantik |
| Code-Generierung (AOT/JIT) | 15 % | 2 | Wasm-AOT existiert; nativer JIT fehlt; SERIL ist Textformat |
| Nebenläufigkeit | 10 % | 5 | spawn + Channels ok; RT/SPSC/Affinity fehlen |
| Grafik-Subsystem | 10 % | 7 | wgpu/SSGE stark; Emulationspfad (Command-Level) fehlt |
| FFI/System-Bridge | 10 % | 3 | Plugin-Loader da; `host_dispatch` in 9.5 Stub, v10 C-ABI unfertig |
| Numerik/Bit-Exaktheit | 5 % | 8 | Breiten-Typen + Soft-Float überdurchschnittlich |
| Tooling/Docs | 10 % | 5 | LSP/Debugger-Ansätze; viele Docs sind Aspirations-Prosa |
| Reife/Produktionsstatus | 10 % | 3 | Dev-Kandidat, keine Release-Abnahme, Windows-Pfad offen |
| **Gesamt (gewichtet)** | | **≈ 4,4 / 10** | **Solide Sprachbasis, fehlende Systemtiefe** |

**Interpretation:** SSPL v10 ist als *Sprache* weiter als v9.5 suggeriert —
mit bemerkenswerten Stärken (Numerik-Modell, SSGE/wgpu, Plugin-System).
Für Emulation fehlt aber exakt die Schicht, die Emulatoren ausmacht:
**echtes Raw-Memory, Code-Generierung und deterministisches Timing.**
Ohne diese drei Blöcke kann SSPL nur die Orchestrierung, nicht den Kern liefern.

---

## 4. Was konkret an SSPL entwickelt werden muss

Priorisiert nach Wirkung für den Emulator-Fall (und generalisierbar für
weitere Systeme: N64, PS1, Wii):

### P0 — Blocker (ohne sie kein Emulator-Kern in SSPL)
1. **Echtes Speichermodell (`sspl.mem`)**: `VirtualAlloc`/`mmap`-Backed Arenen,
   typisierte Big-/Little-Endian-Views, Schutz gegen Bereichsverletzung als
   opt-in (Emulator will bewusst ungeschützte Fast-Pfade).
2. **Nativer Hot-Path**: Entweder
   (a) **Wasm-AOT als Codegen-Ziel** ausbauen (wasmtime → native Performance,
       bereits angelegt), oder
   (b) **Dynarec-Bibliothek als natives Plugin** (`libloading` vorhanden)
       mit PPC-Frontend in SSPL. — Empfehlung: (a) für Portabilität, (b) für
       Maximaltempo; beides über dieselbe IR-Schnittstelle abstrahieren.
3. **Deterministischer Event-Scheduler als Runtime-Primitive**:
   Zyklengenauer Timer (`rdtsc`/`clock_gettime`-Granularität),
   Event-Queue mit Prioritäten, `sleep_until`, Tick-Synchronisation
   zwischen Komponenten.

### P1 — Produktnotwendig
4. **Lock-free IPC**: SPSC/MPMC-Ringbuffer (für CPU↔GPU-FIFO, DSP-Sample-Queue),
   Speicher-Abbildungen zwischen Threads, Affinity/Priorität-API.
5. **Emulations-GPU-Pfad in SSGE**: Zugriff *unterhalb* der High-Level-SSGE-
   Pipeline — eigene Command-Buffer, Compute-Shader für TEV, Texture-Upload,
   Framebuffer-Readback (EFB→RAM). Wgpu ist dafür geeignet.
6. **Audio-Ausgabe-Contract**: Sample-genauer Stream an cpal mit Ringpuffer.
7. **C-ABI-FFI vollständig**: Strukturen by-value, Callbacks *in* SSPL
   (für MMIO-Hooks, Input-Polling), Plattform-Tabelle (Win32/Unix).

### P2 — Qualitäts-/Differenzierungsfeatures
8. **Savestate-Serialisierung**: `Value`-Snapshot + native Seiten diffen;
   Determinismus-Garantie für Netplay (Input-Lockstep).
9. **Emulator-Tooling**: Disassembler, Watchpoints, Trace-Ringbuffer,
   Chrono-Debugger auf Maschinenebene (nicht nur SSPL-Ebene).
10. **Hardware-Test-Harness**: Abgleich gegen reale GC (Testsuite: bekannte
    Instruktions-/Timing-Verhalten), plus Homebrew-Kompatibilitätsliste.

### Architektur-Empfehlung (Hybrid)
```
┌─ Neo-Cube OS (bestehendes Web-UI) ────────────────────────────┐
│  Electron/pywebview → Python-API → IPC zum Core               │
└──────────────┬────────────────────────────────────────────────┘
               │ Launch/Config/Status-Events
┌──────────────▼────────────────────────────────────────────────┐
│  SSPL v10 Orchestrierung (CBA-Komponenten, Event-Bus)         │
│  neo_cube.sspl-Dialekt → auf v10-Grammatik harmonisieren      │
│  • Komponenten: GekkoCPU, FlipperGPU, MacaronDSP, DVD, SI…    │
│  • Kommunikation: Channels/Event-Bus                          │
└──────┬───────────────┬──────────────────┬─────────────────────┘
       │ native FFI    │ native FFI       │ native FFI
┌──────▼───────┐ ┌─────▼───────┐ ┌────────▼────────┐ ┌──────────┐
│ neocore_cpu  │ │ neocore_gpu │ │ neocore_dsp_dvd │ │ neocore_ │
│ (Rust/Crate) │ │ (Rust/wgpu) │ │ (Rust Crate)    │ │ platform │
│ Interpreter+ │ │ CP-FIFO, XF,│ │ HLE-Ucode, DI,  │ │ mem,thr, │
│ Wasm-JIT     │ │ TEV→WGSL    │ │ EXI, SI, VI     │ │ input,net│
└──────────────┘ └─────────────┘ └─────────────────┘ └──────────┘
```
- **SSPL bleibt die Produkt- und Orchestrierungssprache** (Komponenten, Config,
  Powermode, NCL, UI-Bridge) — das ist die Stärke des CBA-Modells.
- **Die silicon-nahen Hotpaths werden als SSPL-native Module (Rust-Crates)
  implementiert** und über das Plugin/FFI-System eingebunden — konform zur
  bereits angelegten `vrf.aufruf`/Native-Bridge-Idee, aber mit *echten*
  Implementierungen statt Stubs.
- Damit bleibt das Produkt „ein SSPL-Emulator" — mit korrekter Arbeitsweise
  unter der Haube.

---

## 5. Sofortmaßnahmen (konkret, aus dem Ist-Zustand)

1. **Dialekt-Angleichung**: Die Neo-Cube-`*.sspl`-Dateien nutzen eine Syntax,
   die v9.5/v10 nicht parst (`paket` vs. `MODUL`, `komponente` vs. `STRUKTUR`,
   `waehrend` vs. `SOLANGE`). Entweder Grammatik erweitern oder Dateien auf
   v10-Syntax portieren.
2. **`host_dispatch` real implementieren** (v10: natives Plugin statt `return 0`).
3. **Speicher-Primitive zuerst**: Ein einziges natives Modul `neocore_mem`
   mit Arena + Endian-Views ersetzt den kompletten Stub-Pfad und ist die
   kleinste Einheit mit echtem Emulator-Nutzen.
4. **Boot-Pfad-Entscheidung**: HLE-Boot (DOL direkt laden, kein BIOS nötig)
   vs. IPL-Boot (`boot_roms/` vorhanden, aber urheberrechtlich nicht
   auslieferbar → HLE als Default).
5. **SSGE-Evaluierung**: Prüfen, ob wgpu-Pfad die nötige Command-Buffer-
   Granularität hergibt; falls nicht, Vulkan-Direktanbindung als natives
   Modul vorsehen.

---

*Erstellt durch Code-Inspection der installierten SSPL v9.5-Toolchain
(`bin/sspl.exe`, Schmiede-Module, vrf-api) sowie des v10-Workspaces
(`C:\Dev\SSPL-v10`, Rust-Quellbaum, Dokumentation). Alle Aussagen sind an
beobachtbares Verhalten/Quelltext gebunden; keine Aussage basiert auf
Marketing-Dokumenten.*
