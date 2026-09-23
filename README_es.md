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

**La experiencia de emulación de GameCube de próxima generación.**

Neo-Cube es un emulador de GameCube revolucionario, con precisión de ciclos, construido nativamente sobre el framework **SSPL v9.5**. A través de su Arquitectura Basada en Componentes (CBA), Neo-Cube ofrece un rendimiento absoluto, una interfaz de usuario premium y capacidades de red sin precedentes.

> [!NOTE]
> **Característica Exclusiva:** ¡Neo-Cube soporta nativamente la emulación de **FlippyDrive Deluxe**! Transmite ISOs directamente desde tu PC/NAS a través de un adaptador Ethernet virtual y juega títulos multijugador LAN en todo el mundo mediante P2P.

## 🌟 Características Principales
- **CBA:** Núcleo modular con ejecución síncrona de CPU, GPU y DSP.
- **Modo Energía Neo-Cube (Power-Mode):** ¡Libera el máximo rendimiento! Este modo proporciona RAM expandida y un núcleo de CPU Dolphin ultra optimizado para la mayor fidelidad y velocidad de fotogramas posibles.
  
  <img src="docs/powermode_config.jpg" alt="Power-Mode Config" width="800">

- **SSGE Vulkan Backend:** Integración nativa con el nuevo SonnerStudioGraficEngine (`vulkan_backend.sspl`) para un renderizado impecable.
- **FlippyDrive Ethernet Addon:** Emulación virtual de LAN/NAS (`ethernet_addon.sspl`) para cargar juegos ISO sin problemas.
- **Interfaz de DVD Nativa:** Nuevo analizador de ciclo exacto para sistemas de archivos ISO (`dvd_interface.sspl`).
- **Neo-Cube Live (NCL):** Multijugador en línea mediante túnel LAN.
- **CubeWave VoIP:** Chat de voz integrado en el sistema operativo para crear grupos.
- **Modo Flippy-OS Deluxe:** Arranca directamente en el firmware original de CubeBoot.
- **Dashboard Premium "Neo-XMB":** Una impresionante interfaz de usuario en modo oscuro Glassmorphism 4K que funciona a 60 FPS fluidos. Construido para la máxima exclusividad.
  
  <img src="docs/system_settings.jpg" alt="System Settings" width="800">


## 🚀 Próximamente: SuperCube-Mode 64
Actualmente estamos desarrollando la evolución definitiva: **SuperCube-Mode 64**. Esta capa de emulación de 64 bits de próxima generación contará con un rediseño completo de hardware virtual (CPU Neo-Gekko-64, GPU Neo-Flipper-64, Audio Neo-Macaron-64). Prepárate para capacidades de resolución 8K, inyección de activos, audio espacial (incluida la afinación audiófila a 432 Hz) y tiempos de carga cero a través de la emulación NVMe PCIe 4.0. *Más detalles próximamente...*

---
*Desarrollado con pasión por SonnerStudio. Reservados todos los derechos.*
