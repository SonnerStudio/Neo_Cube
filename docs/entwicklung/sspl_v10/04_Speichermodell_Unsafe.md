# S-04 · Speichermodell & Unsafe-Vertrag

**Zweck:** Das P0-Fundament — ersetzt die heutigen Placebo-Primitiven (`alloc`→Fake, `ptr_read`→0, `ptr_write`→No-Op) durch ein echtes, geprüftes Speichermodell. Voraussetzung für SERIL-Backend, Emulator-Adressräume und FFI.

## 1. Speicherarchitektur (drei Ebenen)

| Ebene | Mechanik | Einsatz |
|---|---|---|
| **Besitz-Speicher** | Stack/`arena`-allokierte Werte, Move/Borrow-Regeln (S-03 §3) | Normale Programme |
| **Arenen** | `arena<T>`: Puffer-basiert, `neu`, `zuruecksetzen()`, Generationen-Check | Hotpaths, Emulator-RAM, Frame-Daten |
| **Roher Speicher** | `*T`, `*mut T`, `zeiger`, `speicher.lese/schreibe_*`, `view<T>` | `unsafe`, MMIO, FFI, Gast-Adressräume |

**Beschluss: kein Garbage Collector.** Lebenszeit = Besitz + Arenen + `arc`/`gemeinsam` für geteilte Referenzen. Determinismus und Latenz (Emulator!) schließen einen tracenden GC aus; ein optionales RC-basiertes `gemeinsam<T>` deckt bequeme Fälle ab.

## 2. Arena-API (Stdlib `speicher.arena`)

```sspl
var ram = arena<byte>.mit_kapazitaet(0x1800000);   // 24 MiB
var p: view<byte> = ram.fenster(0x8000_0100, 16); // bounds-geprüft
p.schreibe_u32be(0, 0xDEADBEEF als u32be);         // Endian-Schreibzugriff
ram.zuruecksetzen();                                // O(1)-Freigabe
```

- Arena-Handles: `zeiger`/Views dürfen die Arena nicht überleben → `E4010` (Generationen-Tag in Debug, statisch wo möglich).
- Backends: `arena` allokiert via `mmap`/`VirtualAlloc` (Page-aligned, optional `huge_pages`), Wachstum nur via `mit_wachstum()`-Opt-in.

## 3. `view<T>` — der Emulator-Zugriffstyp

`view<T>` = (Basis-Pointer, Länge, Endian-Tag) als Fat-Pointer, **nicht besitzend**, bounds-geprüft in Debug, ungeprüft in Release-Hotpaths via `view.ungeprueft()` im `unsafe`-Kontext.

```sspl
view<u32be> ram32 = bus.mem1_view();       // Big-Endian-Fenster auf Gast-RAM
let x: u32 = ram32[idx].von_be();          // Lesezugriff: idx*4 bound-check
```

- `view<T>` darf nur aus `arena`, `memory`-Deklaration oder FFI-Puffer entstehen.
- Aliasing: `view` auf identische Region erlaubt (MMIO-Spiegelungen!); Schreib-Konflikte sind Programmierverantwortung innerhalb `unsafe`-Deklarationen, nicht Sprachfehler.

## 4. `zeiger`/`ptr` und der `unsafe`-Vertrag

```sspl
unsafe {
    var p: *mut u8 = speicher.alloziere(4096);     // real mmap'd
    p.schreibe(0xAB);
    speicher.freigeben(p);
}
```

- `zeiger` (Neo-Cube-Kompat) = `*mut byte`; typisierte Form `zeiger<T>` bzw. `*T`/`*mut T`.
- Außerhalb `unsafe`: `zeiger`-Werte sind opake Handles (vergleichbar, ausgebbar) — Dereferenzierung nur im `unsafe`-Block → `E4020`.
- Intrinsics (Stdlib `speicher`): `alloziere`, `freigeben`, `lese_*`/`schreibe_*` (`u8..u64`, `f32/f64`, `*_be/*_le`, `bytes`), `kopiere`, `setze`, `volatil_lese/schreibe`.
- **`aktiviere_hardware_translation(view, gast_basis)`** — Neo-Cube-Semantik real machen: registriert die Arena/View als Gast-Region im `Adressraum`-Dienst (S-08); kein echtes Host-Page-Fault-Forwarding in v10.0 (späterer RFC; würde `mmap`-Probing erfordern — als Erweiterung eingeplant).

## 5. Hardware-Deklarationen (Semantik der Grammatik §4.4)

```sspl
hardware Flipper {
    register CP_FIFO: u32 @ 0xCC008000;
    memory MEM1: byte[0x1800000] @ 0x80000000;
    port UART0: u8 @ 0x10;
}
```

- `register` erzeugt **volatiles** Feld, das beim `hardware`-Instanziieren gegen einen `Adressraum`-Eintrag/FFI-Callback gebunden wird. Lese-/Schreibzugriff ruft den registrierten Handler (MMIO) — im Interpreter Callback, im Backend direkter Fn-Pointer (ABI S-07).
- `memory` deklariert eine Region mit Basisadresse; im `sspl-emu`-Profil mappt sie auf den Gast-Adressraum, außerhalb auf eine Host-Arena (gleiche API → Emu-Code testbar als natives Programm).
- `@volatile` auf Feldern = jeder Zugriff ist ein `volatil_lese/schreibe` (kein Caching, kein Reordering).

## 6. Layout & Endianness

- `@layout("C")` auf `struktur`: Feldreihenfolge = Deklaration, ABI-Alignment, keine Umordnung — Pflicht für alle FFI- und MMIO-Strukturen.
- `@layout("packed")`: Alignment 1.
- `@layout("big_endian")` auf `struktur`/`enum`: (De)Serialisierung und `view`-Zugriffe lesen/schreiben BE — Gast-Strukturen (PPC-Registerblöcke, Disc-Header) deklarativ.
- Alignment-Verletzungen: `E4030`; `view`-Zugriff mit `idx` prüft `alignof(T)` in Debug.

## 7. Garantien & Verbote

- Kein Data-Race in `Send`-Kontexten: `&mut`/Arena-Handles nicht `Send` außer `arc<mutex<arena>>`-Pattern.
- Null-Pointer: `zeiger`/`ptr` dürfen `null` sein; Deref ohne Check → Laufzeit-Panic in Debug.
- Use-after-`zuruecksetzen()` einer Arena: Generation-Tag → Panic in Debug; in Release undefiniert (dokumentiert wie `unsafe` in Rust).
- Keine automatische `memcpy`-Optimierung über `volatile`-Zugriffe hinweg.

## 8. Migration (v9.5-Primitiven → v10)

| v9.5/Neo-Cube | v10 kanonisch |
|---|---|
| `speicher.lese_u32(adr)` | `speicher.lese_u32(view_oder_adr)` — gleiche Signatur, echte Implementierung; Typ `u32`↔`u32be` per Zielregion |
| `speicher.schreibe_u32(adr, w)` | identisch, echt |
| `speicher.schreibe_bytes(adr, payload, n)` | `view.schreibe_bytes(adr, payload)` / Shim identisch |
| `zeiger` | `zeiger` (=` *mut byte`) |
| `byte[N]` Feld | Arena/`view<byte>` im Komponenten-`var` |

## 9. Akzeptanzkriterien

- Arena+View+Endian-Suite: byte-identische Ergebnisse x86_64↔aarch64; Bounds-Fehler deterministisch.
- `unsafe`-Missbrauchs-Suite: jede Regel oben je 1 `E4xxx`-Negativfall.
- Leistung: `view.schreibe_u32be` ≤ 3 Host-Instruktionen nach Inlining (Rev-Benchmark); `speicher.lese_u32` auf `view` ≥ 1 GB/s sequentiell.
