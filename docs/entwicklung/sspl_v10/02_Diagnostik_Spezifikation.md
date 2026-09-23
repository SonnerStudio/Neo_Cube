# S-02 · Diagnostik-Spezifikation

**Zweck:** Ein einheitliches, **stabiles und maschinenlesbares** Fehlerformat für Compiler, LSP, `verified apply` und KI-Werkzeuge. Diagnostik ist ein Vertrag — Codes und Schema sind semver-versioniert.

## 1. Grundsätze

1. Jede Diagnose hat einen **stabilen Code** (`Exxxx`), nie nur Prosa.
2. Jede Diagnose trägt **Source-Spans** (Datei, Zeile, Spalte, Länge) — primary + beliebig viele sekundäre.
3. **Mehrere Fehler pro Lauf** (Recovery-Pflicht, mindestens 25 Diagnosen ohne Abbruch).
4. **Deterministische Ausgabe:** gleiche Eingabe → gleiche Diagnosen in gleicher Reihenfolge (sortiert nach Datei, Span, Code).
5. Zwei Ausgabemodi: `menschenlesbar` (Terminal, Snippets, Farbe) und `json` (maschinell, Zeile-pro-Diagnose = JSONL).
6. **Fix-Hints sind strukturierte Edits**, kein Freitext — damit KI-Werkzeuge sie direkt anwenden können.

## 2. Code-Taxonomie

| Bereich | Präfix | Beispiele |
|---|---|---|
| Lexikalisch | `E0xxx` | `E0001` ungültiges Zeichen · `E0002` unterminierter String · `E0003` ungültige Zahl |
| Syntaktisch | `E1xxx` | `E1001` Token erwartet · `E1002` unerwarteter Terminator · `E1010` fehlende Klammer `wenn (...)` · `E1011` fehlendes `;` |
| Namen/Module | `E2xxx` | `E2001` unbekannter Bezeichner · `E2002` unbekanntes Paket · `E2003` Importzyklus · `E2004` Sichtbarkeit verletzt |
| Typsystem | `E3xxx` | `E3001` Typ-Mismatch · `E3002` unbekannter Typ · `E3010` `match` nicht erschöpfend · `E3020` Trait-Bound nicht erfüllt · `E3030` Endian-Konversion implizit |
| Besitz/Lebenszeit | `E4xxx` | `E4001` Nutzung nach `verschiebe` · `E4002` `&mut` aliasiert · `E4010` `view` überlebt Arena · `E4020` `zeiger` außerhalb `unsafe` |
| Effekte/Unsafe | `E5xxx` | `E5001` `nativ`-Effekt in `rein`-Funktion · `E5002` `unsafe`-Op ohne `unsafe`-Block |
| Hardware/Target | `E6xxx` | `E6001` `register` außerhalb `hardware`-Block · `E6002` Ziel unterstützt Feature nicht |
| Komponenten/Events | `E7xxx` | `E7001` unbekanntes `ereignis` · `E7002` Port-Typ inkompatibel · `E7003` `on` ohne Deklaration |
| Intent/KI | `E8xxx` | `E8001` Proof-Obligation unbewiesen · `E8002` Capability-Verletzung im generierten Code |
| Intern | `E9xxx` | `E9001` interner Compilerfehler (ICE — immer mit Backtrace-Artefakt) |

Warnungen: `Wxxxx` parallel (`W1001` ungenutzte Variable, `W2001` `objekt`-Typ verwendet (Migration), `W3001` deprecated-API, `W7001` stringbasiertes `ereignis.sende`).

**Regel:** Codes werden nie wiederverwendet; veraltete Codes erhalten `superseded_by`-Eintrag im Registry-File (`compiler/diagnostik_registry.json`).

## 3. JSON-Schema (JSONL, eine Diagnose pro Zeile)

```json
{
  "schema": "sspl-diag/1",
  "severity": "fehler|warnung|hinweis",
  "code": "E3001",
  "message": "Typ-Mismatch: erwartet `u32`, gefunden `text`",
  "primary": { "datei": "core/mmu_bus.sspl", "zeile": 25, "spalte": 34, "laenge": 4 },
  "sekundaer": [
    { "span": { "datei": "core/mmu_bus.sspl", "zeile": 25, "spalte": 20, "laenge": 10 },
      "label": "Parameter `wert: u32`" }
  ],
  "notizen": ["Endian-Typen konvertieren nicht implizit."],
  "fixes": [
    { "beschreibung": "`als u32` hinzufügen",
      "edits": [ { "datei": "core/mmu_bus.sspl", "zeile": 25, "spalte": 38, "laenge": 0, "einfuegen": " als u32" } ] }
  ],
  "phase": "typpruefung",
  "verursacht_durch": null
}
```

Pflichtfelder: `schema`, `severity`, `code`, `message`, `primary`. `fixes[].edits` sind exakte, anwendbare Text-Edits (Position+Ersetzung). CLI: `sspl check --format=json`.

## 4. Menschenlesbares Format

```
fehler[E3001]: Typ-Mismatch: erwartet `u32`, gefunden `text`
  --> core/mmu_bus.sspl:25:34
   |
25 |         speicher.schreibe_u32(adresse, wert);
   |                    ------------------ ^^^^ gefunden `text`
   |                    |
   |                    Parameter `wert: u32`
   |
   = notiz: Endian-Typen konvertieren nicht implizit.
hilfe: `wert als u32`
```

## 5. Recovery-Verhalten (Parser)

- Fehler produzieren **Fehlerknoten** im AST (`Expr::Fehler`, `Stmt::Fehler`) — der Parse läuft weiter.
- Synchronisationspunkte: `;`, `}`, Statement-Keywords, Item-Keywords.
- Kein Kaskaden-Spam: Folgediagnosen, die kausal auf denselben Fehlerknoten zurückgehen, werden als `verursacht_durch: "E1xxx"` markiert und im Terminal aggregiert.

## 6. Verbraucher-Verträge

| Verbraucher | Nutzung |
|---|---|
| `verified apply` (KI) | Patch gilt als sauber, wenn `severity==fehler`-Diagnosen = 0; `fixes` werden als Code-Actions angeboten, nie automatisch angewendet ohne Gate. |
| LSP | `PublishDiagnostics` 1:1 aus JSON-Schema; `fixes` → `CodeAction`. |
| `sspl migriere` | Migrationshinweise als `W2xxx`-Diagnosen mit `fixes`. |
| Konformitätssuite | Negativfälle referenzieren exakt einen erwarteten `code`. |

## 7. Akzeptanzkriterien

- `sspl check` auf einer Datei mit 10 absichtlichen Fehlern liefert ≥10 Diagnosen, stabil sortiert, JSONL-parsebar.
- Jede Produktion der EBNF hat mindestens einen Negativfall mit zugeordnetem `E1xxx`-Code.
- `fixes` aus `W2xxx` (Migrationswarnungen) sind auf ≥95 % der Fixtures anwendbar ohne Konflikt.
