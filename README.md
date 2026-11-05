# Daemoshot (v15.0.0-custom)

**Daemoshot** es una versión personalizada y extendida de captura de pantalla y anotación explicativa en tiempo real para Windows (x86_64), creada a partir del análisis de ingeniería inversa con **REA (`rea-agents`) + Ghidra** y el código fuente abierto de **[Flameshot](https://github.com/flameshot-org/flameshot)**.

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
   - Se verificó la arquitectura `PE32+ (amd64)` del ejecutable original `flameshot.exe` (`SHA-256: 40a9bb0c006fe47f7a79dd1a3f6e8f956b89a4632cbf0adfc99b069cefb198cb`), compilado con MSVC y enlazado dinámicamente contra **Qt 6.9.3** (`Qt6Core.dll`, `Qt6Gui.dll`, `Qt6Widgets.dll`, `Qt6Network.dll`, `Qt6Svg.dll`).
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

#### B. Modo de Dibujo en Pantalla Viva (`ScreenDrawWidget` — `Alt+Shift+D`)
- Implementado en [`src/widgets/screendrawwidget.h`](flameshot/src/widgets/screendrawwidget.h) y [`src/widgets/screendrawwidget.cpp`](flameshot/src/widgets/screendrawwidget.cpp).
- Superpone un lienzo transparente multimonitor siempre visible sobre tus aplicaciones en ejecución.
- Incluye barra flotante interactiva con:
  - **✏️ Lápiz (`P`)**, **🖍️ Resaltador (`M`)**, **✨ Láser (`L`)**, **↗️ Flecha (`A`)**, **📏 Línea (`D`)**, **⬜ Rectángulo (`R`)**, **⭕ Círculo (`C`)**.
  - Conmutador **⏳ `Se desvanece (F)` / 📌 `Se mantiene (F)`** (también accesible con clic derecho).
  - Conmutador **🖍️ `Glow ON/OFF (H)`**.
  - Paleta rápida de 7 colores neón, ajuste de grosor con la rueda del ratón, **Deshacer (`Ctrl+Z`)**, **Limpiar (`X`)** y **Salir (`Esc`)**.

### 3. Identidad Propia (`Daemoshot`), Icono Personalizado e Inicio Automático con Windows
- Se integró el icono personalizado [`ico.png`](ico.png) convirtiéndolo en un recurso `.ico` multirresolución de 32 bits (`16×16`, `32×32`, `48×48`, `64×64`, `128×128`, `256×256`) incrustado en `daemoshot.exe` y en el sistema de recursos de Qt (`graphics.qrc`).
- Se habilitó por defecto el **inicio automático con Windows** (`startupLaunch = true`), registrando automáticamente `Daemoshot` en `HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`.

---

## ⌨️ Atajos Rápidos Principales

| Atajo | Acción |
| :--- | :--- |
| **`PrtSc`** (o `Meta+Shift+X`) | Iniciar captura directa en todos los monitores |
| **`Alt+Shift+D`** | Abrir/Cerrar **Modo Dibujo Explicativo en Pantalla Viva** |
| **`Ctrl+D`** *(en captura)* | Pasar de la captura al **Modo Dibujo en Pantalla Viva** |
| **`F`** | Alternar entre **⏳ Se desvanece** y **📌 Se mantiene** |
| **`H`** | Activar/Desactivar **🖍️ Efecto Resaltador (Glow)** en cualquier herramienta |
| **`L`** | Seleccionar herramienta **✨ Puntero/Trazo Láser** |
| **`P` / `M` / `A` / `D` / `R` / `C`** | Lápiz (`P`), Resaltador (`M`), Flecha (`A`), Línea (`D`), Rectángulo (`R`), Círculo (`C`) |
| **`X`** | Limpiar todos los trazos dibujados |
| **`Rueda del ratón`** | Cambiar grosor de la herramienta activa |
| **`Ctrl+Z`** | Deshacer último trazo |
| **`Esc`** | Salir de la captura o del modo dibujo en pantalla |

---

## 📦 Instalación y Uso

### Opción 1: Usar el Instalador (`Daemoshot-Setup.exe`)
Ejecuta **`Daemoshot-Setup.exe`**:
1. Instala **Daemoshot** en `%LOCALAPPDATA%\Programs\Daemoshot`.
2. Crea accesos directos en el **Escritorio** y en el **Menú Inicio** con el icono oficial de Daemoshot.
3. Configura el **inicio automático al arrancar Windows** (`HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\Daemoshot`).
4. Registra el desinstalador en *Aplicaciones instaladas* de Windows y deja **Daemoshot** ejecutándose en la bandeja del sistema.

### Opción 2: Ejecución Directa Portable (`daemoshot/`)
También puedes ejecutar directamente [`daemoshot/daemoshot.exe`](daemoshot/daemoshot.exe). Al iniciarse por primera vez, queda residente en la bandeja del sistema y se registra para iniciar con Windows.
