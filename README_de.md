<p align="center">
  <img src="docs/neo_cube_logo.jpg" alt="Neo-Cube Logo" width="600">
</p>

# Neo-Cube 
<div align="center">
  <a href="README.md"><img src="docs/flags/gb.png" width="16" alt="EN"> English</a> |
  <a href="README_de.md"><img src="docs/flags/de.png" width="16" alt="DE"> Deutsch</a> |
  <a href="README_es.md"><img src="docs/flags/es.png" width="16" alt="ES"> Español</a> |
  <a href="README_ja.md"><img src="docs/flags/jp.png" width="16" alt="JA"> 日本語</a> |
  <a href="README_zh.md"><img src="docs/flags/cn.png" width="16" alt="ZH"> 中文</a> |
  <a href="README_ru.md"><img src="docs/flags/ru.png" width="16" alt="RU"> Русский</a> |
  <a href="README_fr.md"><img src="docs/flags/fr.png" width="16" alt="FR"> Français</a>
</div>

**Die Next-Generation GameCube Emulations-Erfahrung.**

Neo-Cube ist ein revolutionärer, zyklusgenauer GameCube-Emulator, der nativ auf dem **SSPL v9.5 (Universal Edition)** Framework basiert. Durch die hybride Component-Based Architecture (CBA) bietet Neo-Cube absolute Performance, ein Premium Konsolen-UI und beispiellose Netzwerkfähigkeiten.

> [!NOTE]
> **Exklusives Highlight:** Neo-Cube unterstützt nativ die **FlippyDrive Deluxe** Emulation! Streame ISOs direkt von deinem PC/NAS über ein virtuelles Ethernet Addon und spiele LAN Multiplayer-Titel (wie *Mario Kart: Double Dash!!*) weltweit über P2P!

## 🌟 Kern-Features

- **Component-Based Architecture (CBA):** Der Emulator-Kern ist komplett modular. GekkoCPU, FlipperGPU, MacaronDSP und der Event-Bus operieren absolut synchron.
- **Neo-Cube Power-Mode:** Entfessle maximale Leistung! Dieser Modus bietet erweiterten Arbeitsspeicher und einen stark optimierten Dolphin-CPU-Kern für die bestmögliche Darstellung und höchste Framerates.
  
  <img src="docs/powermode_config.jpg" alt="Power-Mode Config" width="800">

- **SSGE Vulkan Backend:** Native Integration mit der neuen SonnerStudioGraficEngine (`vulkan_backend.sspl`) für makelloses Rendering.
- **FlippyDrive Ethernet Addon:** Virtuelle LAN/NAS Emulation (`ethernet_addon.sspl`) für nahtloses Laden von ISOs.
- **Natives DVD Interface:** Neuer zyklusgenauer Parser für ISO-Dateisysteme (`dvd_interface.sspl`).
- **Neo-Cube Live (NCL):** Das virtuelle GameCube-LAN wird über das Internet getunnelt (UDP Hole Punching), um Online-Multiplayer zu ermöglichen.
- **CubeWave Voice & Party System:** Erstelle Lobbys und unterhalte dich mit deinen Freunden über das integrierte VoIP-System, während du spielst.
- **Physischer SD-Karten Passthrough:** Lese originale GameCube Homebrew und ISOs direkt von deinem physischen SD-Kartenleser (Raw Block I/O).
- **Premium "Neo-XMB" Dashboard:** Eine atemberaubende 4K Glassmorphism Dark-Mode Benutzeroberfläche, die mit 60FPS läuft. Entwickelt für absolute Exklusivität (PlayStation/Xbox Niveau).
  
  <img src="docs/system_settings.jpg" alt="System Settings" width="800">


## 🚀 Ausblick: SuperCube-Mode 64
Wir entwickeln derzeit die ultimative Evolution: den **SuperCube-Mode 64**. Dieser Next-Gen 64-Bit-Emulations-Layer umfasst ein komplettes virtuelles Hardware-Redesign (Neo-Gekko-64 CPU, Neo-Flipper-64 GPU, Neo-Macaron-64 Audio).

Klassische GameCube-Spiele werden durch heuristisches Hashing und "On-The-Fly" Asset-Injection simultan transformiert: Statische Rasterization wird durch dynamisches Raytracing (Global Illumination & PBR-Materialien) ersetzt. Das System skaliert intern von 1080p bis hin zu ultra-scharfen 8K-Auflösungen, gestützt durch KI-Upscaling (DLSS/MetalFX).

Gleichzeitig eröffnet das neue SuperCube SDK völlig neue Dimensionen für die Homebrew-Community: Entwickelt Next-Gen-Titel mit unlimitiertem Speicher und Compute-Shadern, während ihr dem nostalgischen und allseits beliebten GameCube-Stil treu bleibt! Mache dich bereit für Spatial Audio (inkl. audiophilem 432 Hz Tuning) und Zero-Load-Times via NVMe PCIe 4.0 Emulation. *Weitere Details folgen in Kürze...*

---
*Mit Leidenschaft entwickelt von SonnerStudio. Verfügbar für Windows 11 & macOS (M-Serie).*
