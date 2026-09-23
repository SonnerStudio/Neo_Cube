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

**L'expérience d'émulation GameCube de nouvelle génération.**

Neo-Cube est un émulateur GameCube révolutionnaire, précis au cycle près, construit nativement sur le framework **SSPL v9.5**. Grâce à son architecture basée sur des composants (CBA), Neo-Cube offre des performances absolues, une interface utilisateur premium et des capacités réseau sans précédent.

> [!NOTE]
> **Fonctionnalité exclusive :** Neo-Cube prend en charge nativement l'émulation **FlippyDrive Deluxe** ! Diffusez des ISO directement depuis votre PC/NAS via un module Ethernet virtuel et jouez à des jeux multijoueurs LAN dans le monde entier via P2P.

## 🌟 Fonctionnalités principales
- **CBA :** Cœur modulaire avec exécution synchrone du CPU, du GPU et du DSP.
- **Neo-Cube Power-Mode :** Libérez les performances maximales ! Ce mode fournit une RAM étendue et un cœur de processeur Dolphin ultra-optimisé pour la fidélité et les fréquences d'images les plus élevées possibles.
  
  <img src="docs/powermode_config.jpg" alt="Power-Mode Config" width="800">

- **Backend Vulkan SSGE :** Intégration native avec le nouveau SonnerStudioGraficEngine (`vulkan_backend.sspl`) pour un rendu impeccable.
- **Module complémentaire Ethernet FlippyDrive :** Émulation LAN/NAS virtuelle (`ethernet_addon.sspl`) pour un chargement transparent des jeux ISO.
- **Interface DVD native :** Nouvel analyseur précis pour les systèmes de fichiers ISO (`dvd_interface.sspl`).
- **Neo-Cube Live (NCL) :** Multijoueur en ligne via un tunnel LAN.
- **CubeWave VoIP :** Chat vocal intégré au système d'exploitation pour créer des groupes.
- **Mode Flippy-OS Deluxe :** Démarrez directement sur le firmware d'origine CubeBoot.
- **Tableau de bord Premium "Neo-XMB" :** Une interface utilisateur époustouflante en mode sombre Glassmorphism 4K fonctionnant à 60 FPS fluides. Construit pour un maximum d'exclusivité.
  
  <img src="docs/system_settings.jpg" alt="System Settings" width="800">


## 🚀 À venir : SuperCube-Mode 64
Nous développons actuellement l'évolution ultime : **SuperCube-Mode 64**. Cette couche d'émulation 64 bits de nouvelle génération comprend une refonte complète du matériel virtuel (Neo-Gekko-64 CPU, Neo-Flipper-64 GPU, Neo-Macaron-64 Audio).

Les jeux GameCube classiques sont simultanément transformés via un hachage heuristique et une injection d'actifs à la volée : la rastérisation statique est remplacée par le raytracing dynamique (Illumination Globale & matériaux PBR). Le système passe en interne de 1080p jusqu'aux résolutions 8K ultra-nettes, optimisé par l'IA (DLSS/MetalFX).

Parallèlement, le nouveau SDK SuperCube ouvre des dimensions inédites pour la communauté homebrew : développez des titres de nouvelle génération avec une mémoire et des shaders de calcul illimités, tout en restant fidèle au style nostalgique et bien-aimé de la GameCube ! Préparez-vous à l'Audio Spatial (y compris le réglage audiophile 432 Hz) et à des temps de chargement nuls via l'émulation NVMe PCIe 4.0. *Plus de détails bientôt...*

---
*Développé avec passion par SonnerStudio. Tous droits réservés.*
