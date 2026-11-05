# Daemoshot (v15.0.0-custom)

**Daemoshot** es una versión personalizada y extendida de captura de pantalla, anotación explicativa en tiempo real y grabación de video para Windows (x86_64), creada a partir del análisis de ingeniería inversa con **REA (`rea-agents`) + Ghidra** y el código fuente abierto de **[Flameshot](https://github.com/flameshot-org/flameshot)**.

---

## 🔗 Referencia al Proyecto Original

Este proyecto está basado en **[Flameshot](https://github.com/flameshot-org/flameshot)**, desarrollado originalmente por Alejandro Sirgo Rica y la comunidad de colaboradores de `flameshot-org`:

- **Repositorio oficial de referencia**: [https://github.com/flameshot-org/flameshot](https://github.com/flameshot-org/flameshot)
- **Licencia**: [GNU General Public License v3.0 o posterior (GPLv3+)](https://www.gnu.org/licenses/gpl-3.0.html)
- **Versión base analizada**: Binario instalado `Flameshot v14.0.0.1` (`C:\Program Files\Flameshot\bin\flameshot.exe`) + rama principal (`v15.0.0-dev`, commit `f4f2e4e`).

---

## 🔍 Qué se analizó con REA (Reverse Engineer Anything)

Utilizando el servidor MCP de **REA (`rea-agents@6.8.0`)** junto con **Ghidra 12.1.4** en **Antigravity**:

1. **Inspección de Aplicación de Escritorio y Binario PE (`inspect_desktop_application`, `inspect_pe_imports`, `inspect_pe_resources`)**:
   - Se verificó la arquitectura `PE32+ (amd64)` del ejecutable original `flameshot.exe`, compilado con MSVC y enlazado dinámicamente contra **Qt 6.9.3** (`Qt6Core.dll`, `Qt6Gui.dll`, `Qt6Widgets.dll`, `Qt6Network.dll`, `Qt6Svg.dll`).
2. **Identificación del cambio de comportamiento en v14.0+ (Selector de Monitor Obligatorio)**:
   - Se localizaron los símbolos y rutinas `ScreenGrabber::grabEntireDesktop`, `ScreenGrabber::selectMonitorAndCrop` y `MonitorPreview`.
   - En Flameshot v14.0, al invocar una captura gráfica (`gui`), `ScreenGrabber::grabEntireDesktop` capturaba el escritorio completo pero llamaba de forma obligatoria a `selectMonitorAndCrop(screenshot, ok)`, abriendo una ventana previa (`MonitorPreview`) que obligaba al usuario a elegir primero un monitor y recortaba la captura a una sola pantalla antes de mostrar `CaptureWidget`.

---

## 🚀 Qué se Hizo y Nuevas Funcionalidades en Daemoshot

### 1. Captura Directa de Todos los Monitores Simultáneamente (Sin Selector Previo)
Al iniciar una captura (`PrtSc`, clic en el icono de bandeja o `daemoshot.exe gui`), **Daemoshot** toma instantáneamente todos los monitores conectados y despliega el lienzo de selección unificado sobre todas las pantallas a la vez (como en las versiones clásicas de Flameshot), permitiendo arrastrar y seleccionar el área directamente en cualquier monitor sin pasos intermedios.

- **Archivos modificados**:
  - [`src/utils/screengrabber.cpp`](flameshot/src/utils/screengrabber.cpp):
    - En `ScreenGrabber::grabEntireDesktop`: salta `selectMonitorAndCrop` cuando `captureAllMonitors` está activo (habilitado por defecto) y conserva `m_selectedMonitor = -1`.
    - En `ScreenGrabber::windowsScreenshot`: compone todos los monitores sobre la geometría virtual unificada (`desktopGeometry()`) respetando los desplazamientos lógicos `(screenGeom.topLeft() - geometry.topLeft())` y el `devicePixelRatio` máximo para lograr alineación 1:1 en configuraciones multimonitor.
  - [`src/widgets/capture/capturewidget.cpp`](flameshot/src/widgets/capture/capturewidget.cpp):
    - Dimensiona y posiciona `CaptureWidget` abarcando `grabber.desktopGeometry()` cuando `selectedScreen == nullptr` y registra las regiones individuales de cada monitor en `m_buttonHandler`.
  - [`src/utils/confighandler.h`](flameshot/src/utils/confighandler.h), [`src/utils/confighandler.cpp`](flameshot/src/utils/confighandler.cpp), [`src/config/generalconf.h`](flameshot/src/config/generalconf.h) y [`src/config/generalconf.cpp`](flameshot/src/config/generalconf.cpp):
    - Añade la opción configurable `captureAllMonitors` (`true` por defecto) en la pestaña *General* de Configuración.

### 2. Herramientas de Dibujo Explicativo y Señalamiento con Modo Resaltador (`Se desvanece` o `Se mantiene`)
Se añadieron capacidades completas de anotación explicativa estilo *presentador / señalador* tanto **dentro del editor de capturas** como **en vivo sobre cualquier aplicación de escritorio**:

#### A. En el Editor de Captura (`CaptureWidget`)
- **Nueva Herramienta Puntero / Resaltador Láser (`TYPE_LASER`, tecla `L`)**:
  - Implementada en [`src/tools/laser/lasertool.h`](flameshot/src/tools/laser/lasertool.h) y [`src/tools/laser/lasertool.cpp`](flameshot/src/tools/laser/lasertool.cpp).
  - Renderiza curvas cuadráticas suavizadas con triple capa de resplandor neón y núcleo incandescente.
- **Barra Flotante Explicativa Superior (`m_explanatoryBar`)**:
  - **⏳ `Se desvanece (F)` / 📌 `Se mantiene (F)`**: Permite alternar en tiempo real si los trazos dibujados con **cualquier herramienta** (*Lápiz `P`, Láser `L`, Marcador `M`, Flecha `A`, Línea `D`, Rectángulo `R`, Círculo `C`*) se desvanecen automáticamente tras ~2.8 segundos o permanecen fijos en la captura.
  - **🖍️ `Resaltador ON/OFF (H)`**: Activa un halo resaltador luminoso debajo de cualquier herramienta de dibujo (Lápiz, Flecha, Línea, Rectángulo, Círculo).
  - **🗑️ `Limpiar trazos (X)`**: Borra todos los trazos dibujados con una sola tecla.
  - **🖥️ `Dibujar en Pantalla Viva (Ctrl+D)`**: Cambia directamente al modo de anotación transparente sobre el escritorio en vivo.

#### B. Modo de Dibujo en Pantalla Viva (`ScreenDrawWidget` — `Alt+Shift+D` o Menú de Bandeja)
- Implementado en [`src/widgets/screendrawwidget.h`](flameshot/src/widgets/screendrawwidget.h) y [`src/widgets/screendrawwidget.cpp`](flameshot/src/widgets/screendrawwidget.cpp).
- Superpone un lienzo transparente multimonitor siempre visible sobre tus aplicaciones en ejecución.
- Incluye barra flotante interactiva con:
  - **✏️ Lápiz (`P`)**, **🖍️ Resaltador (`M`)**, **✨ Láser (`L`)**, **↗️ Flecha (`A`)**, **📏 Línea (`D`)**, **⬜ Rectángulo (`R`)**, **⭕ Círculo (`C`)**.
  - Conmutador **⏳ `Se desvanece (F)` / 📌 `Se mantiene (F)`** (también accesible con clic derecho).
  - Conmutador **🖍️ `Glow ON/OFF (H)`**.
  - Paleta rápida de 7 colores neón, ajuste de grosor con la rueda del ratón, **Deshacer (`Ctrl+Z`)**, **Limpiar (`X`)** y **Salir (`Esc`)**.

### 3. Grabación de Video y Pizarra en Vivo (H.265 / HEVC o GIF) con Modo de Interacción de Escritorio
Se integró directamente en el menú flotante de la pizarra (`ScreenDrawWidget`) la capacidad de grabar en tiempo real e interactuar con el PC:

- **Modo Usar PC / Interacción de Escritorio (`🖱️ Usando PC (Tab)` / `✏️ Dibujando (Tab)`)**:
  - Resuelve la necesidad de usar programas, hacer clics en ventanas, navegar por internet o escribir texto **mientras se graba la pantalla o mientras la pizarra está abierta**.
  - Al activar este modo (mediante el botón en la barra o presionando la tecla **`Tab`**), la pizarra entra en modo *Mouse Passthrough* (`WM_NCHITTEST -> HTTRANSPARENT`). Todos los clics y movimientos del ratón pasan directamente a las aplicaciones de Windows.
  - La barra flotante permanece activa y clickeable para pausar, cambiar de herramienta, volver al modo dibujo o detener la grabación en cualquier momento.
  - Al seleccionar cualquier herramienta de dibujo (`P`, `L`, `M`, `A`, `D`, `R`, `C`), se reactiva automáticamente el modo pizarra para dibujar de inmediato.
- **Formatos soportados con un clic**:
  - **`🎬 H.265 (MP4)`**: Video de ultra alta compresión utilizando el códec moderno HEVC / H.265 (`libx265`, `-preset ultrafast`, tag `hvc1`), totalmente compatible con el reproductor nativo de Windows y navegadores modernos.
  - **`🎞️ GIF`**: Grabación en formato GIF animado mediante cuantización y dithering de paleta optimizada en FFmpeg, y con respaldo nativo mediante motor integrado en código fuente (`src/utils/gif.h`) para funcionar de forma autónoma sin depender de ejecutables externos.
- **Controles en la barra de herramientas de la pizarra**:
  - Botón selector de interacción: **`✏️ Dibujando (Tab)`** / **`🖱️ Usando PC (Tab)`**.
  - Botón selector de formato: **`🎬 H.265 (MP4)`** / **`🎞️ GIF`**.
  - Botón de grabación en tiempo real con cronómetro: **`🔴 REC (V)`** / **`⏹ REC [mm:ss] (V)`** con estilo visual distintivo en rojo.
  - Atajo de teclado: Tecla **`V`** para alternar iniciar / detener grabación inmediatamente sin levantar el cursor.
- **Captura enriquecida**:
  - Graba a 15 FPS la pantalla actual junto con todos los trazos explicativos en vivo, trazos que se desvanecen, puntero láser, cursor del ratón con indicador luminoso y badge superior de grabación.
- **Finalización y acceso directo**:
  - Guarda automáticamente los videos en la carpeta `Videos/Daemoshot_Recordings` con marca de tiempo única.
  - Al detener la grabación, muestra un diálogo resumen con opciones de un solo clic para **`▶ Reproducir / Abrir`** o **`📁 Abrir Carpeta`**.

### 4. Identidad Propia (`Daemoshot`), Icono Personalizado e Inicio Automático con Windows
- Se integró el icono personalizado [`ico.png`](ico.png) convirtiéndolo en un recurso `.ico` multirresolución de 32 bits (`16×16`, `32×32`, `48×48`, `64×64`, `128×128`, `256×256`) incrustado en `daemoshot.exe` y en el sistema de recursos de Qt (`graphics.qrc`).
- Se habilitó por defecto el **inicio automático con Windows** (`startupLaunch = true`), registrando automáticamente `Daemoshot` en `HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`.

---

## 📦 Instalador Autónomo para Windows (`Daemoshot-Setup.exe`)

Se generó un instalador autónomo ejecutable para Windows:

- **Ruta del Instalador**: [`Daemoshot-Setup.exe`](Daemoshot-Setup.exe) (ubicado en la raíz de este directorio).
- **Características del Instalador**:
  - Interfaz gráfica moderna con icono oficial de Daemoshot y barra de progreso.
  - Instala en `%LocalAppData%\Programs\Daemoshot` (no requiere privilegios elevados de Administrador).
  - Crea accesos directos en el **Menú Inicio** y en el **Escritorio**.
  - Configura automáticamente el inicio al encender Windows (`HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`).
  - Soporta instalación desatendida mediante el flag `--silent`.
  - Inicia inmediatamente Daemoshot en segundo plano tras la instalación.
