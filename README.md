<p align="center">
  <img src="docs/neo_cube_logo.jpg" alt="Neo-Cube Logo" width="600">
</p>

# Neo-Cube
<div align="center">
  <a href="README.md">:uk: English</a> |
  <a href="README_de.md">:de: Deutsch</a> |
  <a href="README_es.md">:es: Español</a> |
  <a href="README_ja.md">:jp: 日本語</a> |
  <a href="README_zh.md">:cn: 中文</a> |
  <a href="README_ru.md">:ru: Русский</a> |
  <a href="README_fr.md">:fr: Français</a>
</div>

**The Next-Generation GameCube Emulation Experience.**

Neo-Cube is a revolutionary, cycle-accurate GameCube emulator built natively on the **SSPL v9.5 (Universal Edition)** framework. Designed from the ground up using a hybrid Component-Based Architecture (CBA), Neo-Cube delivers absolute performance, a premium console-level UI, and unprecedented networking capabilities.

> [!NOTE]
> **Exclusive Highlight:** Neo-Cube natively supports **FlippyDrive Deluxe** emulation! Stream ISOs directly from your PC/NAS over a virtual Ethernet Addon and play LAN multiplayer games (like *Mario Kart: Double Dash!!*) across the globe via P2P.

## 🌟 Core Features

- **Component-Based Architecture (CBA):** The emulator core is completely modular. GekkoCPU, FlipperGPU, MacaronDSP, and the Event-Bus operate in extreme synchronicity.
- **Neo-Cube Power-Mode:** Unleash maximum performance! This mode provides expanded RAM and an ultra-optimized Dolphin CPU core for the highest fidelity and frame rates possible.
- **Neo-Cube Live (NCL):** Say goodbye to local-only multiplayer. NCL tunnels your virtual GameCube's LAN traffic over the internet (UDP Hole Punching).
- **CubeWave Voice & Party System:** Create lobbies and chat with friends using the built-in VoIP functionality while playing.
- **Physical SD-Card Passthrough:** Reads original GameCube homebrew and ISOs directly from your PC's physical SD-card reader via Raw Block I/O.
- **Premium "Neo-XMB" Dashboard:** A breathtaking, 4K Glassmorphism Dark-Mode user interface running at a fluid 60FPS. Built for maximum exclusivity.

## 🚀 Installation & Setup

1. **Clone the repository:**
   ```bash
   git clone https://github.com/SonnerStudio/Neo_Cube.git
   ```
2. **Build with SSPL:**
   ```bash
   sspl schmiede neo_cube.sspl
   ```
3. **Run the Emulator:**
   ```bash
   sspl run neo_cube.sspl.ssplx
   ```

## 🎮 Flippy-OS Deluxe Mode
Launch the emulator with the `--deluxe` flag to boot directly into the original **CubeBoot** firmware (emulating the Raspberry Pi Pico boot sequence), giving you the ultimate authentic hardware experience!

---
*Built with passion by SonnerStudio. Available for Windows 11 & macOS (M-Series).*
