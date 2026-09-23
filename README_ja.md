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

**次世代のゲームキューブエミュレーション体験。**

Neo-Cubeは、**SSPL v9.5**フレームワーク上でネイティブに構築された、サイクル精度の高い革新的なゲームキューブエミュレータです。ハイブリッドコンポーネントベースアーキテクチャ（CBA）により、絶対的なパフォーマンス、プレミアムなUI、前例のないネットワーク機能を提供します。

> [!NOTE]
> **限定機能:** Neo-Cubeは**FlippyDrive Deluxe**エミュレーションをネイティブサポートしています！仮想イーサネットアドオンを介してPC/NASから直接ISOをストリーミングし、P2Pを介して世界中でLANマルチプレイヤーゲームをプレイできます。

## 🌟 主な機能
- **CBA:** CPU、GPU、DSPの同期実行を備えたモジュラーコア。
- **Neo-Cube パワーモード:** 最大限のパフォーマンスを解き放ちます。このモードは、拡張されたRAMと超最適化されたDolphin CPUコアを提供し、可能な限り最高の忠実度とフレームレートを実現します。
  
  <img src="docs/powermode_config.jpg" alt="Power-Mode Config" width="800">

- **SSGE Vulkan バックエンド:** 完璧なレンダリングのための新しい SonnerStudioGraficEngine (`vulkan_backend.sspl`) とのネイティブ統合。
- **FlippyDrive イーサネット アドオン:** シームレスな ISO ゲーム ロードのための仮想 LAN/NAS エミュレーション (`ethernet_addon.sspl`)。
- **ネイティブ DVD インターフェイス:** ISO ファイルシステム用の新しいサイクル アキュレート パーサー (`dvd_interface.sspl`)。
- **Neo-Cube Live (NCL):** LANトンネリングによるオンラインマルチプレイ。
- **CubeWave VoIP:** パーティーを作成するためのOS統合ボイスチャット。
- **Flippy-OSデラックスモード:** オリジナルのCubeBootファームウェアに直接起動。
- **プレミアム "Neo-XMB" ダッシュボード:** 息を呑むような 4K グラスモーフィズムのダークモード ユーザー インターフェイスで、60 FPS でスムーズに動作します。最高の独占性のために構築されています。
  
  <img src="docs/system_settings.jpg" alt="System Settings" width="800">

---
*SonnerStudioによって開発されました。全著作権所有。*
