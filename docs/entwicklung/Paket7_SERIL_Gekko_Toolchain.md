# Paket 7 — SERIL→Gekko-Toolchain und geschlossener Kreis

Stand: abgeschlossen (SERIL-Subset → Gekko-Maschinencode → Ausführung auf
der SSPL-Gekko-CPU verifiziert).

## Architektur

```
SSPL-Quelle
  → sspl_frontend/Parser.sspl   (AST als Map)
  → seril/Lower.sspl            (SERIL-Modul-Map)
  → seril/GekkoCodegen.sspl     (Liste von PPC/Gekko-Instruktionsworten)
  → Gast-RAM (SystemBus)
  → neo_cube.core.GekkoCPU      (Ausführung)
```

`GekkoCodegen.sspl` liegt im SSPL-Compiler-Repo
(`F:\Build SSPL Produktion\seril\`) — es ist Compiler-Toolchain, nicht
Emulator-Hardware.

## Unterstütztes SERIL-Slice

- `ConstI64` (I64 wird auf u32 trunkiert — Gekko ist 32-bit)
- `LocalGet`/`LocalSet` (Frame-Locals ab `r1+8`, 4-Byte-Slots)
- `Dup`/`Drop`, `INeg`/`INot`/`LNot`
- `IAdd` `ISub` `IMul` `IDiv` `IRem` `IAnd` `IOr` `IXor`
- `IShl` `IShr` `IShrU` (`slw`/`sraw`/`srw`)
- `IEq`..`IGe` (`cmpw` + Materialisierung über `bc`)
- `Jump`, `JumpIfFalse`, `Call` (statisch, ≤8 Parameter in r3..r10), `Ret`
- `IntCast` (i8→extsb, i16→extsh, u8/u16→rlwinm-Maske), `IntToI64`
- `CallHost`: nur die Gast-Shims `shl(a,b)`→`slw`, `shr(a,b)`→`srw`,
  `len(x)`→`rt:len`

## Gast-Laufzeit (Datenobjekt-Slice)

Dynamische Werte laufen vollständig im Gast — die Runtime-Routinen sind
emittierter Gekko-Maschinencode hinter dem Funktionscode, kein Host-Rückgriff.

- **Wert-Repräsentation**: ints roh (u32), Objektzeiger = `addr|1`,
  `nil` = `0xFFFFFFFF`. Zeiger-Erkennung = odd UND `v & 0xFE000000 ==
  0x80000000` (ungerade ints bleiben ints).
- **Heap**: Bump-Allokator, HP-Zeiger @`0x8100FFF0`, Heap @`0x81100000`,
  Scratch @`0x8100FE00`/`r1-16`.
- **Layouts** (Big-Endian-Worte): Str `[1][bytelen][bytes]`; Liste
  `[2][cap][len][elem…]` (Push mit ×2-Grow, Zeiger-Write-back);
  Map `[3][cap][n][key,val…]` (linearer Scan über `rt:streq`, ×2-Grow);
  Obj `[4][klass][felder-map][res]` — die Felder-Map enthält `__klasse`
  (Interpreter-Parität für `CallMethod`).
- **Routinen**: `rt:alloc/streq/strcat/listnew/listget/listset/listpush/
  mapnew/mapget/mapset/objnew/methode/vgl/truthy/idxget/idxset/len`.
- **Methoden-Dispatch**: `rt:mtab` (Datensektion) = `[n][klass,name,fn]*`
  aus `Klasse.meth`-Funktionsnamen; `CallMethod` liest `__klasse` aus
  Objekt-Header bzw. Map, sucht `(klass,name)` per `streq`, ruft via
  `mtctr`/`bctrl`.
- **Truthiness**: `rt:truthy` spiegelt den Interpreter (0, nil, "",
  leere Liste/Map falsch; Objekt wahr).
- **Calling Convention**: Leaf-Routinen clobbern nur r0,r3-r11;
  Non-Leafs nutzen r1-Frames (`stwu r1,-48`, LR in **eigenem** Frame
  +44 — nie im Caller-Frame, sonst Kollision mit dem LR-Slot des
  generierten 16-Byte-Frames).

Alles andere (`F*`, `Ymm*`, `Soft*`, sonstige `CallHost`) erzeugt einen
**expliziten Fehler** — kein Raten.

## Aufrufkonvention

- `r1` = Aufrufstapel (abwärts, Frame = 16-aligned: `[r1+4]`=LR,
  Locals ab `r1+8`)
- `r31` = SERIL-Operandenstapel-Pointer (aufwärts, 4-Byte-Slots)
- `r3`/`r4` = Operanden/Ergebnis, `r5` = Scratch (`IRem`)
- Args in `r3`..`r(2+argc)`; Rückgabe in `r3`
- Wrapper (Wort 0..5): `lis r1,0x8020` / `lis r31,0x8010` / `bl main` /
  `stw r3,0x300(r11)` (r11=0x8000) / `b .`

Ergebnis liegt danach in Gast-Adresse `0x80000300`.

## Verifikation

`gekko_seril_test.sspl` (Neo-Cube-Root, benötigt `SSPL_MODULE_PATH` auf das
Compiler-Repo):

```
SERIL-Referenz main() = 103      # seril.ausfuehren (Host)
Codegen: 319 Instruktionen
Gekko-Ergebnis main() = 103      # GekkoCPU, 3804 Gäste-Schritte
=> 0
```

Probe: `gekko_seril_prog.sspl` — `fib` (Rekursion), `summe` (Schleife mit
Rückwärtssprung), `max` (Vergleich/Branch), `shl`, `%`.

Objekt-Laufzeit-Probe (`gekko_obj_test.sspl` + `gekko_obj_prog.sspl`):
Listen-Iteration, Map-Get/Set mit Grow, String-Concat/-Gleichheit,
`neu Paar` + `initialisiere`/`summe`-Dispatch, Feld-Set auf Objekt,
Truthiness von `""`/`null`, `len` polymorph — Host 87 == Gekko 87,
2977 Schritte, 0 unbekannte.

## CPU-Änderung: `fuehre_aus` Idle-Erkennung

`b .` (0x48000000, Selbst-Sprung) gilt als Bare-Metal-Programmende:
einmal ausführen, dann bricht `fuehre_aus` ab — die interpretierte CPU
verbrät kein Schrittbudget mehr im Idle-Loop.

## Repo-übergreifende Imports: `SSPL_MODULE_PATH`

Neue Auflösungsebene in `src/interpreter.rs`: schlägt die cwd-relative
Suche (`seril/Lower.sspl`) fehl, werden die `;`-separierten Wurzeln aus
`SSPL_MODULE_PATH` durchsucht — und zwar sowohl für `a.b.C`-Dateipfade als
auch für den `neo_cube.*`-Komponenten-Scan. Damit läuft der
Closed-Loop-Test aus dem Neo-Cube-Repo heraus gegen die Compiler-Quellen.

Aufruf:

```bash
cd "C:\Dev\Repos\SonnerStudio\Gamecube Emulator"
SSPL_MODULE_PATH="F:/Build SSPL Produktion" \
  sspl-run gekko_seril_test.sspl
```

## Bekannte Grenzen

- 32-bit-Domäne: `I64` wird trunkiert; `u64`-Arithmetik nicht exakt.
- Kein `F*`-Slice im Backend (die CPU kann FPU/Paired Singles — die
  SERIL-`F*`-Ops sind noch nicht angebunden).
- `CallHost` beschränkt auf `shl`/`shr`/`len`-Shims.
- Die interpretierte SSPL-CPU schafft nur ~10–60 Gäste-Instr./s —
  ausreichend für Korrektheitstests, nicht für reale Software
  (Performance-Roadmap-Schritt).

## Schritt 4 — Hostseitige SERIL-Ausführung von `gekko_cpu.sspl`

Die Gekko-CPU selbst wird jetzt durch den self-hosted Lowerer gesenkt
und im Rust-SERIL-Interpreter ausgeführt — derselbe Quelltext, dieselbe
Semantik, ~18× schneller als der AST-Tree-Walker.

### Neue Lowerer-Fähigkeiten (`seril/Lower.sspl`)

- **Modul-Konstanten**: `konstante NAME = <literal>` werden in einer
  `konstanten`-Map gesammelt und beim Identifier-Lowering inline als
  `ConstI64`/`ConstF64`/`ConstString` emittiert (Ausnahmevektoren,
  MSR-Masken in `gekko_cpu.sspl`).
- **Namespace-Calls**: `speicher.lese_u32(...)`, `ereignis.sende(...)`,
  `sspl.konsole.schreibe(...)` etc. senken zu `CallHost` mit dem
  gepunkteten Namen; Rust löst über die Env-Namespace-Maps auf.
- **Typed-Array-Felder**: `byte[N]`/`u8[N]`-Komponentenfelder emittieren
  die Größe als `ConstI64`-Token — `speicher.aktiviere_hardware_translation`
  allokiert die Gastregion Rust-seitig.
- **Int-Familien-Kompatibilität**: `u8..u64`/`i8..i32`/`I64` sind
  untereinander kompatibel (SERIL rechnet auf I64).
- **F64-Vergleiche**: `Ge`/`Le`/`Lt`/`Gt`/`Eq`/`Ne` auf F64-Operanden
  (Parität in `src/seril/lower.rs` und `src/seril/interp.rs`).
- **`BindMethod`**: `diese.methode` als Wert (Event-Handler) senkt zu
  `BindMethod` — erzeugt `SerilValue::BoundMethod(obj_rc, name)` und
  bewahrt die Objekt-Identität.

### Host-Brücke (`src/interpreter.rs`, `src/seril/*`)

- `SerilValue::BoundMethod` + Opcode `BindMethod` (IR, Interpreter,
  Validator, `seril_op_aus`).
- `seril_zu_value`/`value_zu_seril`: BoundMethod/Obj → Marker-Map
  (`__seril_meth__`, `__seril_obj_ref__`) + Obj-Registry im Interpreter.
  `execute_value_call` erkennt die Marker-Map und re-entert SERIL —
  dadurch funktioniert `ereignis.registriere("CPU_INTERRUPT",
  diese.externer_interrupt)` + `ereignis.sende` vollständig.
- `seril.feld(obj_marker, name)`: Live-Feldzugriff auf geparkte Objekte
  (Marker-Maps sind Snapshots).
- `zeit.jetzt_ms()`: neues Zeit-Native für Laufzeitmessung.
- `value_matches_type`: Int-Familie (`IntW`↔`I64`/`Fixed`) kompatibel.
- `run_func`-Step-Limit 1M → 100M (Batch-Ausführung `fuehre_aus(N)`).

### Gemessenes Ergebnis (`gekko_seril_perf_test.sspl`)

```
Lower OK: 77 Funktionen (gekko_cpu + mmu_bus)
SERIL Programm 1: r5=12 mem=12 schritte=5     # == Tree-Walker
SERIL Programm 3: r3=10 mem=10                # Schleife korrekt
Event-Callback: int_ausstehend=wahr           # BindMethod OK
SERIL: 200035 Schritte in ~6.5–7.3 s (~28–31k Gäste-Instr/s)
TW:      2015 Schritte in ~1.2–1.4 s (~1.4–1.7k Gäste-Instr/s)
Speedup SERIL vs TW: ~18×
```

Geschlossene Schleife unverändert grün: `gekko_seril_test` (103),
`gekko_min_test` (11), `gekko_obj_test` (87), `gekko_exec_test` (0),
`gekko_fpu_test` (48/0), `gekko_sys_test` (35/0), Smoke-Test.

### Aufruf

```bash
cd "C:\Dev\Repos\SonnerStudio\Gamecube Emulator"
SSPL_MODULE_PATH="F:/Build SSPL Produktion" \
  sspl-run gekko_seril_perf_test.sspl
```
