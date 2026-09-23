# SSPL v10 — Spezifikations-Index & Entscheidungsregister

**Zweck dieses Ordners:** Alle verbindlichen Entscheidungen und Spezifikationen, die die Implementierung von SSPL v10 leiten. Jede Spezifikation (S-xx) endet mit messbaren Akzeptanzkriterien — sie sind die vertragliche Grundlage des Honesty-Gates (S-11 §4).

## Spezifikations-Landkarte

| Spec | Titel | Roadmap-Phase | Kernfrage |
|---|---|---|---|
| `docs/entwicklung/SSPL_v10_Grammatik_EBNF.md` | Kanonische Grammatik (Freeze v0.1) | 0 | Wie sieht die eine Syntax aus? |
| S-01 `01_Feature_Truth_Table.md` | Feature-Truth-Table | 0 | Was existiert wirklich? |
| S-02 `02_Diagnostik_Spezifikation.md` | Diagnostik-Format | 0 | Wie sprechen Compiler & KI über Fehler? |
| S-03 `03_Typsystem_Semantik.md` | Typsystem & Semantik | 1/2 | Besitz, Generics, Effekte, Numerik |
| S-04 `04_Speichermodell_Unsafe.md` | Speicher & Unsafe | 2 | Arenen, Views, Endian, MMIO-Zugriff |
| S-05 `05_SERIL_IR_Backends.md` | SERIL-SSA & Backends | 3 | Echte Codegenerierung |
| S-06 `06_Laufzeit_Nebenlaeufigkeit_Zeitmodell.md` | Laufzeit & Zyklusmodell | 4 | Deterministische Zeit & Tasks |
| S-07 `07_Komponenten_FFI_ABI.md` | Komponenten, FFI, ABI v1 | 5 | Verdrahtung & native Grenzen |
| S-08 `08_Emulator_Profil.md` | `sspl-emu` | 6/7 | Emulatoren für beliebige Systeme |
| S-09 `09_KI_Entwicklungsplattform.md` | KI-First-Plattform | 10 | Verifizierte KI-Entwicklung |
| S-10 `10_Tooling_Zielplattformen_SSGE.md` | Tooling, Targets, SSGE-Spiel | 8/9 | IDE, Pakete, PC & Apple-Silicon-Spiele |
| S-11 `11_Migration_Release_Policy.md` | Migration & Release | 11/12 | Übergang, Versionierung, Honesty-Gate |
| S-12 `12_Konformitaetsstufe_Spezifikation.md` | Konformitätsstufen (`sspl-konform`) | 0→alle | Zertifizierungslevel KS-0..KS-7 + Profil-Stufen |

## Entscheidungsregister (D-Nummern)

| # | Entscheidung | Spec | Status |
|---|---|---|---|
| D-001 | Neo-Cube-Dialekt in kanonische Grammatik aufgenommen; `komponente` = Sprachbaustein | EBNF | ✅ gefällt |
| D-002 | `{}`-Blöcke kanonisch; `ENDE*`-Terminatoren nur via `migriere` | EBNF §1 | ✅ gefällt |
| D-003 | Aliase = Lexer-Mapping, nie eigene AST-Knoten | EBNF §3 | ✅ gefällt |
| D-004 | Klammern bei `wenn`/`waehrend` verpflichtend | EBNF §8.1 | ✅ gefällt |
| D-005 | `>>`/`<<` Doppelbelegung: Typ-Disambiguierung, kein Rename | EBNF §8.2 | ✅ gefällt |
| D-006 | `/* */`-Kommentare, `fange(e[:Typ])`, `;`-Pflicht, `fuer`→Werte | EBNF §8.3–8.6 | ✅ gefällt |
| D-007 | Diagnostik = stabile `E*`-Codes + JSONL-Schema `sspl-diag/1` | S-02 | ✅ spezifiziert |
| D-008 | Besitz = „Ownership light" (Move + lokale Borrows, kein Full-Borrow-Checker), kein GC | S-03 §3 | ✅ spezifiziert |
| D-009 | Keine impliziten numerischen Konversionen; Endian-Typen eigenständig | S-03 §4 | ✅ spezifiziert |
| D-010 | Generics = Monomorphisierung; kein Funktions-Overloading | S-03 §5 | ✅ spezifiziert |
| D-011 | Effekt-System explizit (`rein/io/nativ/unsicher/zyklisch`) | S-03 §7 | ✅ spezifiziert |
| D-012 | `comptime` statt Makrosystem (v10.0) | S-03 §10 | ✅ spezifiziert |
| D-013 | `view<T>` = Emulator-Zugriffstyp (Fat-Pointer + Endian-Tag); `zeiger` nur in `unsafe` | S-04 | ✅ spezifiziert |
| D-014 | Cranelift = Primär-Backend; LLVM optional; Wasm & Interpreter bleiben | S-05 §1 | ✅ spezifiziert |
| D-015 | Endianness = Attribut an load/store, nicht Registertyp | S-05 §2 | ✅ spezifiziert |
| D-016 | Virtueller `zyklus`-Zähler + deadline-Event-Queue als Sprachprimitive; Interrupts darauf abgebildet | S-06 §4 | ✅ spezifiziert |
| D-017 | Determinismus-Modus + Replay via SSOB (Netplay/Chrono-Debug) | S-06 §5 | ✅ spezifiziert |
| D-018 | `komponente` = Referenztyp + Task-Scope; typisierte `ereignis`/`on`; Ports = spsc-Kanäle | S-07 §1 | ✅ spezifiziert |
| D-019 | FFI = C-ABI bidirektional; `sspl-abi/1` versioniert; MMIO-Handler < 20 ns Ziel | S-07 §3/4 | ✅ spezifiziert |
| D-020 | `sspl-emu` systemgenerisch: `Adressraum`/`CpuKern`/`GastMmu`/DMA/Taktdomänen; Gekko = Referenz | S-08 | ✅ spezifiziert |
| D-021 | Interpreter = Referenz; Dynarec muss differential beweisen | S-08 §1 | ✅ spezifiziert |
| D-022 | `verified apply`: KI-Patch→Gates→menschliche Freigabe; Compiler-Autorität letzte Instanz | S-09 §3 | ✅ spezifiziert |
| D-023 | `absicht` = Proof-Obligations (erfordert/sichert/invariant → Property-Tests/Asserts) | S-09 §4 | ✅ spezifiziert |
| D-024 | Tier-1-Ziele: `x86_64-pc-windows-msvc`, `aarch64-apple-darwin`, `x86_64-unknown-linux-gnu` | S-10 §2 | ✅ spezifiziert |
| D-025 | Formatter: 4 Spaces, LF, idempotent — KI-Diff-Freundlichkeit | S-10 §4 | ✅ spezifiziert |
| D-026 | Honesty-Gate: keine Feature-Behauptung ohne Test-Artefakt; `development_checks_ok` ≠ `full_production_ok` | S-11 §4 | ✅ spezifiziert |
| D-027 | Kompat-Paket `sspl-kompat` für `vrf.aufruf`/String-Events — opt-in, deprecated | S-11 §2 | ✅ spezifiziert |
| D-028 | Konformität = abgestufte Zertifizierung KS-0..KS-7 + Profil-Stufen (emu/spiel/ki/freestanding); `sspl konform` ist Selbsttest und Fremdzertifizierung | S-12 | ✅ spezifiziert |
| D-029 | Feature-Status `impl` nur bei grüner Fixture in erreichter KS-Stufe (Honesty-Kopplung) | S-12 §5 | ✅ spezifiziert |

## Umsetzungs-Reihenfolge (aus Roadmap, mit Spec-Abhängigkeiten)

```
Phase 0: EBNF✅ + S-01✅ + S-02▶ (Diagnostik implementieren)
Phase 1: S-03 → Frontend (Parser auf kanonische Grammatik, Name-Resolution, Typcheck)
Phase 2: S-04 → Arena/View/Endian/unsafe in Runtime + Backend-Verträge
Phase 3: S-05 → SERIL-SSA-Validator, Cranelift-Tier, Objekt/Link, .ssplx v2
Phase 4: S-06 → Executor, spsc/mpsc, zyklus-Queue, Determinismus
Phase 5: S-07 → komponente/ereignis/Ports, extern/ABI, Plugins
Phase 6: S-08 → sspl-emu: Adressraum, CpuKern, Gekko-Interpreter, MMIO
Phase 7: S-08 §3 → Dynarec via SERIL→Cranelift, Block-Linking, SMC
Phase 8–9: S-10 → Stdlib, ssplx-Registry, LSP, DAP, VRF-Profiler, SSGE-Bindung
Phase 10: S-09 → grammatik.json, verified-apply, absicht-Obligations, Provenance
Phase 11/12: S-11 → migriere, kompat-Shim, Signierung, Release-Gates
```

## Leseempfehlung für Implementierung

1. Parser-Arbeit: EBNF → S-02 (Diagnostik-Codes) → S-01 (was schon existiert)
2. Runtime-Arbeit: S-04 → S-06 → S-07
3. Emulator-Arbeit: S-04 → S-06 → S-07 → S-08
4. KI-Tooling: S-02 → S-09 → S-10 §4–5

**Änderungsregel:** Jede Spec-Änderung = RFC-Dokument + Update des Entscheidungsregisters + Versions-Bump der betroffenen Versionierungs-Oberfläche (S-11 §3).
