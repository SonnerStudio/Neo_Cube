# SSPL v10 — Kanonische Grammatik (EBNF)

**Status:** ✅ Grammar-Freeze v0.1 (Phase 0 beschlossen) · **Geltungsbereich:** SSPL v10 Compiler-Frontend
**Beschlüsse:** (1) Neo-Cube-Dialekt (`paket`, `komponente`) ist in die kanonische Grammatik aufgenommen. (2) Alle Detailentscheidungen in §8 sind gefällt — die Grammatik ist damit implementierungsreif. Änderungen ab hier nur noch als versionierter RFC.

Dieses Dokument definiert die **eine** kanonische Syntax von SSPL v10. Alle historischen Dialekte (Großschreib-Schmiede, Klein/Klammer, M4-Sicherheitsdialekt, Neo-Cube) werden über die Alias-Tabelle (§3) bzw. `sspl migriere` darauf abgebildet. Notation: ISO/EBNF — `{ }` Wiederholung, `[ ]` optional, `|` Alternative, `" "` Terminal.

---

## 1. Grundsätze

1. **Eine kanonische Form.** Aliase sind reine Lexer-Abbildungen auf dieselben Token — sie erzeugen **keine** eigenen AST-Knoten und keine semantischen Unterschiede.
2. **Blöcke sind `{}`-delimitiert.** Die v9.5-Keyword-Terminatoren (`ENDEFUNKTION`, `ENDEWENN`, `ENDESTRUKTUR`, `ENDEKLASSE`, `ENDEFUER`, `GIBT x ZURUECK`, `RUECKGABE`-Blockstil) sind **Migrationsformen** — der kanonische Parser akzeptiert sie nicht; `sspl migriere` übersetzt sie.
3. **Deutsch-first, pragmatisch.** Kanonisch ist deutsch, wo die Dialekte deutsch sind (`funktion`, `wenn`, `waehrend`, `komponente`). Etablierte Lehnwörter bleiben kanonisch (`match`, `enum`, `spawn`, `async`, `await`, `unsafe`, `trait`, `impl`, `pub`, `use`, `type`), mit deutschen Aliasen.
4. **Kein syntaktisches Rauschen.** v9.5-Füllwörter (`DANN`, `JEDES`, `MACHE`, `ALS` im Import) entfallen im Kanonischen; der Lexer toleriert sie in der Alias-Schicht.
5. **Case:** Keywords sind klein geschrieben; Großschreib-Formen (`FUNKTION`, `WENN`, `WAHR` …) werden als Aliase akzeptiert.
6. **Erweiterbarkeit:** Reservierte Keywords (§7) dürfen nicht als Bezeichner verwendet werden — die Grammatik ist damit für Phasen 4–6 der Roadmap vorgedacht.

## 2. Lexikalische Struktur

```ebnf
(* Bezeichner: Unicode XID, wie bisher *)
ident        = xid_start , { xid_continue } ;
qualified    = ident , { "." , ident } ;          (* neo_cube.core.SystemBus *)

(* Literale — entsprechen dem bestehenden v10-Lexer *)
int_lit      = digit , { digit | "_" }
             | "0x" , hex , { hex }
             | "0b" , ("0"|"1") , { "0"|"1" } ;
float_lit    = digit , { digit } , "." , digit , { digit } ;
string_lit   = '"' , { char | escape } , '"' ;
sq_string    = "'" , { char | escape } , "'" ;    (* wie bisher akzeptiert *)
template_lit = "`" , { char | "${" , expr , "}" } , "`" ;
bool_lit     = "wahr" | "falsch" ;                (* Aliase: true/false, WAHR/FALSCH *)
null_lit     = "null" ;                            (* Aliase: NULL, nil, nichts *)

comment      = "//" , { any_char }
             | "/*" , { any_char } , "*/" ;       (* Block-Kommentare: beschlossen §8.3 *)
attribute    = "@" , ident , [ "(" , attr_args , ")" ] ;  (* @no_mangle, @layout("C"), @target("arm64") *)
```

## 3. Keyword-Tabelle (kanonisch → Aliase)

| Kanonisch | Aliase (Lexer-Mapping, gleiches Token) |
|---|---|
| `paket` | `modul`, `MODUL`, `package` |
| `importiere` | `import`, `IMPORTIEREN`, `use`, `importiere`+`ALS`-Füllwort |
| `komponente` | `component`, `KOMPONENTE` |
| `struktur` | `struct`, `STRUKTUR`, `klasse`†, `KLASSE`†, `sichere_klasse`† |
| `enum` | `aufzaehlung`, `ENUM` |
| `trait` | `schnittstelle`, `TRAIT` |
| `impl` | `implementiere`, `IMPL` |
| `funktion` | `fn`, `FUNKTION`, `function`, `native_funktion`‡ |
| `var` | `variable`, `VARIABLE`, `mut`‡ |
| `let` | `sei`, `LET` (immutable Bindung) |
| `konstante` | `const`, `KONSTANTE` |
| `wenn` | `falls`, `if`, `WENN`, `DANN`-Füllwort |
| `sonst` | `else`, `SONST` (`sonst wenn` = else-if) |
| `waehrend` | `while`, `solange`, `SOLANGE` |
| `fuer` | `for`, `FUER`, `FUER JEDES` |
| `schleife` | `loop`, `LOOP` |
| `rueckgabe` | `return`, `RUECKGABE`, `ZURUECKGEBEN` |
| `abbruch` | `break`, `ABBRUCH` |
| `weiter` | `continue`, `WEITER` |
| `match` | `matche`, `MATCH` |
| `in` | `IN`, `in` |
| `diese` | `self`, `DIESE`, `this` |
| `neu` | `new`, `NEU` |
| `und` / `oder` / `nicht` | `&&` / `||` / `!`, `UND`/`ODER`/`NICHT` |
| `unsafe` | `unsicher`, `UNSAFE` |
| `versuche` / `fange` | `try` / `recover`, `VERSUCHE`/`FANGE` |
| `spawn` | `starte` |
| `async` / `await` | `asynchron` / `warte` |
| `pub` | `oeffentlich`, `oeffentliche`, `exportiere` |
| `type` | `typ`, `TYPE` |
| `ereignis` | `event`, `EREIGNIS` |
| `extern` | `nativ`, `EXTERN` |
| `absicht` | `ABSICHT`, `natural`, `intent` |
| `static` | `statisch`, `STATIC` |

† `klasse`/`sichere_klasse` sind Migrations-Aliase: sie desugarieren zu `struktur` + `impl`-Block (Semantik §6.4).
‡ `native_funktion` → `extern funktion`; `mut` → `var` (kanonische Mut-Bindung).

**Nicht-Keyword-Builtins** (Stdlib-Namespaces, keine Grammatik): `sspl`, `speicher`, `ereignis`-Objekt, `vrf`, `datei`, `konsole`, `SSGE`. Sie bleiben als Kompatibilitäts-Shims erhalten; kanonische Stdlib-Pfade folgen in Phase 8.

## 4. EBNF — Übersetzungseinheit

```ebnf
compilation_unit = package_decl , { import_decl } , { item } ;

package_decl     = "paket" , qualified , ";" ;
import_decl      = "importiere" , ( qualified | string_lit )
                 , [ "als" , ident ] , ";" ;

item             = component_decl
                 | struct_decl
                 | enum_decl
                 | trait_decl
                 | impl_block
                 | function_decl
                 | extern_block
                 | const_decl
                 | static_decl
                 | type_alias
                 | hardware_decl          (* reserviert, Phase 2/6 *)
                 | intent_decl ;          (* reserviert, Phase 10 *)

vis              = [ "pub" ] ;
attrs            = { attribute } ;
generics         = [ "<" , ident , { "," , ident } , [ "where" , bounds ] , ">" ] ;
```

### 4.1 Komponente (Neo-Cube-Baustein, kanonisch)

```ebnf
component_decl   = attrs , vis , "komponente" , ident , generics
                 , "{" , { component_member } , "}" ;

component_member = field_decl
                 | function_decl
                 | ereignis_decl
                 | on_handler
                 | port_decl ;            (* reserviert, Phase 5 *)

field_decl       = vis , "var" , ident , ":" , type , [ "=" , expr ] , ";" ;

(* Typisierte Ereignis-Deklaration — ersetzt stringbasierte ereignis.sende langfristig *)
ereignis_decl    = vis , "ereignis" , ident , [ "(" , params , ")" ] , ";" ;

(* Handler: on NAME(...) — kanonische Form von ereignis.registriere("NAME", diese.h) *)
on_handler       = "on" , ident , "(" , [ params ] , ")" , block ;

(* Ports: deklarierte Ein-/Ausgänge für Komponenten-Verdrahtung (Phase 5) *)
port_decl        = vis , "port" , ident , ":" , ( "eingang" | "ausgang" )
                 , type , ";" ;
```

`komponente` hat Referenzsemantik (Heap-Objekt mit Lifecycle `initialisiere`/`starte`/`stoppe`), Felder sind komponentenprivat unless `pub`, Methoden erhalten impliziten `diese`-Parameter.

### 4.2 Strukturen, Enums, Traits

```ebnf
struct_decl      = attrs , vis , "struktur" , ident , generics
                 , "{" , { field_decl | "var" ident ":" type "," } , "}" ;

enum_decl        = attrs , vis , "enum" , ident , generics
                 , "{" , variant , { "," , variant } , [ "," ] , "}" ;
variant          = ident , [ "(" , type_list , ")" | "{" , named_fields , "}" | "=" , int_lit ] ;

trait_decl       = attrs , vis , "trait" , ident , generics
                 , "{" , { trait_member } , "}" ;
trait_member     = function_sig , ( ";" | block ) | type_alias | const_decl ;

impl_block       = attrs , "impl" , generics , type , [ "fuer" , type ]
                 , "{" , { function_decl | const_decl | type_alias } , "}" ;
                 (* "impl Trait fuer Typ" — Aliase: "for", "für" *)
```

### 4.3 Funktionen, Extern, Konstanten

```ebnf
function_decl    = attrs , vis , [ "async" ] , "funktion" , ident , generics
                 , "(" , [ params ] , ")" , [ "->" , type ] , block ;
function_sig     = [ "async" ] , "funktion" , ident , generics
                 , "(" , [ params ] , ")" , [ "->" , type ] ;
params           = param , { "," , param } ;
param            = [ "var" ] , ident , ":" , type ;   (* var = mutables Argument *)

extern_block     = attrs , "extern" , [ string_lit ]     (* extern "C" *)
                 , "{" , { function_sig , ";" } , "}" ;
                 (* einzeln: extern funktion name(...) -> typ; *)

const_decl       = vis , "konstante" , ident , ":" , type , "=" , expr , ";" ;
static_decl      = attrs , vis , "static" , ident , ":" , type , "=" , expr , ";" ;
type_alias       = vis , "type" , ident , generics , "=" , type , ";" ;
```

### 4.4 Hardware-/Bare-Metal-Deklarationen (reserviert — Semantik Phase 2/6)

```ebnf
hardware_decl    = "hardware" , ident , "{" , { hw_member } , "}" ;
hw_member        = register_decl | memory_decl | port_io_decl ;

(* Register: fester Gast-/MMIO-Platz, volatile Zugriffe *)
register_decl    = "register" , ident , ":" , type , "@" , expr , ";" ;
memory_decl      = "memory" , ident , ":" , "byte" , "[" , expr , "]"
                 , "@" , expr , ";" ;                    (* Region @ Basisadresse *)
port_io_decl     = "port" , ident , ":" , type , "@" , expr , ";" ;
```

Beispiel Zielsyntax (informiert über die Grammatik, Semantik folgt):
`register SI_POLL: u32 @ 0xCC006400;` · `memory MEM1: byte[0x1800000] @ 0x80000000;`

### 4.5 Intent-/KI-Block (reserviert — Semantik Phase 10)

```ebnf
intent_decl      = attrs , "absicht" , ident , "(" , [ params ] , ")"
                 , [ "->" , type ] , intent_body ;
intent_body      = "{" , { intent_stmt } , "}" ;
intent_stmt      = spec_stmt | function_decl ;
spec_stmt        = ( "erfordert" | "sichert" | "invariant" )
                 , expr , ";" ;   (* pre/post/invariant — Proof-Obligations *)
```

### 4.6 Anweisungen

```ebnf
block            = "{" , { stmt } , "}" ;

stmt             = let_stmt | var_stmt | assign_stmt | expr_stmt
                 | if_stmt | while_stmt | for_stmt | loop_stmt
                 | match_stmt | return_stmt | break_stmt | continue_stmt
                 | spawn_stmt | try_stmt | unsafe_stmt | item ;

let_stmt         = "let" , pattern , [ ":" , type ] , "=" , expr , ";" ;
var_stmt         = "var" , pattern , [ ":" , type ] , [ "=" , expr ] , ";" ;
assign_stmt      = lvalue , assign_op , expr , ";" ;
assign_op        = "=" | "+=" | "-=" | "*=" | "/=" | "%="
                 | "&=" | "|=" | "^=" | "<<=" | ">>=" ;
expr_stmt        = expr , ";" ;

if_stmt          = "wenn" , "(" , expr , ")" , block
                 , { "sonst" , "wenn" , "(" , expr , ")" , block }
                 , [ "sonst" , block ] ;
                 (* Klammern sind verpflichtend — beschlossen §8.1 *)

while_stmt       = "waehrend" , "(" , expr , ")" , block ;
loop_stmt        = "schleife" , block ;
for_stmt         = "fuer" , pattern , "in" , expr , block ;

match_stmt       = "match" , expr , "{" , { match_arm } , "}" ;
match_arm        = pattern , [ "wenn" , expr ] , "=>" , ( expr , "," | block ) ;
pattern          = literal | ident | "_" | qualified
                 | qualified , "(" , [ pattern , { "," , pattern } ] , ")"
                 | qualified , "{" , ident , { "," , ident } , "}"
                 | pattern , "|" , pattern
                 | "[" , [ pattern , { "," , pattern } ] , "]"
                 | "(" , pattern , { "," , pattern } , ")" ;

return_stmt      = "rueckgabe" , [ expr ] , ";" ;
break_stmt       = "abbruch" , [ expr ] , ";" ;
continue_stmt    = "weiter" , ";" ;

spawn_stmt       = "spawn" , expr , ";" ;
try_stmt         = "versuche" , block , "fange" , "(" , ident , ")" , block ;
unsafe_stmt      = "unsafe" , block ;
```

### 4.7 Ausdrücke

```ebnf
expr             = pipe_expr ;

pipe_expr        = compose_expr , { ( "|>" | "<|" ) , compose_expr } ;
compose_expr     = range_expr , { ( ">>" | "<<" ) , range_expr } ;
range_expr       = logic_or , [ ".." , logic_or ] ;
logic_or         = logic_and , { ( "||" | "oder" ) , logic_and } ;
logic_and        = equality , { ( "&&" | "und" ) , equality } ;
equality         = comparison , { ( "==" | "!=" ) , comparison } ;
comparison       = bitor , { ( "<" | "<=" | ">" | ">=" ) , bitor } ;
bitor            = bitxor , { "|" , bitxor } ;
bitxor           = bitand , { "^" , bitand } ;
bitand           = shift , { "&" , shift } ;
shift            = additive , { ( "<<" | ">>" ) , additive } ;   (* §8: Konflikt mit Komposition *)
additive         = multiplicative , { ( "+" | "-" ) , multiplicative } ;
multiplicative   = unary , { ( "*" | "/" | "%" ) , unary } ;

unary            = ( "-" | "!" | "nicht" | "~" | "&" | "*" | "await" | "spawn" )
                 , unary
                 | postfix ;

postfix          = primary , { postfix_op } ;
postfix_op       = "(" , [ args ] , ")"              (* Aufruf *)
                 | "." , ident                        (* Feldzugriff: diese.bus *)
                 | "." , ident , "(" , [ args ] , ")" (* Methodenaufruf *)
                 | "[" , expr , "]"                   (* Index: payload[0] *)
                 | "?"                                (* Fehler-Propagation *)
                 | "!"                                (* Makro/Intrinsic-Aufruf *)
                 | "als" , type ;                     (* Cast — Alias: "as" *)
args             = expr , { "," , expr } ;

primary          = literal | qualified | "diese"
                 | "(" , expr , ")"
                 | new_expr | array_expr | map_expr
                 | closure | if_expr | match_expr
                 | block | async_block ;

new_expr         = "neu" , type , ( "(" , [ args ] , ")"        (* neu Typ() *)
                                  | "[" , expr , "]" ) ;        (* neu byte[8] *)
array_expr       = "[" , [ expr , { "," , expr } ] , "]" ;
map_expr         = "{" , [ ident , ":" , expr , { "," , ident , ":" , expr } ] , "}" ;
closure          = "|" , [ params_bare ] , "|" , ( expr | block ) ;
params_bare      = ident , [ ":" , type ] , { "," , ident , [ ":" , type ] } ;

if_expr          = "wenn" , "(" , expr , ")" , block , "sonst" , block ;
match_expr       = "match" , expr , "{" , { match_arm } , "}" ;
async_block      = "async" , block ;
```

**Präzedenz** (hoch → niedrig): postfix → unary → `* / %` → `+ -` → `<< >>`(Shift) → `&` → `^` → `|` → Vergleich → Gleichheit → `&&` → `||` → `..` → `>>`/`<<`(Komposition) → `|>`/`<|`. Linksassoziativ. **Beschlossen (§8.2):** Kompositions-`>>`/`<<` teilen sich die Token mit Shift; die Disambiguierung erfolgt über Operandentypen (Funktionstyp vs. Integer). Der Formatter normalisiert Leerzeichen; bei statischer Mehrdeutigkeit meldet der Parser einen Fehler mit Hinweis.

### 4.8 Typen

```ebnf
type             = primitive | qualified | generic_type
                 | array_type | slice_type | tuple_type
                 | fn_type | ptr_type | ref_type | view_type ;

primitive        = "u8"|"u16"|"u32"|"u64"|"u128"|"u256"
                 | "i8"|"i16"|"i32"|"i64"|"i128"|"i256"
                 | "int" | "float"                      (* int=i64, float=f64 *)
                 | "f32"|"f64" | "f32x2" | "f32x4"      (* Vektoren, Phase 6 *)
                 | endian_int                            (* u16be, u32be, u64be, u16le … *)
                 | "bool" | "byte" | "text" | "zeiger"
                 | "objekt"                             (* dynamisch, Migrations-Typ *)
                 | "nichts" ;                           (* void/unit — Alias: () *)

endian_int       = ("u"|"i") , ("16"|"32"|"64") , ("be"|"le") ;

generic_type     = qualified , "<" , type , { "," , type } , ">" ;
array_type       = type , "[" , int_lit , "]" ;           (* byte[25165824] *)
slice_type       = type , "[" , "]" ;                     (* byte[] *)
tuple_type       = "(" , type , { "," , type } , ")" ;
fn_type          = "funktion" , "(" , [ type_list ] , ")" , [ "->" , type ] ;
ptr_type         = "zeiger" , "<" , type , ">"            (* typisiert *)
                 | "*" , [ "mut" ] , type ;               (* raw, nur unsafe *)
ref_type         = "&" , [ "mut" ] , type ;
view_type        = "view" , "<" , type , ">" ;            (* reserviert, Phase 2 *)
type_list        = type , { "," , type } ;
```

**Mapping des Neo-Cube-Typvokabulars:** `text`→String (UTF-8), `zeiger`→untypisierter Host-Zeiger (`zeiger<T>` bevorzugt), `byte`→`u8`, `objekt`→dynamischer Typ (`dyn`/`Value`-Box, nur Migration), `byte[N]`/`byte[]`→Array/Slice, `u32`/`int`/`float`/`bool`→kanonische Breiten.

## 5. Attribute (kanonisch `@name`, Argumente optional)

| Attribut | Bedeutung | Phase |
|---|---|---|
| `@freestanding`, `@no_std` | Bare-Metal-Übersetzungseinheit | 2/6 |
| `@no_mangle`, `@link_section("name")` | Symbol-/Layout-Kontrolle | 3/5 |
| `@entry`, `@interrupt`, `@naked`, `@panic_handler` | Einsprung-/ISR-Konventionen | 6 |
| `@target("aarch64" \| "x86_64" \| …)` | bedingte Kompilierung | 3 |
| `@layout("C" \| "packed" \| "big_endian")` | Struktur-Layout-Garantien | 2 |
| `@volatile` | volatile Feld-/Speicherzugriffe | 2 |
| `@inline`, `@cold` | Optimierungs-Hints | 3 |
| `@doc("...")`, `@deprecated("...")` | Metadaten/Docs | 1/8 |
| `@test`, `@eigenschaft` | Test-/Property-Test-Marker | 0/11 |
| `@verifiziert`, `@sandbox` | KI-Provenance/Capability | 10 |

## 6. Semantische Anker (Stichpunkte für den Type-Checker)

1. `komponente` = Referenztyp; `neu K()` erzeugt Instanz; `diese` nur in Komponenten-/Impl-Kontext gebunden. Lifecycle-Methoden `initialisiere`, `starte`, `stoppe` sind konventionell (trait `Lebenszyklus`, Phase 5).
2. `struktur` = Werttyp (Copy/Move nach Feldtyp); `klasse`-Alias erzeugt `struktur` + automatischen `impl`-Block — Migrationskrücke, nicht für Neuentwicklung.
3. `ereignis NAME(args);` in einer Komponente deklariert ein typisiertes Event; `sende`/`on` werden dagegen getypt (stringbasierte `ereignis.sende("X")`-Aufrufe bleiben über die Stdlib-Shim lauffähig, markiert `@deprecated`).
4. `extern`-Blöcke/-Funktionen sind `unsafe` am Aufrufpunkt, sofern nicht `@sicher` annotiert.
5. `view<T>`, `ptr`, `memory`-/`register`-Deklarationen sind nur in `unsafe`-Kontexten oder `@freestanding`-Einheiten voll nutzbar.
6. Endian-Typen (`u32be`) sind eigene Typen mit definierten Konversionen (`als`, `.zu_endian()`) — keine impliziten Umwandlungen.

## 7. Reservierte Keywords (für spätere Phasen — jetzt schon als Bezeichner verboten)

`zyklus`, `kanal`, `arena`, `view`, `eingang`, `ausgang`, `erfordert`, `sichert`, `invariant`, `lese`/`schreibe` (Port-Ops), `ssml`, `material`, `fenster`, `schwarm`, `qubit`, `vernichte`, `signal`, `maske`, `verschiebe`.

## 8. Beschlossene Detailentscheidungen (Freeze v0.1)

1. **Klammern bei `wenn`/`waehrend` — verpflichtend.** `wenn (c) { }` ist die einzige kanonische Form (Neo-Cube-Stil). Begründung: eindeutig, KI-freundlich, formatter-stabil. Klammerlose Formen sind Migrationsfälle für `sspl migriere`.
2. **`>>`/`<<`-Doppelbelegung — beibehalten.** Shift und Funktionskomposition teilen sich die Token; Disambiguierung über Operandentypen. Statisch unauflösbare Fälle erzeugen einen Parser-Fehler mit Hinweis (`Verwende explizite Typannotation oder (f >> g)`).
3. **Block-Kommentare `/* */` — aufgenommen** (siehe §2).
4. **`fange`-Bindung — getypt optional:** `fange(e: FehlerTyp)` filtert auf den Typ; `fange(e)` fängt dynamisch (`objekt`/Fehler-Union).
5. **Semikolon-Pflicht — ja.** `;` terminiert Deklarations- und Ausdrucks-Statements; Blöcke schließen ohne `;`. Begründung: klare Diffs, robuste Fehler-Recovery, einfache KI-Patches.
6. **`fuer x in sammlung` — iteriert über Werte.** Maps liefern Schlüssel nur explizit (`fuer k in m.schluessel()`, `fuer (k,v) in m.eintraege()`); keine implizite Key-Iteration.

## 9. Konformitätssuite (Anforderung an Phase 0)

Pro EBNF-Produktion: ≥2 Positivfälle, ≥1 Negativfall mit erwartetem Fehlercode. Dialekt-Fixtures: alle bestehenden Neo-Cube-`*.sspl`-Dateien müssen nach `sspl migriere` (oder direkt, wo sie schon kanonisch sind) fehlerfrei parsen. Golden-ASTs für: `neo_cube.sspl`, `mmu_bus.sspl`, `serial_interface.sspl`, `flipper_gpu.sspl`. Parser-Fuzzing: ≥10 Mio. Token-Mutationen ohne Panic.

---

### Anhang A — Minimalbeispiel kanonisch

```sspl
paket neo_cube.core;

importiere sspl.kern;
importiere sspl.speicher;
importiere sspl.ereignis;

komponente SystemBus {

    var haupt_ram: byte[25165824];      // 24 MB (1T-SRAM)
    var a_ram: byte[16777216];          // 16 MB ARAM

    ereignis HwRegisterSchreiben(adresse: u32, wert: u32);

    funktion initialisiere() {
        speicher.aktiviere_hardware_translation(diese.haupt_ram, 0x80000000);
        speicher.aktiviere_hardware_translation(diese.a_ram, 0x81800000);
    }

    funktion lese_32(adresse: u32) -> u32 {
        rueckgabe speicher.lese_u32(adresse);
    }

    funktion schreibe_32(adresse: u32, wert: u32) {
        wenn (adresse >= 0xCC000000 und adresse <= 0xCC00FFFF) {
            sende HwRegisterSchreiben(adresse, wert);
        } sonst {
            speicher.schreibe_u32(adresse, wert);
        }
    }
}
```

### Anhang B — Migrations-Mapping (Auswahl)

| Alt (beliebiger Dialekt) | Kanonisch v10 |
|---|---|
| `MODUL x;` / `@modul x` / `paket x;` | `paket x;` |
| `IMPORTIEREN "p" ALS a;` | `importiere p als a;` |
| `FUNKTION f() GIBT T ZURUECK ... ENDEFUNKTION` | `funktion f() -> T { ... }` |
| `WENN c DANN ... SONST WENN ... ENDEWENN` | `wenn (c) { } sonst wenn (…) { }` |
| `FUER JEDES x IN s MACHE ... ENDEFUER` | `fuer x in s { }` |
| `STRUKTUR S ... ENDESTRUKTUR` | `struktur S { }` |
| `KLASSE K ... ENDEKLASSE` | `struktur K { }` + `impl K { }` |
| `variabel v = x` / `VARIABLE v: T = x` | `var v: T = x;` |
| `versuche { } fange(e) { }` (bereits kanonisch) | unverändert |
| `native_funktion f(...)` | `extern funktion f(...);` |
| `vrf.aufruf("name", a, b)` | `vrf::aufruf("name", a, b)` (Shim) → später `extern`-Aufruf |
| `ereignis.sende("NAME", a)` | `sende NAME(a)` (getypt) oder Shim |
| `ereignis.registriere("NAME", diese.h)` | `on NAME(...) { }` in Komponente |
| `drucken(x)` / `sspl.konsole.schreibe(x)` | `konsole.schreibe(x)` (Stdlib) |
