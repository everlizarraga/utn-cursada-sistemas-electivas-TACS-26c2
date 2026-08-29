# 🐳 Clase desde Cero — Docker · Setup en Windows

**Serie:** Clase desde Cero — Docker · Archivo de setup (se hace una sola vez) · El mapa está en `clase-cero-docker-roadmap.md`
**Si tu máquina es Mac:** este no es tu archivo — andá a `clase-cero-docker-setup-mac.md`.

---

### Sobre este documento

**Qué cubre:** las cuatro terminales de Windows y cuál vamos a usar (decisión incluida) · qué es WSL y por qué Docker lo necesita · cómo diagnosticar una máquina en estado desconocido (quizás instalaste cosas hace años y no te acordás — este archivo lo contempla) · instalar WSL desde cero si hace falta · el kit de comandos para administrar tus Linux · instalar Docker Desktop por **dos vías posibles** (elegís una), con una mini-guía de winget para quien no lo conozca · el paso que todo el mundo se olvida · y la verificación de punta a punta.

**Qué NO cubre:** conceptos. Este archivo es instrumental: no hay contenido de parcial, así que no vas a ver marcas 🔴🟡🟢 ni bloques 🎓. Solo ⚠️ (trampas) y 🕳️ (madrigueras).

**Cuándo hacerlo:** cuando quieras, pero **obligatorio antes del módulo 4**. Los módulos 1 a 3 se leen sin nada instalado.

**El orden importa:** WSL primero (secciones 3-5, común a todo), Docker Desktop después (sección 6, con sus dos vías).

---

## 1. El mapa: qué vas a tener al final

```
┌───────────────────────────────────────────────────────────────┐
│                       TU PC (Windows)                         │
│                                                               │
│  ┌──────────────────┐        ┌───────────────────────────────┐│
│  │ Terminal Ubuntu  │        │  WSL 2 (Linux REAL, liviano)  ││
│  │                  │        │  ┌─────────────┐ ┌───────────┐││
│  │  docker ...      │────────┼─▶│ Ubuntu      │ │ docker-   │││
│  │  (el cliente)    │        │  │ (tu distro) │ │ desktop   │││
│  └──────────────────┘        │  └─────────────┘ │ ← acá van │││
│                              │                  │ a vivir   │││
│  ┌──────────────────┐        │                  │ tus       │││
│  │ Docker Desktop   │────────┼─────────────────▶│ containers│││
│  │ (app/dashboard)  │        │                  └───────────┘││
│  └──────────────────┘        └───────────────────────────────┘│
└───────────────────────────────────────────────────────────────┘
```

Dos piezas: **WSL 2** (un Linux de verdad corriendo dentro de Windows) y **Docker Desktop** (la app, que usa ese Linux para hacer vivir a los containers). ¿Por qué Docker necesita un Linux? Versión corta: los containers necesitan un kernel de Linux y Windows no lo tiene. La explicación completa — incluido qué es exactamente una "distro" acá adentro y por qué `docker-desktop` no es "una VM adentro de otra" — está en el módulo 2, sección 6.3. Este setup es, de contrabando, tu primer contacto con ese hecho.

- **WSL (Windows Subsystem for Linux):** el mecanismo oficial de Microsoft para correr Linux dentro de Windows, sin dual boot ni VirtualBox. La versión que importa es **WSL 2**, que trae un kernel Linux real dentro de una VM muy liviana e integrada al sistema. Docker requiere la 2.
- **Distribución (distro):** un "sabor" de Linux — Ubuntu, Debian, etc. Dentro de WSL podés tener varias instaladas a la vez, cada una con sus archivos y su mundo propio, todas compartiendo el mismo kernel de la VM.

## 2. Las cuatro terminales de Windows — y la decisión

Windows tiene cuatro consolas posibles, y no dan lo mismo:

| Terminal | Qué es en realidad | ¿Para esta serie? |
|---|---|---|
| **CMD** | La consola histórica de Windows; su propio lenguaje (`dir`, `%cd%`) | ❌ Fuera del material |
| **PowerShell** | La consola moderna de Windows; otro lenguaje más (`${PWD}`, otras comillas) | ❌ Fuera del material — **excepto** para administrar WSL (secciones 3-5), que es territorio Windows |
| **Git Bash** | Un *imitador* de terminal Linux sobre Windows. Buen imitador — pero traduce rutas por debajo, y con Docker esa "ayuda" rompe | ⚠️ Compatible con dos asteriscos (recuadro abajo) |
| **Terminal de Ubuntu (WSL)** | Un **Linux real**. No imita: es bash de verdad, rutas de verdad | ✅ **La recomendada.** Todo el material corre acá tal cual está escrito |

**La decisión del material:** los comandos de la serie son shell Unix y corren idénticos en la Terminal de Mac, en cualquier Linux, y en la **terminal de Ubuntu** en Windows. Si usás esa, jamás vas a tener un "¿por qué en Mac anda y acá no?". PowerShell y CMD quedan afuera: Docker en sí funciona ahí, pero toda la sintaxis de alrededor es otro idioma. La única excepción: los comandos `wsl ...` de las secciones que siguen son administración *de* Windows y se tiran desde PowerShell — una vez configurado todo, no volvés a necesitarla.

> ⚠️ **Si insistís con Git Bash** (una sola vez, acá, y no se repite más): dos trampas conocidas. **(1) Traducción de rutas:** cuando un comando lleva una ruta Linux (`/algo`), Git Bash la convierte a ruta de Windows antes de pasarla — y Docker recibe cualquier cosa. Arreglo: anteponer `MSYS_NO_PATHCONV=1` al comando. **(2) Modo interactivo:** los comandos con `-it` a veces se cuelgan; arreglo: anteponer `winpty`. Funciona, pero son dos asteriscos permanentes. La terminal de Ubuntu no tiene ninguno.

## 3. Diagnóstico: ¿en qué estado está tu máquina?

Quizás instalaste WSL hace años siguiendo un tutorial, experimentaste, formateaste, y hoy no te acordás de nada. Perfecto — no asumimos nada. Abrí **PowerShell** (menú inicio → escribí "PowerShell" → Enter) y corré:

```console
> wsl -l -v
```

*( `-l` = list, `-v` = verbose: listá lo instalado, con detalle )*

**Interpretación de cada salida posible:**

**Caso A — error tipo `wsl: no se reconoce...` / `command not found`:**
No hay WSL. Andá a la sección 4 (instalación desde cero).

**Caso B — un texto de ayuda o "no tiene distribuciones instaladas":**
WSL existe pero no hay ningún Linux adentro. Instalá una distro:
```console
> wsl --install -d Ubuntu
```
y seguí en la sección 4 desde el paso del reinicio.

**Caso C — una tabla como esta:**
```console
  NAME              STATE           VERSION
* Ubuntu            Stopped         2        # ← el * marca tu distro por defecto
  Debian            Stopped         2        # ← restos de experimentos pasados: inofensivos
```
Ya tenés WSL y al menos una distro. Chequeá dos cosas: que **VERSION diga 2** (si dice 1: `wsl --set-version Ubuntu 2` y esperá — puede tardar), y que WSL esté al día:
```console
> wsl --update
```
Con eso, saltá directo a la sección 5 (el kit) o 6 (instalar Docker).

⚠️ `STATE: Stopped` **no es un problema** — las distros arrancan solas al abrirlas y se detienen solas al rato de no usarse. No hay que "prenderlas" a mano.

## 4. Instalación de WSL desde cero

1. PowerShell **como administrador**: menú inicio → "PowerShell" → click derecho → **Ejecutar como administrador**.
2. ```console
   > wsl --install
   ```
   Un solo comando: instala WSL 2 completo **y** Ubuntu como distro por defecto (Ubuntu es solo el default — con `-d` se elige otra).
3. **Reiniciá la máquina** cuando lo pida.
4. Al volver, se abre sola una consola de Ubuntu que termina la instalación (si no se abre, abrila vos: menú inicio → "Ubuntu"). Te pide crear un **usuario y contraseña de Linux** — es una identidad nueva, independiente de tu usuario de Windows. Usuario en minúsculas.

⚠️ **La contraseña no se ve mientras la tipeás.** Ni asteriscos, ni puntos, nada — el cursor no se mueve. La terminal no está rota: es el estilo Unix de pedir contraseñas. Tipeá y Enter.

⚠️ Si el instalador se queja de **virtualización deshabilitada**: tu CPU la soporta pero está apagada en el BIOS/UEFI. Chequeo rápido: Administrador de tareas → pestaña Rendimiento → CPU → abajo dice "Virtualización: Habilitada/Deshabilitada". Si está deshabilitada, hay que entrar al BIOS al prender la máquina (la tecla depende del fabricante: F2, F10, Supr…) y habilitar "Intel VT-x", "AMD-V" o "SVM". Es un toggle, se hace una vez.

## 5. El kit de administración de WSL

Todo esto se corre desde PowerShell. Es tu panel de control de los Linux instalados:

```console
> wsl -l -v                       # listar lo instalado, con estado y versión
> wsl -l -o                       # listar qué distros hay DISPONIBLES para instalar (online)

> wsl                             # abrir la distro por defecto (la del *)
> wsl -d Debian                   # abrir una distro específica por nombre
                                  #   (también: menú inicio → "Ubuntu" → Enter)
$ exit                            # salir de la sesión y volver a PowerShell

> wsl -t Ubuntu                   # -t = terminate: apagar UNA distro ya
> wsl --shutdown                  # apagar TODO el subsistema WSL (todas las distros + la VM)

> wsl --install -d Debian         # instalar OTRA distro (conviven sin problema)
> wsl --set-default Debian        # cambiar cuál es la distro por defecto (mueve el *)

> wsl --unregister Debian         # ⚠️ ELIMINAR una distro. Borra TODO su contenido,
                                  #   sin papelera, sin confirmación. Irreversible.
> wsl --update                    # actualizar el propio WSL
```

> 🕳️ **Madriguera — ¿dónde viven los archivos?**
> Cada distro guarda su mundo en un disco virtual (un archivo grande en tu disco de Windows). Desde adentro de Ubuntu, tus discos de Windows aparecen montados en `/mnt/c`, `/mnt/d`, etc. Alcanza con saber que existe.
> *Volvé al camino.*

## 6. Instalar Docker Desktop — dos caminos, elegí UNO

Con WSL listo, falta la app. Hay dos formas, y **las dos terminan exactamente en el mismo lugar**. Nadie hace las dos:

- **Vía A — la clásica:** ir al sitio oficial, descargar, instalar. La de toda la vida, cero requisitos. Si querés instalar Docker y seguir con tu vida, esta.
- **Vía B — con winget:** instalar con una línea de comando. Si ya conocés winget, andá directo al comando. Si no, la sección 6.2 te lo presenta en un minuto — es el pariente de Windows del Homebrew de los usuarios de Mac.

### 6.1 Vía A — Instalación clásica (descarga directa)

**docker.com** → descarga para Windows. La mayoría de las PCs son **x86_64/AMD64** (elegí ese); si tu notebook es de las nuevas con ARM, elegí ARM64. ¿Dudas? Configuración → Sistema → Información → "Tipo de sistema". Ejecutá el instalador; si te ofrece la opción **"Use WSL 2 instead of Hyper-V"**, dejala tildada. Puede pedir cerrar sesión al final. Seguí en la sección 7.

> 🕳️ **Madriguera — Hyper-V**
> El hipervisor propio de Windows, el camino que Docker usaba antes de WSL 2. Hoy el camino es WSL 2 y no hay que tocar nada de Hyper-V.
> *Volvé al camino.*

### 6.2 Vía B — Instalación con winget

**¿Qué es winget? (30 segundos).** El gestor de paquetes **oficial de Microsoft**: un catálogo desde el que instalás y actualizás programas con una línea de comando, sin peregrinar por sitios web. Viene incluido en Windows 10/11 moderno. A diferencia de Homebrew, no distingue entre programas de consola y aplicaciones con ventana — instala todo por igual, más simple.

**¿Ya lo tengo?** En PowerShell:

```console
> winget --version
v1.x.x               # ← responde con versión = lo tenés
```

Si no responde: se instala/actualiza gratis desde Microsoft Store buscando **"App Installer"**.

**El estilo, en cuatro comandos:**

```console
> winget search docker            # buscar en el catálogo
> winget install Git.Git          # instalar Git, por ejemplo
> winget list                     # qué hay instalado en la máquina
> winget upgrade --all            # actualizar todo lo actualizable
```

> 🕳️ **Madriguera — winget a fondo**
> Fuentes, manifiestos, la Store como backend… nada de eso hace falta acá. Si te dio curiosidad, explorálo en otro chat.
> *Volvé al camino.*

**Instalar Docker Desktop:**

```console
> winget install Docker.DockerDesktop
```

Aceptá lo que pida. Puede requerir cerrar sesión al final.

### 6.3 Primer arranque (común a las dos vías)

Abrí **Docker Desktop**, aceptá los términos, y esperá a que el ícono de la **ballenita** en la bandeja del sistema (abajo a la derecha, quizás dentro del `^`) se quede quieto: animada = arrancando, quieta = lista.

## 7. ⚠️ EL paso que todo el mundo se olvida: la integración con tu Ubuntu

Docker Desktop y tu distro Ubuntu son vecinos, pero hay que presentarlos. Sin este paso, el síntoma es desconcertante: el dashboard anda perfecto, pero adentro de Ubuntu `docker: command not found`.

1. Docker Desktop → engranaje ⚙️ (**Settings**) → **Resources** → **WSL integration**.
2. Tildá **"Enable integration with my default WSL distro"**.
3. Abajo aparece el interruptor por distro: **prendé Ubuntu** (y cualquier otra donde quieras usar Docker).
4. **Apply & Restart**.

## 8. El regalo escondido

Ahora corré de nuevo el diagnóstico en PowerShell:

```console
> wsl -l -v
  NAME              STATE           VERSION
* Ubuntu            Running         2
  docker-desktop    Running         2        # ← ¡apareció una distro nueva que vos no instalaste!
```

Esa `docker-desktop` la creó Docker Desktop para sí mismo: es la burbuja donde vive el motor de Docker — y adentro de ella, tus containers. **No es otra VM**: es una distro más, enchufada al mismo kernel compartido que tu Ubuntu (módulo 2, sección 6.3 — burbujas anidadas, como carpetas). Tu propia máquina te está mostrando en una lista el "Linux escondido" de la serie. (Si además aparece `docker-desktop-data`, es de versiones anteriores — mismo rol, repartido en dos.) ⚠️ No las toques ni las elimines con `--unregister`: son de Docker, él las administra.

## 9. La cuenta (Docker ID) — opcional, y sin misterio

La app te invita a iniciar sesión; podés saltearlo. Si te registrás (con mail o con GitHub), creás un **Docker ID**: una cuenta de **Docker Hub**, el sitio público de donde se descargan imágenes — GitHub, pero de imágenes. Sirve para descargar con límites más generosos y, más adelante, subir imágenes tuyas. **No sincroniza nada local**: si usás el mismo Docker ID en otra máquina (una Mac, por ejemplo), no se cruza absolutamente nada — lo único compartido será lo que vos deliberadamente subas al Hub.

## 10. Verificación — de punta a punta, desde Ubuntu

Abrí la **terminal de Ubuntu** (menú inicio → "Ubuntu"). De acá en adelante, esta es tu casa: todos los comandos de la serie se tiran acá.

**Test 1 — ¿el cliente ve al servidor?**

```console
$ docker version
Client:
 Version:    29.x.x
 OS/Arch:    linux/amd64       # ← fijate: adentro de Ubuntu, el cliente también es linux
 Context:    default

Server: Docker Desktop
 Engine:
  Version:   29.x.x
  OS/Arch:   linux/amd64       # ← el servidor responde: la cadena Ubuntu → Docker funciona
```

⚠️ Si dice **`docker: command not found`** → te salteaste la sección 7 (la integración). Si dice **`Cannot connect to the Docker daemon`** → Docker Desktop está cerrado o todavía arrancando; abrilo, ballenita quieta, reintentá. (El *daemon* es el proceso servidor de Docker; tu comando `docker` es un cliente que le habla.)

**Test 2 — el circuito completo:**

```console
$ docker run hello-world
Unable to find image 'hello-world:latest' locally   # ← no la tenés: la va a buscar a Docker Hub
latest: Pulling from library/hello-world
...
Status: Downloaded newer image for hello-world:latest

Hello from Docker!                                   # ← ✅ ESTA línea es el éxito
This message shows that your installation appears to be working correctly.
...
```

En una línea (la anatomía completa es del módulo 4): descargó una imagen de prueba desde Docker Hub, creó un container con ella, lo ejecutó, el container saludó y terminó. Si ves el saludo, toda la cadena funciona: Ubuntu → Docker → internet → Hub → container.

Corrélo de nuevo: ya no dice `Unable to find image` — la imagen quedó local y no se re-descarga. Primer indicio de algo que va a importar mucho.

**Test 3 — el dashboard cuenta lo mismo:** en Docker Desktop, pestaña **Images**: `hello-world`. Pestaña **Containers**: uno con nombre inventado tipo `brave_hopper` (Docker inventa nombres: adjetivo + científico/a célebre) en estado **Exited**. La app de Windows y tu terminal de Ubuntu ven exactamente lo mismo, porque hablan con el mismo Docker.

## 11. Apagar todo

- **Docker:** ballenita de la bandeja → click derecho → **Quit Docker Desktop**.
- **WSL entero** (si querés silencio absoluto): en PowerShell, `wsl --shutdown`. Todo vuelve a arrancar solo la próxima vez que lo uses.

## ✅ Checklist de salida

- [ ] `wsl -l -v` muestra Ubuntu en VERSION 2
- [ ] Docker Desktop instalado (por la vía que sea — son equivalentes) y la ballenita llega a quedarse quieta
- [ ] La integración con Ubuntu está prendida (Settings → Resources → WSL integration)
- [ ] En la **terminal de Ubuntu**: `docker version` muestra la sección Server
- [ ] En la **terminal de Ubuntu**: `docker run hello-world` imprime el saludo
- [ ] Vi la distro `docker-desktop` aparecer en `wsl -l -v`

Todo tildado → listo. Volvé al módulo que estabas leyendo, o directo al 4 si ya venís con 1-3 hechos.

---

**FIN DEL SETUP — Docker en Windows**
