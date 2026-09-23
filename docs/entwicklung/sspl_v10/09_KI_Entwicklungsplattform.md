# S-09 · KI-First-Entwicklungsplattform

**Zweck:** SSPL wird die Sprache, in der KI-generierter Code **nachweisbar am zuverlässigsten** ist — nicht durch Marketing, sondern durch Architektur: maschinenlesbare Grammatik, strukturierte Diagnostik, verifizierende Pipeline, Provenance.

## 1. Warum SSPL für KI geeignet sein kann (Design-Assets)

| Eigenschaft | KI-Nutzen |
|---|---|
| Kanonische Grammatik + stabiler Formatter | eindeutige Diffs, kein Stilrauschen |
| `E*`-Diagnostik mit strukturierten `fixes` | Modelle bekommen anwendbare Reparaturen, keine Prosa |
| Starkes Typsystem + Effekte | semantische Fehler werden vor Ausführung gefangen |
| `comptime`/Interpreter-Sandbox | sichere Probeausführung ohne Host-Risiko |
| Deterministische Builds/AST-Hashes | reproduzierbare Evaluation |
| `absicht`-Blöcke + Proof-Obligations | Spezifikation und Code bleiben gekoppelt |
| Symbolgraph/LSP-Index | präziser Repo-Kontext statt Textfenster |

## 2. Maschinenlesbare Sprachartefakte (Compiler-Ausgaben)

1. **`grammatik.json`** — die EBNF (Freeze v0.1) als maschinenlesbares Regelwerk (Terminals, Produktionen, Präzedenz, Aliase). Konsum: Prompt-Kontext, Parser-Generierung, Konformitätstests.
2. **`ast.schema.json` + `ast.canonical`** — stabile AST-Serialisierung (kanonischer Hash `ast_hash` für Caching, Patch-Verankerung, Dedup).
3. **`diagnostik_registry.json`** — alle `E*/W*`-Codes mit Bedeutung + typischen `fixes` (S-02).
4. **`symbolgraph`** (LSP-Export): Definitionen, Referenzen, Typsignaturen — Repo-Kontext für Agenten.
5. **`api.manifest`** pro Paket: öffentliche Signaturen + Docs — kompakter Importkontext.

## 3. `verified apply` — der KI-Patch-Gate (Beschluss)

```
KI-Vorschlag ─► Formatter ─► Parse ─► Namen/Typ/Effekt-Check ─► Sandbox-Eval (opt.)
            ─► Tests/Property ─► Diff-Report ─► MENSCHLICHE FREIGABE ─► Write
```

- **Compiler-Autorität ist letzte Instanz.** Kein generierter Text gilt als Code, bevor er alle Gates besteht; ein Modell kann Gates nicht überstimmen.
- **Sandbox-Eval:** Interpreter-Run mit Capability-Profil (S-11): `fs:lesen_eingeschraenkt`, `net:kein`, `mem:N MiB`, `zeit:N ms`, `nativ:verboten` — generierter Code ist standardmäßig `nativ`-frei.
- **Diff-Report:** AST-Diff (nicht Text), Semantik-Notizen (Effekt-Änderungen, neue `unsafe`), Risiko-Score.
- **Freigabe-Modi:** `immer` (Default), `batch` (Sammelreview), `auto` nur innerhalb `sandbox`-Modulen mit `@ki_freigegeben`-Attribut — nie für `unsafe`/`extern`/`nativ`-Effekt-Code.

## 4. `absicht`-Blöcke — Intent → verifizierter Code

```sspl
absicht lade_dol(pfad: text, bus: SystemBus) -> u32 {
    erfordert pfad != "";
    sichert ergebnis >= 0x80000000 und ergebnis < 0x81800000;
    invariant bus.haupt_ram.integritaet();

    funktion lade_dol(pfad: text, bus: SystemBus) -> u32 { ... }  // KI-generiert
}
```

- `erfordert`/`sichert`/`invariant` = Proof-Obligations → kompilieren zu: (a) statischer Check wo möglich, (b) generierte Property-Tests (`@eigenschaft`), (c) Laufzeit-Assertions in Debug.
- Unbewiesene Obligation → `E8001` (kein stiller Erfolg!). Der Intent-Text (Kommentar/`natural`-Beschreibung) geht als Prompt-Kontext an das NLP-Backend; der generierte Body durchläuft `verified apply`.
- Damit wird die v9.5-`ABSICHT`-Idee ehrlich: die Absicht ist **prüfbarer Vertrag**, nicht Dekoration.

## 5. Provenance & Audit

- **Patch-Manifest** (SSOB, append-only): `patch_hash, ast_hash_vorher/nachher, modell{id,version}, prompt_hash, werkzeug, freigabe{nutzer,zeit}, gates[parse,typ,test,review]`.
- `@verifiziert`-Attribut am Code-Artefakt verweist auf Manifest-Eintrag; Build kann `require_verified` erzwingen (Policy in `sspl.toml`).
- Audit-Log für `auto`-Freigaben mit erhöhtem Detailgrad (vollständiger Prompt, Diff, Testausgaben-Hash).

## 6. KI-Evaluationskultur (Messlatte statt 210-NLP-Fragen)

- **`sspl-bench-ki`:** reale Aufgaben-Suite (Bugfix, Feature, Refactoring, Emu-Domäne: „implementiere `lwarx`-Instruktion", „baue SI-Handler") — Metrik = **% verifiziert übernommene Patches** + Regression-Rate, nicht Prompt-Trefferquoten.
- Baseline-Läufe pro Release; Modell-Wechsel dokumentiert im Manifest.
- NLP-210-Suite bleibt als Sprachverständnis-Smoke-Test, ist aber **nicht** das Qualitätskriterium.

## 7. Prompt-Injection & Sicherheit

- Generierter Code ist **Daten bis zum Gate**: nie direkt `eval`'t auf dem Host (außer Sandbox), nie mit Schreibrechten.
- `unsafe`/`extern`/`nativ`/`comptime`-IO in generiertem Code → hartes Review-Gate + `E8002` bei Capability-Verletzung.
- Keine Modell-gesteuerten Shell-Aufrufe; `sys_exec` in generiertem Code = verboten (Policy).
- Repository-Kontext läuft über `symbolgraph`, nicht über ungefiltertes Datei-Dumping (Injection-Oberfläche kleiner).

## 8. Werkzeug-Integration

- `sspl ki vorschlag|pruefe|anwenden|audit` CLI; MCP/stdio-Protokoll für Agenten (diagnostik-json, symbolgraph, patch-endpoints).
- NLP-Provider-Abstraktion (existiert: llama.cpp/Ollama) bleibt austauschbar; KI-Features sind **opt-in-Feature-Flags**, nie Build-Pflicht.

## 9. Akzeptanzkriterien

- `verified apply` weist 100 % der Injektions-Fixtures zurück (unsafe-Ops, sys_exec, nativ-Effekte ohne Gate).
- `absicht` mit unbewiesener Obligation → `E8001`; bewiesene Obligation generiert bestehende Property-Tests.
- `sspl-bench-ki` liefert reproduzierbare Metrik auf ≥50 Aufgaben; Manifest-Audit-Trail vollständig.
- `grammatik.json` deckt 100 % der EBNF-Produktionen ab und parst die Konformitätssuite identisch zum Referenz-Parser.
