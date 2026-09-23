# S-11 · Migration v9.5→v10 & Release-Policy

**Zweck:** Der Übergang — Migrationswerkzeug, Kompatibilitätsstrategie, Versionierung und das „Honesty-Gate", das die v9.5-Krankheit (Behauptung ≠ Beweis) strukturell verhindert.

## 1. `sspl migriere` — Migrationswerkzeug

- Eingabe: beliebiger Dialekt (Großschreib-Schmiede, Klein/Klammer, M4, Neo-Cube) — Erkennung per Dialekt-Heuristik (Keyword-Dichte, Terminatoren).
- Ausgabe: kanonisches v10 + **Migrationsbericht** (JSONL im S-02-Diagnostikformat, `W2xxx`).
- Regeln: Anhang B der Grammatik-Spezifikation; nicht überführbare Stellen → `W2999` mit `MIGRATE:`-Kommentar statt stiller Verfälschung.
- Idempotent: `migriere(migriere(x)) == migriere(x)`; Formatter läuft anschließend automatisch.
- Semantik-Notfälle (hardcoded `structDefaults`-Verhalten, `rufe_nativ_auf` ohne echten Handler): werden als `W2900` „manuelle Prüfung" markiert — nie implizit „weitermachen".

## 2. Kompatibilitäts-Shim `sspl-kompat`

| Alt | Shim | Ziel |
|---|---|---|
| `vrf.aufruf("x", …)` | `vrf::aufruf` dispatcht auf `extern`-Tabelle | `extern`-Deklarationen |
| `ereignis.sende("X",…)` / `registriere` | String-Bus-Shim | typisiertes `ereignis`/`on`/`sende` |
| `drucken`/`sspl.konsole.schreibe` | `konsole.schreibe` | Stdlib `konsole` |
| `speicher.lese/schreibe_*` | gleiche Namen, echte Impl (S-04) | `view`-basierte Zugriffe |
| `objekt`-Payloads | dynamische Box | typisierte Ereignis-Daten |

Shim-Paket ist opt-in (`features=["kompat"]`), jede Nutzung erzeugt `W3001`-Deprecations mit `fixes`-Migration.

## 3. Versionierung & Stabilitätsversprechen

| Oberfläche | Versioniert als | Stabilität |
|---|---|---|
| Grammatik | `grammatik vX.Y` (Freeze-Tag) | Ergänzungen rückwärtskompatibel; Brechungen nur Major |
| Sprachsemantik | `edition="v10"` in `sspl.toml` | Edition-Mechanik (wie Rust) für künftige Brechungen |
| ABI | `sspl-abi/1` | Major-Version bei Layout-/Konventionsänderung |
| `.ssplx`-Format | Container-Version im Header | Reader abwärtskompatibel 1 Major |
| Diagnostik-Schema | `sspl-diag/1` | additive Felder ok |
| Stdlib-API | semver je Paket | deprecated ≥1 Minor vor Entfernung |

## 4. Release-Kanäle & Gates

- Kanäle: `nightly` (main), `beta` (monatlich), `stabil` (quartalsweise). Feature-Flags für Experimentelles — nie in `stabil` aktiviert.
- **Honesty-Gate (Pflicht für jedes Release):** jede Feature-Behauptung im Release-Manifest verlinkt ein Test-/Bench-Artefakt; `full_production_ok` darf nur true sein, wenn *alle* P0/P1-Akzeptanzkriterien der Spezifikationen S-01..S-11 grün sind. `development_checks_ok` und `full_production_ok` bleiben getrennte Felder (v10-Praxis wird formalisiert).
- **CI-Matrix:** 3 Tier-1-Targets × (check, test, conformance, emu_test, bench) + Fuzzing-Nightly (Parser/Lexer/SERIL) + Sanitizer-Lauf (UBSan/ASan auf Unsafe-Kernen).
- **Performance-Gates:** Bench-Regression > 5 % blockiert Merge (jit_compile, spsc, view-Zugriff, fib-Baseline).
- **Konformitäts-Gate:** 100 % Suite, davon ≥80 % auch unter Tier-1-Backend (nicht nur Interpreter).

## 5. Sicherheits- & Supply-Chain-Policy

- Signierte `.ssplx` (Ed25519), Lockfile-Pflicht, `minimumReleaseAge=7d` (Registry-Enforcement), SBOM.
- Keine Build-Schritte, die Netz ohne Opt-in ziehen; reproduzierbarer Build = Default bei `--release`.
- Vulnerability-Prozess: `SECURITY.md`, koordinierte Disclosure, Patch-Releases auf letzte 2 Minor-Linien.

## 6. Dokumentations-Pflicht je Feature

- Spec-Referenz (S-xx-Abschnitt), Akzeptanztest-Verweis, Migrationsnotiz (falls v9.5-Parallele), Beispiel. Fehlt eines → `docs_ok=false` im Manifest → kein `stabil`.

## 7. Neo-Cube-Migrationspfad (konkret)

1. `sspl migriere` über `*.sspl`-Bestand → kanonische Dateien (`komponente`, `ereignis` typisiert, `falls`→`wenn`, `waehrend` bleibt, `return`→`rueckgabe`).
2. Shim-Lauf (`kompat`) lässt Orchestrierung früh ausführbar bleiben, während Hotpaths native Module bleiben (S-07-Plugin: `neocore_ppc`, `neocore_gpu` …).
3. Schrittweise Absorption: mit Phase 6/7 wandern `SystemBus`, `FlipperGPU` usw. von Shim zu echten `sspl-emu`-Komponenten — getrieben durch `emu_test`-Abdeckung, nicht durch Enthusiasmus.

## 8. Definition of Done (v10.0 „Produktion")

Erst wenn erfüllt, darf `full_production_ok=true`:

- [ ] S-02..S-11 Akzeptanzkriterien vollständig grün auf allen Tier-1-Targets
- [ ] Neo-Cube-P0 (S-08 §9): `hello.dol` nativ, Differential-Suite, Savestate, MMIO-Bench
- [ ] Konformitätssuite 100 %, Fuzzing-Nightly sauber 30 Tage
- [ ] `sspl-bench-ki` Baseline publiziert, `verified apply` auditiert
- [ ] `.ssplx`-Signierung + Registry-Policy live, Docs-Pflicht erfüllt
- [ ] Ein externes Review der ABI- und Unsafe-Spezifikationen (S-04, S-07)
