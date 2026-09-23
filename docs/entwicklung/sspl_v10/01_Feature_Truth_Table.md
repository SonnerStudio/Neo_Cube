# S-01 · Feature-Truth-Table — SSPL v9.5 & v10

**Zweck:** Ehrlicher Status je Feature. Regel: Ein Feature gilt nur als `implemented`, wenn ein verlinkbarer Test/lauffähiges Artefakt es beweist. Manifeste, Hashes und „OK"-Logs zählen nicht.

**Status-Legende:** `impl` = nachweisbar implementiert · `partial` = teilweise, Lücken benannt · `stub` = Aufruf existiert, keine Semantik · `decl` = nur deklariert/dokumentiert · `n/a` = nicht vorhanden

## A. Sprachkern & Compiler-Pipeline

| Feature | v9.5 | v10 (Rust) | Evidenz v10 | Roadmap-Phase |
|---|---|---|---|---|
| Lexer | stub (zählt 6 Keywords, leere Tokenliste) | **impl** (logos, ~100 Tokenarten) | `src/lexer.rs` | 0/1: kanonische Tokens ergänzen |
| Parser | n/a (String-Matching) | partial (AST vorhanden, Spans/Recovery unklar) | `src/parser.rs`, `parser_chunk.rs` | 1 |
| Namensauflösung/Module | stub | partial | type_checker/interpreter | 1 |
| Typprüfung | n/a | partial (`type_checker.rs` vorhanden, Umfang unbewiesen) | `src/type_checker.rs` | 1 |
| Interpreter | stub (Marker-Interpreter + Mock-Structs) | **impl** (Tree-Walk, reiches `Value`-Modell) | `src/interpreter.rs` | — (Bleibt: Skript/REPL/Debug) |
| IR (SERIL) | stub (Marker-Strings) | partial (`seril/ir.rs`, `lower.rs`, `interp.rs`, `validator.rs`) | `src/seril/` | 3: SSA-Neudefinition |
| Nativer Codegen | **decl** (`MCODE:`-Textmarker) | n/a (kein x86/ARM-Backend) | — | 3: Cranelift |
| Wasm-Backend | n/a | partial (`wasm-target`, AOT) | `src/wasm/` | 3: härten |
| Linker/Object-Output | n/a | n/a | — | 3 |
| `.ssplx`-Paketformat | stub (Text+SHA256) | n/a | — | 3: binärer Container v2 |
| Selfbuild | stub (csc.exe/Kopie) | n/a | — | 11 |
| Diagnostik | stub (Print-Ausgaben) | n/a (kein strukturiertes Format) | — | 0: S-02 |

## B. Laufzeit & Speicher

| Feature | v9.5 | v10 | Evidenz | Phase |
|---|---|---|---|---|
| `alloc`/`ptr_read`/`ptr_write` | stub (Fake-Adresse, `0`, No-Op) | **stub** (identisches Muster im Interpreter) | `src/interpreter.rs` | **2 (P0)** |
| Speichermodell (Ownership/Arenen) | n/a | n/a | — | 2: S-04 |
| Endian-Typen | decl (`ByteFeld`) | partial (Bitbreiten-Laufzeit `IntTagged`/`IntLimbs`) | `src/runtime/`, `interpreter.rs` | 2 |
| Vektoren/SIMD | decl (`OP_CMP_EQ_256`-Marker) | partial (`seril/ymm.rs`, Soft-Typen) | `src/seril/ymm.rs` | 2/6 |
| Volatile/MMIO-Zugriffe | decl | decl (Keywords lexed: `volatile`,`register`,`memory`,`port`) | `src/lexer.rs` | 2/6 |
| GC/Deterministische Speicherung | decl („Antimaterie-GC") | n/a — Beschluss: **kein GC**, Ownership+Arenen | — | 2 |

## C. Nebenläufigkeit & Zeit

| Feature | v9.5 | v10 | Evidenz | Phase |
|---|---|---|---|---|
| `spawn`/Threads | decl (`OP_THREAD_SPAWN`-Marker) | partial (OS-Threads + mpsc-Channels) | `src/interpreter.rs` | 4 |
| Channels | n/a | partial (mpsc; kein spsc/select) | `Value::Channel` | 4 |
| async/await | decl | partial (Keywords + `Value::Future`) | lexer/interpreter | 4 |
| Atomics/Ordering | n/a | n/a | — | 4 |
| Deterministischer Scheduler | decl („5D-Timeline"-Marker) | n/a | — | **4 (P0 Emu)** |
| Zyklus-/Event-Modell | decl (Chrono-Debugger-Idee) | partial (debugger: snapshots/timeline) | `src/debugger/` | 4 |

## D. Komponenten, FFI, Pakete

| Feature | v9.5 | v10 | Evidenz | Phase |
|---|---|---|---|---|
| `komponente`-Baustein | decl (nur Neo-Cube-Dialekt) | n/a (Grammatik Freeze v0.1 definiert) | `docs/entwicklung/SSPL_v10_Grammatik_EBNF.md` | 5 |
| Ereignis-Bus | stub (Queue ohne Verteilung) | n/a | — | 5: typisierte `ereignis`/`on` |
| `vrf.aufruf`/Native-Calls | stub (`host_dispatch` → `0`) | n/a | — | 5: `extern`-Modell ersetzt |
| FFI/ABI | decl | partial (Plugin via libloading, kein ABI-Vertrag) | `src/plugin/mod.rs` | 5: ABI v1 |
| Paketmanager/Registry | n/a | n/a | — | 8 |

## E. Grafik, UI, Plattformen

| Feature | v9.5 | v10 | Evidenz | Phase |
|---|---|---|---|---|
| SSGE (wgpu-Renderer) | decl (nur `.sspl`-Spezifikationen) | partial→impl (Frame-Graph, Meshlets, ReSTIR, HWRT — großes Modulverzeichnis) | `src/ssge/` | 9: Low-Level-Pfad + Profiler |
| GUI (`window`,`material`,SSML) | decl | partial (`gui`-Feature, Tokens lexed) | lexer, `src/gui*` | 9 |
| Zielplattformen | decl („darwin-universal") | partial (`freestanding-target`, `windows-hardware`, Wasm) | Cargo features | 9: S-10 |
| Audio | decl | partial (SSGE-Audio-Modul) | `src/ssge/audio*` | 6/9 |

## F. KI-/Entwicklerplattform

| Feature | v9.5 | v10 | Evidenz | Phase |
|---|---|---|---|---|
| NLP-Übersetzung | stub (Ollama-Brücke über tote Native-Calls) | partial (llama.cpp/Ollama-Provider, Preview-Gate; Qualität 55–154/210) | `src/nlp/` + Statusdok | 10 |
| `absicht`/Intent-Blöcke | decl (Keyword + Resolver-Stub) | n/a | — | 10 |
| `verified apply` (Patch→Check→Test→Gate) | n/a | partial (nlp-preview vorhanden) | `src/nlp/` | 10 |
| Provenance/Audit | decl („Pre-Crime") | partial (`verify`-Modul) | `src/verify/` | 10/11 |
| Strukturierte Diagnostik für KI | n/a | n/a | — | 0: S-02 |
| LSP | n/a | partial (`lsp`-Feature) | Cargo features | 9 |
| Property-Tests | n/a | partial (`property_test`-Modul) | `src/property_test/` | 0/11 |
| Debugger | n/a | partial (snapshots/timeline/ui) | `src/debugger/` | 9: DAP |

## G. Sicherheit & Verifikation

| Feature | v9.5 | v10 | Evidenz | Phase |
|---|---|---|---|---|
| Capability-Profile | stub (festes Profil-String) | partial (`security`-Modul) | `src/security/` | 10/11 |
| Sandbox-Ausführung | n/a | partial (wasm-Pfad prinzipiell sandboxbar) | `src/wasm/` | 10 |
| Signierte Pakete | stub (SHA256-Footer) | n/a | — | 8/11 |
| Fuzzing/Sanitizer-CI | n/a | partial (`property_test`) | — | 11 |

## H. Emulator-Eignung (Zielbild)

| Fähigkeit | Stand | Phase |
|---|---|---|
| Echte Arenen + `view<T>` + `u32be` | ❌ | 2 |
| `memory`/`register`-Mapping mit realer Semantik | ❌ | 2/6 |
| MMIO-Callback-ABI < ~20 ns | ❌ | 5 |
| Zyklus-Scheduler + Event-Queue | ❌ | 4 |
| Dynarec-Framework (Code-Puffer, Block-Cache, SMC) | ❌ | 6/7 |
| Paired-Single-/Vektorsemmantik | ❌ | 6 |
| Snapshot/Serialisierung (SSOB) | ❌ | 6 |
| Differential-Test-Harness | ❌ | 6 |

**Fazit:** v10 ist die einzig tragfähige Basis (echter Lexer/Parser/Interpreter + SSGE + Wasm). Alle emulator-kritischen P0-Lücken sind in den Roadmap-Phasen 2–6 abgedeckt; kein P0-Item existiert heute.
