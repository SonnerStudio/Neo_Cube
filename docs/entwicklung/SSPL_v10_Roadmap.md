# SSPL v10 — Entwicklungs-Roadmap

**Status:** Entwurf · **Datum:** 2026 · **Geltungsbereich:** SSPL v10 (`C:\Dev\SSPL-v10`), Migration von v9.5, Zielprojekt Neo-Cube

Diese Roadmap leitet aus einer vollständigen Untersuchung von SSPL v9.5 (installierte Schmiede + C#-Host `sspl_host.cs`, ~2.215 Zeilen) und dem Rust-Workspace v10 ab, welche Entwicklungsschritte nötig sind, um

1. einen **echten, leistungsfähigen Compiler** zu erzeugen,
2. eine **hochmoderne Programmiersprache** hervorzubringen, die für **Emulatorenentwicklung** außerordentlich geeignet ist, und
3. die **Referenzsprache für KI-gestützte Programmentwicklung** zu werden.

Vorbemerkung zur Beweislage: Dokumente, Manifeste, Hash-Artefakte und „OK"-Ausgaben der v9.5-Schmiede gelten **nicht** als Implementierungsnachweise. Bewertet wird ausschließlich, was im Code nachweisbar ausgeführt wird. Ergänzend: `docs/entwicklung/SSPL_Emulator_Faehigkeitsanalyse.md` (erste Bewertung, ~4,4/10).

---

## Teil 1 — Gesamtaudit SSPL v9.5 (Fähigkeiten und Veranlagungen)

### 1.1 Was die Sprache konzeptionell vorsieht (die „Veranlagungen")

Die Schmiede-Quellen zeigen eine erstaunlich vollständige Sprachvision — allerdings über **mindestens vier inkonsistente Dialekte** verteilt:

| Dialekt | Beispiel | Ort |
|---|---|---|
| Großschreib-Blockdialekt | `MODUL`, `FUNKTION ... ENDEFUNKTION`, `STRUKTUR ... ENDESTRUKTUR`, `GIBT x ZURUECK` | `kern/`, `system/`, `kompilierer/` |
| Klein/Klammer-Dialekt | `importiere`, `exportiere KLASSE`, `variabel`, `versuche/fange`, `->` | `ai/ollama_bruecke.sspl`, `nlp/` |
| M4-/Sicherheitsdialekt | `@modul`, `oeffentliche sichere_klasse`, `native_funktion`, `ki_funktion`, `schwarm {}`, `vernichte` | `SSPL_Syntax_Richtlinien.md` |
| Neo-Cube-Dialekt | `paket`, `komponente`, `funktion`, `vrf.aufruf(...)` | Emulator-Repo (`*.sspl`) |

**Kein einziger Dialekt wird vollständig geparst.** Der installierte v9.5-Lexer zählt nur 6 Schlüsselwörter (`FUNKTION, MODUL, IMPORTIEREN, WENN, ABSICHT, PROGRAMM`); der Neo-Cube-Dialekt erzeugt „0 SSPL-Keywords erkannt".

Konzeptionell deklarierte Sprachmittel: Module mit Alias-Importen, Strukturen und Klassen mit Konstruktoren, Generics (`Liste<T>`), Intent-Blöcke (`ABSICHT`), Quanten-Keywords (`Qubit`, `SUPERPOSITION`), NLP-Vorverarbeitung (`#NLP_MODUS: SEMANTISCH`), Ereignis-Bus, Komponenten-Registry, Capability-Sicherheitsprofile, Bare-Metal-Keywords (`hardware`, `volatile`, `register`, `memory`, `port`, `cpu`, `platform`) samt Attributen (`@freestanding`, `@no_mangle`, `@interrupt`, `@link_section`).

### 1.2 Was die v9.5-Laufzeit tatsächlich tut

| Bereich | Behauptung/Dokumentation | Verifizierter Zustand |
|---|---|---|
| `run` | „führt SSPL aus" | Text-Matching-Interpreter: `drucken()` gibt aus, `VARIABLE` wird als String gespeichert, Funktionsaufrufe liefern **hartkodierte Mock-Structs** (Schulen, Spenden, Ledgers, Support-Fälle). Kein AST, keine Ausdruckssemantik außer `+`-Verkettung. |
| `baue` / „nativ" | „kompiliert zu nativem Binary" | Schreibt `MCODE:x86_64_AVX2` + SERIL-Text in `.out`. **Keine Codegenerierung.** |
| `.ssplx` | „Binärpaket" | Quelltext + `SSPLX`-Header + SHA-256-Footer. Lader validiert Marker, dann Textausführung. |
| `lexer.sspl` | „Lexikalischer Analysator" | Gibt immer leere Tokenliste zurück; Scanner auskommentiert („würde hier..."). |
| `seril_emitter.sspl` | „IR/Bytecode-Emission" | Erzeugt feste Marker-Strings (`[SERIL:OP_CONST] ...`). Opcode-Namen existieren, keine Semantik. |
| `rufe_nativ_auf` / `host_dispatch` | „Native-Brücke" | Stub → konstant `0`. |
| `alloc`/`ptr_read`/`ptr_write` | „Speicherprimitiven" | Fake-Adresse (arithmetisch), `ptr_read` → `0`, `ptr_write` → No-Op. |
| `selfbuild` | „Selbstkompilierung" | Hasht Quellen, ruft `csc.exe` für den C#-Host; Fallback: kopiert `sspl.exe`. Kein SSPL-in-SSPL-Compiler. |
| `ssge`/`ssob`/`endgame`/`pro` | Phase-3..7-Reife | Zeremonielle Prüfung auf Dateiexistenz + Hash-Artefakte + Manifeste. Keine Laufzeitsemantik. |
| `vrf` | „C++-Engine" | `vrf-api.exe` ist ein echter REST-Dienst (Health/Telemetrie), aber **keine** JIT-/GPU-Engine. |
| `doctor` | „Systemdiagnose" | Prüft Dateiexistenz und Keyword-Vorkommen — nicht Verhalten. |
| `entwickle` | „Dev-Server + Hot-Reload" | Filewatcher, der `run` neu auslöst — funktioniert real. |
| NLP/Ollama | „Semantische Code-Generierung" | `ollama_bruecke.sspl` ist konzeptionell sauber (System-Prompt, Auto-Fix-Loop), ruft aber `http.post`/`json` über die Stub-Native-Brücke → nicht ausführbar. |

### 1.3 Erhaltenswerte Ideen aus v9.5

Unabhängig vom Implementierungsstand sind diese Konzepte echte Differenzierer und gehören in v10 fortgeführt:

- **Intent-/KI-nahe Sprachmittel** (`ABSICHT`, NLP-Präprozessor, Auto-Fix-Schleife) — Vorstufe zu „verified apply".
- **Ereignis-Bus + Komponenten-Registry** — passt zur CBA-Architektur von Neo-Cube.
- **SSOB** (binäres Objektformat, „JSON verboten auf dem Draht") — als Paket-/IPC-Format weiterdenken.
- **VRF als Observability-Layer** — als echter Profiler/Tracer für Sprache und Emulator ausbauen.
- **Bare-Metal-Keywords + Attribute** — in v10 bereits lexikalisch verankert; das ist genau das Systems-Profil, das ein Emulator braucht.
- **Deutsche Syntax als First-Class-Feature** — beibehalten, aber auf **eine** kanonische Grammatik vereinheitlichen.

---

## Teil 2 — Ist-Stand SSPL v10 (Rust-Workspace)

Verifiziert vorhanden:

- **Lexer** (logos): umfangreich — `fn/FUNKTION`, `let/VARIABLE`, `mut`, `if/WENN`, `else/SONST`, `for/FUER`, `while/SOLANGE`, `match`, `trait`, `impl`, `async/await`, `spawn`, `try/recover`, `unsafe`, `import/IMPORTIEREN`, `enum`, `type`, Pipe-Operatoren (`|>`, `<|`, `>>`, `<<`), Hex-/Binär-/Template-Literale, Bare-Metal-Keywords und `@`-Attribute, SSML/`window`/`material`-Tokens.
- **Parser + Type-Checker + Tree-Walk-Interpreter** mit reichem `Value`-Modell (getaggte Ints/Floats, `IntLimbs` 256-Bit-Softint, `FloatSoft`, Listen/Maps, Enum, Result/Option, Futures, Channels, Qubit, NativeFunction).
- **Nebenläufigkeit:** `spawn` → OS-Threads, mpsc-Channels, Netzwerk-Channels.
- **SERIL-Modul** in Rust: `ir.rs`, `lower.rs`, `interp.rs`, `validator.rs`, `ymm.rs` — deutlich realer als v9.5, weiterhin IR ohne nativen Backend-Ausgang.
- **Wasm-AOT-Backend** (`wasm-target`), **Freestanding-Target**, **Plugin-System** (libloading), **Debugger-Grundgerüst** (snapshots/timeline/ui), **LSP**, **SSGE** (wgpu-Renderer: Frame-Graph, Meshlets, ReSTIR, HWRT), **NLP** (llama.cpp/Ollama, Preview-Gate), Property-Tests, `verify`-Modul.
- **Stdlib:** ~43 Native-Funktionen (fs, http, json, list/map/string, sys, uuid, sleep).

Verifiziert fehlend oder Platzhalter:

- **Kein echtes Raw-Memory** (`alloc`/`ptr_read`/`ptr_write` wie in v9.5 deklariert → simuliert).
- **Kein nativer Code-Generator** (x86_64/ARM64). Wasm ist der einzige reale Backend-Pfad.
- **Kein Dynarec/JIT-Framework** — für einen Emulator zwingend.
- **Kein deterministischer Zyklus-/Event-Scheduler.**
- **Kein Speichersicherheitsmodell** (Ownership/Borrowing/Arenen) — trotz `unsafe`-Keyword.
- **Kein ABI-Vertrag** für hochfrequente Host↔SSPL-Aufrufe (MMIO-Callbacks!).
- Dialekt-Fragmentierung wie in v9.5 weiterhin ungelöst (Lexer „skippt" deutsche Rausch-Tokens statt sie zu normieren).

---

## Teil 3 — Roadmap

Legende: **P0** = Blocker (ohne sie kein echter Compiler/Emulator), **P1** = Qualitäts-/Produktivitätskern, **P2** = Differenzierung/Ausbau. Jede Phase nennt Liefergegenstand und Akzeptanzkriterium (messbar, nicht manifest-artig).

### Phase 0 — Wahrheits- und Spezifikations-Basis (P0)

**Ziel:** Eine Grammatik, eine Semantik, ein ehrlicher Statusbericht.

1. ✅ **EBNF-Gesamtgrammatik v10** — abgeschlossen: `docs/entwicklung/SSPL_v10_Grammatik_EBNF.md` (Freeze v0.1, alle §8-Detailentscheidungen gefällt).
2. **Dialekt-Entscheid fällen:** ✅ **Gefallen:** Der Neo-Cube-Dialekt wird in die kanonische Grammatik aufgenommen — `komponente` ist echter Sprachbaustein (Phase 5), `paket` die Modul-Deklaration. Spezifiziert in `docs/entwicklung/SSPL_v10_Grammatik_EBNF.md`.
3. **Konformitätssuite** — ✅ spezifiziert: `docs/entwicklung/sspl_v10/12_Konformitaetsstufe_Spezifikation.md` (Stufen KS-0..KS-7 + Profil-Stufen, Fixture-Format, Coverage-Anforderungen, Zertifizierungs-Gates). Implementierung im v10-Workspace steht aus.
4. ✅ **Feature-Truth-Table** — abgeschlossen: `docs/entwicklung/sspl_v10/01_Feature_Truth_Table.md`. Regel: „behauptet ≠ bewiesen", nur verlinkte Tests zählen.
5. ✅ **Diagnostics-Format** — abgeschlossen: `docs/entwicklung/sspl_v10/02_Diagnostik_Spezifikation.md` (`sspl-diag/1`, `E*`-Codes, strukturierte `fixes`).

**Akzeptanz:** 100 % der Konformitätssuite grün; `sspl check` meldet jeden Grammatikbruch mit Span + Fehlercode; kein Feature ohne Tabelleneintrag.

> 📐 **Alle Phasen-Spezifikationen:** `docs/entwicklung/sspl_v10/` (S-01..S-11 + Entscheidungsregister D-001..D-027 in `00_Index_Entscheidungsregister.md`). Diese sind verbindlich für die Implementierung; Änderungen nur per RFC.

### Phase 1 — Produktions-Frontend (P0)

1. Parser härten: vollständige Source-Spans, Fehler-Recovery (mehrere Fehler pro Lauf), deterministische AST-Serialisierung (stabiler Hash für Caching/KI-Diffs).
2. **Namensauflösung & Modulsystem:** lexikalische Scopes, Sichtbarkeit, Aliase, Paket-Grenzen; Zyklen-Erkennung.
3. Type-Checker vervollständigen: Inferenz, Generics-Monomorphisierung oder -Dictionary, `trait`-Bound-Prüfung, `match`-Exhaustiveness, numerische Breitenkorrektheit (u8..u256, i*, f32/f64 + Soft-Varianten).
4. **Eingebaute Metadaten:** Dok-Kommentare → maschinenlesbare API-Docs; `@target`-Attribut semantisch machen.

**Akzeptanz:** Typsystem fängt in der Suite alle definierten Fehlerklassen ab; zwei über die Stdlib kompilierbare Beispielprogramme laufen identisch im Interpreter und späteren Backend.

### Phase 2 — Speicher- und Sicherheitsmodell (P0)

Der wichtigste Sprachbeschluss. Empfohlene Architektur:

- **Ownership + Borrowing light:** lineare/`move`-Semantik für Ressourcen, Borrow-Checking nur wo nötig; `Rc`/`arena`/`pool` als **explizite Allokatoren** (`arena<T>` als Stdlib-Typ), kein globaler GC — passt zu „kein GC"-Vision und zu Emulator-Determinismus.
- **`unsafe`-Blöcke** mit echtem Vertrag: Raw-Pointer (`*T`, `*mut T`), `alloc/dealloc` auf echten Arenen (mmap/VirtualAlloc), `ptr_read/ptr_write` **real** implementieren, Bounds-/Align-Checks im Debug.
- **Endianness als Typ-/Layout-Eigenschaft:** `u32be`, `u16be` bzw. `layout(C, big_endian)` auf Strukturen — für PowerPC-Gast-Adressraum essenziell.
- **Bitfelder & gepackte Layouts** für Hardware-Register (`register`-Keyword semantisch machen: `volatile`, MMIO-Mapping auf Host-Adresse).
- **Sicht-Typen:** `view<T>` = nicht-besitzendes Fenster auf Arena (MEM1/ARAM-Mapping für den Emulator).

**Akzeptanz:** Fuzz-Test schreibt/liest `u32be`-Werte in eine Arena und liefert auf x86 und ARM byte-identische Ergebnisse; Miri-artige UB-Prüfung (oder sanitizer) auf den Unsafe-Kernen grün.

### Phase 3 — IR, Optimierung und native Backend (P0)

1. **SERIL neu definieren** als getypte SSA-IR (das Rust-Modul ist der Startpunkt): Blöcke, Phi, Typen, Effekte annotiert; Validator als Pflichtpass.
2. **Backend-Strategie (Empfehlung):** **Cranelift** als Baseline-Codegenerator (schnell, JIT-tauglich, x86_64 + AArch64) + optional LLVM für Release-Builds. Wasm-Backend bleibt als drittes Ziel.
3. **Object/Link-Pfad:** echte Objektdateien (COFF/ELF/Mach-O via `object`-Crate), System-Linker-Anbindung; `sspl build --release` erzeugt ausführbares Binary.
4. **Paketformat `.ssplx` v2:** binärer Container (Header, Symbole, Abschnitte, Signatur) — löst die Textdatei ab.
5. **Optimierungs-Pipeline:** Inlining, GVN, DCE, Loop-Passes auf SERIL; Profil-geführte Tiers.

**Akzeptanz:** `hello.sspl` → natives PE-Executable, dessen Ausgabe zeitidentisch mit dem Interpreter ist; Fibonacci-Benchmark nativ ≥ 50× schneller als Tree-Walk; deterministischer Byte-Output bei gleichem Input (reproduzierbarer Build).

### Phase 4 — Laufzeit, Nebenläufigkeit, deterministisches Zeitmodell (P0 für Emulator)

1. **Strukturierte Nebenläufigkeit:** `spawn` in Scopes; Kanäle: SPSC/MPMC lock-free für Hotpaths, async/await auf eigenem Executor (kein externes Tokio-Abhängigkeitsgewicht im Kern).
2. **Determinismus-Modus:** seedbarer Scheduler — gleiche Eingaben → identische Interleavings (Netplay/Tests/Debugging).
3. **Virtuelle Zeit als Sprachprimitive:** `cycle`-Zähler, `schedule_at(cycles, event)`, deadline-sorted Event-Queue — das Herzstück für taktgenaue CPU/GPU/DSP-Synchronisation.
4. **Interrupt-Modell:** `signal`/`raise`/`mask`-Konstrukte, die auf die Event-Queue abbilden.
5. **Atomics & Memory-Ordering** explizit (`acquire/release/seqcst`), kein implizites Locking.

**Akzeptanz:** Referenz-Test „zwei Komponenten tauschen 1 Mio. Events über Cycle-Queue" deterministisch; Netplay-artige Replay-Tests (Aufzeichnen → Wiederholen → Bit-Identität).

### Phase 5 — Komponentenmodell & FFI/ABI (P0/P1)

1. **`komponente` als Sprachbaustein** (übernimmt den Neo-Cube-Dialekt): deklarierte Ports/Events, Lifecycle (`init/start/stop`), isolierter Zustand — kompiliert auf Task + Kanal-Endpunkte.
2. **FFI v1:** `extern "C"` beidseitig — SSPL ruft C/Rust, C/Rust ruft SSPL-Callbacks (für MMIO-Read/Write-Handler!). Automatischer Binding-Generator aus Headern.
3. **ABI-Stabilität:** versionierte Aufrufkonvention; Plugin-System (existiert via libloading) auf dokumentiertes ABI heben; `repr(C)`-Layouts garantieren.
4. **Callback-Schnellpfad:** MMIO-Handler müssen < ~20 ns/Aufruf kosten → direkter Funktionszeiger, kein Serialisierungsweg.

**Akzeptanz:** Roundtrip-Test: SSPL-Komponente registriert nativen Callback; Rust-Testrufer feuert 1 Mio. MMIO-Reads; Latenz und Korrektheit gemessen.

### Phase 6 — Emulator-Profil (`sspl-emu`) (P1, produktkritisch für Neo-Cube)

Eine **Profil-Schicht** aus Sprachmitteln + Runtime-Bibliothek, kein separater Dialekt:

- **Adressraum-Modell:** `address_space` mit Regionen (RAM, MMIO, ROM), Big-Endian-Views, Mirror-Mapping (MEM1@0x8000_0000/0xC000_0000).
- **Dynarec-Framework:** Code-Puffer (W^X-konform, `mmap`+`mprotect`/VirtualAlloc+FlushInstructionCache), Block-Cache, Block-Linking, Invalidate-on-Write (SMC); Translationseinheit = Basisblock mit Gast-PC-Map für Debugger.
- **Host-Feature-Detektion:** AVX2/AVX-512/NEON, Timer-Auflösung, Core-Pinning — via `platform`-Keyword real gemacht.
- **Vektoren/Paired-Singles:** `f32x2`/`f32x4` Typen mit definierter Semantik (Gekko-Paired-Single-Verhalten: gemischte Präzision!).
- **Float-Konfiguration:** IEEE-Modus + „GC-Modus" (Non-IEEE-Feineinstellungen wie auf Gekko dokumentiert), deterministisch über `FloatSoft`-Pfad für Tests.
- **Snapshot/Serialisierung:** vollständiger Emulator-Zustand SSOB-serialisierbar (Savestates, Netplay-Sync, Chrono-Debugger!).
- **DMA/Interrupt-Verdrahtung** über Phase-4-Event-Queue.

**Akzeptanz:** Referenz-Emulator-Milestone: DOL-Loader lädt Homebrew-`hello.dol` in `address_space`, Gekko-Interpreter (P0) bzw. Dynarec (P1) führt Integer-Code aus, VI-Stub liefert Frame-Callback. Differential-Test gegen bekannter PPC-Testsuite (z. B. Teile aus Dolphin-Tests/ppc-instruction-tests).

### Phase 7 — JIT/Dynarec-Vertiefung (P1)

1. Zwei-Stufen-Strategie: korrekter Interpreter zuerst (Referenz), dann Dynarec PPC→Host via eigenem kleinen IR→Cranelift-Pfad.
2. Gast-spezifische Optimierungen: Branch-Link-Cache, Condition-Register-Lazy-Eval, PS-Load/Store-Fusion.
3. Sicherheitsgrenzen: Gast-Code kann Host nicht korrumpieren (Sandbox-Checks am Adressraumrand).
4. ARM64-Primärziel (Apple Silicon) + x86_64 — beide Pfad in CI.

**Akzeptanz:** Gleiche PPC-Testsuite in beiden Modi bit-identisch; Leistungsziel ≥ 10× Interpreter auf Dekoder-Benchmark.

### Phase 8 — Standardbibliothek & Ökosystem (P1)

- `core` (no_std-fähig): numerische Breiten, bit-ops, Endianness, arenas, slices, simd.
- `std`: collections (inkl. ringbuffer/pool für Emulator), io, net (TCP/UDP/HTTP), time (echte + virtuelle), crypto, ssob, log/trace (VRF-Events als Schema).
- `ssplx`-Paketmanager: Abhängigkeitsgraph, Lockfile, reproduzierbare Builds, signierte Artefakte.
- Dokumentationsgenerator + „Language Server Index" für Repo-weite Symbolgraphen (KI-Kontext!).

### Phase 9 — Tooling & Developer Experience (P1)

1. **LSP vollenden:** Completion, Hover mit Typen, Rename, Code-Actions, Inlay-Hints; Symbolgraph-Export für KI-Indexing.
2. **Formatter** mit kanonischer Ausgabe (stabile Diffs — KI-freundlich).
3. **DAP-Debugger:** Breakpoints, Steps, `chrono`-Ringbuffer-Replay (v9.5-Idee, jetzt real via Snapshots-Modul).
4. **VRF 2.0 als echter Profiler/Tracer:** strukturierte Event-Streams (span-basiert, kein REST-Polling), Flamegraphs, Timeline-Zoom — die 10-ns-Zoom-Idee wird zur Cycle-Zoom für den Emulator.
5. **`sspl doctor`**: prüft Verhalten statt Dateiexistenz.

### Phase 10 — KI-First-Plattform (P1, Alleinstellungsmerkmal)

Ziel: die Sprache, in der KI-generierter Code **am zuverlässigsten** ist.

1. **Maschinenlesbare Spezifikation:** EBNF + `spec`-Artefakt pro Release; KI konsumiert Grammatik, nicht Folklore.
2. **Strukturierte Diagnostik** (aus Phase 0) mit vorgeschlagenen Code-Actions → Auto-Fix-Loop (v9.5-Idee) auf echter Basis.
3. **`verified apply`:** KI-Patch → format → parse → typecheck → test → sandbox-run → Diff-Report → menschliche Freigabe. Kompiler-Autorität bleibt letzte Instanz, nie das Modell.
4. **Intent-Blöcke (`ABSICHT`) real machen:** deklarative Absicht + generierter Code + **Proof-Obligations** (Type-/Effekt-/Property-Checks), die der Compiler verifiziert. Scheitern = sauberer Compile-Fehler.
5. **Provenance & Audit:** jede KI-Änderung mit Signatur (Modell, Prompt-Hash, Patch-Hash) im Build-Manifest.
6. **Prompt-Injection-Schutz:** generierter Code läuft im Sandbox-Interpreter mit Capability-Profil (v9.5s `generiere_sicherheits_profil` — jetzt echt).
7. **Evaluationskultur:** reale Coding-Aufgaben-Suite (nicht nur NLP-210); Metrik = % verifiziert übernommene Patches.

### Phase 11 — Sicherheit, Verifikation, Release-Engineering (P1)

- Reproduzierbare Builds (hermetisch, Lockfile, Hash-Manifeste — aber mit **echtem** Inhalt).
- Supply-Chain: signierte Pakete, SBOM, `minimumReleaseAge`-Regeln für Registry.
- Sanitizer-/Miri-CI, Parser-Fuzzer, Differential-Tests (Interpreter vs. Backend vs. Wasm).
- **Honesty-Gate im Release-Prozess:** kein `full_production_ok` ohne verlinkte Akzeptanztests (die v10-Dokumentationskultur `development_checks_ok` vs `full_production_ok` ist hierfür die richtige Basis — beibehalten!).
- Semver + Stabilitätsversprechen für Grammatik/ABI/`.ssplx`-Format.

### Phase 12 — Migration v9.5 → v10 & Neo-Cube-Anbindung (P1)

1. `sspl migriere`: übersetzt Schmiede- und Neo-Cube-Dialekte auf kanonisches v10 (größtenteils Keyword-/Terminator-Mapping + `komponente`-Umformung).
2. Neo-Cube-`.sspl`-Dateien bleiben **Orchestrierungsspezifikation**: Komponenten-Deklarationen, Event-Verdrahtung, Konfiguration. Silicon-nahe Hotpaths laufen als native Module über die Phase-5-ABI (Rust/C++), bis Phasen 6–7 sie in SSPL absorbieren können.
3. Schmiede-Module (Paketierer, Loader, Dev-Server, SSOB, NLP-Brücke) als v10-Ports mit echtem Verhalten nachbauen — nicht als Marker-Texte.

---

## Teil 4 — Kritischer Pfad & Abhängigkeiten

```
Phase 0 (Grammatik/Wahrheit)
 └─ Phase 1 (Frontend/Semantik)
     ├─ Phase 2 (Speicher-/Sicherheitsmodell) ─┐
     └─ Phase 3 (SERIL-SSA + Cranelift + Linker)
         └─ Phase 4 (Runtime/Concurrency/Zeitmodell)
             └─ Phase 5 (Komponenten + FFI/ABI)
                 └─ Phase 6 (sspl-emu-Profil) → Neo-Cube P0-Blocker gelöst
                     └─ Phase 7 (Dynarec-Vertiefung)
Phase 8–9 (Stdlib/Tooling) parallel ab Phase 3
Phase 10 (KI-First) parallel ab Phase 0, braucht 1+3 für verified-apply
Phase 11–12 quer über alles
```

**Früheste Emulator-Befähigung:** Ende Phase 6. Bis dahin bleibt die vorherige Architekturempfehlung gültig: Neo-Cube-Hotpaths nativ (Rust), SSPL orchestriert.

## Teil 5 — Risiken

| Risiko | Mitigation |
|---|---|
| Speichermodell zu komplex (Ownership light ≠ Borrow-Checker) | Erst Arenen + `unsafe`, Borrow-Checking inkrementell; einfache Fälle müssen einfach bleiben |
| Cranelift allein reicht nicht (PS-Ops, exakte FP-Flags) | Eigenes Mini-Backend für Emu-Intrinsics zulässig; LLVM-Option offenhalten |
| Dialekt-Fork (Neo-Cube vs. Schmiede vs. M4) | Phase-0-Beschluss hat Veto-Charakter: eine Grammatik, Aliase nur lexikalisch |
| „Feature-Inflation" (Qubit, Schwarm, 5D-Timeline) | Nur implementieren, was Akzeptanztests spezifiziert; Vision-Keywords bleiben reserviert, nicht halbgebaut |
| KI-Fähigkeiten ohne Verifikation = v9.5-Wiederholung | Phase-10-Punkte 3/7 sind nicht verhandelbar |

## Teil 6 — Non-Goals (v10-Scope)

- Kein eigener OS-Kernel / keine Hypervisor-Ebene.
- Keine echte Quanten-Hardware-Anbindung (Qubit-Typ bleibt Simulations-/Lern-Feature, klar gekennzeichnet).
- Kein Ersatz für SSGE — SSGE bleibt Grafik-Stack, `sspl-emu` liefert die Gast-GPU-Semantik (Flipper→SSGE-Mapping ist Neo-Cube-Aufgabe).
- Keine vollautonome KI-Codeübernahme ohne Compiler-/Test-Gate.

---

## Nächster konkreter Schritt

Phase-0-Aufgabe 1+2: Die kanonische v10-EBNF schreiben und den Neo-Cube-Dialekt-Beschluss fällen (`komponente` aufnehmen vs. migrieren). Das ist eine Design-Entscheidung, die ich mit dir abstimmen sollte, bevor Code entsteht — sie legt fest, ob die bestehenden `*.sspl`-Dateien von Neo-Cube später 1:1 lauffähig werden oder übersetzt werden müssen.
