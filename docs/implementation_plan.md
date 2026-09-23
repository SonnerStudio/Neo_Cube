# Neo-Cube Emulator - Implementierungsplan (Final)

## Zielsetzung
Erstellung des "Neo-Cube" GameCube-Emulators für Windows 11 und macOS (Apple Silicon M-Prozessoren) in **SSPL v9.5**. Der Emulator soll eine hochpräzise Emulation der GameCube-Hardware bieten und das "Flippy-Drive" inkl. echtem **physischem SD-Karten-Passthrough** simulieren. Modernste Technologien wie Ubershader und zyklusgenaues Timing verhindern die Fehler älterer Emulatoren.

## Architektur & Technologie-Stack
Wir verwenden eine **hybride Architektur**, die das Beste aus beiden Welten vereint:

1. **Frontend / Benutzeroberfläche: Neo-Cube-OS (MVP/MVVM)**
   Für das User Interface importieren wir die Neo-Cube-OS Basis aus dem externen Projekt (`GameCube_DOL-001_Projekt`). Das Frontend wird in SSPL mit stark verbesserter nativer Auflösung (4K UI) integriert. Der Presenter regelt die Kommunikation zwischen der Neo-Cube-OS Oberfläche und dem Emulator-Kern.
   
2. **Emulator-Kern (Backend): Component-Based Architecture (CBA) & Event-Driven**
   Für die eigentliche Hardware-Emulation setzen wir auf eine komponentenbasierte Architektur. Eine Spielkonsole besteht aus diskreten Hardware-Chips (CPU, GPU, Speicherkontroller). Wir modellieren jeden Chip als eigenständige SSPL-Komponente (z.B. `GekkoCPU`, `FlipperGPU`, `FlippyDriveInterface`).
   Die Komponenten kommunizieren über einen zentralen Event-Bus. Nur so lässt sich ein zyklusgenaues (cycle-accurate) Hardware-Timing realisieren, was zwingend erforderlich ist, um bekannte Abstürze oder Audio-Desyncs älterer Emulatoren zu vermeiden.

**SSPL v9.5 Backend:**
- **Kern-Emulation:** **SSPL v9.5** (Component-Based) mit VRF C++-Engine für Direct I/O (SD-Karte) und die ressourcenintensive JIT-Kompilierung.
- **Grafik-Backend:** **Vulkan** (Win11) / **Metal** (macOS) mit Ubershadern (asynchrones Pipeline-Caching) gegen Stottern.

**Neo-Cube-Powermode (Custom Hardware Mode):**
Ein spezieller, zuschaltbarer Betriebsmodus, der die Limitierungen der originalen GameCube-Hardware aufhebt:
- **Dolphin-CPU Boost:** Übertaktung der virtuellen Gekko-CPU auf ein Vielfaches (z.B. 1.5+ GHz) zur Vermeidung von Ingame-Rucklern bei komplexen Szenen.
- **Speicher-Erweiterung:** Anhebung des Hauptspeichers (RAM) von 24 MB auf 128 MB (oder mehr) und des ARAMs, was extrem hochauflösende 8K-Texture-Packs und komplexe Homebrew-Mods ermöglicht.
- **Entkoppelte Framerates:** Umgehung der 30/60 FPS Limitierungen des Originalsystems durch asynchrone VBLANK-Generierung (bis zu 120/144 FPS).

**Flippy-Drive Emulation Subsystem (Deluxe Edition):**
- Logische Nachbildung der Raspberry Pi Pico und ESP32 Controller (Component-Based).
- **Ethernet Add-on Emulation:** Volle Unterstützung der FlippyDrive Deluxe Hardware, inklusive Netzwerk-Passthrough für Game-Streaming via NAS/PC und LAN-Gaming-Funktionen.
- Exklusiver Raw I/O Zugriff auf reale SD-Kartenleser des Hosts.
- Dynamisches In-Memory-Patching des GameCube-IPLs (CubeBoot).

## Vorgeschlagene Umsetzungsschritte (Phasen)

Aus Sicht des Projektmanagements setzen wir auf das **Minimum Viable Product (MVP)** Prinzip.

### Phase 1: Projekt-Setup & Minimum Viable Product (Core)
- Initialisierung als SSPLX-Paket (`sspl schmiede`) mit VRF-Bridge (`sspl vrf start`).
- Implementierung der Kern-Komponenten (`GekkoCPU`, `MMU`) nach CBA-Prinzipien.
- Ziel: Ein MVP-Core, der die ersten Instruktionen des GameCube IPL lesen kann.

### Phase 2: Audio & Video der nächsten Generation
- Aufbau des Flipper-Grafik-Prozessors über Vulkan/Metal (Ubershader-Architektur).
- LLE-Audio (Macaron DSP) für sauberen Sound.
- Anbindung des MVP-Frontends an die Emulator-Komponenten.

### Phase 3: Flippy-Drive Deluxe & Physischer SD-Karten-Support
- Emulation der Disc-Befehle (DVD-Interface Component).
- **SD-Slot Integration:** SSPL-Low-Level-Treiber für Block-Reads einer echten SD-Karte, transparent durchgereicht an das Flippy-Drive Subsystem.
- **Ethernet / NAS-Streaming:** Implementierung des FlippyDrive Deluxe Netzwerk-Stacks, um ISOs direkt von lokalen PC-Freigaben (SMB/NAS) oder für LAN-Multiplayer laden zu können.
- Test mit GameCube-Homebrew ("Swiss") via echter SD-Karte und Netzwerk.

### Phase 4: Optimierung & Neo-Cube-Powermode
- **JIT-Compiler:** Umstellung des CPU-Interpreters auf JIT (ARM64 & x86_64) für durchgängig 60 FPS (und mehr).
- **Powermode-Implementierung:** Einbau der zuschaltbaren RAM-Erweiterung und der dynamischen "Dolphin-CPU"-Übertaktung in den Kern.

### Phase 5: GPU Deep-Dive & Advanced Rendering
- **Shader-Analyse:** Low-Level Programmierung der GameCube Texture Environment (TEV) Einheiten in Vulkan/Metal Compute-Shadern.
- **Next-Gen Features:** Implementierung von Raytracing/Path-Tracing Hooks, 16x Anisotropischer Filterung und Widescreen-Heuristik auf Basis der 3D-Geometrie.

### Phase 6: Neo-Cube-OS Integration
- **UI-Portierung:** Importieren der Grundlagen aus dem `GameCube_DOL-001_Projekt` (Neo-Cube-OS).
- **Auflösungs-Upgrade:** Anpassen der OS-Assets und Schriften für eine gestochen scharfe, hochauflösende 4K-Darstellung.
- **Frontend-Backend Bridge:** Verknüpfung der Neo-Cube-OS GUI mit dem Emulator-Core via Event-Bus.

### Phase 7: Neo-Cube Live (WAN-Multiplayer & Voice-Chat)
- **Netplay-Tunneling:** Erweiterung des FlippyDrive Deluxe Ethernet-Stacks um eine WAN-Bridge (UDP Hole Punching/P2P). LAN-Pakete der originalen Spiele werden über das Internet getunnelt, um mehrere Neo-Cube Instanzen weltweit zu verbinden.
- **CubeWave Voice- & Party-System:** Ein im Neo-Cube-OS verankertes VoIP-System (Voice over IP). Spieler können (ähnlich wie bei PlayStation oder Xbox) Lobbys erstellen, sich über Headset in Echtzeit unterhalten und nahtlos gemeinsam Spiele starten.

### Phase 8: Premium Rollout & Flippy-OS Deluxe
- **NeoCube-OS Premium UI (Konsolen-Niveau):** Neugestaltung des OS auf dem Qualitätsniveau moderner Konsolen (PlayStation 5 / Xbox Series X). Ein exklusives, tiefes Dark-Theme mit subtilen Nebeleffekten (Glassmorphismus), fließenden 60FPS Micro-Animations und einer übersichtlichen Dashboard-Navigation ("Neo-XMB"). Keine Retro-Lila-Töne, sondern pure, moderne Exklusivität.
- **Flippy-OS Implementierung:** Direkte Emulation und Einbindung des originalen "Flippy-Drive" OS (CubeBoot) für den Deluxe Mode. Benutzer erleben das exakte Menü der physischen Hardware.
- **QA & Packaging:** Letzte Stabilitätstests, Bugfixing und Erstellung der Installationspakete (.exe/.dmg) für den finalen Rollout.

### Phase 9: Dev-Server Deployment & GitHub Launch
- **Dev-Server Preview:** Starten des Emulators (`neo_cube.sspl.ssplx`) im Hintergrund (Daemon-Modus), damit die UI und Funktionalität geprüft werden kann.
- **GitHub Repository Setup:** Initialisierung von Git.
- **Premium Readmes:** Verfassen von hochprofessionellen README-Dateien in mehreren Sprachen (Deutsch, Englisch), welche die CBA-Architektur, das Neo-Cube Live Netzwerk und den SD-Karten Passthrough bewerben.
- **Grafik- & Asset-Generierung:** KI-gestützte Erstellung von beeindruckenden Header-Grafiken und Logo-Mockups (via Image-Generator).

### Phase 10: Closed-Source Release & Marketing
- **Proprietäres Lizenz-Modell (All Rights Reserved):** Erstellung strikter Lizenzdateien in Englisch, Deutsch, Spanisch, Japanisch, Chinesisch, Russisch und Französisch. Diese sichern SonnerStudio alle geistigen Eigentumsrechte, untersagen Reverse-Engineering und binden die Endnutzer an unsere Bedingungen. 
- **Quellcode-Schutz (Secret Source):** Anpassung der `.gitignore`, sodass strikt **kein Quellcode** (`*.sspl`, C++ Sourcen, NeoCubeOS-Core Dateien) ins Git gelangt. Auf GitHub wird nur das Release-Paket (`*.ssplx`), die READMEs und die Lizenzen veröffentlicht.
- **Git History Bereinigung:** Entfernen des Quellcodes aus dem bisherigen lokalen Git-Index, damit bei einem Push absolute Geheimhaltung gewährleistet ist.
- **Internationale Dokumentation:** README-Dateien in allen oben genannten 7 Sprachen.
- **Facebook Marketing-Artikel:** Erstellung eines professionellen, werbewirksamen Social-Media-Posts (als Artifact), der alle bahnbrechenden Features des Neo-Cubes für die Öffentlichkeit vorstellt.

## Verification Plan
### Automatisierte Tests
- SSPL Unit-Tests für isolierte Hardware-Komponenten (`GekkoCPU`, `FlippyDriveInterface`).
### Manuelle Verifikation
- Entwickler-Umgebung (`sspl entwickle`) starten, echte SD-Karte einlegen und überprüfen, ob das CubeBoot-Menü flüssig lädt.
