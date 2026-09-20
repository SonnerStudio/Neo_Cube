<div align="right">
  <a href="README.md">🇬🇧 English</a> |
  <a href="README_de.md">🇩🇪 Deutsch</a> |
  <a href="README_es.md">🇪🇸 Español</a> |
  <a href="README_ja.md">🇯🇵 日本語</a> |
  <a href="README_zh.md">🇨🇳 中文</a> |
  <a href="README_ru.md">🇷🇺 Русский</a> |
  <a href="README_fr.md">🇫🇷 Français</a>
</div>

<p align="center">
  <img src="docs/neo_cube_logo.jpg" alt="Neo-Cube Logo" width="600">
</p>

# Neo-Cube 
**Die Next-Generation GameCube Emulations-Erfahrung.**

Neo-Cube ist ein revolutionärer, zyklusgenauer GameCube-Emulator, der nativ auf dem **SSPL v9.5 (Universal Edition)** Framework basiert. Durch die hybride Component-Based Architecture (CBA) bietet Neo-Cube absolute Performance, ein Premium Konsolen-UI und beispiellose Netzwerkfähigkeiten.

> [!NOTE]
> **Exklusives Highlight:** Neo-Cube unterstützt nativ die **FlippyDrive Deluxe** Emulation! Streame ISOs direkt von deinem PC/NAS über ein virtuelles Ethernet Addon und spiele LAN Multiplayer-Titel (wie *Mario Kart: Double Dash!!*) weltweit über P2P!

## 🌟 Kern-Features

- **Component-Based Architecture (CBA):** Der Emulator-Kern ist komplett modular. GekkoCPU, FlipperGPU, MacaronDSP und der Event-Bus operieren absolut synchron.
- **Neo-Cube Power-Mode:** Entfessle maximale Leistung! Dieser Modus bietet erweiterten Arbeitsspeicher und einen stark optimierten Dolphin-CPU-Kern für die bestmögliche Darstellung und höchste Framerates.
- **Neo-Cube Live (NCL):** Das virtuelle GameCube-LAN wird über das Internet getunnelt (UDP Hole Punching), um Online-Multiplayer zu ermöglichen.
- **CubeWave Voice & Party System:** Erstelle Lobbys und unterhalte dich mit deinen Freunden über das integrierte VoIP-System, während du spielst.
- **Physischer SD-Karten Passthrough:** Lese originale GameCube Homebrew und ISOs direkt von deinem physischen SD-Kartenleser (Raw Block I/O).
- **Premium "Neo-XMB" Dashboard:** Eine atemberaubende 4K Glassmorphism Dark-Mode Benutzeroberfläche, die mit 60FPS läuft. Entwickelt für absolute Exklusivität (PlayStation/Xbox Niveau).

## 🚀 Installation & Setup

1. **Repository klonen:**
   ```bash
   git clone https://github.com/SonnerStudio/Neo_Cube.git
   ```
2. **Mit SSPL kompilieren:**
   ```bash
   sspl schmiede neo_cube.sspl
   ```
3. **Emulator starten:**
   ```bash
   sspl run neo_cube.sspl.ssplx
   ```

## 🎮 Flippy-OS Deluxe Mode
Starte den Emulator mit dem Parameter `--deluxe`, um direkt in die originale **CubeBoot** Firmware zu booten. Der Emulator spiegelt hierbei exakt die Boot-Sequenz des Raspberry Pi Picos wider!

---
*Mit Leidenschaft entwickelt von SonnerStudio. Verfügbar für Windows 11 & macOS (M-Serie).*
