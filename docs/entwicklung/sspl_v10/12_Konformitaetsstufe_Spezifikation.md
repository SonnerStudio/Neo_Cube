# S-12 · Konformitätsstufen-Spezifikation (`sspl-konform`)

**Zweck:** Definiert das abgestufte Zertifizierungsmodell, mit dem SSPL-Implementierungen (Interpreter, Cranelift-Tier, LLVM-AOT, Wasm, zukünftige Fremdimplementierungen) **messbar** auf Konformität zur kanonischen Grammatik und Semantik geprüft werden. Diese Spezifikation konkretisiert Roadmap-Phase 0, Punkt 3 (Konformitätssuite) und ist das operative Rückgrat des Honesty-Gates (S-11 §4).

## 1. Begriffe

- **Konformitätsstufe (KS-x):** ein abgeschlossenes Zertifizierungslevel; jede Stufe baut auf der vorherigen auf und ist kumulativ.
- **Fixture:** eine Testeinheit im Suite-Verzeichnis (Quelltext + Erwartungs-Artefakt).
- **Konforme Implementierung:** ein Build/Backend, das eine Stufe vollständig (100 % Pflicht-Fixtures, alle Negativfälle mit korrektem `E*`-Code) besteht.
- **Profil:** optionaler Zusatz-Scope (`emu`, `spiel`, `freestanding`, `ki`), der eigene Stufenanforderungen addiert.

## 2. Die Stufen

| Stufe | Name | Was bewiesen wird | Mindestumfang |
|---|---|---|---|
| **KS-0** | Lexikalisch | Token-Strom exakt: alle kanonischen + Alias-Keywords, Literale, Operatoren, Kommentare, Spans | 300 Positiv-Token-Fixtures, 60 Negativfälle (`E0xxx`) |
| **KS-1** | Syntaktisch | Jede EBNF-Produktion parst; AST identisch zum Golden-AST (kanonische Serialisierung + `ast_hash`) | ≥2 Positiv- + ≥1 Negativ-Fixture je Produktion (`E1xxx`); Neo-Cube-Golden-Files |
| **KS-2** | Semantisch-statisch | Namensauflösung, Typprüfung, Effekte, Match-Exhaustiveness, Move/Borrow-Regeln — akzeptiert Valides, lehnt Invalides mit korrektem Code ab | ≥200 Fälle; jeder `E2xxx`–`E5xxx`-Code ≥1 Negativfall |
| **KS-3** | Verhaltensgleichheit | Ausführbare Semantik: Interpreter-Referenz; Ausgaben, Exit-Codes, Laufzeitfehler identisch zur Erwartung | ≥150 Programme; Determinismus über 5 Wiederholungen |
| **KS-4** | Backend-Parität | Tier-1 (Cranelift) und AOT liefern **bit-identische** Ergebnisse zum Interpreter — inkl. Float, Overflow, Endian | 100 % der KS-3-Fixtures × jedes Backend; ≥80 % davon Pflicht-Parität |
| **KS-5** | Determinismus & Zeit | `zyklus`-Queue, Event-Ordnung, Replay-Bitidentität, `waehle`-Tiebreaks unter Seed | 40 Scheduler-Fixtures; Replay-Roundtrips |
| **KS-6** | Speicher & Unsafe | Arenen/Views/Endian-Zugriffe, Bounds-/Align-Verhalten, Generationen-Fehler, `hardware`-/`register`-Bindung | 60 Fixtures + UB-Suite (je Regel S-04 §7 ein Negativfall) |
| **KS-7** | Komponenten & ABI | `komponente`-Lifecycle, typisierte `ereignis`, Ports, `extern`-Roundtrips, Plugin-ABI-Handshake | 50 Fixtures + C-Gegenstelle (clang) auf Tier-1-Zielen |

### Profil-Stufen (additiv)

| Profil | Zusatz-Stufe | Inhalt |
|---|---|---|
| `emu` | **KS-E** | `adressraum`-Mapping (Regionen/Spiegel/MMIO), Taktdomänen, Snapshot-Roundtrip, Differential-Lauf (Interpreter↔Dynarec), MMIO-Latenz-Budget < 20 ns p99 |
| `spiel` | **KS-G** | SSGE-Kontext, `fenster`/`eingabe`/`rahmen`-Loop, identische Szenen-Ausgabe Vulkan↔Metal (Golden-Image-Toleranz) |
| `ki` | **KS-K** | `grammatik.json`-Abgleich, `verified-apply`-Gates, Injektions-Fixtures (100 % zurückgewiesen), Provenance-Manifeste |
| `freestanding` | **KS-F** | `@freestanding`-Unit baut ohne std, `@entry`/`@interrupt` erzeugen korrekte Symbole |

## 3. Fixture-Format (verbindlich)

```
tests/konform/
  ks1_syntaktisch/
    positiv/
      komponente_minimal.sspl          # Quelltext
      komponente_minimal.ast           # kanonischer Golden-AST (+ ast_hash)
    negativ/
      wenn_ohne_klammer.sspl           # Quelltext
      wenn_ohne_klammer.err            # erwartete Diagnose: code=E1010, span, severity
  ks3_verhalten/
    fib_rekursiv.sspl
    fib_rekursiv.expect                # stdout + exit_code + stderr
  ks5_determinismus/
    zwei_komponenten_eventflut.sspl
    .expect.hash                       # Endzustands-Hash (SSOB)
  ksE_emu/
    ppc_ori_add.trace                  # ISA-Golden-Trace
```

- `.ast` = kanonische AST-Serialisierung (S-09 §2.2); `.err` = S-02-JSONL-Zeile; `.expect` = Ausgabe + `exit_code`; `.expect.hash` = deterministischer Zustands-Hash.
- Jede Fixture trägt Front-Matter: `// konform: stufe=KS-1, code=E1010, spec=EBNF§4.6` — maschinell auswertbar, verlinkt Spec-Abschnitt.
- **Negativfälle müssen den exakten Fehlercode referenzieren** — „schlägt fehl" ohne Code gilt nicht.

## 4. Coverage-Anforderungen

| Regel | Anforderung |
|---|---|
| Grammatik | 100 % der EBNF-Produktionen durch Positiv-Fixtures belegt; Alias-Tabelle vollständig gespiegelt (jedes Alias ≥1×) |
| Semantik | jeder `E*`-Code der Registry ≥1 Negativfall; jede `fixes`-Art ≥1 anwendbarer Fall |
| Migration | alle Neo-Cube-`*.sspl`-Bestandsdateien als Fixtures; `sspl migriere`-Roundtrip-Parität |
| Fuzzing | Lexer/Parser/SERIL-Validator: ≥10 Mio. Mutationen/Lauf ohne Panic, Crash = Blocker-Bug |
| Regression | jeder gefundene Bug erzeugt vor dem Fix eine scheiternde Fixture („test-first" Pflicht) |

## 5. Zertifizierung & Gates

- `sspl konform --stufe KS-4` führt die Suite gegen eine Implementierung aus und schreibt `konform_bericht.json` (pro Stufe: bestanden/quote/abdeckung + Artefakt-Hashes).
- **Kanal-Regeln:** `nightly` ≥ KS-3 · `beta` ≥ KS-4 · `stabil` = KS-0..KS-7 vollständig + Profil-Stufen der ausgelieferten Features.
- **Honesty-Kopplung:** ein Feature darf in der Truth-Table (S-01) erst `impl` heißen, wenn die zugehörigen Fixtures in der erreichten Stufe grün sind. Claims ohne Fixture werden im Release-Gate zurückgewiesen.
- Drittimplementierungen (z. B. alternativer Interpreter) zertifizieren sich über denselben `konform`-Lauf — das macht die Suite zur echten Sprachdefinition neben der EBNF.

## 6. CI-Einbindung

- PR-Gate: KS-0..KS-3 (schnell, < 5 Min) · Nightly: KS-4..KS-7 + Fuzzing + Sanitizer · Release: volle Matrix auf allen Tier-1-Targets.
- Fixture-Änderungen erfordern Spec-Verweis; AST-/Semantik-Änderungen ohne Spec-RFC werden abgelehnt (freeze-Schutz).
- Statistik-Artefakt `konform_status.md` wird pro Lauf generiert: Stufen-Fortschritt, Abdeckung, offene Codes — ersetzt die v9.5-Manifest-Zeremonie durch echte Messung.

## 7. Akzeptanzkriterien für die Suite selbst

- Die Suite läuft selbst-hostend via `sspl konform` (kein separates Testframework im Repo-Kern).
- Falsch-Positive-Rate = 0: jede grüne Fixture wurde gegen Referenz-Verhalten verifiziert, nicht gegen sich selbst.
- Neue EBNF-Produktion ohne Fixture → `sspl konform --abdeckung` meldet Lücke als Fehler.
- Neo-Cube-Bestandsdateien: 100 % parsebar nach Migration, Golden-ASTs eingefroren.

## 8. Beziehung zu anderen Specs

| Nutzt | Für |
|---|---|
| EBNF (Freeze v0.1) | KS-0/KS-1-Produktionsabdeckung, Golden-ASTs |
| S-02 | `E*`-Codes in Negativ-Fixtures, `.err`-Format |
| S-03/S-04 | KS-2/KS-6-Semantikfälle |
| S-05 | KS-4-Backend-Parität, `.seril`-Golden-IR |
| S-06 | KS-5-Determinismus/Replay |
| S-07 | KS-7-ABI-Roundtrips |
| S-08 | KS-E-Emu-Profil |
| S-09 | KS-K + `grammatik.json`-Konsument der Suite |
| S-11 | Zertifizierungs-Gates, Honesty-Kopplung |
