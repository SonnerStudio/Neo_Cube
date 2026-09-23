# S-07 · Komponentenmodell, FFI & ABI v1

**Zweck:** Phase-5-Entscheide. `komponente` wird vom Neo-Cube-Dekorationswort zum echten Laufzeitkonstrukt; `extern`/`vrf`-Shim bekommt einen stabilen ABI-Vertrag — inklusive des MMIO-Schnellpfads, der die Emulator-Performance entscheidet.

## 1. `komponente` — Semantik (Beschluss)

```sspl
komponente SystemBus {
    var haupt_ram: byte[25165824];
    ereignis HwRegisterSchreiben(adresse: u32, wert: u32);
    port ram_ausgang: ausgang view<byte>;

    funktion initialisiere() { ... }
    funktion schreibe_32(adr: u32, wert: u32) { sende HwRegisterSchreiben(adr, wert); }
    on HwRegisterSchreiben(adr: u32, wert: u32) { ... }
}
```

- **Referenztyp** mit implizitem Task-Scope (S-06): `neu K()` erzeugt Instanz + Scope; `stoppe()`/Scope-Ende beendet Kinder geordnet.
- **Lifecycle-Trait** `Lebenszyklus`: `initialisiere`, `starte`, `stoppe` — konventionell, kein Zwang; Komponenten-Registry ruft vorhandene Methoden in definierter Reihenfolge.
- **Isolation:** Felder sind instanzprivat; Zustand fremder Komponenten nur über Methodenaufrufe, Ports oder `ereignis` zugänglich → natürliche Thread-Sicherheit für Emu-Komponenten.
- **Verdrahtung:** `verbinde bus.ram_ausgang -> gpu.ram_eingang` (Port-Kompatibilität statisch geprüft, `E7002`); Ports = typisierte Kanal-Endpunkte (`spsc` Default).
- **Typisierte Events:** `ereignis` deklariert Signatur; `sende NAME(args)` = Broadcast an `on`-Handler derselben + fremder Komponenten (`abonniere(bus.HwRegisterSchreiben, handler)`); FIFO-Reihenfolge, zustellung über `spsc`-Kanäle → determinstisch, replay-fähig.
- **String-Shim:** `ereignis.sende("NAME", ...)` / `ereignis.registriere("NAME", f)` bleibt als `W7001`-deprecatiertes Shim für Migration (dynamisch, `objekt`-Payload).

## 2. `extern` & FFI (Beschluss)

```sspl
extern "C" {
    funktion jit_execute(start: u32) -> u32;
    funktion lese_host_puffer(p: *byte, len: u32) -> view<byte>;   // @sicher optional
}
```

- Aufrufkonvention: **C-ABI** des Ziels; Typen müssen ABI-fähig sein (`@layout("C")`, primitive, `zeiger`, Fat-Slices als `ptr+len`, `funktion`-Pointer).
- **Bidirektional:** SSPL→C via `extern`; C→SSPL via `extern`-exportierte `funktion` (`@no_mangle`) oder `extern`-Callback-Typ `extern "C" funktion(u32) -> u32`.
- `vrf.aufruf("name", ...)` → Shim: dispatcht auf registrierte `extern`-Tabelle (`vrf::registriere("name", fn)`); `W3001` deprecated — Zielbild: direkte `extern`-Deklarationen.
- `nativ`-Effekt (S-03 §7) markiert alle FFI-Aufrufe; `@sicher`-Annotation an `extern` erlaubt Aufruf ohne `unsafe` (Verantwortung beim Deklarierenden).

## 3. ABI v1 — Vertrag (versioniert: `sspl-abi/1`)

| Konstrukt | Repräsentation |
|---|---|
| `u8..u64,i*,f32/f64,bool,byte` | wie C (bool = `_Bool`) |
| `u128/u256`, Endian-Typen | Array von u64-Limbs (limb-order LE im Speicher, BE-Typen serialisieren BE) |
| `text`, `T[]` | Fat-Pointer `{*T, usize len}` (nicht null-terminiert!) |
| `struktur @layout("C")` | C-Layout |
| `option/result` | Tag u8 + Payload (nur mit `@layout("C")` garantiert) |
| `komponente`-Ref | opake `u64`-Handle (Registry) |
| `funktion` | C-fn-ptr (nur `extern`-deklarierte/nicht-capturing) |
| `view<T>` | `{*T, usize len}` + Host-seitig Region-Metadaten |
| Strings/Ereignisse | niemals null-terminiert annehmen; Längen immer explizit |

**Regeln:** Kein ABI über nicht-`@layout("C")`-Strukturen (`E6100`); Panics dürfen FFI-Grenze nicht kreuzen (Abbruch-Wrapper); `extern` in `async`/Tasks nur via `spawn_blocking`.

## 4. MMIO-Schnellpfad (Emu-kritisch)

```sspl
@layout("C") struktur MmioHandler {
    lese:  extern "C" funktion(ctx: zeiger, adr: u32, breite: u8) -> u64;
    schreib: extern "C" funktion(ctx: zeiger, adr: u32, breite: u8, wert: u64);
    ctx: zeiger;
}
```

- Direkter Fn-Pointer-Aufruf, **kein Marshalling**; Ziel < ~20 ns/Aufruf (gemessen in Bench).
- `hardware`/`register`-Zugriffe (S-04 §5) lowern auf `MmioHandler`-Tabellen; Interpreter nutzt dieselbe Tabelle mit Komponenten-Dispatch.
- Hotpath-Garantie: keine Allokation, kein Lock im Handler-Pfad; Verletzungen → `W6110` Performance-Lint.

## 5. Plugin-/Modul-System

- Plugin = `.ssplx`-Paket mit `plugin.manifest`: `name, version, abi: "sspl-abi/1", einstieg: "sspl_plugin_init"`.
- `sspl_plugin_init(ctx) -> i32` Einstieg; Version-Handshake (ABI-Mismatch → sauberer Fehler, nie Crash).
- Dynamisches Laden (libloading existiert): `plugin.lade("neocore_ppc.ssplx")` → Registry-Handles; Entladen nur ohne lebende Handles.
- Hot-Reload (Dev): Datei-Watch → `plugin.neulade()` mit State-Transfer-Hook (`migriere_zustand(alt)->neu`).

## 6. Registry & Orchestrierung (Neo-Cube-Architektur)

- `komponenten.registry`: `registriere(name, instanz)`, `hole<T>(name)`, `alle()` — ersetzt die v9.5-Dateiexistenz-Registry durch echte Instanzverwaltung.
- Boot-Sequenz: `anwendung { komponenten: [...] }`-Manifest → Topologie-Validierung (Ports/Events) schon zur Compilezeit prüfbar.

## 7. Akzeptanzkriterien

- Roundtrip: SSPL-Komponente registriert `MmioHandler`; Rust/C-Rufer feuert 1 Mio. Reads → korrekt + Latenz < 20 ns p99.
- Plugin laden/entladen mit ABI-Mismatch-Fall sauber; Hot-Reload mit State-Transfer.
- Typisiertes `ereignis` liefert FIFO über 3 Komponenten; `waehle`-Integration bewiesen.
- ABI-Konformität: C-Testprogramm (clang) ruft `extern`-Export korrekt auf x86_64 + aarch64.
