# S-08 · Emulator-Profil `sspl-emu`

**Zweck:** Das domänenspezifische Profil, das SSPL zur Emulatorsprache macht — **systemgenerisch** (GameCube ist erste Instanz; Design trägt beliebige Gast-ISAs und Busse). Es bündelt S-04 (Speicher), S-06 (Zeit), S-07 (MMIO/FFI) zu einem kohärenten Framework.

## 1. Architekturprinzipien

1. **Gast ≠ Host, strikt.** Gast-Adressen sind Werte (`GastAdr = u32/u64`), nie Host-Pointer. Zugriff nur über `Adressraum`/Views — ein Gast-Pointer-Bug kann den Host nicht korrumpieren.
2. **Ein Zyklusmodell, viele Einheiten.** Alle Komponenten teilen die Event-Queue (S-06); Timing entsteht durch `plane_nach`-Latenzmodelle, nicht durch Threads.
3. **Interpreter zuerst, Dynarec gleichrangig später.** Der Interpreter ist die *Referenz-Implementierung* — jede Dynarec-Optimierung muss differential gegen ihn beweisen.
4. **Zustand ist serialisierbar.** Gesamter Emulatorzustand (RAM, Register, Queue, Gerätezustände) SSOB-serialisierbar → Savestates, Netplay, Chrono-Debugging.

## 2. Adressraum-Modell

```sspl
var bus = adressraum.neu(gekko_32);
bus.mappe_region(0x8000_0000, 0x0180_0000, ram_view, "MEM1");
bus.mappe_spiegel(0xC000_0000, 0x8000_0000, 0x0180_0000);   // uncached Alias
bus.mappe_mmio(0xCC00_0000, 0x0001_0000, hw_handler);        // MmioHandler-Tabelle
bus.lese_u32(0x8000_0100);  bus.schreibe_u32be(0xCC00_6800, v);
```

- Region-Typen: `ram`, `rom`, `mmio`, `spiegel`, `ungueltig` (Fault → Geräte-Fehler-Event).
- **Endianness pro Region/Gast-ISA** — PPC-Gast → `u32be`-Default-Views.
- Page-Table optional (`bus.mmu_gekko()`): BAT/Seitentabellen-Emulation als austauschbarer Übersetzer (`GastMmu`-Trait).
- Performance-Vertrag: RAM-Zugriff ≤ 3 Host-Instruktionen (Bounds+Endian); MMIO ≤ Handler-Call (S-07 §4).

## 3. CPU-Kern-Framework

```sspl
trait CpuKern {
    funktion schritt() -> u32;                 // führt 1 Instruktion aus, liefert Zykluskosten
    funktion fuehre_bis(ziel_zyklus: u64);     // Batch-Ausführung mit Event-Check
    funktion signal_pruefen();                 // Interrupt-Gate
    funktion zustand_ssob() -> block;          // Snapshot
}
```

- **Interpreter-Kern** (Pflicht zuerst): Dekoder via `comptime`-Tabellen (S-03 §10) — aus Opcode-Feldextraktion generierte Sprungtabellen.
- **Dynarec-Kern** (`DynarecKern` implementiert `CpuKern`): Gast-Basisblöcke → Gast-IR → SERIL → Cranelift → `CodePuffer` (S-05 §7).
  - Block-Linking, `invalidate(bereich)` für SMC, PC-Map (Host↔Gast für Debugger/Savestates).
  - **Sicherheitsgrenze:** generierter Code greift nur über `Adressraum`-Trampoline zu — kein Gast-Zugriff auf Host-Speicher außerhalb der Arenen.
  - Lazy-Eval für Condition-Register/Flags; `zyklus.vorruecken(kosten)` am Blockende (Batch) mit Event-Schwellen-Check.

## 4. Gekko/PPC-Instanz (Neo-Cube)

| Feature | Spezifikation |
|---|---|
| GPR r0–r31 | `u32`-Array, Host-Register-Mapping via mem2reg |
| FPR + **Paired-Singles** | `f32x2`-Typ mit Gekko-Semantik: ps0/ps1, gemischte Präzision, `ps_madd` etc. als Intrinsics — **keine** IEEE-Ausnahmen, definierte Rundung |
| SPRs (LR, CTR, CR, XER, SRR, MSR…) | Felder im Kernzustand; CR-Lazy-Eval im Dynarec |
| MMU | BAT + Segment/Seitentabelle via `GastMmu`-Plugin |
| Exceptions | `signal`-Kanal `GAST_EXC`, Vektor-Sprung am Instruktionsbeginn |
| FP-Modus | `fp_modus.gc` (NICHT strikt IEEE: kein Denormal-Flush-Verhalten wie x86; `FloatSoft`-Referenzpfad für Tests) |
| Dekodierung | `comptime`-Tabelle aus Bitfeld-Extraktionen (PPC-Format A/B/D/X/XL…) |

## 5. Weitere Systemkomponenten (generisch)

- **DMA:** `dma.neu(quell_view, ziel_view, laenge, latenz_modell)` → plant `DMA_FERTIG`-Event; konkurrierende Zugriffe sichtbar über Region-Lock-Trace (Debug).
- **Interrupt-Controller:** `intr.leitung(name, prio)`; Geräte `signal.ausloesen`, CPU `signal_pruefen` am Blockende — Masken/Prioritäten über Queue-Klassen (S-06).
- **FIFO/Kommando-Streams** (CP/GX-artig): `fifo<T>` = `spsc`+DMA-Quelle; GPU-Komponente konsumiert.
- **Taktdomänen:** je Komponente `takt(hz)` → `plane_nach` skaliert Zyklen in Einheits-Zeit (`zyklus` zählt Referenz-Hz, Umrechnung per Frequenz-Faktor — Powermode = Faktor-Override).

## 6. Snapshot & Serialisierung

- `emulator.zustand() -> ssob` / `emulator.lade(ssob)`: RAM-Views, Kern-Register, Queue (Event-IDs + Zeitstempel + registrierte Handler-IDs — Payload-Funktionen müssen **registriert** sein: `ereignis`-Handler haben stabile IDs), Gerätezustände.
- Unverträglichkeit = Versions-Feld + Hash der Komponenten-Tabelle; Mismatch → sauberer Fehler.
- Netplay: Savestate-Sync + deterministischer Scheduler = Lockstep ohne Rollback-Framework (Rollback später über Ring-Snapshots möglich).

## 7. Test- & Verifikationsharness

```sspl
emu_test {
    kern = GekkoInterpreter();
    gold = lade_trace("ppc_isa_tests/ori.trace");
    vergleiche_pro_instruktion(kern, gold);   // Register-Diff je Schritt
}
```

- **Differential-Tests** (Pflicht): Interpreter↔Dynarec↔`FloatSoft` auf PPC-ISA-Testsuite; Zustands-Hash je Instruktion (`zustand_hash()`).
- **Trace-Format:** kanonisches `.trace` (PC, regs, mem-diffs) — auch gegen externe Referenzen (Dolphin-Testdaten) diffbar.
- **Property-Tests:** zufällige Instruktionssequenzen → Interpreter==Dynarec; Speicher-Fuzz auf Adressraum-Regionen.
- CI-Matrix: x86_64 + aarch64 (Paired-Single-Semantik muss auf NEON identisch bleiben → Soft-Referenzpfad).

## 8. Übertragbarkeit auf andere Systeme

`sspl-emu` ist ISA-agnostisch: neue CPU = `CpuKern`-Impl + Dekodiertabelle + `GastMmu`; neuer Bus = `adressraum`-Konfiguration; neue Taktdomäne = `takt(hz)`. GameCube (PPC/Gekko) ist die Referenz-Instanz; dasselbe Framework trägt später z. B. ARM-/MIPS-/68k-Gäste ohne Sprachänderung.

## 9. Akzeptanzkriterien (Neo-Cube-P0)

1. `hello.dol`-Homebrew: Interpreter lädt DOL via `adressraum`, führt Integer-PPC aus, schreibt ins VI-Framebuffer-MMIO — Host-Fenster zeigt Ausgabe.
2. Differential: PPC-ISA-Suite Interpreter==Dynarec (wo implementiert), bit-identisch.
3. Zyklusmodell: VI-Interrupt feuert zur geplanten Frame-Grenze ±0 Zyklen.
4. Savestate: Save→Run-1000-Instr→Load→Run → identischer Zustands-Hash wie durchlaufend.
5. MMIO-Pfad < 20 ns/Aufruf p99 (S-07-Bench).
