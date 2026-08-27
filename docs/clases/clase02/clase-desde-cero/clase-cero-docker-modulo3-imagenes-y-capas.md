# 🐳 Clase desde Cero — Docker · Módulo 3
## La imagen: el disco que viaja con tu aplicación

**Serie:** Clase desde Cero — Docker · Módulo 3 de 7 · El mapa completo está en `clase-cero-docker-roadmap.md`

---

### Sobre este documento

**Qué cubre:** el trío de conceptos que sostiene todo Docker — **imagen, container, registry** — y el desarrollo a fondo del primero: qué es una imagen, cómo está construida por dentro (**capas** apiladas, identificadas por **hashes**), la **receta** que la produce (el **Dockerfile**, instrucción por instrucción), las consecuencias de que las capas se congelen, las imágenes base (acá vuelven las distros del módulo 2), y la mirada al microscopio con los comandos de inspección.

**Qué NO cubre:** construir y correr la imagen con tus manos — eso abre el módulo 4. Este módulo cierra dos hilos grandes: **H3** (qué es el disco que ve el container) y **H2** (el "en mi máquina funciona" del módulo 1). Se puede leer entero sin tener Docker instalado; si ya hiciste el setup, los comandos de la sección 7 son probables ahí mismo.

### De dónde venís

Del módulo 2 traés: el container es un proceso común envuelto en una **burbuja** (namespaces + cgroups) · el kernel es compartido · el namespace de mount le muestra al container "su propio disco" — y quedó abierta la pregunta de qué hay en ese disco · y la distinción kernel / sistema operativo / **distro** (el ajuar alrededor del motor), que acá vuelve con rol estelar.

---

## 1. 🔴 El trío: imagen, container, registry

Todo Docker se para sobre tres conceptos, y diferenciarlo bien es lo primero que se espera de cualquiera que diga "sé Docker":

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

Respuesta: **ese disco es la imagen.** Una imagen es un file system completo, empaquetado y congelado — todos los archivos que tu app necesita para vivir, listos y en su lugar. Cuando Docker lanza un container, el namespace de mount hace su magia: le muestra al proceso, como si fuera "el disco de la máquina", el contenido de la imagen. El container no está *dentro* de otra computadora — está **mirando un disco de utilería**, montado por el kernel para él.

Y ahora atá el cabo con el módulo 2, porque ya conocés este truco: una imagen de "Ubuntu" es **el ajuar de Ubuntu sin el motor** — las carpetas, el `apt`, las herramientas, las librerías… y ningún kernel, porque el kernel lo pone el host. ¿Te suena? Es *exactamente* el mismo formato que las distros de WSL (módulo 2, §6.3): el séquito empaquetado, listo para enchufarse a un kernel ajeno. La industria entera funciona empaquetando "Linux sin kernel" — WSL lo hace con distros enteras, Docker lo hace con imágenes. Por eso un container "parece un Ubuntu": tiene los *archivos* de Ubuntu. El motor es prestado.

**Hilo H3: cerrado.** El disco del container es la imagen, servida por el namespace de mount.

Dos propiedades de la imagen que van a gobernar todo lo que sigue:

1. **Es de solo lectura.** Nadie escribe sobre una imagen, nunca. (¿Y entonces cómo hace la app para escribir un archivo? Excelente pregunta — es el gancho del módulo 4. Aguantala.)
2. **Es inmutable: no se edita, se construye otra.** ¿Querés cambiar algo? Construís una imagen nueva. Las imágenes se versionan como el código — de hecho, vas a ver que se *construyen desde* código.

## 3. 🔴 Las capas: cómo está armado el paquete por dentro

Acá viene la decisión de diseño más importante de Docker, la que explica su eficiencia y también sus trampas. Una imagen **no es un archivo gigante**: es una **pila de capas**, apiladas una sobre otra, donde cada capa aporta archivos y el conjunto se ve como un único file system:

```
        LA IMAGEN, POR DENTRO — una pila de capas read-only
       ┌─────────────────────────────────────────────┐
   ④   │  capa: app.py                    (2 kB)     │  ← tu código
       ├─────────────────────────────────────────────┤
   ③   │  capa: python3 + pip instalados  (140 MB)   │  ← el runtime
       ├─────────────────────────────────────────────┤
   ②   │  capa: actualizaciones del sistema (30 MB)  │
       ├─────────────────────────────────────────────┤
   ①   │  capa: base ubuntu:22.04         (78 MB)    │  ← el ajuar de la distro
       └─────────────────────────────────────────────┘
              ▲
              │  el container mira desde arriba y ve UN file system:
              │  la SUMA de todas las capas superpuestas
```

- **Overlay** ("superposición"): la técnica de file system que logra ese efecto — capas transparentes apiladas, como filminas: mirás desde arriba y ves la combinación de todas. Cuando el container busca un archivo, el sistema lo busca **de la capa más alta hacia abajo** y usa la primera versión que encuentra; si no está en ninguna, no existe.

¿Y cómo se identifica cada capa? Con un **hash**:

- **Hash:** una huella digital calculada a partir del contenido — un código tipo `a3f2c884b1...`. Mismo contenido → mismo hash, siempre; un byte de diferencia → hash completamente distinto. El hash no es un nombre que alguien eligió: es la **identidad matemática del contenido**.

Esto convierte a las capas en piezas **reutilizables y compartibles**: si dos imágenes usan la misma capa base de `ubuntu:22.04` (mismo hash), tu máquina la guarda y la descarga **una sola vez**. Cien imágenes paradas sobre Ubuntu no cuestan cien Ubuntus — cuestan uno más cien diferencias. Guardá el concepto de hash con cariño: en el módulo 5 va a ser el protagonista de la caché de build (hilo H4, que se abre acá).

## 4. 🔴 El Dockerfile: la receta que construye la imagen

¿Y de dónde salen las capas? De una **receta escrita en un archivo de texto** llamado `Dockerfile` (así, literal, sin extensión). Cada línea es una **instrucción**; Docker las ejecuta en orden y va **congelando capas** a medida que avanza. La imagen es el resultado de cocinar la receta.

Sigamos con la app de la serie. Primero, la versión Python del servidor del módulo 1 (mismo espíritu, otro lenguaje — lo importante es la receta, no el lenguaje):

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

Y ahora la receta que empaqueta esa app **con todo su mundo**:

```dockerfile
# Dockerfile — la receta, línea por línea

FROM ubuntu:22.04
# └─ TODA receta arranca con FROM: el punto de partida. No es una capa nueva:
#    es un PUNTERO a otra imagen ya existente (con sus propias capas adentro).
#    Acá: el ajuar de Ubuntu 22.04. Sobre eso construimos lo nuestro.

LABEL maintainer="equipo-tacs"
# └─ Metadata: una etiqueta informativa (quién mantiene esto). No agrega archivos.

RUN apt-get update && apt-get install -y python3 python3-pip && rm -rf /var/lib/apt/lists/*
# └─ RUN ejecuta un comando ADENTRO de la imagen en construcción y congela el
#    resultado como capa. Acá: actualizar el índice de paquetes, instalar Python,
#    y borrar los archivos temporales del índice. ¿Por qué las TRES cosas
#    encadenadas con && en UNA instrucción? Sección 5 — es una lección entera.

WORKDIR /app
# └─ "De acá en adelante, parate en la carpeta /app" (la crea si no existe).
#    Afecta a las instrucciones siguientes y al container cuando corra.

COPY app.py .
# └─ COPY lleva archivos DESDE tu carpeta del proyecto HACIA la imagen.
#    Acá: app.py a /app (el "." es el WORKDIR actual). Congela una capa.

EXPOSE 8080
# └─ DOCUMENTA que la app escucha en el 8080. ⚠️ No abre ni publica nada:
#    es un cartel informativo. La conexión real del puerto llega en el módulo 4.

CMD ["python3", "app.py"]
# └─ El comando que va a correr cuando un container nazca de esta imagen.
#    No se ejecuta ahora (en el build): queda ANOTADO para el futuro.
#    Este comando será el famoso PID 1 del módulo 2. Desarrollo: módulo 4.
```

La correspondencia receta → paquete, que es la idea para retener:

```
   Dockerfile (receta)                    Imagen (pila resultante)
   ─────────────────────                  ─────────────────────────
   CMD ["python3","app.py"]  ──anota──▶   (metadata: comando de arranque)
   EXPOSE 8080               ──anota──▶   (metadata: puerto documentado)
   COPY app.py .             ──congela─▶  ④ capa: app.py
   WORKDIR /app              ──anota──▶   (metadata: carpeta de trabajo)
   RUN apt-get ...           ──congela─▶  ③ capa: python3 instalado
   LABEL maintainer=...      ──anota──▶   (metadata: etiqueta)
   FROM ubuntu:22.04         ──apunta──▶  ①② capas de la imagen base
```

Fijate la distinción: las instrucciones que **tocan archivos** (`RUN`, `COPY`) congelan capas con peso; las demás (`LABEL`, `WORKDIR`, `EXPOSE`, `CMD`) solo **anotan metadata** — capas de 0 bytes, como vas a comprobar en la sección 7.

🟡 Un atajo que existe y conviene conocer: en vez de instalar Python a mano sobre Ubuntu, podés arrancar de una imagen que **ya lo trae**:

```dockerfile
FROM python:3.11-slim      # Python ya instalado sobre un Debian recortado
WORKDIR /app
COPY app.py .
EXPOSE 8080
CMD ["python3", "app.py"]
```

Misma app, receta de cinco líneas. Esta versión corta es la que vas a construir con tus manos en el módulo 4.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué es un Dockerfile y qué relación tiene con las capas de la imagen?** Es el archivo de texto con la receta para construir una imagen: una secuencia de instrucciones que Docker ejecuta en orden. Las instrucciones que modifican archivos (RUN, COPY) congelan cada una una capa read-only nueva sobre las anteriores; las demás (LABEL, WORKDIR, EXPOSE, CMD) registran metadata. FROM no crea una capa: apunta a una imagen base existente sobre la que se apila todo. La imagen final es la pila completa, y cada capa queda identificada por el hash de su contenido.

## 5. 🔴 Las consecuencias de congelar

Las capas son read-only y se congelan **en el momento en que su instrucción termina**. Eso tiene tres consecuencias que separan a quien usa Docker de quien lo entiende — y la cátedra evalúa exactamente esta diferencia en el TP.

**Consecuencia 1: por eso el `RUN` triple encadenado.** Volvé a mirar la línea:

```dockerfile
RUN apt-get update && apt-get install -y python3 python3-pip && rm -rf /var/lib/apt/lists/*
```

`apt-get update` descarga índices de paquetes — archivos temporales que solo sirven para instalar. Si la limpieza (`rm -rf ...`) fuera una instrucción `RUN` **aparte**, llegaría tarde: la capa anterior ya se congeló *con los temporales adentro*. La capa de la limpieza solo puede taparlos, no destaparlos — y los archivos viajan igual, inflando la imagen para siempre. Encadenado con `&&`, todo ocurre dentro de **una** capa: los temporales nacen y mueren antes del congelamiento, y la capa queda limpia.

⚠️ El encadenamiento tiene un precio diferido que te va a morder en el módulo 4: como el índice de paquetes se borró, si algún día entrás a un container de esta imagen y tirás `apt-get install loquesea`, va a fallar — hasta que corras `apt-get update` de nuevo. No es un bug: es la consecuencia exacta de la limpieza. Cuando te pase, vas a saber por qué.

**Consecuencia 2: borrar no achica.** Generalizando lo anterior: **eliminar archivos en una capa superior no reduce el tamaño de la imagen.** El archivo sigue existiendo en la capa de abajo (congelada); la capa de arriba solo lo *marca como borrado* y lo esconde de la vista. La imagen pesa la suma de todas sus capas, incluyendo lo que ya nadie ve. La versión en vivo de este fenómeno — qué pasa cuando un *container* borra archivos de la imagen — es plato fuerte del módulo 5.

**Consecuencia 3: la imagen no se edita — se reconstruye.** ¿Cambiaste una línea de `app.py`? La imagen vieja no se "actualiza": construís una nueva, con sus capas nuevas donde corresponda. ¿Y hay que reconstruir *todo* cada vez, las capas pesadas incluidas? Ahí Docker tiene un as bajo la manga que involucra a los hashes de la sección 3… y es exactamente el hilo H4. Módulo 5.

## 6. 🟡 Imágenes base: las distros vuelven, ahora como mercadería

Toda receta arranca con `FROM`, así que toda imagen desciende de alguna **imagen base**. ¿Cuáles existen? Las distros del módulo 2, empaquetadas:

| Imagen base | Qué trae | Peso aprox.* | Cuándo usarla |
|---|---|---|---|
| `ubuntu` | El ajuar completo de Ubuntu: `apt`, herramientas estándar | ≈ 78 MB | Familiaridad total; sobra de todo |
| `debian` | El padre de Ubuntu; ajuar clásico y estable | ≈ 120 MB (o ≈ 75 MB en variante `slim`) | Base tradicional de muchas imágenes oficiales |
| `alpine` | Una distro **mínima**, pensada para containers; su propio gestor (`apk`) | ≈ **5-8 MB** | Cuando el peso importa (casi siempre, a escala) |
| `busybox` | Lo mínimo de lo mínimo: un puñado de herramientas y nada más | ≈ 1-4 MB | Casos extremos y pruebas |

*\* Los pesos varían según versión y arquitectura de tu máquina — tomalos como órdenes de magnitud. La diferencia Ubuntu vs Alpine (¡diez veces!) es real y es la que importa: multiplicá por cada versión de cada imagen de cada servicio que viaja por la red de una empresa, y entendés por qué existe Alpine.*

También existen bases **con el runtime ya puesto** — `python:3.11-slim`, `node:22-slim`, etc. — mantenidas oficialmente: la distro más el lenguaje, listos. El sufijo `slim` significa "recortada: lo esencial para correr, sin extras". Sobre el nombre completo (`python:3.11-slim` — ¿qué es cada parte, qué pasa si no ponés nada después de los dos puntos?): eso es el sistema de **tags** y tiene su desarrollo en el módulo 7. Por ahora: lo que va después de `:` elige la versión.

🟡 **Y una práctica de equipos grandes que la cátedra mira con cariño:** si tu equipo tiene cinco servicios Python, en vez de que cada Dockerfile repita la misma instalación, se construye **una imagen base propia** (`python-tacs:1.0`, con el runtime, las herramientas y los estándares del equipo) y los cinco servicios hacen `FROM python-tacs:1.0`. Una sola fuente de verdad, versionada, con ciclo de vida propio. Para el TP de la materia — donde las buenas prácticas de imágenes se evalúan explícitamente — tenerlo en el radar suma.

## 7. 🟡 La imagen bajo el microscopio

Tres comandos para *ver* todo lo anterior con tus ojos. Como siempre: las salidas están acá, leélas — y si ya hiciste el setup, probalos (la práctica en serio arranca en el módulo 4).

**Descargar una imagen — y ver las capas viajar:**

```console
$ docker pull python:3.11-slim
3.11-slim: Pulling from library/python
a2318d6c47ec: Pull complete      # ← ¡MIRÁ! No baja "un archivo": baja CAPA por CAPA
c78ef38ec2c1: Pull complete      #    cada línea es una capa, identificada por su hash
d19f0e0ce1cb: Pull complete
e2fb3a0b4c8d: Pull complete
9e35fe52a63c: Pull complete
Digest: sha256:9f35f3ad48...     # ← el hash de la imagen completa
Status: Downloaded newer image for python:3.11-slim
```

Cinco líneas `Pull complete` = cinco capas. La pila de la sección 3, cruzando la red pieza por pieza. Y si otra imagen compartiera alguna de esas capas, esa línea diría `Already exists` — la reutilización por hash, en vivo.

**Listar lo que tenés — y comparar pesos:**

```console
$ docker image ls
REPOSITORY    TAG         IMAGE ID       CREATED        SIZE
python        3.11-slim   9f35f3ad48e2   3 weeks ago    ≈130MB    # ← Debian recortado + Python
ubuntu        22.04       53a843653cbc   5 weeks ago    ≈78MB
alpine        latest      a8cbb8c69ee7   6 weeks ago    ≈8MB      # ← 10 veces menos que Ubuntu
hello-world   latest      5dd0d3e6e255   5 months ago   ≈20kB     # ← la del setup: casi nada
```

⚠️ Detalle que confunde a todos la primera vez: `CREATED` **no** es cuándo la descargaste vos — es cuándo sus autores la **construyeron**. Las imágenes viajan con su fecha de nacimiento.

**Ver la pila de una imagen — la receta al revés:**

```console
$ docker history python:3.11-slim
CREATED BY                                      SIZE      # (salida recortada a lo que importa)
CMD ["python3"]                                 0B        # ← arriba: lo ÚLTIMO de la receta
RUN ... instalación de python ...               ≈12MB
RUN ... dependencias del sistema ...            ≈3MB
ENV PATH=/usr/local/bin:...                     0B        # ← metadata: capa de 0 bytes
ADD file:... in /                               ≈75MB     # ← abajo: la base Debian
```

`docker history` te muestra la pila **de arriba hacia abajo** — o sea, la receta *al revés*: la primera línea es la última instrucción. Y ahí están, con tus propios ojos, las dos clases de instrucción de la sección 4: las que pesan (RUN, ADD/COPY) y las de **0B** (CMD, ENV… — pura metadata). Toda imagen del mundo, incluidas las oficiales, es esto: una receta congelada que podés auditar.

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
```

La lista entera de dependencias del módulo 1 — el runtime en su versión exacta, las librerías, las herramientas — vive ahora **adentro del paquete**. Tus seis compañeros no instalan nada de eso: descargan la imagen (por su hash, idéntica al byte) y la corren. El servidor de la nube: lo mismo. "En mi máquina funciona" muere porque **la máquina — el ajuar completo — viaja con la app**.

Con un asterisco honesto que planta el módulo 6: *casi* todo viaja adentro. Lo que cambia entre ambientes a propósito — la contraseña de la base de datos, la URL del servicio de clima, las claves — **no** se hornea dentro de la imagen (¡viajaría a todos lados, incluida la imagen que compartís!). Eso entra desde afuera, al momento de correr. Cómo, en el módulo 6.

**Hilo H2: cerrado.** El más viejo de la serie.

## 9. 🔴 Síntesis — y la puerta del módulo 4

Lo conquistado hasta acá, sumando módulos:

| Del deseo del módulo 1… | Estado | Mecanismo |
|---|---|---|
| Aislamiento | ✅ módulo 2 | Burbuja: namespaces + cgroups |
| Eficiencia y arranque instantáneo | ✅ módulo 2 | Proceso común, kernel compartido |
| "Corre igual en cualquier máquina" | ✅ **este módulo** | La imagen: el file system viaja con la app |

La tercera columna del módulo 1 está completa **en los papeles**. Falta hacerla vivir: tomar la imagen y parir el proceso — el container — con tus manos en la consola. Y apenas nazca, va a chocar con la pregunta que quedó picando en la sección 2: la imagen es de solo lectura… **¿y cuando la app escribe?** ¿Dónde cae el archivo que tu servidor loguea, el temporal que crea, el dato que guarda? La respuesta es una capa que todavía no conocés — la única con permiso de escritura en todo el edificio — y es la protagonista del módulo 4.

---

## ✅ Checkpoint del Módulo 3

*Sin mirar el material. Las que no salgan marcan qué sección releer. (Las respuestas no están acá a propósito.)*

1. Definí imagen, container y registry, y la relación entre los tres.
2. ¿Qué es exactamente "el disco" que el namespace de mount le muestra al container? ¿Por qué un container "parece un Ubuntu" sin traer sistema operativo?
3. ¿En qué se parece una imagen de Ubuntu a una distro de WSL?
4. ¿Cómo se busca un archivo en un file system de capas superpuestas? ¿Qué pasa si el archivo está en dos capas a la vez?
5. ¿Qué es un hash y por qué usar hashes como identidad de capa permite compartir y reutilizar capas entre imágenes?
6. Recitá la receta: ¿qué hacen FROM, RUN, COPY, WORKDIR, EXPOSE y CMD? ¿Cuáles congelan capas con peso y cuáles solo anotan metadata?
7. ¿Por qué `apt-get update`, la instalación y la limpieza van encadenados en UN solo RUN? ¿Qué pasaría si la limpieza fuera un RUN aparte?
8. ¿Por qué borrar archivos en una capa superior no achica la imagen?
9. ¿Qué diferencia hay entre `ubuntu` y `alpine` como imágenes base, y por qué esa diferencia le importa (en plata) a una empresa?
10. ¿Qué es una imagen base propia de equipo (estilo `python-tacs:1.0`) y qué problema resuelve?
11. En `docker history`, ¿en qué orden aparecen las capas respecto del Dockerfile? ¿Qué significan las líneas de 0B?
12. Explicá con la imagen en la mano por qué "en mi máquina funciona" deja de ser un problema. ¿Qué es lo único que deliberadamente NO viaja adentro de la imagen?

---

## Qué viene en el Módulo 4

Se acabó la teoría pura: el módulo 4 es consola. Vas a construir la imagen de este módulo con `docker build`, parir containers con `docker run`, verlos vivir y morir (`ps`, `logs`, `stop`, `rm`), meterte adentro de uno con `exec` y comprobar con tus ojos la doble identidad del PID 1 del módulo 2. Y vas a conocer a la única capa con permiso de escritura — la **capa read-write** — que responde la pregunta que este módulo dejó abierta y abre, ella misma, el problema más traicionero de todos: dónde están tus datos. (Spoiler: en peligro.)

**FIN DEL MÓDULO 3**
