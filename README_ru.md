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

**Эмуляция GameCube следующего поколения.**

Neo-Cube — это революционный, тактово-точный эмулятор GameCube, нативно созданный на фреймворке **SSPL v9.5**. Благодаря гибридной компонентной архитектуре (CBA) Neo-Cube обеспечивает абсолютную производительность, премиальный пользовательский интерфейс и беспрецедентные сетевые возможности.

> [!NOTE]
> **Эксклюзивная функция:** Neo-Cube изначально поддерживает эмуляцию **FlippyDrive Deluxe**! Потоковая передача ISO напрямую с вашего ПК/NAS через виртуальный адаптер Ethernet и многопользовательские игры по локальной сети по всему миру через P2P.

## 🌟 Основные функции
- **CBA:** Модульное ядро с синхронным выполнением CPU, GPU и DSP.
- **Режим мощности Neo-Cube:** Раскройте максимальную производительность! Этот режим обеспечивает расширенную оперативную память и ультра-оптимизированное ядро процессора Dolphin для максимально возможной точности и частоты кадров.
  
  <img src="docs/powermode_config.jpg" alt="Power-Mode Config" width="800">

- **SSGE Vulkan Backend:** Встроенная интеграция с новым SonnerStudioGraficEngine (`vulkan_backend.sspl`) для безупречного рендеринга.
- **Дополнение FlippyDrive Ethernet:** Виртуальная эмуляция LAN/NAS (`ethernet_addon.sspl`) для плавной загрузки игр ISO.
- **Нативный интерфейс DVD:** Новый точный до цикла парсер для файловых систем ISO (`dvd_interface.sspl`).
- **Neo-Cube Live (NCL):** Многопользовательская онлайн-игра через туннелирование LAN.
- **CubeWave VoIP:** Встроенный в ОС голосовой чат для создания групп.
- **Flippy-OS Deluxe Mode:** Прямая загрузка оригинальной прошивки CubeBoot.
- **Премиум-панель "Neo-XMB":** Захватывающий дух пользовательский интерфейс 4K Glassmorphism Dark-Mode, работающий со скоростью 60 кадров в секунду. Создан для максимальной эксклюзивности.
  
  <img src="docs/system_settings.jpg" alt="System Settings" width="800">


## 🚀 В разработке: SuperCube-Mode 64
В настоящее время мы разрабатываем идеальную эволюцию: **SuperCube-Mode 64**. Этот 64-битный уровень эмуляции следующего поколения включает полный редизайн виртуального оборудования (Neo-Gekko-64 CPU, Neo-Flipper-64 GPU, Neo-Macaron-64 Audio).

Классические игры GameCube одновременно преобразуются с помощью эвристического хеширования и внедрения ассетов на лету: статическая растеризация заменяется динамической трассировкой лучей (Global Illumination и PBR-материалы). Система масштабируется от 1080p до сверхчеткого разрешения 8K благодаря ИИ-апскейлингу (DLSS/MetalFX).

В то же время, новый SuperCube SDK открывает совершенно новые измерения для сообщества homebrew: разрабатывайте проекты следующего поколения с неограниченной памятью и вычислительными шейдерами, оставаясь верными ностальгическому и всеми любимому стилю GameCube! Приготовьтесь к пространственному звуку (включая аудиофильскую настройку 432 Гц) и нулевому времени загрузки с помощью эмуляции NVMe PCIe 4.0. *Подробности скоро...*

---
*Разработано с энтузиазмом SonnerStudio. Все права защищены.*
