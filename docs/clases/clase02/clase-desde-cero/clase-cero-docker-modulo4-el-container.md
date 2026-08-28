# 🐳 Clase desde Cero — Docker · Módulo 4
## El container: nace, vive, escribe y muere

**Serie:** Clase desde Cero — Docker · Módulo 4 de 7 · El mapa completo está en `clase-cero-docker-roadmap.md`

---

### Sobre este documento

**Qué cubre:** qué pasa exactamente cuando un container nace — incluida la respuesta a la pregunta del módulo 3: la **capa read-write** · el build y el run con tus manos: `build`, `run`, `ps`, `logs`, `exec`, `stop`, `rm` · el binding de puertos · la vida adentro de la burbuja (y la comprobación en vivo del PID 1 del módulo 2) · `CMD` vs `ENTRYPOINT` · y el **ciclo de vida** completo del container, con la diferencia crucial entre parar y borrar.

**Qué NO cubre:** la caché de build y el costo del copy-on-write (módulo 5), la persistencia de verdad y las redes entre containers (módulo 6).

**Este módulo se hace, no solo se lee.** Es el primero que necesita Docker instalado (tu archivo de setup, si todavía no). Igual mantiene la regla de la serie: toda salida esperada está escrita — podés leerlo entero sin la terminal y entender todo, y ejecutar después para fijar.

### De dónde venís

Del módulo 3 traés: imagen = pila de capas congeladas + ficha técnica · el build cocina la receta una vez; el run solo sirve · el `CMD` quedó **anotado** en la ficha, esperando · y la pregunta abierta: si la imagen es de solo lectura, ¿dónde escribe la app?

---

## 1. 🔴 El nacimiento: qué hace exactamente `docker run`

Cuando ejecutás `docker run mi-app`, pasan cuatro cosas, en este orden y en milisegundos:

1. Docker busca la **imagen** (local; si no está, la baja del registry).
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

Aparecieron **fantasmas**: los hello-world del setup, en estado `Exited`. Acá está, en tu propia máquina, una de las distinciones centrales de todo Docker: **existir no es estar corriendo.** Un container `Exited` es un proceso que terminó *pero cuya capa read-write y ficha siguen guardadas en disco* — muerto, pero no enterrado. Se puede revivir (`docker start`), inspeccionar, o borrar. Por eso tu dashboard decía "Showing 2 items" y a la vez "No containers are running": dos existentes, cero corriendo.

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

⚠️ Trampa menor de regalo: el comando `ps` vive en un paquete que se llama **`procps`** — pedir "ps" a secas no lo encuentra. Y ahora, el momento para el que el módulo 2 te preparó:

```console
root@f3a91c07d2e8:/app# ps aux
USER   PID  ...  COMMAND
root     1  ...  python3 app.py     # ← PID 1: TU APP. Tal cual lo prometido.
root    28  ...  bash               # ← tu sesión de exec
root    36  ...  ps aux             # ← este mismo comando
```

**Tres procesos. Ese es el universo entero visto desde adentro.** Tu app es el PID 1 — el primer proceso de un mundo que existe solo para ella — y no hay rastro de tu navegador, tus otros containers, ni los cientos de procesos reales de la máquina. La doble identidad del módulo 2, comprobada con tus manos.

🖥️ **Según tu sistema — la otra mitad de la comprobación.** ¿Y desde afuera? En un **Linux nativo**, `ps aux | grep app.py` en la terminal del host te muestra ese mismo proceso con un PID alto cualquiera — las dos identidades a la vez. En **Mac y Windows**, si lo probás en tu terminal… no aparece nada. No es magia ni error: tu terminal es de macOS/Windows, y el proceso no vive ahí — vive **adentro de la VM** (módulo 2, sección 6). El "host" del container es el Linux escondido, no tu escritorio. Que no lo encuentres es, en sí mismo, la prueba de dónde vive.

Salí con `exit`. Detalle importante: **salir de tu bash no mata al container** — tu shell era un proceso *extra*; el container vive mientras viva su PID 1, que sigue siendo la app.

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

En criollo: **CMD es una sugerencia; ENTRYPOINT es una decisión.** CMD dice "si no pedís otra cosa, corré esto"; ENTRYPOINT dice "esta imagen ES este programa — lo que escribas serán sus argumentos". El combo (tercera fila) es el patrón de las imágenes bien hechas: ejecutable fijo, argumentos por defecto reemplazables. Cuando tengas que decidir para el TP, mirá cómo lo resuelven las imágenes oficiales de tu stack — son la referencia de buenas prácticas que la cátedra espera.

*(Y el porqué del formato de lista `["python3","app.py"]` que quedó pendiente del módulo 3: escrito así, tu programa ES el PID 1 directamente y recibe bien las señales — importa en la sección que sigue. Escrito sin lista, Docker mete una shell en el medio y las señales le llegan a la shell, no a tu app.)*

> 🎓 **Para el parcial, si te preguntan**
> **¿Diferencia entre CMD y ENTRYPOINT?** Ambos definen qué se ejecuta al arrancar el container (el PID 1). CMD es el comando por defecto y se **reemplaza por completo** si el usuario pasa un comando en `docker run`. ENTRYPOINT fija el ejecutable principal, y lo que el usuario pase en el `run` se **agrega como argumentos** de ese ejecutable. Se usan combinados: ENTRYPOINT fija el programa y CMD aporta los argumentos por defecto, reemplazables sin tocar el ejecutable.

## 6. 🔴 El ciclo de vida: parar NO es borrar

Todo lo que viste se ordena en un mapa de estados:

```
                docker run (= create + start)
                        │
   ┌─────────┐          ▼            docker pause ▶ ┌────────┐
   │ created │──────▶ ┌─────────┐ ◀ docker unpause  │ paused │
   └─────────┘        │ RUNNING │ ◀─────────────────└────────┘
    (armado, sin      └────┬────┘
     arrancar)             │  el PID 1 termina (solo, por error,
        ▲                  │  por Ctrl+C, o por docker stop)
        │                  ▼
        │             ┌─────────┐        docker start
        │             │ EXITED  │────────(revive: mismo container,
        │             └────┬────┘         misma capa read-write)──▶ RUNNING
        │                  │
        │                  │  docker rm
        │                  ▼
        │             ┌─────────┐
        └─── ✗ ────── │ REMOVED │   ← ya no existe: capa read-write
                      └─────────┘     y ficha, borradas para siempre
```

Las transiciones que importan, con sus matices:

**`docker stop mi-servidor`** — el buen final. No desenchufa: le manda al PID 1 una **señal de terminación** (un mensaje del sistema: "cerrá ordenado, por favor"). La app tiene unos segundos de gracia para terminar lo que estaba haciendo — cerrar conexiones, soltar archivos: un **graceful shutdown** (apagado elegante). Si no termina a tiempo, Docker la corta por las malas. Por eso importaba el formato de lista del CMD: la señal tiene que llegarle a TU app, no a una shell intermediaria.

**`docker start mi-servidor`** — la resurrección. Un container `Exited` no es un cadáver: es un proceso terminado con su capa read-write **intacta en disco**. `start` lo vuelve a lanzar — mismo container, misma capa, mismos archivos que hubiera escrito. **Parar no pierde datos.**

**`docker rm mi-servidor`** — el entierro. Borra la capa read-write y la ficha. Ahora sí: **todo lo que ese container escribió, desapareció para siempre.** (La imagen no se toca: era compartida y congelada. Podés parir otro container igual — pero nace con la capa read-write VACÍA: es *otro* container.)

La distinción merece marco:

> ⚠️ **`stop` ≠ `rm`.** Stop detiene el proceso y conserva los datos de la capa read-write. Rm borra el container y sus datos, irreversiblemente. El descuido clásico es el inverso: creer que "ya lo paré" limpia — no: los `Exited` se acumulan (mirá tus hello-worlds), cada uno reteniendo su capa en disco. Limpieza fina: `docker rm serene_easley loving_ptolemy` (por nombre o ID). Limpieza gruesa y el medidor de cuánto ocupa todo: módulo 5.

Dos yapas del día a día: `docker run --rm ...` crea un container que **se auto-borra** al terminar (ideal para pruebas de un solo uso — el `ls /app` de la sección 5 merecía un `--rm`), y `docker pause/unpause` congela y descongela un container vivo sin matarlo (existe, se usa poco).

> 🎓 **Para el parcial, si te preguntan**
> **Describí el ciclo de vida de un container.** Created (armado, sin arrancar) → Running (el PID 1 vive; `docker run` = create + start) → Paused (congelado con `pause`, reversible) → Exited (el PID 1 terminó — solo, por error o por `docker stop`, que envía una señal de terminación y da unos segundos de gracia para un graceful shutdown) → Removed (`docker rm` borra la capa read-write y la metadata). Clave: stop conserva la capa read-write y el container puede revivir con `start` sin perder lo escrito; rm la borra irreversiblemente. El container vive exactamente lo que vive su PID 1.

## 7. 🔴 Síntesis — y las dos cuentas pendientes

Lo que este módulo te deja en las manos:

| Comando | Qué hace |
|---|---|
| `docker build -t nombre:tag .` | Cocina la receta de la carpeta actual |
| `docker run -d --name X -p H:C imagen` | Pare un container: de fondo, con nombre, con puente de puertos |
| `docker ps` / `docker ps -a` | Censo de corriendo / de existentes |
| `docker logs X` (`-f`) | Lo que el PID 1 dijo por stdout |
| `docker exec -it X bash` | Entrar a la burbuja de un container vivo |
| `docker stop X` / `docker start X` | Buen final (datos a salvo) / resurrección |
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
6. Existir ≠ correr: ¿qué es un container `Exited`, qué conserva, y con qué comando se lista? ¿Con cuál se revive?
7. ¿Qué muestra `docker logs` exactamente? ¿Cuál es la buena práctica de logging para una app en un container?
8. `docker exec -it X bash`: ¿qué hace, y por qué salir de esa shell no mata al container?
9. Contá la secuencia completa de la trampa: por qué `ps` no estaba, por qué `apt-get install procps` falló primero, y por qué `apt-get update` lo cura. ¿Cómo se llama el paquete de `ps`?
10. Desde adentro, tu app es PID 1. ¿Qué ves desde la terminal del host en Linux nativo? ¿Y en Mac/Windows — y por qué esa ausencia es en sí misma una prueba?
11. CMD vs ENTRYPOINT: ¿qué pasa con cada uno cuando pasás un comando en el `docker run`? ¿Cuál es el patrón combinado y para qué sirve?
12. ¿Por qué el CMD en formato de lista importa para el `docker stop`? ¿Qué es un graceful shutdown?
13. Dibujá el ciclo de vida completo con sus transiciones. ¿Qué conserva `stop` y qué destruye `rm`?
14. ¿Para qué sirve `--rm` y en qué tipo de containers conviene?

---

## Qué viene en el Módulo 5

Dos experimentos con final sorpresa. Primero: rebuild tras cambiar una letra del código — y la revelación de la **caché de capas**: por qué el orden del Dockerfile es la diferencia entre builds de segundos y builds de minutos, multiplicada por cada build de cada servicio de una empresa (la cátedra evalúa exactamente esto en tu TP). Segundo: qué pasa *de verdad* cuando un container modifica o borra un archivo de la imagen congelada — el **copy-on-write**, su costo oculto, el comando `docker diff` para espiar la capa read-write, y el medidor (`docker system df`) y la escoba (`prune`) para que Docker no se coma tu disco.

**FIN DEL MÓDULO 4**
