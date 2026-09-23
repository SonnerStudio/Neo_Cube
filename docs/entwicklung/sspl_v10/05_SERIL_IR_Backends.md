# S-05 · SERIL-IR & Backend-Architektur

**Zweck:** Der echte Compiler-Kern. Ersetzt die v9.5-Marker-Strings durch eine getypte SSA-Zwischendarstellung mit validiertem Text-/Binärformat, Optimierungs-Pässen und drei Ausgabezielen (Cranelift-JIT, LLVM-AOT, Wasm).

## 1. Architektur-Übersicht

```
.sspl ──► Lexer ─► Parser ─► AST ─► Namensauflösung ─► TAST (typannotiert)
   ─► Lowering ─► SERIL-SSA ─► [Opt-Pässe] ─► ┬─ Cranelift ─► Maschinencode (JIT/AOT)
                                             ├─ LLVM (Release-Opt) ─► Objektdatei
                                             └─ Wasm ─► .wasm (AOT/Runtime)
   Objektcode ─► System-Linker ─► Executable / .ssplx-Container
```

**Entscheidungen:**
- **Cranelift ist das Primär-Backend** (Rust-nativ, schnelle Kompilierzeiten, x86_64+aarch64, JIT-tauglich — identischer IR-Konsum für Dynarec-Pfad in S-08).
- **LLVM optional** als Release-Optimierer hinter Feature-Flag (Abhängigkeitsgewicht akzeptabel).
- **Interpreter bleibt** als Tier-0 (REPL, Debugging, `comptime`, Sandbox).
- **Wasm bleibt** Target für Web/Plugin-Sandbox.

## 2. SERIL-SSA — Modell

```
modul := { typ_dekl, global_dekl, funktion }
funktion := @name (params) -> typ [effekte] { block+ }
block := label: { phi*, instr*, terminator }
instr := v = op [typ] args | call | load/store [endian] | allok | cast | intrinsic
terminator := sprung | br_wenn | rueckkehr | unerreichbar
```

- **SSA strikt:** jede `v`-Definition einmal; `phi` nur an Block-Eingängen; Dominanz validiert.
- **Typen spiegeln S-03:** `u8..u256`, `i*`, `f32/f64`, `v2f32/v4f32`, `u32be` (als `u32` + Endian-Attribut an load/store), `zeiger`, Aggregate, Fat-Pointer-Paare (Slice = `{ptr,len}`).
- **Endianness ist Zugriffsattribut:** `lade u32be [p]` / `speichere u32be [p], v` — kein Endian-Typ im Register, sauber für Host-Codegen.
- **Intrinsics:** `speicher.*`, `zyklus.*`, `dynarec.*`, `simd.*`, `comptime.*`, `atomar.*` — deklarierte Schnittstelle, backend-geprüft.
- **Metadaten:** Source-Spans je Instruktion (Debugger/Profiler!), Effekt-Tags, `@layout`-Layout-Tabellen.

## 3. Formate

- **Textform** `.seril`: kanonisch, stabil, diff-freundlich — Pflichtformat für Tests & KI-Inspektion.
- **Binärform** in `.ssplx` v2 (Container: Header, Symbole, SERIL-Sektion, native Code-Sektionen je Target, SSOB-Ressourcen, Signatur).
- **Validator** (existiert als `seril/validator.rs` — erweitern): SSA-Dominanz, Typen, Endian-Attribute, Terminator-Regeln, Intrinsic-Arity. Build schlägt bei Invalidität fehl — kein „interpretiere trotzdem".

## 4. Optimierungs-Pipeline (v1-Umfang)

| Pass | Zweck | Emulator-Nutzen |
|---|---|---|
| `mem2reg` | SSA-Lifting von Allokas | Register-Mapping Gekko→Host |
| `gvn` + `dce` | Redundanz-/Totcode | Dekodierte Konstanten |
| `inline` (Kostenmodell) | MMIO-/Helper-Inlining | `lese_u32be` → 2–3 Instr. |
| `licm` | Loop-Hoisting | Fetch-Loops |
| `endian_fold` | `be→host`-Faltung, Byte-Swap-Verschmelzung | PPC-Datenzugriffe |
| `simd_lower` | `v2f32/v4f32` → AVX2/NEON | Paired-Singles |
| `branch_fold` | Sprungketten | Block-Linking-Vorbereitung |

Pass-Reihenfolge fest (deterministisch); `--opt=0..3`; `O2` = kanonischer Release-Modus.

## 5. Tiering-Strategie

| Tier | Technik | Zweck |
|---|---|---|
| 0 | Tree-Walk-Interpreter | REPL, `comptime`, Sandbox-Eval, Debugging |
| 1 | Cranelift-Baseline-JIT | schnelle Startzeit, dev-run |
| 2 | Cranelift+Opt / LLVM-AOT | Release |
| (Emu) | Dynarec: Gast-IR→SERIL→Cranelift | PPC→Host (S-08) |

## 6. Ausgabe & Verlinkung

- Objektemission: COFF/ELF/Mach-O via `object`-Crate; `sspl baue --release` → System-Linker (MSVC `link`, `ld`, `ld64`/`lld`); kein eigener Linker.
- `sspl baue --ziel wasm32` → `.wasm`; `--ziel aarch64-apple-darwin` Cross-Compile via `zig cc`/osxcross als Toolchain-Plugin (S-10).
- Reproduzierbarkeit: `--deterministisch` (Timestamps=0, sortierte Sektionen, Build-ID=Inhaltshash) — Pflicht für Release-Manifeste.

## 7. JIT-Infrastruktur (geteilt mit Dynarec)

- `CodePuffer`: W^X-konforme Seiten (`mmap` RW→RX-Flip / `VirtualAlloc`+`VirtualProtect`+`FlushInstructionCache`), Block-Verwaltung, Relocation-Tabelle.
- `BlockCache`: PC→Host-Adr-Map, Patchbare Jump-Slots (Block-Linking), Invalidierung (`invalidate(bereich)` für SMC).
- Tier-1 nutzt dieselbe Infrastruktur — kein zweiter JIT-Stack.

## 8. Akzeptanzkriterien

- `fib(35)` nativ ≥ 50× Tree-Walk; Compile-Zeit Baseline ≤ 50 ms/1k Fn.
- SERIL-Validator lehnt 100 % der konstruierten Invaliditäts-Fixtures ab.
- Semantik-Parität Interpreter↔Tier-1↔AOT auf der Sprach-Suite (bit-identisch inkl. Float).
- `.seril`-Textform roundtrip-stabil (parse→emit→parse = gleicher Text).
