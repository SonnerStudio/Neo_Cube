# S-03 · Typsystem & Semantik

**Zweck:** Vollständige Entscheidungsgrundlage für Type-Checker und Inferenz (Phase 1). Ziel: stark, statisch, vorhersagbar — einfache Programme bleiben einfach, Systemprogrammierung bleibt ausdrucksstark.

## 1. Typklassen

| Klasse | Typen | Semantik |
|---|---|---|
| Primitiv (Copy) | `u8..u256`, `i*`, `f32/f64`, `f32x2/f32x4`, Endian-`u*be/le`, `bool`, `byte` | Wertsemantik, `Copy` |
| Aggregat (Wert) | `struktur`, `enum`, Tupel, Arrays `T[N]` | Wertsemantik, Move-default; `Copy` nur via `impl Kopier` |
| Referenztypen | `komponente`, `kanal<T>`, `future<T>` | Heap-Objekt, Besitz regelt Lebenszeit (§3) |
| Text/Sequenzen | `text` (UTF-8, immutable), `T[]` Slice, `liste<T>`, `map<K,V>` | `text`/`T[]` Copy-Slices; `liste`/`map` Wertcontainer mit Copy-on-Write bei Aliasing |
| Funktionen | `funktion(A..)->R`, Closures `\|x\| e` | Umgebungs-Capture per Move oder `&`-Borrow |
| Dynamisch | `objekt` | getaggte Box (Migrations-/Interop-Typ; `W2001`-Warnung im Neucode) |
| Spezial | `zeiger<T>`/`ptr` (unsafe), `view<T>`, `arena<T>`, `option<T>`, `result<T,E>`, `nichts` | siehe S-04 |

## 2. Bindungen & Mutabilität

```sspl
let x = 5;              // immutable, Typ inferiert
var y: u32 = 0x20;      // mutable
konstante MAX: u32 = 486000000;
```

- `let` = immutable Bindung (Wert selbst kann `var`-Felder enthalten — Mutabilität ist Bindungs-, nicht Typsache; Feldzugriff `x.feld = 1` auf `let x` ist `E4012`).
- Parameter sind immutable außer `var`-Markierung: `funktion f(var zaehler: u32)`.
- `&T`/`&mut T` Referenzen nur aus `var`-Bindungen bzw. Feldern; `&mut` exklusiv (§3).

## 3. Besitzmodell (Beschluss: „Ownership light + explizites Teilen")

Entscheidung — kein voller Borrow-Checker, aber kein GC:

1. **Move-Semantik** für nicht-`Copy`-Typen: Bindung/Argument/Rückgabe überträgt Besitz; Nutzung danach → `E4001`. `verschiebe x` macht es explizit (reserviertes Keyword).
2. **Borrows** `&T` (viele) / `&mut T` (exklusiv) nur in **lokalem Scope**: Referenzen dürfen nicht in Feldern/Closures/Rückgaben gespeichert werden (kein Lifetime-System nötig — `E4015` „Referenz entkommt"). Das deckt ~90 % der Fälle bei einfacher Mentalmodel-Kosten.
3. **Geteiltes** nur explizit: `arc<T>` (atomar gezählt) und `gemeinsam` (task-lokal `rc<T>`) als Stdlib-Typen. Komponenten-Referenzen zwischen Komponenten nur via `arc` oder Ports.
4. **Arenen** `arena<T>` für Hotpaths/Emulator: `arena.neu(T) -> &T`, `arena.zuruecksetzen()`; Views `view<T>` sind gebundene, bounds-geprüfte Fenster — Details S-04.
5. **`nichts`**: `()`-Unit; `option<T>`/`result<T,E>` statt `null` für neue APIs; `null` existiert nur für `objekt`/Migration.

## 4. Numerik (Beschluss)

- **Keine impliziten Konversionen** — auch nicht `i32 → i64` oder `int → float`. Alles via `als` (`x als u32`) oder `.zu_*()`. Ausnahme: ungetypte Integer-Literale inferieren zum erwarteten Typ (`var x: u8 = 7` ok).
- Overflow: Debug = Panic/`fange`-bar, Release = Wrap (`+%%`-explizite Wrap-Ops für Hotpaths: `+%%`, `-%%`, `*%%`).
- `f32x2` = Gekko-Paired-Single-Semantik (in `sspl-emu` dokumentiert, S-08): gemischte interne Präzision, keine IEEE-Ausnahmen.
- Endian-Typen sind **eigene Typen**: `u32be` ≠ `u32`; Zuweisung `u32be → u32` erfordert `als`/`von_be()`; `E3030` bei implizitem Versuch. Vergleiche `==`/`<` zwischen gleichen Endian-Typen erlaubt.

## 5. Generics & Traits

- **Monomorphisierung** (wie Rust): `liste<T>` instanziiert pro `T`; Code-Bloat-Kontrolle via `@generisch_dyn` (v1.1, optional). Keine Type-Erasure im Default — nötig für `u32be`-Layout und Inlining.
- `trait` = Interface mit Default-Implementierungen; Bound-Syntax: `funktion f<T: Lesbar + Debug>(x: T)`. `where`-Klausel: `funktion f<T>(x: T) where T: Serialisierbar`.
- **Kohärenz (Orphan-Rule):** `impl T fuer U` nur, wenn `T` oder `U` im eigenen Paket definiert ist.
- Operatoren über Traits: `Add`, `Sub`, `Eq`, `Ord`, `Index`, `BitOp` — kein freies Operator-Overloading außerhalb dieser Traits (`E3050`).
- Kein Überladen von Funktionen nach Signatur (kein overloading; eindeutige Namen) — AI- und Parser-Freundlichkeit.

## 6. Kontrollfluss-Semantik

- `match` muss **erschöpfend** sein (`E3010`); `_` als Rest-Arm; Guards `wenn`.
- `wenn (c)` erfordert `bool` — keine Truthiness (`E3060`); `wenn let Some(x) = opt` als Pattern-Form erlaubt.
- `fuer x in e`: `e: IntoIter`; kanonisch Werte (§8.6 Freeze). `fuer (k,v) in m.eintraege()`.
- `versuche { } fange(e[: Typ]) { }`: Panic-ähnliche Laufzeitfehler + `result<T,E>` für erwartete Fehler; `?` propagiert `result`.
- Schleifen sind Ausdrücke mit `abbruch wert;` (Rust-Stil).

## 7. Effekt-System (Beschluss: explizite, geprüfte Effekte)

Funktionen tragen Effekt-Marker; Aufrufer müssen die Effekte „haben":

| Effekt | Bedeutung | Default |
|---|---|---|
| `rein` | keine IO/allocation-side-effects sichtbar | — |
| `io` | Datei/Netz/Konsole | von `io`-Kontexten vererbbar |
| `nativ` | ruft `extern`/FFI | `unsafe` nötig am Aufrufpunkt (außer `@sicher`) |
| `unsicher` | dereferenziert `zeiger`/`view` roh | `unsafe`-Block Pflicht |
| `zyklisch` | greift auf virtuelle Zeit/Event-Queue zu | — |

`rein`-Funktionen mit `io`/`nativ` → `E5001`. Effekte propagieren statisch; `absicht`-Blöcke dürfen nur deklarierte Effekte fordern (KI-Sicherheit, S-09).

## 8. Closures & Funktionswerte

- Syntax `|x| expr` / `|x: T| -> R { }`; Capture per Move (default) oder `&`/`&mut` wenn nur gelesen/geschrieben im selben Scope.
- `spawn`-Closures müssen `Send` erfüllen: nur `Copy`-, `arc`-, `kanal`- und move-captures; `gemeinsam`/Rc-Verbot → `E5020`.
- Methodenreferenzen `diese.on_vblank` erzeugen gebundene Funktionswerte (Komponenten-Ref + Funktion) — benötigt für `ereignis`-Registrierung.

## 9. `objekt` & Interop

`objekt` = dynamischer `Value`-Summentyp (wie im heutigen Interpreter). Regeln: Feldzugriff dynamisch (Laufzeitfehler möglich), `als T` checked-cast, `W2001` in Neucode. Existenzgrund: Migration, `ereignis`-Daten-Payloads, Host-Brücken. **Nicht** für Emulator-Hotpaths.

## 10. Kompilierzeit (`konstante`, `comptime`)

- `konstante` = compile-time evaluierbarer Ausdruck (reine Faltung).
- `comptime`-Blöcke (Phase 1.5): `comptime { }` führt eingeschränkten Interpreter aus — erlaubte Ops: Arithmetik, Stringbau, Tabellen. Kein IO. Einsatz: Lookup-Tabellen, Dekodier-Tabellen für CPU-Emulatoren (Gekko-Opcode-Tabelle aus Feldextraktion generieren!).
- Hygienische Makros: **kein** freies Makrosystem in v10.0 (Beschluss); `comptime`-Funktionen + generische Konstanten decken die Emulator-Fälle ab.

## 11. Fehlerklassen (Auszug)

`E3001` Mismatch · `E3010` nicht-erschöpfend · `E3020` Bound · `E3030` implizite Endian-Konversion · `E3050` unzulässiges Operator-Overload · `E3060` Nicht-`bool`-Bedingung · `E4001` use-after-move · `E4015` entkommene Referenz · `E5001` Effekt-Verletzung · `E5020` nicht-`Send`-Capture

## 12. Akzeptanzkriterien (Phase-1-Abnahme)

- Typsuite: ≥200 Fälle (Inferenz, Generics, Traits, Match-Exhaustiveness, Effekte, Move/Borrow-Regeln) — alle `E*`-Codes oben je ≥1 Negativfall.
- Identische Semantik Interpreter↔Backend auf der Semantik-Suite (Determinismus, Overflow-Verhalten, Float).
- `comptime`-Gekko-Dekodiertabelle kompiliert und liefert identische Einträge zur Referenz-CSV.
