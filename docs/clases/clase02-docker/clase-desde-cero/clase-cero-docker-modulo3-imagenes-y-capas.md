# 🐳 Clase desde Cero — Docker · Módulo 3
## La imagen: el disco que viaja con tu aplicación

**Serie:** Clase desde Cero — Docker · Módulo 3 de 7 · El mapa completo está en `clase-cero-docker-roadmap.md`

---

### Sobre este documento

**Qué cubre:** el trío de conceptos que sostiene todo Docker — **imagen, container, registry** — y el desarrollo a fondo del primero: qué es una imagen y qué NO es (no es el mount, no es el container), cómo está construida por dentro (**capas** apiladas + una **ficha técnica**), dónde vive físicamente, la **receta** que la produce (el **Dockerfile**), los **dos momentos** de la vida de una imagen (build y run — separarlos lo cambia todo), las consecuencias de que las capas se congelen, las imágenes base, y la mirada al microscopio.

**Qué NO cubre:** construir y correr la imagen con tus manos — eso abre el módulo 4. Este módulo cierra dos hilos grandes: **H3** (qué es el disco que ve el container) y **H2** (el "en mi máquina funciona" del módulo 1). Se puede leer entero sin tener Docker instalado; si ya hiciste el setup, los comandos de la sección 7 son probables ahí mismo.

### De dónde venís

Del módulo 2 traés: el container es un proceso común envuelto en una **burbuja** (namespaces + cgroups) · el kernel es compartido · el namespace de mount le muestra al container "su propio disco" — y quedó abierta la pregunta de qué hay en ese disco · y la distinción kernel / sistema operativo / **distro** (el equipamiento alrededor del motor), que acá vuelve con rol estelar.

---

## 1. 🔴 El trío: imagen, container, registry

Todo Docker se para sobre tres conceptos, y diferenciarlos bien es lo primero que se espera de cualquiera que diga "sé Docker":

| Concepto | Qué es | Analogía | Se desarrolla en |
|---|---|---|---|
| **Imagen** | El **paquete**: una foto de solo lectura del file system completo que tu app necesita — código, runtime, librerías, herramientas | El molde / la clase | **Este módulo** |
| **Container** | La **instancia en ejecución** de una imagen: un proceso vivo con su burbuja, parado sobre ese file system | Lo horneado / el objeto | Módulo 4 |
| **Registry** | El **repositorio** donde las imágenes se publican y desde donde se descargan (el público por defecto: Docker Hub) | GitHub, pero de imágenes | Módulo 7 |

Si venís de la programación orientada a objetos, la primera analogía te va a rendir muchísimo: **la imagen es la clase, el container es el objeto**. De una imagen podés instanciar uno, cinco o cien containers — todos idénticos al nacer, cada uno con su vida propia después. Y así como una clase no cambia porque sus objetos cambien, la imagen no cambia jamás por lo que hagan sus containers.

> 🎓 **Para el parcial, si te preguntan**
> **Diferenciá imagen, container y registry.** La imagen es un template de solo lectura del file system que la aplicación necesita (código, runtime, librerías, configuración), construido en capas. El container es la instancia en runtime de una imagen: un proceso aislado con namespaces y cgroups que ve ese file system como propio. El registry es el repositorio donde se almacenan y distribuyen imágenes (Docker Hub es el público por defecto). Relación: de una imagen se instancian N containers; las imágenes se comparten vía registries.

## 2. 🔴 Qué es una imagen — y el cierre del hilo H3

El módulo 2 te dejó mirando una puerta cerrada: el namespace de mount le muestra al container "su propio disco", con carpetas de un Ubuntu que tu máquina no tiene. ¿Qué es ese disco? ¿Quién lo llenó?

Respuesta: **ese disco es la imagen.** Una imagen es un file system completo, empaquetado y de solo lectura — todos los archivos que tu app necesita para vivir, listos y en su lugar. Cuando Docker lanza un container, el namespace de mount le muestra al proceso, como si fuera "el disco de la máquina", el contenido de la imagen. El container no está *dentro* de otra computadora — está **mirando un disco de utilería**, montado por el kernel para él.

Y como acá conviven tres cosas parecidas que NO son lo mismo, separémoslas de una vez:

| Pieza | Qué es | Su rol |
|---|---|---|
| **La imagen** | Contenido: la pila de archivos, guardada en disco | **QUÉ** se ve |
| **El namespace de mount** | Mecanismo del kernel: decide qué árbol de archivos ve un proceso | **La ventana** por la que se ve |
| **"El disco del container"** | La ilusión resultante: lo que el proceso encuentra al mirar por su ventana | **Lo visto** |

La analogía para colgar en la pared: **el cine**. La imagen es **el rollo de película** — existe en un depósito, es uno solo, nadie lo modifica. El namespace de mount es **el proyector de tu sala** — apunta el contenido a tus ojos, y solo a los tuyos. "El disco del container" es **lo que ves en pantalla**. Dos containers de la misma imagen son dos salas proyectando el mismo rollo: cada una ve la película completa, el rollo sigue siendo uno, y nadie raya el original.

Y ahora atá el cabo con el módulo 2, porque ya conocés este truco: una imagen de "Ubuntu" es **Ubuntu sin el kernel — todo el equipamiento, sin el motor**: las carpetas, el `apt`, las herramientas, las librerías… y ningún kernel, porque el kernel lo pone el host. ¿Te suena? Es *exactamente* el mismo formato que las distros de WSL (módulo 2, §6.3): el equipamiento empaquetado, listo para enchufarse a un kernel ajeno. La industria entera funciona empaquetando "Linux sin kernel" — WSL lo hace con distros enteras, Docker lo hace con imágenes. Por eso un container "parece un Ubuntu": tiene los *archivos* de Ubuntu. El motor es prestado.

**Hilo H3: cerrado.** El disco del container es la imagen, mostrada por el namespace de mount.

Dos propiedades de la imagen que van a gobernar todo lo que sigue:

1. **Es de solo lectura.** Nadie escribe sobre una imagen, nunca. (¿Y entonces cómo hace la app para escribir un archivo? Excelente pregunta — es el gancho del módulo 4. Aguantala.)
2. **Es inmutable: no se edita, se construye otra.** ¿Querés cambiar algo? Construís una imagen nueva. Las imágenes se versionan como el código — de hecho, vas a ver que se *construyen desde* código.

## 3. 🔴 Las capas: cómo está armado el paquete por dentro

Acá viene la decisión de diseño más importante de Docker, la que explica su eficiencia y también sus trampas. Una imagen **no es un archivo gigante**: es una **pila de capas**, apiladas una sobre otra, donde cada capa aporta archivos y el conjunto se ve como un único file system.

Antes de la primera capa, la palabra que va a aparecer mil veces: **congelar**. En esta serie, congelar una capa significa **sellarla como solo-lectura para siempre**: la capa se llena de archivos una única vez y, desde ese momento, nadie la modifica jamás — ni Docker, ni vos, ni el container. Todo lo que veas de acá en adelante se apoya en ese sellado.

```
        LA IMAGEN, POR DENTRO — una pila de capas congeladas
       ┌─────────────────────────────────────────────┐
   ④   │  capa: app.py                    (2 kB)     │  ← tu código
       ├─────────────────────────────────────────────┤
   ③   │  capa: python3 + pip instalados  (140 MB)   │  ← el runtime
       ├─────────────────────────────────────────────┤
   ②   │  capa: actualizaciones del sistema (30 MB)  │
       ├─────────────────────────────────────────────┤
   ①   │  capa: base ubuntu:22.04         (78 MB)    │  ← el equipamiento de la distro
       └─────────────────────────────────────────────┘
              ▲
              │  el container mira desde arriba y ve UN file system:
              │  la SUMA de todas las capas superpuestas
```

- **Overlay** ("superposición"): la técnica de file system que logra ese efecto — capas transparentes apiladas, como filminas: mirás desde arriba y ves la combinación de todas. Cuando el container busca un archivo, el sistema lo busca **de la capa más alta hacia abajo** y usa la primera versión que encuentra; si no está en ninguna, no existe.

¿Y cómo se identifica cada capa? Con un **hash**:

- **Hash:** una huella digital calculada a partir del contenido — un código tipo `a3f2c884b1...`. Mismo contenido → mismo hash, siempre; un byte de diferencia → hash completamente distinto. El hash no es un nombre que alguien eligió: es la **identidad matemática del contenido**.

Esto convierte a las capas en piezas **reutilizables y compartibles**: si dos imágenes usan la misma capa base de `ubuntu:22.04` (mismo hash), tu máquina la guarda y la descarga **una sola vez**. Cien imágenes paradas sobre Ubuntu no cuestan cien Ubuntus — cuestan uno más cien diferencias. Guardá el concepto de hash con cariño: en el módulo 5 va a ser el protagonista de la caché de build (hilo H4, que se abre acá).

**¿Y dónde viven las capas? En disco — no en memoria.** Convención de la serie desde acá: cuando algo ocupa espacio, decimos dónde. Las capas, las imágenes y todo lo que Docker guarda viven en una carpeta del Linux (`/var/lib/docker`). Ahora, ¿dónde está *ese* disco según tu máquina?

| Tu sistema | ¿Dónde vive `/var/lib/docker`? | ¿Y eso dónde está físicamente? |
|---|---|---|
| **Linux nativo** | En tu disco, directamente | Tu disco físico — una carpeta más |
| **Mac** | En el disco **de la VM** de Docker Desktop | La VM guarda su disco como **un archivo grande** en tu disco físico |
| **Windows** | En el disco **de la distro** `docker-desktop` | Cada distro de WSL guarda su mundo como un archivo grande en tu disco físico |

O sea: "el disco de la VM" no es humo — es un archivo real ocupando gigas reales de tu SSD. Cuando bajás una imagen de 500 MB, tu disco físico pierde 500 MB de verdad, solo que adentro de ese archivo-valija de la VM. La RAM se usa como siempre: solo lo que los procesos cargan al trabajar. En el módulo 5 llega el comando para ver cuánto te está comiendo Docker — y el de limpiar.

## 4. 🔴 El Dockerfile: la receta que construye la imagen

¿Y de dónde salen las capas? De una **receta escrita en un archivo de texto** llamado `Dockerfile` (así, literal, sin extensión). Cada línea es una **instrucción**, y hay **dos clases** — distinguirlas de entrada evita el mareo después:

- Instrucciones que **tocan archivos** (`RUN`, `COPY`): cada una **congela una capa** con lo que cambió.
- Instrucciones que **anotan** (`LABEL`, `WORKDIR`, `EXPOSE`, `CMD`, `ENV`… y `FROM`, que es caso aparte): no crean capas — escriben un renglón en la **ficha técnica** de la imagen.

Porque — y esto es estructura real, no dibujo didáctico — una imagen completa son **dos partes**:

```
   LA IMAGEN, completa
   ┌─────────────────────────────────────┐
   │  LA PILA DE CAPAS (los archivos)    │ ← la construyen RUN y COPY
   │  ④ app.py                           │   (instrucciones que TOCAN archivos)
   │  ③ python instalado                 │
   │  ①② base ubuntu                     │
   ├─────────────────────────────────────┤
   │  LA FICHA TÉCNICA (metadata)        │ ← la escriben CMD, EXPOSE,
   │  · comando de arranque: python3...  │   WORKDIR, LABEL, ENV...
   │  · puerto documentado: 8080         │   (instrucciones que ANOTAN)
   │  · carpeta de trabajo: /app         │
   └─────────────────────────────────────┘
```

Existe incluso un comando — `docker inspect` — que te muestra la ficha técnica tal cual, en formato JSON: el renglón del CMD, el del WORKDIR, y la lista de hashes de las capas de la pila. El diagrama es el plano del edificio, no una simplificación.

Ahora sí, la receta. Sigamos con la app de la serie — primero la versión Python del servidor del módulo 1 (mismo espíritu, otro lenguaje; en la sección 4.2 está el equivalente en Node, y vas a ver que la receta casi ni se entera del lenguaje):

```python
# app.py — un servidor HTTP mínimo en Python (el primo del server.js del módulo 1)
from http.server import HTTPServer, BaseHTTPRequestHandler

class Saludo(BaseHTTPRequestHandler):
    def do_GET(self):                                  # esta función atiende cada GET que llega
        self.send_response(200)                        # respondemos "200 OK"
        self.end_headers()
        self.wfile.write("Hola desde Docker!".encode())  # el cuerpo de la respuesta

HTTPServer(("", 8080), Saludo).serve_forever()         # escuchar en el 8080, para siempre
```

Y la receta que empaqueta esa app **con todo su mundo**:

```dockerfile
# Dockerfile — la receta, línea por línea

FROM ubuntu:22.04
# └─ TODA receta arranca con FROM: el punto de partida. No es una capa nueva:
#    es un PUNTERO a otra imagen ya existente (con sus propias capas adentro).
#    Acá: el equipamiento de Ubuntu 22.04. Sobre eso construimos lo nuestro.

LABEL maintainer="equipo-tacs"
# └─ ANOTA en la ficha: una etiqueta informativa (quién mantiene esto).

RUN apt-get update && apt-get install -y python3 python3-pip && rm -rf /var/lib/apt/lists/*
# └─ RUN ejecuta un comando ADENTRO de la imagen en construcción y, al terminar,
#    CONGELA el resultado como capa. Acá pasan tres cosas encadenadas con &&:
#    1) apt-get update  → OJO: no actualiza programas. Descarga el CATÁLOGO de la
#       "tienda" de paquetes de Ubuntu (qué paquetes existen, en qué versión, de
#       dónde bajarlos) y lo guarda en la carpeta /var/lib/apt/lists/.
#       Sin catálogo, apt no sabe qué es "python3" ni dónde conseguirlo.
#    2) apt-get install → usa el catálogo para bajar e instalar Python de verdad.
#    3) rm -rf /var/lib/apt/lists/* → borra el catálogo: una vez instalado Python,
#       son decenas de MB de listas que tu app jamás va a leer. Peso muerto.
#    ¿Por qué las tres JUNTAS en una instrucción? Sección 5 — es una lección entera.

WORKDIR /app
# └─ ANOTA en la ficha: "de acá en adelante, parate en la carpeta /app"
#    (la crea si no existe). Afecta a las instrucciones siguientes y al container.

COPY app.py .
# └─ COPY lleva archivos DESDE tu carpeta del proyecto HACIA la imagen.
#    Acá: app.py a /app (el "." es el WORKDIR actual). CONGELA una capa
#    que contiene el archivo entero.

EXPOSE 8080
# └─ ANOTA en la ficha: "esta app escucha en el 8080". ⚠️ No abre ni publica
#    nada: es un cartel informativo. La conexión real del puerto: módulo 4.

CMD ["python3", "app.py"]
# └─ ANOTA en la ficha el comando de arranque: el comando `python3 app.py`
#    escrito como lista de palabras (el programa, y su argumento).
#    NO se ejecuta ahora: queda anotado para cuando un container nazca de esta
#    imagen. Ese comando, al ejecutarse, será el famoso PID 1 del módulo 2.
#    Por qué el formato de lista y qué pasa al arrancar: módulo 4.
```

La correspondencia receta → paquete, para retener de un vistazo:

```
   Dockerfile (receta)                    Imagen (resultado)
   ─────────────────────                  ─────────────────────────
   CMD ["python3","app.py"]  ──anota──▶   ficha: comando de arranque
   EXPOSE 8080               ──anota──▶   ficha: puerto documentado
   COPY app.py .             ──congela─▶  pila: ④ capa con app.py
   WORKDIR /app              ──anota──▶   ficha: carpeta de trabajo
   RUN apt-get ...           ──congela─▶  pila: ③ capa con python3
   LABEL maintainer=...      ──anota──▶   ficha: etiqueta
   FROM ubuntu:22.04         ──apunta──▶  pila: ①② capas de la base
```

### 4.1 🔴 Los dos momentos: build y run (la distinción que ordena todo)

Pregunta trampa que conviene hacerse ahora: ¿**cuándo** se ejecuta el Dockerfile? ¿Cada vez que corrés la app? No — y separar los dos momentos es lo que le da sentido a todo el sistema:

```
  MOMENTO 1 — BUILD (una sola vez)           MOMENTO 2 — RUN (N veces)
  «docker build»                              «docker run»
  ┌─────────────────────────────┐            ┌───────────────────────────┐
  │ acá SÍ se ejecuta el        │            │ acá NO se ejecuta NADA    │
  │ Dockerfile, línea por línea:│            │ del Dockerfile:           │
  │  RUN apt-get ▶ corre, y AL  │   imagen   │ · toma la pila congelada  │
  │  TERMINAR congela capa ③    │  ───────▶  │ · lee la ficha técnica    │
  │  COPY app.py ▶ congela ④    │  (lista)   │ · lanza el CMD → PID 1    │
  │  CMD ...     ▶ solo ANOTA   │            │                           │
  └─────────────────────────────┘            └───────────────────────────┘
     acá se COCINA (tarda minutos)              acá solo se SIRVE (miliseg.)
```

El **build** cocina la receta **una vez**, y su resultado es la imagen (pila + ficha), lista y congelada. El **run** no re-cocina nada: agarra la pila, la muestra por la ventana del mount, y ejecuta el comando anotado. Punto. Recién ahora cierra del todo la promesa del módulo 2: el container arranca en **milisegundos** porque al momento de correr *no se instala nada* — todo se instaló en el build. Si el Dockerfile se ejecutara en cada run, arrancar tardaría lo que tarda `apt-get install`, y el edificio entero se caería.

Y adentro del build, el orden es **estrictamente secuencial**: las instrucciones corren una por una, y la siguiente **no arranca hasta que la anterior terminó por completo** — el `apt-get` corre, instala todo, el proceso finaliza, la capa se congela, y recién entonces empieza la línea que sigue. No existe el escenario "Python instalándose mientras la próxima línea ya pide dependencias". Si venís de JavaScript: es como si cada línea tuviera un `await` implícito — nada queda colgando. Por eso la línea N+1 puede confiar ciegamente en que existe todo lo de la línea N.

### 4.2 🟡 La receta en otros lenguajes — y una regla de oro

Existe un atajo para la receta de arriba: arrancar de una imagen que **ya trae** el runtime:

```dockerfile
FROM python:3.11-slim      # Python ya instalado sobre un Debian recortado
WORKDIR /app
COPY app.py .
EXPOSE 8080
CMD ["python3", "app.py"]
```

Misma app, cinco líneas. Esta versión corta es la que vas a construir con tus manos en el módulo 4.

**¿Y en Node?** La misma película con otros nombres — fijate qué poco cambia:

```dockerfile
FROM node:22-slim          # Node y npm ya instalados
WORKDIR /app
COPY package*.json .       # primero el "qué necesito" (package.json y su lock)
RUN npm ci                 # instala las dependencias → congela SU propia capa
COPY . .                   # después, el código
EXPOSE 8080
CMD ["node", "server.js"]
```

(Y con Java/Spring es la misma historia con Maven o Gradle.) Dos cosas para notar: hay **varios** `RUN`/`COPY` — varias capas — y está perfecto: el encadenado con `&&` del ejemplo anterior era por un motivo puntual (la limpieza, sección 5), no una regla de "todo en una línea". Y el orden `package.json` primero / código después no es capricho: es la jugada maestra del módulo 5.

> ⚠️ **Regla de oro: las dependencias se hornean en el build — JAMÁS se instalan al arrancar el container.** La tentación existe: "el `node_modules` pesa, ¿y si la imagen viaja liviana y cada container hace `npm install` al arrancar?". No. Tres razones: (1) el arranque pasa de milisegundos a minutos — muere la promesa del módulo 2; (2) el container pasa a necesitar internet y un npm funcionando *para poder existir* — un servidor sin salida a la red no arranca tu app; (3) cada container instala "lo que haya disponible ese día" → dos containers de la misma imagen pueden terminar distintos… y "en mi máquina funciona" vuelve por la ventana. La imagen ES la garantía de identidad; vaciarla de dependencias es vaciarla de sentido. El peso se maneja de otra forma: instalando solo dependencias de producción, y con la reutilización de capas — la capa de dependencias se congela una vez y solo se reconstruye cuando cambia el `package.json` (módulo 5).

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué es un Dockerfile y qué relación tiene con la imagen?** Es el archivo de texto con la receta para construir una imagen: una secuencia de instrucciones que Docker ejecuta en orden, una por una, durante el build (una sola vez — nunca al correr el container). Las instrucciones que modifican archivos (RUN, COPY) congelan cada una una capa read-only; las demás (LABEL, WORKDIR, EXPOSE, CMD) anotan metadata en la ficha de la imagen. FROM apunta a la imagen base sobre la que se apila todo. La imagen resultante = pila de capas + metadata, y cada capa queda identificada por el hash de su contenido.

## 5. 🔴 Las consecuencias de congelar

Las capas se congelan **en el momento en que su instrucción termina**. Eso tiene tres consecuencias que separan a quien usa Docker de quien lo entiende — y la cátedra evalúa exactamente esta diferencia en el TP.

**Consecuencia 1: por eso el `RUN` triple encadenado.** Ya sabés qué hace cada pieza (el catálogo, la instalación, la limpieza — está en los comentarios de la receta). Ahora el porqué del encadenado: si la limpieza (`rm -rf /var/lib/apt/lists/*`) fuera una instrucción `RUN` **aparte**, llegaría tarde — la capa anterior ya se congeló *con el catálogo adentro*. La capa de la limpieza solo puede taparlo, no destaparlo, y esos MB viajan igual, inflando la imagen para siempre. Encadenado con `&&`, todo ocurre dentro de **una** capa: el catálogo nace y muere antes del congelamiento, y la capa queda limpia.

El criterio general, para que no te quede "todo va en una línea": **agrupá lo que forma una unidad** (instalar + limpiar SU basura), **separá lo que cambia a ritmos distintos** (las dependencias por un lado, tu código por otro — como en la receta de Node). Qué significa "ritmos distintos" y en qué orden conviene: módulo 5, entero.

🟡 **¿Y la basura de las otras herramientas?** Cada gestor tiene su propia papelera: `apt` deja el catálogo en `/var/lib/apt/lists`, `npm` guarda cachés de descarga en una carpeta propia, `pip` en otra, Maven en otra. No hay basurero universal — pero tampoco hace falta memorizarlos: para cada stack existe un **patrón estándar de Dockerfile** que ya incluye su limpieza, y los Dockerfiles de las imágenes oficiales son la chuleta pública. Lo que sí hay que tener claro es la distinción: **basura = lo que no hace falta para correr** (catálogos, cachés de descarga) → se limpia en su misma capa; **dependencia = lo que sí hace falta** (Python, tu `node_modules`) → se queda adentro, por la regla de oro de la sección 4.2.

⚠️ El encadenamiento tiene un precio diferido que te va a morder en el módulo 4: como el catálogo se borró, si algún día entrás a un container de esta imagen y tirás `apt-get install loquesea`, va a fallar — hasta que corras `apt-get update` de nuevo (bajar el catálogo otra vez). No es un bug: es la consecuencia exacta de la limpieza. Cuando te pase, vas a saber por qué.

**Consecuencia 2: borrar no achica.** Generalizando: **eliminar archivos en una capa superior no reduce el tamaño de la imagen.** El archivo sigue existiendo en la capa de abajo (congelada); la capa de arriba solo lo *marca como borrado* y lo esconde de la vista. La imagen pesa la suma de todas sus capas, incluyendo lo que ya nadie ve. La versión en vivo de este fenómeno — qué pasa cuando un *container* borra archivos de la imagen — es plato fuerte del módulo 5.

**Consecuencia 3: la imagen no se edita — se reconstruye.** ¿Cambiaste una línea de `app.py`? La imagen vieja no se "actualiza": construís una nueva, con sus capas nuevas donde corresponda. ¿Y hay que re-cocinar *todo* cada vez, las capas pesadas incluidas? Ahí Docker tiene un as bajo la manga que involucra a los hashes de la sección 3… y es exactamente el hilo H4. Módulo 5.

## 6. 🟡 Imágenes base: las distros vuelven, ahora como mercadería

Toda receta arranca con `FROM`, así que toda imagen desciende de alguna **imagen base**. ¿Cuáles existen? Las distros del módulo 2, empaquetadas:

| Imagen base | Qué trae | Peso aprox.* | Cuándo usarla |
|---|---|---|---|
| `ubuntu` | El equipamiento completo de Ubuntu: `apt`, herramientas estándar | ≈ 78 MB | Familiaridad total; sobra de todo |
| `debian` | El padre de Ubuntu; equipamiento clásico y estable | ≈ 120 MB (o ≈ 75 MB en variante `slim`) | Base tradicional de muchas imágenes oficiales |
| `alpine` | Una distro **mínima**, pensada para containers; su propio gestor (`apk`) | ≈ **5-8 MB** | Cuando el peso importa (casi siempre, a escala) |
| `busybox` | Lo mínimo de lo mínimo: un puñado de herramientas y nada más | ≈ 1-4 MB | Casos extremos y pruebas |

*\* Los pesos varían según versión y arquitectura de tu máquina — tomalos como órdenes de magnitud. La diferencia Ubuntu vs Alpine (¡diez veces!) es real y es la que importa: multiplicá por cada versión de cada imagen de cada servicio que viaja por la red de una empresa, y entendés por qué existe Alpine.*

También existen bases **con el runtime ya puesto** — `python:3.11-slim`, `node:22-slim`, etc. — mantenidas oficialmente: la distro más el lenguaje, listos. El sufijo `slim` significa "recortada: lo esencial para correr, sin extras". Sobre el nombre completo (`python:3.11-slim` — ¿qué es cada parte, qué pasa si no ponés nada después de los dos puntos?): eso es el sistema de **tags** y tiene su desarrollo en el módulo 7. Por ahora: lo que va después de `:` elige la versión.

🟡 **Y una práctica de equipos grandes que la cátedra mira con cariño:** si tu equipo tiene cinco servicios Python, en vez de que cada Dockerfile repita la misma instalación, se construye **una imagen base propia** (`python-tacs:1.0`, con el runtime, las herramientas y los estándares del equipo) y los cinco servicios hacen `FROM python-tacs:1.0`. Una sola fuente de verdad, versionada, con ciclo de vida propio. Para el TP de la materia — donde las buenas prácticas de imágenes se evalúan explícitamente — tenerlo en el radar suma.

## 7. 🟡 La imagen bajo el microscopio

Tres comandos para *ver* todo lo anterior con tus ojos. Como siempre: las salidas están acá, leélas — y si ya hiciste el setup, probalos (la práctica en serio arranca en el módulo 4). Las salidas van en el formato clásico, el que vas a encontrar en la mayoría de la documentación y de las máquinas; al final hay un recuadro sobre las variantes modernas.

**Descargar una imagen — y ver las capas viajar:**

```console
$ docker pull python:3.11-slim
3.11-slim: Pulling from library/python
8c773fdc99b6: Pull complete      # ← ¡MIRÁ! No baja "un archivo": baja CAPA por CAPA
baef1f7c3bab: Pull complete      #    cada línea es una capa, identificada por su hash
bf7af0229701: Pull complete
715211e58574: Pull complete
cedb5d0a039e: Pull complete
Digest: sha256:1042b61448...     # ← el hash de la imagen completa
Status: Downloaded newer image for python:3.11-slim
```

Cada línea `Pull complete` = una capa. La pila de la sección 3, cruzando la red pieza por pieza. Y si otra imagen compartiera alguna de esas capas, esa línea diría `Already exists` — la reutilización por hash, en vivo.

**Listar lo que tenés — y comparar pesos:**

```console
$ docker image ls
REPOSITORY    TAG         IMAGE ID       CREATED        SIZE
python        3.11-slim   1042b61448fe   4 days ago     ≈215MB    # ← Debian recortado + Python
ubuntu        22.04       53a843653cbc   5 weeks ago    ≈78MB
alpine        latest      a8cbb8c69ee7   6 weeks ago    ≈8MB      # ← 10 veces menos que Ubuntu
hello-world   latest      5dd0d3e6e255   5 months ago   ≈20kB     # ← la del setup: casi nada
```

(La tabla muestra `ubuntu` y `alpine` para que compares pesos — no hace falta que las descargues: la comparación es para leer.)

⚠️ Detalle que confunde a todos la primera vez: `CREATED` **no** es cuándo la descargaste vos — es cuándo sus autores la **construyeron**. Las imágenes viajan con su fecha de nacimiento.

**Ver la pila de una imagen — la receta al revés:**

```console
$ docker history python:3.11-slim
CREATED BY                                      SIZE      # (salida recortada a lo que importa)
CMD ["python3"]                                 0B        # ← arriba: lo ÚLTIMO de la receta
RUN ... instalación de python ...               ≈52MB
RUN ... dependencias del sistema ...            ≈5MB
ENV PYTHON_VERSION=3.11.16                      0B        # ← anota en la ficha: 0 bytes
ENV PATH=/usr/local/bin:...                     0B
ADD file:... in /                               ≈109MB    # ← abajo: la base Debian
```

`docker history` te muestra los pasos de la receta **de arriba hacia abajo** — o sea, la receta *al revés*: la primera línea es la última instrucción. Y ahí están, con tus propios ojos, las dos clases de instrucción de la sección 4: las que congelaron capas con peso (RUN, ADD/COPY) y las que anotaron en la ficha (CMD, ENV… — **0B**, porque un renglón de metadata no pesa). Toda imagen del mundo, incluidas las oficiales, es esto: una receta congelada que podés auditar.

> ⚠️ **Tu salida puede verse distinta — y está bien.** El formato exacto varía según la versión de Docker (y con alternativas como Colima): columnas que cambian de nombre, información de más o de menos. En versiones recientes de Docker Desktop, por ejemplo, `docker image ls` se ve así:
>
> ```console
> IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
> hello-world:latest   5dd0d3e6e255   22.6kB       10.3kB         U
> python:3.11-slim     1042b61448fe   215MB        48.1MB
> ```
>
> Novedades descifradas: **DISK USAGE** es lo que ocupa en tu disco (descomprimida) y **CONTENT SIZE** lo que pesó viajando por la red (comprimida) — el `SIZE` clásico mostraba solo lo primero. La **U** viene con su leyenda en la misma pantalla (`U | In Use`): la imagen tiene al menos un container nacido de ella. Y en `docker history` vas a ver **`<missing>`** en la columna de IDs: significa "faltante" y **no es un error** — en imágenes descargadas, los IDs de los pasos intermedios quedaron en la máquina de quien la construyó y no viajan; solo el ID de la imagen final existe en la tuya. La esencia — capas, hashes, pesos, 0B en la metadata — es idéntica en todos los formatos.

## 8. 🔴 El cierre del hilo H2: "en mi máquina funciona", asesinado

Volvé al diagrama con el que arrancó la serie — lo que tu app es de verdad — y miralo ahora:

```
   MÓDULO 1: el problema                    MÓDULO 3: la solución
   ┌────────────────────────────┐          ┌─────────────────────────────┐
   │ tu código                  │          │        LA IMAGEN            │
   │ + el runtime (¿versión?)   │          │  ┌───────────────────────┐  │
   │ + librerías instaladas     │   ───▶   │  │ código  ✓ adentro     │  │
   │ + librerías del sistema    │          │  │ runtime ✓ adentro,    │  │
   │ + herramientas             │          │  │   en SU versión exacta│  │
   │ + archivos de config       │          │  │ librerías ✓ adentro   │  │
   │ + variables de entorno     │          │  │ herramientas ✓ adentro│  │
   │   ...cada uno distinto     │          │  └───────────────────────┘  │
   │   en cada máquina 💥       │          │  UN paquete, UN hash,       │
   └────────────────────────────┘          │  IDÉNTICO en todos lados    │
                                           └─────────────────────────────┘
                                            viaja idéntica a las 7
                                            máquinas y al servidor
```

La lista entera de dependencias del módulo 1 — el runtime en su versión exacta, las librerías, las herramientas — vive ahora **adentro del paquete**. Tus seis compañeros no instalan nada de eso: descargan la imagen (por su hash, idéntica al byte) y la corren. El servidor de la nube: lo mismo. "En mi máquina funciona" muere porque **la máquina — el equipamiento completo — viaja con la app**.

Con un asterisco honesto que planta el módulo 6: *casi* todo viaja adentro. Lo que cambia entre ambientes a propósito — la contraseña de la base de datos, la URL del servicio de clima, las claves — **no** se hornea dentro de la imagen (¡viajaría a todos lados, incluida la imagen que compartís!). Eso entra desde afuera, al momento de correr. Cómo, en el módulo 6.

**Hilo H2: cerrado.** El más viejo de la serie.

## 9. 🔴 Síntesis — y la puerta del módulo 4

Lo conquistado hasta acá, sumando módulos:

| Del deseo del módulo 1… | Estado | Mecanismo |
|---|---|---|
| Aislamiento | ✅ módulo 2 | Burbuja: namespaces + cgroups |
| Eficiencia y arranque instantáneo | ✅ módulo 2 | Proceso común, kernel compartido |
| "Corre igual en cualquier máquina" | ✅ **este módulo** | La imagen: el file system viaja con la app |

La tercera columna del módulo 1 está completa **en los papeles**. Falta hacerla vivir: cocinar la receta con `docker build` y servirla con `docker run` — el container, con tus manos. Y apenas nazca, va a chocar con la pregunta que quedó picando en la sección 2: la imagen es de solo lectura… **¿y cuando la app escribe?** ¿Dónde cae el archivo que tu servidor loguea, el temporal que crea, el dato que guarda? La respuesta es una capa que todavía no conocés — la única con permiso de escritura en todo el edificio — y es la protagonista del módulo 4.

---

## ✅ Checkpoint del Módulo 3

*Sin mirar el material. Las que no salgan marcan qué sección releer. (Las respuestas no están acá a propósito.)*

1. Definí imagen, container y registry, y la relación entre los tres.
2. Imagen, namespace de mount y "el disco del container": ¿cuál es el rollo, cuál el proyector y cuál la pantalla? ¿Por qué no son lo mismo?
3. ¿Por qué un container "parece un Ubuntu" sin traer sistema operativo? ¿En qué se parece una imagen de Ubuntu a una distro de WSL?
4. ¿Qué significa "congelar" una capa, y en qué momento exacto ocurre?
5. ¿Dónde viven las capas — memoria o disco? En Mac y Windows, ¿dónde termina ese disco físicamente?
6. ¿Qué es un hash y por qué usar hashes como identidad de capa permite compartir y reutilizar capas entre imágenes?
7. Las dos clases de instrucción del Dockerfile: ¿cuáles congelan capas y cuáles anotan en la ficha? ¿Qué dos partes tiene una imagen completa?
8. ¿Cuándo se ejecuta el Dockerfile — en el build, en el run, o en ambos? ¿Qué pasa (y qué NO pasa) en `docker run`? ¿Por qué eso explica el arranque en milisegundos?
9. Dentro del build, ¿las instrucciones corren en paralelo o en secuencia? ¿Puede la línea N+1 asumir que existe todo lo de la línea N?
10. ¿Qué descarga exactamente `apt-get update`, dónde lo guarda, y por qué la limpieza tiene que ir encadenada en el MISMO `RUN`? ¿Cuál es el criterio general para agrupar o separar instrucciones?
11. ¿Qué distingue "basura" de "dependencia"? ¿Las dependencias (`node_modules`, paquetes de pip) van adentro de la imagen o se instalan al arrancar el container? Dá las tres razones.
12. ¿Por qué borrar archivos en una capa superior no achica la imagen?
13. ¿Qué diferencia hay entre `ubuntu` y `alpine` como bases, y por qué esa diferencia le importa (en plata) a una empresa? ¿Qué problema resuelve una imagen base propia de equipo?
14. En `docker history`, ¿en qué orden aparecen los pasos respecto del Dockerfile? ¿Qué significan las líneas de 0B? ¿Y el `<missing>` en imágenes descargadas?
15. Explicá con la imagen en la mano por qué "en mi máquina funciona" deja de ser un problema. ¿Qué es lo único que deliberadamente NO viaja adentro de la imagen?

---

## Qué viene en el Módulo 4

Se acabó la teoría pura: el módulo 4 es consola. Vas a cocinar la receta de este módulo con `docker build`, parir containers con `docker run`, verlos vivir y morir (`ps`, `logs`, `stop`, `rm`), meterte adentro de uno con `exec` y comprobar con tus ojos la doble identidad del PID 1 del módulo 2. Y vas a conocer a la única capa con permiso de escritura — la **capa read-write** — que responde la pregunta que este módulo dejó abierta y abre, ella misma, el problema más traicionero de todos: dónde están tus datos. (Spoiler: en peligro.)

**FIN DEL MÓDULO 3**
