# S-06 · Laufzeit, Nebenläufigkeit & deterministisches Zeitmodell

**Zweck:** Phase-4-Entscheide. Der Emulator braucht cycle-genaue Synchronisation zwischen CPU, GPU, DSP und Interrupts — das Zeitmodell ist **Sprachprimitive**, nicht Bibliothekszufall.

## 1. Task-Modell (Beschluss)

- **Strukturierte Tasks:** `spawn` nur innerhalb `bereich { }`-Scopes (oder Komponenten-Lifecycle); Scope-Ende wartet auf alle Kinder — keine verwaisten Threads.
- Ausführung: Work-Stealing-Pool (eigener Executor, keine Tokio-Abhängigkeit im Kern); `spawn_blocking` für IO.
- `Send`-Regeln aus S-03 §8 erzwingen race-freie Captures.
- Komponenten = Tasks mit Lifecycle (S-07): `komponente`-Instanz besitzt impliziten Scope; `initialisiere`/`starte`/`stoppe` sind die Scope-Gates.

## 2. Kanäle (Beschluss)

```sspl
var ch = kanal<GXKommando>.spsc(1024);       // Lock-free Ring, Single-Prod/Cons
var ch2 = kanal<Ereignis>.mpsc(0);           // 0 = unbegrenzt
ch.sende(k);  var v = ch.empfange();         // blockierend
waehle {                                   // select
    fall v <- ch1 { ... }
    fall t nach 5.ms { ... }               // Timeout-Arm
}
```

- `spsc`/`mpsc` lock-free Ringbuffer (Power-of-2, cache-line-aligned) — Emulator-Hotpaths (CP-FIFO, Audio-DMA) nutzen `spsc` mit Zero-Copy-`view`-Payloads.
- `waehle` = select über Empfängern + `nach`-Timeoutarm; keine Send-Arme in v10.0.
- Kanal-Endpunkte sind `Send`, nicht `Copy`; Schließen via `schliesse()`/`empfange` → `option<T>`.

## 3. Atomics & Ordering

- `atomar<T>` (u8..u64, bool, zeiger): `lade(ord)`, `speichere(ord)`, `tausche`, `vergleiche_tausche`, `fetch_add/sub/und/oder`.
- Orderings: `entspannt`, `erwerb`, `freigabe`, `seqcst` — explizit, kein Default.
- Keine impliziten Locks; `mutex<T>`/`rwlock<T>` als Stdlib für Nicht-Hotpaths.

## 4. Virtuelles Zeitmodell — die Kernprimitive

```sspl
zyklus.plane_bei(cpu_zyklus + 975, || { vi.interrupt(); });  // Event
zyklus.plane_nach(1215, dvd.dma_abschluss);
var jetzt = zyklus.jetzt();                                  // u64 Cycles
zyklus.vorruecken(60);                                       // Host-Tick → Gast-Zeit
```

**Modell:**
- **Ein logischer Takt pro Emulator-Instanz** (`zyklus`-Namespace = systemweiter Zähler `u64`, konfigurierbare Frequenz z. B. 486 MHz Gekko).
- **Deadline-sortierte Event-Queue** (Pairing-Heap): `plane_bei(abs_zyklus, fn)`, `plane_nach(delta, fn)`, `entferne(id)`, `verschiebe(id, neu)`.
- Ausführungsregel: Events an Schwelle `jetzt` in **stabiler FIFO-Reihenfolge** (Insertion-Index als Tiebreak) → deterministisch.
- **Interrupt-Modell darauf abgebildet:** `signal.ausloesen(name)` plant Handler-Event mit Prioritätsklasse (`sofort` = nächste Instruktionsgrenze, `normal` = Event-Zeit); `maske(name, an/aus)` gated Zustellung — kein separates Interrupt-Subsystem.
- **Multi-Core-Gast:** je Gast-CPU eigener `zyklus`-Kontext oder ein geteilter mit `versatz` — Emulator-Profil wählt (S-08: GameCube = Single-Core + gekoppelte Komponenten → ein Zähler, Versatz je Einheit).

## 5. Determinismus-Modus

```sspl
laufzeit.deterministisch(seed: u64);
```

- Ersetzt OS-Scheduling: Tasks laufen kooperativ auf dem virtuellen Scheduler; `zyklus`-Zeit ersetzt `zeit.jetzt()`; Kanal-Reihenfolgen, `waehle`-Arm-Wahl und Event-Tiebreaks folgen dem Seed.
- **Replay:** `laufzeit.zeichne_auf(datei)` speichert Scheduling-Entscheidungen (SSOB-Stream); `laufzeit.replay(datei)` reproduziert bit-identisch → Chrono-Debugger real.
- Einsatz: Netplay-Lockstep, Regressionstests, Bug-Reproduktion.

## 6. Echte Zeit & Host-Integration

- `zeit` (monoton, `zeit.jetzt()`, `zeit.schlafe`) vs. `zyklus` (virtuell) sind **getrennte Namespaces** — Vermischung ist `E6010`-verdächtig (Lint `W6010`).
- Frame-Pacing: `zyklus.rahmen(59.94)` liefert Vblank-Events; Powermode-Skalierung = Frequenz-Faktor auf `plane_nach`-Deltas (konfigurierbar, dokumentiert).

## 7. Fehler- & Abbruchmodell

- Task-Panic isoliert den Task; Scope-Propagiert Fehler an `versuche`-Block des Spawners.
- `abbruch.signal()` = Cancellation-Token; Kanäle/Events respektieren es; `waehle` mit `fall <- abbruch`-Arm.
- Keine Exceptions über Kanäle — `result<T,E>` für erwartete Fehler.

## 8. Emulator-Abbildung (Vorschau S-08)

| Gast-Hardware | Sprachmittel |
|---|---|
| Gekko-Instruktionsbudget | `zyklus.vorruecken(kosten)` pro Instruktion/Block |
| Vblank/VI-Interrupt | `signal.ausloesen("VI")` geplant bei Frame-Grenze |
| DSP/DVD-DMA | `plane_nach(latenz_zyklen, dma_handler)` |
| SI-Polling | periodisches Event (replanning) |
| CPU/GPU-Sync | gemeinsame Queue → Ordnung garantiert |
| Savestate | Queue-Zustand ist serialisierbar (SSOB) — Voraussetzung: Event-Payloads sind SSOB-serialisierbar oder funktionsidentisch registriert |

## 9. Akzeptanzkriterien

- Determinismus: 2 Tasks tauschen 1 Mio. Events → gleiche Zustands-Hashs über 100 Läufe (x86_64+arm64).
- `spsc`-Durchsatz ≥ 200 Mio. Ops/s Single-Pair; `waehle`-Latenz < 1 µs bei 8 Kanälen.
- Queue: 10 Mio. geplante Events, stabile Ordnung, `O(log n)` Insert; Savestate-Roundtrip eines laufenden Schedulers.
- Replay: aufgezeichnete 60-s-Emulator-Session reproduziert identischen Endzustand.
