# 🐳 Clase desde Cero — Docker · Setup en macOS

**Serie:** Clase desde Cero — Docker · Archivo de setup (se hace una sola vez) · El mapa está en `clase-cero-docker-roadmap.md`
**Si tu máquina es Windows:** este no es tu archivo — andá a `clase-cero-docker-setup-windows.md`.

---

### Sobre este documento

**Qué cubre:** instalar Docker Desktop en una Mac por dos vías (elegís una), entender qué acabás de instalar y cómo se prende y se apaga, crear (o no) tu cuenta, y verificar que todo funciona de punta a punta.

**Qué NO cubre:** conceptos. Acá no se explica qué es una imagen ni un container — para eso están los módulos. Este archivo es instrumental: no hay contenido de parcial, así que no vas a ver marcas 🔴🟡🟢 ni bloques 🎓. Solo ⚠️ (trampas) y 🕳️ (madrigueras).

**Cuándo hacerlo:** cuando quieras, pero **obligatorio antes del módulo 4**, que es donde arrancan los comandos. Los módulos 1 a 3 se leen sin nada instalado.

---

## 1. Qué vas a instalar exactamente

Una sola cosa: **Docker Desktop**, una aplicación que trae tres piezas adentro:

```
┌─────────────────────────────────────────────────────┐
│                    TU MAC (macOS)                   │
│                                                     │
│  ┌───────────────┐      ┌─────────────────────────┐ │
│  │  Terminal     │      │  Docker Desktop (app)   │ │
│  │               │      │  · dashboard visual     │ │
│  │  docker ...   │──────┼─▶┌────────────────────┐ │ │
│  │  (el cliente) │      │  │ VM chiquita con    │ │ │
│  └───────────────┘      │  │ LINUX adentro      │ │ │
│                         │  │ ← acá van a vivir  │ │ │
│                         │  │   tus containers   │ │ │
│                         │  └────────────────────┘ │ │
│                         └─────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

¿Por qué una VM con Linux, si el módulo 1 dijo que las VMs eran lo caro? Versión corta: los containers necesitan un kernel de Linux, y macOS no lo tiene — así que Docker Desktop mantiene una VM mínima, una sola y liviana, donde corren *todos* tus containers. La explicación completa de por qué es así es literalmente el módulo 2. Por ahora: es normal, es una sola, y se administra sola.

## 2. Paso 0 — ¿Qué chip tiene tu Mac?

Menú  → **Acerca de esta Mac**. Vas a ver una de dos:

- **"Chip: Apple M1/M2/M3/M4…"** → Apple Silicon (toda Mac de 2021 en adelante, y varias de 2020)
- **"Procesador: Intel…"** → Intel

Guardá el dato: si instalás por descarga directa (vía B), el sitio te va a pedir elegir. Por Homebrew (vía A) no importa — se resuelve solo.

## 3. Vía A — Instalar con Homebrew (si ya usás brew)

- **Homebrew:** el gestor de paquetes más usado en Mac — instalás programas con una línea de comando en vez de andar bajando instaladores. Si no lo tenés y no lo usás, no hace falta instalarlo solo para esto: andá directo a la vía B.

```console
$ brew --version
Homebrew 4.x.x        # ← si responde con un número, tenés brew; si dice "command not found", vía B
```

```console
$ brew install --cask docker-desktop
```

- **Cask:** así llama brew a las aplicaciones con ventana (Chrome, Spotify, Docker Desktop), a diferencia de las fórmulas, que son programas de consola. El `--cask` le dice "quiero la app completa".

⚠️ **Trampa clásica:** `brew install docker` **sin** `--cask` instala *otra cosa* — solamente el cliente de consola, sin la app ni la VM. Te va a dejar un `docker` que no puede conectarse a nada. Si te pasó: `brew uninstall docker` y de nuevo con `--cask`.

⚠️ En guías viejas el comando figura como `brew install --cask docker`. El paquete se renombró; si brew te avisa que el nombre cambió, usá el que te sugiera.

Al terminar: abrí **Docker Desktop** desde Aplicaciones o Spotlight (⌘ + espacio). La primera vez pide aceptar términos y permisos (puede pedir tu contraseña de Mac para instalar componentes privilegiados — es esperado). Seguí en la sección 5.

## 4. Vía B — Instalar por descarga directa

1. Entrá a **docker.com** → botón de descarga para Mac.
2. Elegí según tu chip del paso 0: **Apple Silicon** o **Intel chip**. ⚠️ Si elegís mal, la app no arranca o arranca lenta y rara — no rompe nada, se desinstala y se baja la correcta.
3. Se descarga un `.dmg`. Abrilo y arrastrá la ballena a la carpeta **Applications** — el gesto estándar de instalar apps en Mac.
4. Abrí **Docker Desktop** desde Aplicaciones. Primera vez: aceptar términos, dar permisos, eventualmente tu contraseña de Mac.

## 5. Qué acaba de pasar (y cómo se prende y se apaga)

Cuando la app termina de abrir, la VM de Linux ya está corriendo. El ciclo de vida es este, y es más simple de lo que parece:

- **Abrir la app** → arranca la VM → todo lo Docker disponible.
- **Salir de la app** → la VM se apaga → cero procesos de Docker corriendo.

**Si venís del mundo `brew services`** (por ejemplo, arrancás y parás MySQL con `brew services start/stop mysql`): Docker Desktop **no aparece ahí**, y está bien. `brew services` administra programas de consola que corren de fondo; Docker Desktop se autoadministra — la app *es* el interruptor. No hay servicio que arrancar aparte, ni proceso huérfano que quede vivo cuando salís.

⚠️ **La trampa de Mac: cerrar la ventana no es salir.** El botón rojo (o ⌘W) solo cierra la *ventana* — la app sigue viva y la VM también. Fijate en la **barra de menú** (arriba a la derecha): hay una **ballenita 🐳**. Mientras esté ahí, Docker corre. Para apagar de verdad: click en la ballenita → **Quit Docker Desktop** (o ⌘Q con la app en foco). La ballenita también te habla: animada = arrancando; quieta = lista.

> 🕳️ **Madriguera — "Resource Saver"**
> Abajo a la izquierda del dashboard quizás veas "Resource Saver mode": Docker Desktop reduce solo los recursos de la VM cuando no la usás hace rato. No hay que configurar nada.
> *Volvé al camino.*

## 6. La cuenta (Docker ID) — opcional, y sin misterio

Al abrir el dashboard, la app te invita a iniciar sesión. Podés saltearlo y todo funciona igual. Si te registrás (con mail o con tu cuenta de GitHub — cualquiera de las dos), lo que creás es un **Docker ID**: una cuenta de **Docker Hub**, el sitio público de donde se descargan imágenes. Pensalo como GitHub, pero de imágenes de Docker.

Para qué sirve: descargar con límites más generosos, y más adelante *subir* imágenes tuyas. Qué **no** hace: sincronizar. Tus imágenes, containers y configuración viven en tu máquina y no viajan a ningún lado. Podés usar el mismo Docker ID en todas tus computadoras — como loguearte en GitHub desde dos máquinas — y lo único compartido será lo que vos deliberadamente subas al Hub.

## 7. Verificación — de punta a punta

Con Docker Desktop abierto (ballenita quieta), en una terminal:

**Test 1 — ¿el cliente ve al servidor?**

```console
$ docker version
Client:
 Version:    29.x.x
 OS/Arch:    darwin/arm64      # ← vos: macOS ("darwin"); arm64 en Apple Silicon, amd64 en Intel
 Context:    desktop-linux

Server: Docker Desktop
 Engine:
  Version:   29.x.x
  OS/Arch:   linux/arm64       # ← la línea que importa: del otro lado responde un LINUX
```

Que aparezca la sección **Server** significa que la VM está arriba y respondiendo. Fijate el detalle hermoso: tu cliente es `darwin` (macOS), el servidor es `linux`. Ya estás hablando con el Linux escondido — el módulo 2 te va a contar por qué existe.

⚠️ Si en cambio dice **`Cannot connect to the Docker daemon`**: el servidor no está corriendo — la app está cerrada o todavía arrancando. (El *daemon* es el proceso servidor de Docker que vive en la VM; tu comando `docker` es solo un cliente que le habla.) Abrí Docker Desktop, esperá la ballenita quieta, reintentá.

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

En una línea (la anatomía completa es del módulo 4): descargó una imagen de prueba desde Docker Hub, creó un container a partir de ella dentro de la VM, lo ejecutó, el container imprimió su saludo y terminó. Si ves el saludo, **toda la cadena funciona**: cliente → VM → internet → Hub → container.

Corrélo una segunda vez: ya no aparece el `Unable to find image` — la imagen quedó guardada localmente y no se vuelve a descargar. Primer indicio de algo que va a importar mucho.

**Test 3 — el dashboard cuenta lo mismo:** en la app, pestaña **Images**: `hello-world`, tamaño ínfimo. Pestaña **Containers**: uno (o más, si corriste varias veces) con un nombre gracioso tipo `serene_easley` — Docker inventa nombres (adjetivo + científico/a célebre) cuando no le das uno — y estado **Exited**. Existir sin estar corriendo: eso también es tema del módulo 4.

## 8. Apagar y desinstalar

- **Apagar:** ballenita → Quit Docker Desktop. Nada queda corriendo.
- **Desinstalar del todo** (por si algún día): con brew, `brew uninstall --cask docker-desktop`; a mano, arrastrá la app a la papelera. Antes de eso, dentro de la app: Troubleshoot (🐞) → opciones de limpieza, si querés borrar también imágenes y datos.

## ✅ Checklist de salida

- [ ] Docker Desktop instalado y abre sin errores
- [ ] Sé apagarlo de verdad (ballenita → Quit) y sé que cerrar la ventana no alcanza
- [ ] `docker version` muestra la sección **Server** con `OS/Arch: linux/...`
- [ ] `docker run hello-world` imprime el saludo
- [ ] Veo la imagen y el container en el dashboard

Todo tildado → listo. Volvé al módulo que estabas leyendo, o directo al 4 si ya venís con 1-3 hechos.

---

**FIN DEL SETUP — Docker en macOS**
