# 🐳 Clase desde Cero — Docker · Módulo 4
## El container: nace, vive, escribe y muere

**Serie:** Clase desde Cero — Docker · Módulo 4 de 7 · El mapa completo está en `clase-cero-docker-roadmap.md`

---

### Sobre este documento

**Qué cubre:** qué pasa exactamente cuando un container nace — incluida la respuesta a la pregunta del módulo 3: la **capa read-write** · el build y el run con tus manos: `build`, `run`, `ps`, `logs`, `exec`, `stop`, `rm` · el binding de puertos · la vida adentro de la burbuja (y la comprobación en vivo del PID 1 del módulo 2) · `CMD` vs `ENTRYPOINT`, incluidas **las dos formas de escribirlos** y la shell fantasma que una de ellas esconde · el **ciclo de vida** completo, con sus comandos sobre cada flecha · y qué pasa con tus containers cuando cerrás Docker, suspendés la máquina o la apagás.

**Qué NO cubre:** la caché de build y el costo del copy-on-write (módulo 5), la persistencia de verdad y las redes entre containers (módulo 6).

**Este módulo se hace, no solo se lee.** Es el primero que necesita Docker instalado (tu archivo de setup, si todavía no). Igual mantiene la regla de la serie: toda salida esperada está escrita — podés leerlo entero sin la terminal y entender todo, y ejecutar después para fijar.

### De dónde venís

Del módulo 3 traés: imagen = pila de capas congeladas + ficha técnica · el build cocina la receta una vez; el run solo sirve · el `CMD` quedó **anotado** en la ficha, esperando · y la pregunta abierta: si la imagen es de solo lectura, ¿dónde escribe la app?

---

## 1. 🔴 El nacimiento: qué hace exactamente `docker run`

Cuando ejecutás `docker run mi-app`, pasan cuatro cosas, en este orden y en milisegundos:

1. Docker busca la **imagen** — local; si no está, la baja del **registry** (el depósito de imágenes del módulo 3 — Docker Hub por defecto; su desarrollo completo llega en el módulo 7).
2. El kernel arma la **burbuja**: namespaces (su vista) + cgroups (sus límites).
3. El namespace de mount monta la **pila congelada** de la imagen… **más una capa nueva, finita, VACÍA y con permiso de escritura encima: la capa read-write.**
4. Se ejecuta el comando anotado en la ficha (`CMD`) — y ese proceso es el **PID 1** del container.

El paso 3 es la respuesta que el módulo 3 dejó picando:

```
        EL CONTAINER, POR DENTRO
       ┌─────────────────────────────────────────────┐
  ✏️   │  CAPA READ-WRITE — única escribible,        │ ← nace vacía CON el container
       │  exclusiva de ESTE container                │    y muere CON el container
       ╞═════════════════════════════════════════════╡
  🔒   │  ④ capa: app.py                             │
  🔒   │  ③ capa: python3 instalado                  │  ← LA IMAGEN: congelada,
  🔒   │  ①② capas: base                             │    compartida, intocable
       └─────────────────────────────────────────────┘
        el proceso mira desde arriba y ve UN file system;
        todo lo que ESCRIBE cae en la capa de arriba
```

**Cada escritura del container — el log que genera, el temporal que crea, el archivo que guarda — cae en esa capa de arriba.** La imagen, abajo, jamás se toca: por eso cien containers pueden compartir la misma pila sin pisarse. Cada container tiene **su propia** capa read-write, independiente: si un container borra o escribe algo, los demás ni se enteran — dos salas de cine, mismo rollo, y lo que anotás en tu butaca no aparece en la sala de al lado.

Dónde vive (convención de la serie): **en disco**, al lado de las capas de la imagen — dentro de `/var/lib/docker`, o sea, en el disco de la VM en Mac/Windows, que es un archivo grande en tu disco físico.

Y una advertencia que este módulo planta y el 5 y el 6 cosechan: esa capa es **efímera** — está atada a la vida del container. Guardá la palabra.

## 2. 🔴 Manos a la obra: el build y el primer run

Armá una carpeta con **dos archivos** — los del módulo 3, versión corta:

```python
# app.py
from http.server import HTTPServer, BaseHTTPRequestHandler

class Saludo(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write("Hola desde Docker!".encode())

HTTPServer(("", 8080), Saludo).serve_forever()
```

```dockerfile
# Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY app.py .
EXPOSE 8080
CMD ["python3", "app.py"]
```

Parado en esa carpeta, cociná:

```console
$ docker build -t mi-app:1.0 .
```

Anatomía del comando: `-t mi-app:1.0` le pone **nombre y versión** a la imagen (t de *tag* — sin esto queda un hash pelado imposible de recordar), y el `.` final dice **"la receta y los archivos están en ESTA carpeta"** (el *build context*: donde Docker busca el `Dockerfile` y desde donde puede copiar con `COPY`).

Salida esperada (forma moderna — los tiempos y el orden exacto varían):

```console
[+] Building 9.8s (9/9) FINISHED
 => [internal] load build definition from Dockerfile        # ← lee la receta
 => [internal] load metadata for .../python:3.11-slim
 => [1/3] FROM docker.io/library/python:3.11-slim@sha256:…  # ← la base (ya la tenés: no re-baja)
 => [2/3] WORKDIR /app
 => [3/3] COPY app.py .                                     # ← congela la capa con tu código
 => exporting to image
 => => naming to docker.io/library/mi-app:1.0               # ← bautizada
```

*(En Dockers viejos el build se imprime distinto — `Step 1/5 : FROM…`, `Step 2/5 : …` — misma esencia: la receta, paso a paso. Y fijate que numera `[1/3]`: cuenta solo los pasos que trabajan sobre archivos; EXPOSE y CMD anotan en la ficha y ni figuran como pasos con trabajo.)*

```console
$ docker image ls
REPOSITORY   TAG    ...   SIZE
mi-app       1.0    ...   ≈130MB     # ← tu primera imagen: la base + tu app
```

**Primer run — el modo ingenuo:**

```console
$ docker run mi-app:1.0

```

…y la terminal **se queda ahí, muda**. No está rota: tu servidor corre en *primer plano* — la terminal quedó pegada al container, y como la app no imprime nada, no ves nada. Es incómodo a propósito: cortá con `Ctrl+C` (mata el PID 1 → muere el container) y hagámoslo como se debe:

**El run de verdad:**

```console
$ docker run -d --name mi-servidor -p 8080:8080 mi-app:1.0
f3a91c07d2e8b4...        # ← te devuelve el ID del container y te suelta la terminal
```

Tres flags nuevos, uno por uno:

- **`-d`** (*detached*, despegado): el container corre **de fondo** y la terminal queda libre. El modo normal para servidores.
- **`--name mi-servidor`**: el nombre. Sin esto, Docker inventa uno (adjetivo + científico/a célebre: `serene_easley`, `brave_hopper` — simpático, pero imposible de tipear en serio).
- **`-p 8080:8080`** (*publish*): el puente de puertos, y merece su propio diagrama —

```
   TU MÁQUINA                                LA BURBUJA
   puerto 8080 real ────────puente───────▶  puerto 8080 del container
        │            -p  host:container          │
   el navegador                             la app escucha acá,
   golpea acá                               en SU red privada
```

El módulo 2 te dio a cada container su red privada con sus propios puertos — perfecto para no chocar, pero ahora **nadie de afuera puede entrar**: el 8080 del container vive adentro de la burbuja. `-p host:container` tiende el puente: "lo que llegue al 8080 de mi máquina, pasalo al 8080 del container". Probalo:

```console
$ curl localhost:8080
Hola desde Docker!        # ← tu app, respondiendo desde adentro de la burbuja
```

*(O abrí `http://localhost:8080` en el navegador — mismo resultado. `curl` es una herramienta de terminal para hacer requests HTTP; viene instalada en Mac y en Ubuntu.)*

**Y ahora, el momento revancha.** En el módulo 1, dos servidores en el 8080 = muerte por `EADDRINUSE`. Probá el equivalente Docker:

```console
$ docker run -d --name mi-servidor-2 -p 8080:8080 mi-app:1.0
docker: Error response from daemon: ... Bind for 0.0.0.0:8080 failed: port is already allocated
```

Falló — pero mirá **quién** chocó: no los containers, sino el **puente**: el 8080 de TU máquina ya está tomado por el primer puente. El arreglo es elegir otra puerta de calle:

```console
$ docker run -d --name mi-servidor-2 -p 8081:8080 mi-app:1.0
a77b0c31e9d2...

$ curl localhost:8081
Hola desde Docker!
```

Leé eso de nuevo, despacio: **los dos containers creen estar en el 8080 — y los dos tienen razón**, cada uno en su red privada. Solo los puentes (8080 y 8081 de tu máquina) son distintos. El choque de puertos del módulo 1, disuelto ante tus ojos. (Que dos containers se hablen *entre sí* sin puentes: módulo 6, hilo H6.)

## 3. 🔴 El censo: `ps`, `logs`, y existir ≠ correr

```console
$ docker ps
CONTAINER ID   IMAGE        ...  STATUS         PORTS                    NAMES
a77b0c31e9d2   mi-app:1.0   ...  Up 2 minutes   0.0.0.0:8081->8080/tcp   mi-servidor-2
f3a91c07d2e8   mi-app:1.0   ...  Up 8 minutes   0.0.0.0:8080->8080/tcp   mi-servidor
```

`docker ps` lista los containers **corriendo**. Pero agregale `-a` (*all*):

```console
$ docker ps -a
CONTAINER ID   IMAGE         ...  STATUS                     NAMES
a77b0c31e9d2   mi-app:1.0    ...  Up 3 minutes               mi-servidor-2
f3a91c07d2e8   mi-app:1.0    ...  Up 9 minutes               mi-servidor
656c3e923613   hello-world   ...  Exited (0) 2 days ago      serene_easley     # ← ¡del setup!
65992cf52c5d   hello-world   ...  Exited (0) 3 days ago      loving_ptolemy    # ← también
```

Aparecieron **fantasmas**: los hello-world del setup, en estado `Exited`. Acá está, en tu propia máquina, una de las distinciones centrales de todo Docker: **existir no es estar corriendo.** Un container `Exited` es un proceso que terminó *pero cuya capa read-write y ficha siguen guardadas en disco* — muerto, pero no enterrado. Se puede revivir, inspeccionar, o borrar definitivamente — los tres, con sus comandos, en la **sección 6**; ahora seguimos explorando a los vivos.

🟡 *De paso, el dashboard de Docker Desktop cuenta esta misma historia con puntitos: en la lista de **imágenes**, el círculo verde lleno significa "in use" — al menos un container nació de ella, corriendo o no (es el gemelo visual de la columna `U` de la consola moderna); el círculo vacío, que ninguno. Y en la lista de **containers**, el circulito se pinta solo cuando el container está `Up` — tus fantasmas lo tienen apagado.*

**¿Y qué dice mi app?** Su salida no se pierde por correr despegada:

```console
$ docker logs mi-servidor
127.0.0.1 - - [28/Aug/2026 ...] "GET / HTTP/1.1" 200 -    # ← cada request que atendió

$ docker logs -f mi-servidor     # -f = follow: quedate mirando en vivo (Ctrl+C para salir)
```

`docker logs` muestra todo lo que el PID 1 imprimió por su **salida estándar** (stdout — lo que un programa "dice por pantalla"). Y acá va una buena práctica que la industria toma en serio: **una app en un container loguea a stdout, no a archivos internos** — así `docker logs` (y las herramientas serias de monitoreo) la ven sin abrir nada. Los porqués profundos llegan con los peligros de la capa read-write, en el módulo 5.

## 4. 🔴 Adentro de la burbuja: `docker exec`

Hasta acá miraste el container desde afuera. Ahora entrá:

```console
$ docker exec -it mi-servidor bash
root@f3a91c07d2e8:/app#
```

Anatomía: `exec` ejecuta **un comando adicional adentro de un container que ya corre** — acá, `bash`, o sea una shell. Los flags `-it` (interactivo + terminal) son los que hacen que esa shell te quede *conectada*, tipeable. El prompt cambió: estás **adentro de la burbuja**, parado en `/app` (¿por qué ahí? la ficha: el `WORKDIR` que anotaste en el build).

Explorá:

```console
root@f3a91c07d2e8:/app# ls
app.py                          # ← tu capa ④, vista desde adentro

root@f3a91c07d2e8:/app# ps aux
bash: ps: command not found     # ← ¡¿cómo?!
```

No hay `ps`. Ni `curl`, si probás. Estás en una imagen **slim**: trae lo esencial para *correr la app*, no una caja de herramientas. Instalemos `ps`… y prepará la cara de reconocimiento:

```console
root@f3a91c07d2e8:/app# apt-get install -y procps
E: Unable to locate package procps          # ← LA TRAMPA PLANTADA EN EL MÓDULO 3 💥
```

Falló, y vos ya sabés por qué: la imagen base limpió el **catálogo** de apt en su build (la capa quedó limpia y liviana — el precio diferido es este momento exacto). La cura es re-bajar el catálogo:

```console
root@f3a91c07d2e8:/app# apt-get update
Get:1 http://deb.debian.org/debian ... [descarga el catálogo de nuevo]

root@f3a91c07d2e8:/app# apt-get install -y procps
... Setting up procps ...                   # ← ahora sí
```

⚠️ Trampa menor de regalo: el comando `ps` vive en un paquete que se llama **`procps`** — pedir "ps" a secas no lo encuentra. (Y ya que está en escena, presentémoslo bien: **`ps aux`** es el censo de **procesos** de Linux — `a` = de todos los usuarios, `u` = formato detallado, `x` = incluso los que no tienen terminal. No lo confundas con `docker ps`: mismo apellido, familias distintas — `docker ps` lista *containers*, desde afuera; `ps aux` lista *procesos*, acá adentro.) Y ahora, el momento para el que el módulo 2 te preparó:

```console
root@f3a91c07d2e8:/app# ps aux
USER   PID  ...  COMMAND
root     1  ...  python3 app.py     # ← PID 1: TU APP. Tal cual lo prometido.
root    28  ...  bash               # ← tu sesión de exec
root    36  ...  ps aux             # ← este mismo comando
```

**Tres procesos. Ese es el universo entero visto desde adentro.** Tu app es el PID 1 — el primer proceso de un mundo que existe solo para ella — y no hay rastro de tu navegador, tus otros containers, ni los cientos de procesos reales de la máquina. La doble identidad del módulo 2, comprobada con tus manos.

Salí con `exit`: se cierra **tu** bash (que era un proceso extra) y volvés a la terminal de tu máquina. **El container sigue corriendo** — su PID 1, la app, ni se enteró de tu visita.

🖥️ **Según tu sistema — la otra mitad de la comprobación (con la verdad completa).** ¿Y el mismo proceso, visto desde afuera, con su PID alto? Depende de dónde estés parado:

- **Linux nativo:** directo — `ps aux | grep app.py` en tu terminal muestra el proceso con un PID alto cualquiera. Las dos identidades, a la vista.
- **Mac:** desde tu terminal no vas a ver nada, y **no existe una "terminal de la VM" a la que entrar**: la VM de Docker Desktop está sellada a propósito — sin shell para vos, sin acceso (la asimetría del módulo 2: sustrato privado de la app). El proceso vive ahí adentro, invisible desde tu escritorio.
- **Windows:** ídem — tu terminal de Ubuntu tampoco lo ve, porque `docker-desktop` es otra burbuja hermana, con su propio namespace de PID.

Que no lo encuentres **es, en sí mismo, la prueba de dónde vive**. El módulo te debe esta claridad: la comprobación directa de la doble identidad es un lujo de Linux nativo.

> 🕳️ **Madriguera — espiar la VM igual, por la ventana de Docker**
> Hay un truco elegante si la curiosidad te puede: `docker run --rm --pid=host busybox ps aux`. Es un container descartable al que `--pid=host` le pincha a propósito la parte-PID de la burbuja: en vez de su listita privada, ve **los procesos del host de Docker** — o sea, de la VM. En la salida vas a encontrar tu `python3 app.py` con su PID alto de verdad. (Baja la imagen `busybox`, ~2 MB.) No lo necesitás para nada de la serie — es puro placer de comprobación.
> *Volvé al camino.*

## 5. 🔴 ¿Quién es el PID 1? — CMD, override y ENTRYPOINT

El `CMD` de la ficha es el comando *por defecto*. Pero mirá esto:

```console
$ docker run mi-app:1.0 ls /app
app.py
$
```

Le pasaste un comando después de la imagen y… **reemplazó al CMD por completo**: en vez de `python3 app.py`, el PID 1 fue `ls /app` — listó, terminó, y el container **murió al instante** (quedó `Exited`: la regla de hierro en acción — *el container vive exactamente lo que vive su PID 1*). Un servidor vive "para siempre" porque su proceso nunca termina; un `ls` vive milisegundos.

Existe una segunda pieza para controlar el arranque: **`ENTRYPOINT`**. La diferencia con CMD es qué pasa con lo que escribís en el `run`:

| En la ficha | `docker run imagen` (sin nada) | `docker run imagen X Y` |
|---|---|---|
| `CMD ["python3","app.py"]` | corre `python3 app.py` | corre `X Y` — **CMD reemplazado entero** |
| `ENTRYPOINT ["python3"]` | corre `python3` | corre `python3 X Y` — **lo tuyo se suma como argumentos** |
| Ambos: `ENTRYPOINT ["python3"]` + `CMD ["app.py"]` | corre `python3 app.py` | corre `python3 X Y` — el ejecutable es fijo, CMD era solo el argumento por defecto |

Ahora, la aclaración que evita el malentendido clásico: cuando están los dos, **no se ejecutan dos cosas**. De las dos anotaciones de la ficha se **ensambla UNA sola línea de comando**, y nace UN solo proceso:

```
   ENTRYPOINT ["python3"]   +   CMD ["app.py"]
              └───────────┬───────────┘
                          ▼
      comando final:  python3 app.py       ← UN proceso: el PID 1

   $ docker run imagen otra.py             ← pasaste argumentos
                          ▼
      comando final:  python3 otra.py      ← tus argumentos REEMPLAZARON al CMD;
                                             el ENTRYPOINT quedó fijo
```

CMD no "se ejecuta aparte": **aporta el pedazo reemplazable** de esa única línea. Y como las dos son anotaciones en la ficha — no pasos que corren en secuencia — **el orden en que las escribas en el Dockerfile no importa** (la costumbre es ENTRYPOINT arriba, pero funciona igual al revés). En criollo: **CMD es una sugerencia; ENTRYPOINT es una decisión.** El combo (tercera fila) es el patrón de las imágenes bien hechas: ejecutable fijo, argumentos por defecto reemplazables. Para el TP, la referencia de cómo decidirlo son las **imágenes oficiales de tu stack** — las de Docker Hub mantenidas por cada proyecto (la de `node`, la de `python`, la de MySQL…): abrí su Dockerfile publicado y mirá cómo combinan ENTRYPOINT y CMD; ese es el estándar de buenas prácticas que la cátedra espera ver imitado.

### 5.1 🔴 Las dos formas de escribir el CMD — y la shell fantasma

Falta el misterio del formato de lista, prometido desde el módulo 3. El `CMD` (y el `ENTRYPOINT`) se pueden escribir de **dos formas**:

```dockerfile
CMD ["python3", "app.py"]     # forma LISTA — la que usa esta serie, siempre
CMD python3 app.py            # forma TEXTO — existe, y esconde un intruso
```

Parecen iguales. No lo son. Con la forma **texto**, Docker no ejecuta tu comando directamente: lo envuelve — por debajo corre `/bin/sh -c "python3 app.py"`. Ese `sh` es una **shell**: el mismo tipo de programa que atiende tu terminal (un intérprete de comandos) — solo que acá nace *adentro del container*, sin terminal conectada, invisible: un simple proceso intermediario. Y mirá lo que le hace al árbol:

```
   FORMA LISTA                            FORMA TEXTO
   ┌──────────────────────────┐           ┌─────────────────────────────────┐
   │ PID 1: python3 app.py    │           │ PID 1: sh   ← el intermediario  │
   │        ↑ TU APP manda    │           │   └─ hijo: python3 app.py       │
   └──────────────────────────┘           │             ↑ tu app, relegada  │
                                          └─────────────────────────────────┘
```

¿Y qué importa quién es el PID 1, si la app corre igual? Importa **cuando alguien le habla al PID 1** — y eso es exactamente lo que hace `docker stop`, como vas a ver en la sección 6: le manda un mensaje de "cerrá ordenado". Con la forma lista, el mensaje le llega a **tu app**. Con la forma texto, le llega a la shell intermediaria… que **no se lo reenvía** a tu app — el mensaje muere ahí, tu app nunca se entera, y termina cerrada por la fuerza. Dos aclaraciones para dejarlo redondo: esa shell **no existe en todos los containers** — solo nace si usaste la forma texto (nuestra forma lista = cero intermediarios, tu app es PID 1 directo); y no la confundas con el `bash` del `exec` de la sección 4 — aquel era TU visita interactiva, este es un intruso permanente plantado por la forma de escribir la receta.

> 🎓 **Para el parcial, si te preguntan**
> **¿Diferencia entre CMD y ENTRYPOINT?** Ambos son anotaciones de la ficha que definen el comando de arranque (el PID 1) — de las dos se ensambla una única línea. CMD es el comando (o los argumentos) por defecto y se **reemplaza por completo** si el usuario pasa un comando en `docker run`. ENTRYPOINT fija el ejecutable principal, y lo que el usuario pase se **agrega como argumentos**. Combinados: ENTRYPOINT fija el programa, CMD aporta argumentos por defecto reemplazables. Además, conviene la **forma de lista** (`["python3","app.py"]`): ejecuta la app directamente como PID 1; la forma de texto interpone una shell (`sh -c`) que queda como PID 1 y no reenvía las señales — rompiendo el graceful shutdown de `docker stop`.

## 6. 🔴 El ciclo de vida: parar NO es borrar

Antes del mapa, una confesión de `docker run`: **esconde dos pasos**. Existen por separado, y conocerlos ordena todo el diagrama:

```console
$ docker create --name ensayo -p 8080:8080 mi-app:1.0
b8e2f4a91c...      # ← ARMA el container: burbuja, capa read-write, ficha... SIN arrancarlo.
                   #    Queda en estado "Created": existe, nunca vivió.

$ docker start ensayo
ensayo             # ← y AHORA arranca: lanza el CMD → PID 1 → Running
```

`docker run` = `create` + `start`, en un solo golpe — y es lo que usa todo el mundo (`create` suelto existe para casos finos; te alcanza con saber que está ahí). Pero mirá el regalo conceptual: **`docker start` "arranca un container ya armado"** — y eso vale igual para uno recién creado que para uno que ya vivió y terminó. Revivir un `Exited` es *literalmente el mismo comando*. Con esas piezas, el mapa completo — cada flecha con su comando:

```
                  docker run  ( = docker create + docker start )
                                        │
   ┌─────────┐    docker start    ┌─────▼─────┐    docker pause     ┌────────┐
   │ CREATED │ ─────────────────▶ │  RUNNING  │ ◀─────────────────▶ │ PAUSED │
   └─────────┘                    └─────┬─────┘    docker unpause   └────────┘
        ▲                               │
        │ docker create                 │  el PID 1 termina:
     (imagen)                           │  solo · por error · Ctrl+C · docker stop
                                        ▼
                                  ┌───────────┐     docker start (¡revive!)
                                  │  EXITED   │ ───────────────────────▶ RUNNING
                                  └─────┬─────┘   misma capa read-write,
                                        │         mismos archivos escritos
                                        │  docker rm
                                        ▼
                                  ┌───────────┐
                                  │  REMOVED  │  ← capa read-write y ficha:
                                  └───────────┘    borradas para siempre
```

Las transiciones que importan, con sus matices:

**`docker stop mi-servidor`** — el buen final. No desenchufa: le manda al PID 1 una **señal de terminación** (un mensaje del sistema: "cerrá ordenado, por favor"). La app tiene unos segundos de gracia para terminar lo que estaba haciendo — cerrar conexiones, soltar archivos: un **graceful shutdown** (apagado elegante). Si no termina a tiempo, Docker la corta por las malas. Y acá cobra sentido la sección 5.1: la señal se la mandan **al PID 1** — si tu CMD está en forma lista, el PID 1 es tu app y el mensaje le llega; si está en forma texto, el PID 1 es la shell fantasma, el mensaje muere en ella, y tu app termina cortada por la fuerza igual. La forma de escribir una línea del Dockerfile decide si tu app muere bien o mal.

**`docker start mi-servidor`** — la resurrección (ya lo conocés de arriba). Un container `Exited` no es un cadáver: es un proceso terminado con su capa read-write **intacta en disco**. `start` lo vuelve a lanzar — mismo container, misma capa, mismos archivos que hubiera escrito. **Parar no pierde datos.**

**`docker rm mi-servidor`** — el entierro. Borra la capa read-write y la ficha. Ahora sí: **todo lo que ese container escribió, desapareció para siempre.** (La imagen no se toca: era compartida y congelada. Podés parir otro container igual — pero nace con la capa read-write VACÍA: es *otro* container.)

La distinción merece marco:

> ⚠️ **`stop` ≠ `rm`.** Stop detiene el proceso y conserva los datos de la capa read-write. Rm borra el container y sus datos, irreversiblemente. El descuido clásico es el inverso: creer que "ya lo paré" limpia — no: los `Exited` se acumulan (mirá tus hello-worlds), cada uno reteniendo su capa en disco. Limpieza fina: `docker rm serene_easley loving_ptolemy` (por nombre o ID). Limpieza gruesa y el medidor de cuánto ocupa todo: módulo 5.

Dos yapas del día a día: `docker run --rm ...` crea un container que **se auto-borra** al terminar (ideal para pruebas de un solo uso — el `ls /app` de la sección 5 merecía un `--rm`), y `docker pause/unpause`, que merece su párrafo porque intriga a todo el mundo:

🟡 **`pause` congela los procesos en el aire** — el kernel los deja clavados a mitad de instrucción. El container sigue "vivo" (su capa read-write, su red y su lugar en la guía telefónica siguen ahí), pero adentro nadie respira. La diferencia con `stop` se ve mejor **desde un vecino**: si tu app le pega a una base *pausada*, los requests **quedan colgados** esperando una respuesta que no llega (hasta que el timeout los mate) — el teléfono suena y suena en una casa donde todos están congelados; si la base está *parada* (`Exited`), la conexión **se rechaza al instante** — "número inexistente". En ambos casos el vecino sufre y debe saber reintentar; cambia el sabor del error. ¿Y para qué sirve? Casi nunca: casos nicho, como quitarle la CPU un rato a un container glotón sin matarlo, o congelar algo para inspeccionarlo con calma. Y no tiene nada que ver con suspender tu máquina (sección 6.1): eso congela la VM entera desde afuera; `pause` es un congelador quirúrgico, por container, manual.

### 6.1 🟡 ¿Y si se apaga todo? — cerrar Docker, suspender, apagar la máquina

La ansiedad legítima: tenés containers corriendo y la vida sigue — cerrás la app, cerrás la tapa, apagás. Qué pasa en cada caso:

- **Cerrás Docker Desktop** (Quit / ⌘Q): Docker **detiene ordenadamente** los containers que corren (señal de terminación, como un `stop`) y apaga la VM. Resultado: todos quedan `Exited`, con sus capas read-write intactas en disco — el disco de la VM es un archivo, y los archivos no se apagan. Al reabrir la app: imágenes impecables (congeladas), containers todos presentes en `docker ps -a`, ninguno corriendo, cero corrupción. Levantás lo que necesites con `docker start`. No arrancan solos por defecto — existe una opción para que lo hagan (*restart policies*), pero no la necesitás en esta serie.
- **Suspendés la máquina** (cerrar la tapa, reposo): la suspensión congela **todo** en el lugar — la VM incluida, containers incluidos. Al despertar, siguen `Running` como si el tiempo no hubiera pasado. (Único matiz: conexiones de red hacia afuera pueden haberse cortado por el tiempo dormido — para tu desarrollo local, irrelevante.)
- **Apagás normalmente**: el sistema le pide a cada app que cierre → Docker Desktop hace su apagado ordenado (mismo camino que ⌘Q) → containers `Exited`, datos a salvo, y al prender arrancás donde querías.
- **Apagado violento** (botón apretado, cuelgue, batería muerta): sin tiempo para el apagado elegante. Los archivos en disco quedan — pero una app agarrada a mitad de una escritura puede dejar esa escritura a medias. Es exactamente la clase de riesgo por el que los graceful shutdowns existen. No le temas: sabé que la diferencia entre "apagar" y "desenchufar" también aplica acá adentro.

> 🎓 **Para el parcial, si te preguntan**
> **Describí el ciclo de vida de un container.** Created (armado con `docker create`: burbuja, capa read-write y ficha listas, sin arrancar) → Running (`docker start` lanza el CMD como PID 1; `docker run` = create + start) → Paused (congelado con `pause`, reversible) → Exited (el PID 1 terminó — solo, por error o por `docker stop`, que envía una señal de terminación y da unos segundos de gracia para un graceful shutdown) → Removed (`docker rm` borra la capa read-write y la metadata). Claves: stop conserva la capa read-write y el container revive con `start` sin perder lo escrito; rm la borra irreversiblemente; y el container vive exactamente lo que vive su PID 1.

## 7. 🔴 Síntesis — y las dos cuentas pendientes

Lo que este módulo te deja en las manos:

| Comando | Qué hace |
|---|---|
| `docker build -t nombre:tag .` | Cocina la receta de la carpeta actual |
| `docker run -d --name X -p H:C imagen` | Pare un container: de fondo, con nombre, con puente de puertos |
| `docker create` / `docker start X` | Los dos pasos escondidos del run: armar / arrancar (start también revive) |
| `docker ps` / `docker ps -a` | Censo de corriendo / de existentes |
| `docker logs X` (`-f`) | Lo que el PID 1 dijo por stdout |
| `docker exec -it X bash` | Entrar a la burbuja de un container vivo |
| `docker stop X` | Buen final: señal al PID 1, gracia, datos a salvo |
| `docker rm X` | Entierro: capa read-write y ficha, borradas |
| `docker run --rm ...` | Container descartable, se auto-borra |

Y quedan dos cuentas abiertas, a propósito:

**La primera (H4):** tu build tardó unos segundos con una app de juguete. Cambiale una letra a `app.py` y volvé a construir: ¿re-cocina todo? ¿solo una parte? ¿por qué el orden de la receta decide cuánto pagás? Esa es la caché de build — **módulo 5** — y es de lo más evaluado por la cátedra en el TP.

**La segunda (H5, y es la grave):** todo lo que tu container escribe vive en la capa read-write… que `docker rm` borra para siempre. Ahora proyectalo: tu container es una **base de datos**. Los datos de tus usuarios están en esa capa. Alguien hace `rm`. — Sentiste el frío en la espalda: bien. Con ese frío se entra al módulo 5 (dónde duele) y al 6 (cómo se cura: volúmenes).

---

## ✅ Checkpoint del Módulo 4

*Sin mirar el material. Las que no salgan marcan qué sección releer. (Las respuestas no están acá a propósito.)*

1. Enumerá los cuatro pasos de `docker run`. ¿Qué agrega el paso del mount por encima de la imagen?
2. La capa read-write: ¿de quién es, cuándo nace, cuándo muere, y dónde vive físicamente? ¿Por qué cien containers pueden compartir una imagen sin pisarse?
3. En `docker build -t mi-app:1.0 .` — ¿qué hace el `-t` y qué significa el punto final?
4. ¿Qué hace `-d`? ¿Y `-p 8081:8080` — cuál número es la "puerta de calle" y cuál el puerto del container?
5. ¿Por qué dos containers pueden creer ambos estar en el 8080 sin chocar, pero dos `-p 8080:...` sí chocan? ¿Qué choca exactamente?
6. Existir ≠ correr: ¿qué es un container `Exited`, qué conserva, y con qué comando se lista? ¿Qué indican el círculo verde de una imagen y el circulito de un container en el dashboard?
7. ¿Qué muestra `docker logs` exactamente? ¿Cuál es la buena práctica de logging para una app en un container?
8. `docker exec -it X bash`: ¿qué hace cada parte, y por qué salir con `exit` no mata al container?
9. Contá la secuencia completa de la trampa: por qué `ps` no estaba, por qué `apt-get install procps` falló primero, y por qué `apt-get update` lo cura. ¿Cómo se llama el paquete de `ps`?
10. Desde adentro, tu app es PID 1. ¿Qué ves desde la terminal del host en Linux nativo? ¿Y en Mac/Windows — por qué no hay "terminal de la VM" a la que entrar, y por qué esa ausencia es en sí misma una prueba?
11. CMD vs ENTRYPOINT: ¿qué pasa con cada uno cuando pasás un comando en el `docker run`? Cuando están los dos, ¿cuántos procesos nacen y cómo se ensambla el comando final? ¿Importa el orden de las dos líneas en el Dockerfile?
12. Las dos formas de escribir el CMD: ¿cuál interpone una shell, qué es exactamente esa shell, cuándo existe y cuándo no, y por qué arruina el `docker stop`?
13. `docker run` esconde dos pasos: ¿cuáles, qué hace cada uno, y por qué "revivir" un Exited es uno de esos mismos comandos?
14. Dibujá el ciclo de vida completo con el comando de cada transición. ¿Qué conserva `stop` y qué destruye `rm`?
15. ¿Qué pasa con tus containers al cerrar Docker Desktop? ¿Al suspender la máquina? ¿En un apagado normal vs uno violento?
16. ¿Para qué sirve `--rm` y en qué tipo de containers conviene? ¿Y `pause` — qué congela exactamente, en qué se diferencia de `stop` visto desde un container vecino, y por qué se usa poco?

---

## Qué viene en el Módulo 5

Dos experimentos con final sorpresa. Primero: rebuild tras cambiar una letra del código — y la revelación de la **caché de capas**: por qué el orden del Dockerfile es la diferencia entre builds de segundos y builds de minutos, multiplicada por cada build de cada servicio de una empresa (la cátedra evalúa exactamente esto en tu TP). Segundo: qué pasa *de verdad* cuando un container modifica o borra un archivo de la imagen congelada — el **copy-on-write**, su costo oculto, el comando `docker diff` para espiar la capa read-write, y el medidor (`docker system df`) y la escoba (`prune`) para que Docker no se coma tu disco.

**FIN DEL MÓDULO 4**
