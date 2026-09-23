# S-10 · Tooling, Zielplattformen & SSGE-Spieleentwicklung

**Zweck:** Phase-8/9-Entscheide — Developer-Experience, Paket-Ökosystem, Cross-Compilation und die SSGE-Bindung für Spieleentwicklung auf PC (IBM-kompatibel) und Mac mit Apple-Silicon.

## 1. Projekt- & Paketmodell

```
mein_projekt/
  sspl.toml          # [paket] name, version, edition="v10"; [ziele]; [abhaengigkeiten]
  src/               # .sspl-Quellen, paket-Hierarchie = Ordner-Hierarchie
  tests/             # @test-Module + emu_test/.trace-Fixtures
  sspl.lock          # Lockfile (Versions+Hashes, reproduzierbar)
```

- `sspl.toml` = einzige Build-Authority; Feature-Flags (`features = ["ssge","emu"]`), Ziel-Triples, `[ki]`-Policies (`require_verified`, `auto_freigabe`).
- `sspl paket add/neu/baue/testen/formatieren/pruefen/migriere` — ein CLI, Unterbefehle wie Cargo.

## 2. Zielplattformen (Tier-Matrix)

| Triple | Tier | Grafik (SSGE via wgpu) | Anmerkung |
|---|---|---|---|
| `x86_64-pc-windows-msvc` | **1** | Vulkan/DX12 | Neo-Cube + Spiele Hauptziel |
| `aarch64-apple-darwin` | **1** | Metal | M-Prozessoren; NEON-Pfad Pflicht (Paired-Single-Parität!) |
| `x86_64-unknown-linux-gnu` | 1 | Vulkan | CI/Server/Steam-Deck-Nähe |
| `aarch64-unknown-linux-gnu` | 2 | Vulkan | |
| `wasm32-unknown-unknown` | 2 | WebGPU | Web-Demos, Sandbox-Plugins |
| `freestanding-*` | 3 | — | Bare-Metal-Experimente (`@freestanding`) |

**Cross-Compile-Policy:** Tier-1 in CI nativ gebaut+getestet; Cross via `zig cc`-Toolchain-Plugin als Opt-in; kein Versprechen „alles überall" — Tier-Definition bindet Support-Level ehrlich.

## 3. SSGE-Bindung & Spieleentwicklungs-Profil `sspl-spiel`

- SSGE (wgpu: Frame-Graph, Meshlets, ReSTIR, HWRT, Partikel, Virtual-Texturing, Upscaling, PostFX, Audio) wird als **Stdlib-Namespace `ssge.*`** exponiert — Komponenten-Integration: `SSGE.erstelle_kontext()` liefert `zeiger`/Handle-Typ, kanonisch `ssge::Kontext`.
- `sspl-spiel`-Profil (Library-Profil wie `sspl-emu`): `fenster` (window), `eingabe` (Controller/Tastatur — SI-Datenquelle für Neo-Cube!), `audio` (Buffer-Contract für DSP-Ausgabe), `szene`/`material` (`visual`/`material`-Keywords → SSGE-Objekte), Frame-Loop `rahmen { }` mit `zyklus`-Kopplung.
- **GameCube-Grafik-Mapping (Neo-Cube):** GX-Kommando-Stream → `flipper`-Komponente → SSGE-Render-Passes (TEV→Material-Graph); SSGE liefert die Host-Pipeline, nicht die Gast-Semantik (S-08 bleibt Emulator-seitig).
- `hardware`-/`memory`-Deklarationen in Spielen erlaubt (Modding/Embedded), aber Profil `sspl-spiel` setzt `mem`-/IO-Capabilities restriktiv.

## 4. IDE & LSP

- LSP v1: Completion, Hover (Typ+Docs), Definition/Referenzen, Rename, `fixes`-CodeActions aus S-02-Diagnostik, Inlay-Hints (inferierte Typen), Symbol-Graph-Export (KI-Kontext, S-09).
- Formatter: kanonischer Stil — 4 Leerzeichen, LF, ein Statement/Zeile, `//`-Kommentare, Imports sortiert, Leerzeilen normalisiert; `sspl formatieren --check` in CI. Stabilität = KI-Diff-Freundlichkeit.
- `sspl arzt` (doctor): **Verhaltens-**Prüfung — Toolchain erreichbar, Testprojekt kompiliert+läuft, MMIO-Bench in Toleranz, Toolchain-Versionen gemeldet (nicht Dateiexistenz wie v9.5).

## 5. Debugging & Profiling

- **DAP-Adapter:** Breakpoints, Steps, Watch (inkl. `view`-Inspektion als Hex/Endian-View!), Komponenten-/Task-Übersicht, Kanal-Inspektion.
- **Chrono-Debugger** (v9.5-Idee real): Replay via `laufzeit.zeichne_auf` (S-06) — Debugger spulen deterministisch zurück.
- **VRF 2.0 = Profiler:** strukturierte Span-Events (kein REST-Polling): `vrf.span("jit_block")`, Flame/Timeline-Export, Cycle-Zoom auf `zyklus`-Achse (Emu: Instruktions-Level), Hot-Path-Report in Diagnostik-JSON.
- Emu-spezifisch: Gast-PC↔Host-PC-Map aus Dynarec (S-08) für Breakpoints auf PPC-Adressen + Disassemblierer-View.

## 6. Paket-Registry & Distribution

- `.ssplx` v2 (S-05 §3) als Artefakt: signiert (Ed25519), manifest (`name,ver,abi,targets,hash_baum`), reproduzierbar (`--deterministisch`).
- Registry-Policy: semver erzwungen, `minimumReleaseAge` (7 Tage), yank statt löschen, SBOM je Release, Lockfile-Pflicht für Anwendungen.
- Kompatibilitäts-Shim-Paket `sspl-kompat` für Migrations-APIs (`vrf::aufruf`, string-Events, `objekt`-Helfer) — opt-in, deprecated-markiert.

## 7. Dokumentation & Lernpfad

- `sspl doku` generiert API-Docs aus `@doc`/Kommentaren (Markdown + JSON-Manifest für KI-Kontext).
- Kuratierte Beispiele im Repo: `beispiele/hallo`, `beispiele/emu_minimal` (Bus+Instruktion), `beispiele/ssge_dreieck`, `beispiele/komponenten_chat` — diese sind zugleich Smoke-Tests.

## 8. Akzeptanzkriterien

- `sspl neu demo && sspl baue && sspl testen` funktioniert out-of-box auf allen Tier-1-Zielen.
- Formatter: `formatieren(formatieren(x)) == formatieren(x)` (idempotent); Golden-Style-Tests.
- LSP: alle S-02-`fixes` als CodeActions; Rename auf Repo mit ≥100 Dateien korrekt.
- `sspl arzt` auf sauberem System < 60 s, alle Checks verhaltensbasiert.
- SSGE-Demo: 3D-Szene auf Windows (Vulkan) + macOS ARM (Metal) mit identischer Szene-Datei.
