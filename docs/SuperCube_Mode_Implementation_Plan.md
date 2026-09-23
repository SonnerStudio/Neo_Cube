# Deep-Dive Implementierungsplan: SuperCube-Mode & Neo-Hardware-Stack

Dieses Dokument dient als technisches Blueprint und Spezifikation für die Entwicklung des **SuperCube-Modes** innerhalb des Neo-Cube Emulators. Ziel ist die vollständige Neudefinition der klassischen GameCube-Hardware in ein fiktives, hypermodernes 64-Bit Next-Gen Konsolen-Ecosystem ("Neo-Stack"). 

Diese Architektur erlaubt nicht nur das Abspielen klassischer Spiele in 8K (mit Asset-Injection & Path-Tracing) sowie Spatial Audio, sondern bildet auch das Fundament für "SuperCube-Exclusive" Homebrew-Software mit Next-Gen-Anforderungen.

> [!NOTE]
> **Status:** Der Plan ist **freigegeben**. Die Architektur umfasst nun den kompletten 64-Bit Hardware-Stack inkl. Audio, CPU, Speicher und Netzwerk, sowie die `NeoFlipperAPI` als zentrale Hardware-Abstraktionsschicht (RTX & Apple Silicon).

---

## 1. Der komplette "Neo-Hardware-Stack" (64-Bit Architektur)

Die klassische Hardware (Gekko, Flipper, Macaron, EXI) wird im SuperCube-Mode durch 64-Bit-Gegenstücke ersetzt. Der Hypervisor des Emulators fängt alte 32-Bit-Aufrufe ab und translatiert sie auf dieses neue Level, oder führt nativen 64-Bit-Code direkt aus.

### 1.1 CPU: Neo-Gekko-64
*   **Architektur:** 64-Bit RISC-Architektur, abgeleitet von modernen ARMv9/PowerISA-Konzepten.
*   **Multicore-Prozessor:** Statt des Single-Core Gekko emulieren wir eine 8-Kern (16-Thread) CPU. Klassische Spiele laufen auf Core 0, während der Emulator Mod-Injections, Netzwerk und Physik-Berechnungen dynamisch auf Core 1-7 offloadet.
*   **Erweiterte Register:** 128-Bit und 256-Bit SIMD-Register (`v0` - `v63`). Hardware-beschleunigte Tensor-Register für native Machine-Learning-Berechnungen (z. B. lokales KI-Upscaling, Gegner-KI).
*   **Memory Controller:** **16 GB virtueller Unified Memory** (GDDR6-Äquivalent) statt der klassischen 40 MB. Ermöglicht unkomprimierte 8K-Texturen und riesige Raytracing-BVH-Bäume.

### 1.2 GPU: Neo-Flipper-64
*   **Grafik-Pipeline:** Verzicht auf Fixed-Function-Rasterizer. Native Unterstützung für Mesh-Shaders, Compute-Shaders und Hardware-Raytracing.
*   **Auflösung & Ausgabe:** Nativer Support für 1080p, 1440p, 4K und 8K (7680 × 4320) Ausgabeformate. 
*   **Variable Refresh Rate (VRR):** Die strikte 60Hz-Limitierung wird aufgebrochen (Entkopplung des VBLANK-Interrupts), sodass Spiele mit 120Hz oder 144Hz (G-Sync/FreeSync) laufen.

### 1.3 Audio: Neo-Macaron-64 DSP
*   **Audio-Engine:** Ein 64-Bit Digital Signal Processor für revolutionäres Audio.
*   **Abtastraten & Tiefe:** Unterstützung für 32-Bit Floating-Point Audio bei Abtastraten bis zu 384 kHz.
*   **432 Hz Tuning (Audiophil):** Integrierter, latenzfreier Echtzeit-Pitch-Shifter auf Hardware-Ebene. Dieser transponiert die Audioausgabe automatisch auf die harmonische 432-Hz-Frequenz (Verstimmung um ca. -31,76 Cent), was bei Audio-Enthusiasten für eine natürlichere und entspanntere Klangwahrnehmung geschätzt wird.
*   **Surround & Spatial Audio:** 
    *   Voller Support für Mono, Stereo (mit Binaural-HRTF-Processing für Kopfhörer), 5.1 und 7.1 Surround.
    *   Integration von "Object-Based Audio" (ähnlich Dolby Atmos / DTS:X): Die Audio-Engine berechnet die 3D-Position von Geräuschen in der emulierten Welt und gibt diese an das Windows Sonic / Apple Spatial Audio Backend weiter.

### 1.4 Netzwerk: Neo-Broadband-64
*   **Virtuelles Gigabit-LAN:** Das klassische BBA (Broadband Adapter) Limit von 10/100 MBit/s wird gesprengt. Emulation eines modernen Gigabit-Netzwerk-Controllers.
*   **IPv6 & Wi-Fi 7 Layer:** Volle IPv6-Adressierung für Neo-Cube Live (NCL) Multiplayer, was NAT-Probleme eliminiert.
*   **WebRTC P2P-Integration:** Integrierte Verschlüsselung (DTLS/SRTP) auf Hardware-Ebene für sicheres Matchmaking und Lobbysysteme.

### 1.5 Storage: Neo-EXI-64 (External Interface)
*   **NVMe SSD Emulation:** Die Limitierungen von DVD-Laufwerk (ca. 3 MB/s) und Memory Card (SPI-Bus) entfallen. Der virtuelle I/O-Bus simuliert PCIe 4.0 Geschwindigkeiten. Ladezeiten in Spielen existieren praktisch nicht mehr (Zero-Load-Times).

---

## 2. Die universelle Abstraktionsschicht: `NeoFlipperAPI`

Da wir RTX- und Apple Silicon-Hardware unterstützen, wird der *Neo-Hardware-Stack* nicht direkt gegen Host-Treiber programmiert. Die `NeoFlipperAPI` leitet hochkomplexe Aufrufe asynchron an das jeweils aktive Backend weiter:
*   `nf_api_dispatch_ray()`: Standardisiertes Hardware-Raytracing.
*   `nf_api_execute_tensor_op()`: Matrix-Berechnungen (für Tensor-Cores oder Apple Neural Engine).
*   `nf_api_spatial_audio_emit()`: Sendet ein 3D-Audio-Objekt an die Host-Surround-Engine.

---

## 3. Cross-Platform Rendering Backends (Vulkan & Metal)

### 3.1 Das NVIDIA RTX Backend (Windows / Linux)
*   **Vulkan Extensions:** `VK_KHR_ray_tracing_pipeline` für Full-Scene Path-Tracing.
*   **DLSS 3.5 & Ray Reconstruction:** Das Spiel läuft intern in 1080p (für maximale Path-Tracing-Rays) und wird von den RTX-Tensor-Cores auf 4K/8K hochskaliert und entrauscht.

### 3.2 Das Apple Silicon Backend (Mac M-Prozessoren)
*   **Metal 3 API:** Nutzt den `MPSRayIntersector` der M-Chips für blitzschnelles Hardware-Raytracing.
*   **MetalFX Upscaling:** Apples natives KI-Upscaling (Spatial & Temporal), betrieben durch die Apple Neural Engine (ANE), für lautloses 8K-Gaming auf MacBooks.

---

## 4. Asset-Replacement Pipeline (Die "Konvertierung")

Klassische 480p-Spiele (*Wind Waker*, *Metroid Prime*) werden On-The-Fly in das Next-Gen-Level übersetzt.

*   **Heuristik & Hashing (MurmurHash64):** Jeder alte Draw-Call wird gehasht.
*   **Anchor-Asset-System (Anti-Culling):** Um zu verhindern, dass Mod-Objekte verschwinden, wenn die alte GameCube-Engine sie wegen "Out-of-View" ausblendet, werden statische Welt-Koordinaten als Anker genutzt (identisch zu RTX Remix).
*   **PBR-Materialien in `.scube` Packages:** Modder nutzen **glTF 2.0** oder **USDZ**, um 8K-Meshes, Normal-Maps und Roughness-Maps in den Emulator zu injizieren.

---

## 5. Dynamic Recompilation (JIT) Erweiterung

Der CPU-Kern muss massiv optimiert werden, um den Overhead der Emulation des riesigen Hardware-Stacks zu bewältigen.
*   **SIMD Translation:** GameCube Paired-Singles werden hardwarenah in AVX2/AVX-512 (Intel/AMD) oder NEON (Apple) SIMD-Befehle übersetzt.
*   **Multithreading-Dispatcher:** Die Emulation wird strikt parallelisiert. Core 1: JIT & Game-Logic, Core 2: Audio DSP & 432Hz Pitching, Core 3: Hashing & Asset-Injection, Core 4: Netzwerk-Stack.

---

## 6. Meilensteine & Roadmap (Umsetzung)

1. **Milestone 1: Core 64-Bit Foundation**
   - Schreiben der neuen `core/neo_*.sspl` Komponenten (Gekko-64, Macaron-64, etc.).
   - Implementierung der `NeoFlipperAPI` als Host-Verteiler.
2. **Milestone 2: 8K Rendering & Upscaling**
   - Integration von Vulkan (DLSS) und Metal (MetalFX).
   - Test-Rendern von 3D-Geometrie auf Basis der neuen 64-Bit CPU-Befehle in nativer 8K-Auflösung.
3. **Milestone 3: Spatial Audio & 432 Hz Tuning**
   - Aufbau des `Neo-Macaron-64` DSPs. Echtzeit Audio-Resampling auf 384 kHz inkl. der mathematischen Algorithmen für die harmonische 432 Hz-Konvertierung und Objekt-basiertem Surround.
4. **Milestone 4: Path-Tracing & Asset Injection**
   - Hashing-Engine aktivieren.
   - Eine Test-Injektion: Tausch eines 2002er Low-Poly Assets gegen ein hochdetailliertes glTF-Modell, welches in Echtzeit per Raytracing beleuchtet wird.
