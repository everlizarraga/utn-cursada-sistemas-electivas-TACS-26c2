# 📘 Apunte Maestro — Clase 02: Docker
## Parte 3 — El container en ejecución: ciclo de vida, capa R/W, copy-on-write, volúmenes y redes

**Materia:** TACS · 2C 2026 · **Unidad:** clase02 — Docker
**Parte 3 de 4** · Cubre: qué es un container y su ciclo de vida · correr y gestionar containers (`run`, `ps`, `logs`, `exec`, `stop`, `rm`) con la demo completa de puertos · la capa R/W y los datos efímeros · el patrón stateless · eficiencia en disco: stackable layers + copy-on-write · `docker diff` · volúmenes (named y bind mounts) · networking básico y comunicación por nombre · cheat sheet.

**De las Partes 1-2 se asume:** namespaces + cgroups y la doble identidad del PID · imagen = pila de capas read-only, inmutable · el Dockerfile y la caché · `mi-app:1.0` ya construida (Parte 2, §4) · la convención 👁️ de vistas adelantadas — que en esta parte **se cobra**: acá se enseñan los comandos que las Partes 1 y 2 adelantaron.

---

## 1. 🔴 Qué es un container

La definición completa, ahora que están todas las piezas:

- **Instancia runtime** de una imagen Docker.
- Tiene una **capa R/W (read-write) agregada arriba** de la imagen (que es read-only).
- **Efímero** — al eliminarse, su capa R/W se pierde.
- **Aislado** del resto: no puede ver, modificar ni dañar al host ni a otros containers. Lo logran los **namespaces** (vista privada del sistema: interfaces de red, árbol de PIDs, mountpoints) y los **cgroups** (recursos limitados, mitigan el bad neighbor) — el mecanismo completo está en la Parte 1, §4.

De las cuatro propiedades, la segunda y la tercera son las protagonistas de esta parte: esa **capa R/W** es el único lugar donde el container escribe, y su carácter **efímero** es la fuente del problema más serio de todos (§5).

## 2. 🔴 El ciclo de vida

Cinco estados, y el comando que provoca cada transición:

```
   created  ──▶  running  ──▶  paused  ──▶  stopped  ──▶  removed

   docker create  → created   (armado, sin arrancar)
   docker start   → running   (docker run = create + start, en un solo golpe)
   docker pause   → paused    (procesos congelados, reversible con unpause)
   docker stop    → stopped   (SIGTERM + SIGKILL — ver abajo)
   docker rm      → removed   (borra la capa R/W y la metadata: irreversible)
```

**`docker stop` no desenchufa: negocia.** Le manda al proceso principal una señal **SIGTERM** — el mensaje del sistema que pide "terminá ordenado": que la app deje de recibir requests y cierre lo más *graceful* (elegante) posible. Le da unos segundos; si no termina, la corta con **SIGKILL**, la señal que mata sin preguntar.

**`docker rm` es el final definitivo:** borra la **capa R/W y toda la metadata** del container. La imagen no se toca — era compartida y read-only.

⚠️ **Un container detenido NO se elimina automáticamente** (salvo que se haya usado `--rm`, §3). Se pueden acumular decenas de containers `stopped` ocupando espacio en disco sin que nada lo avise. Detenido ≠ eliminado: el primero conserva su capa R/W y puede volver a arrancar; el segundo no existe más.

> 🎓 **Para el parcial, si te preguntan**
> **Describí el ciclo de vida de un container.** Created (armado con `docker create`, sin arrancar) → Running (`docker start`; `docker run` combina create + start) → Paused (congelado con `pause`, reversible) → Stopped (`docker stop` envía SIGTERM para un cierre graceful y, si no termina en el plazo, SIGKILL) → Removed (`docker rm` elimina la capa R/W y la metadata). Un container detenido conserva su capa R/W y no se elimina solo — se acumulan ocupando disco salvo que se use `--rm`.

## 3. 🔴 Correr y gestionar containers — la demo completa

El repertorio operativo, comando por comando, con las salidas reales. Se trabaja con la imagen `mi-app:1.0` construida en la Parte 2.

**Correr publicando un puerto, de fondo:**

```console
$ docker run -d -p 8080:8080 mi-app:1.0
67ccc01743dee0e34bcc6255addc6173044f72e3a768d921e65d92c4004652c6
```

- **`-d`** (*detached*): el container corre **en segundo plano** y te devuelve el control de la terminal — imprime solo el ID del container. Sin `-d`, el proceso queda **atachado**: si tarda 10 minutos, la terminal queda tomada mostrando su output.
- **`-p 8080:8080`** (*publish*): **binding de puertos** — `host:container`. Lo que llegue al 8080 de tu máquina se pasa al 8080 del container. Es el puente entre la red privada del container (su namespace de red, Parte 1) y el mundo.

```console
$ docker ps
CONTAINER ID   IMAGE        COMMAND            STATUS          PORTS                    NAMES
67ccc01743de   mi-app:1.0   "python3 app.py"   Up 15 seconds   0.0.0.0:8080->8080/tcp   determined_snyder
```

`docker ps` lista los containers **corriendo**: su ID, la imagen, el comando (el CMD de la ficha, Parte 2), el estado, el mapeo de puertos… y un **nombre inventado** (`determined_snyder`): si no se le pone nombre con `--name`, Docker genera uno (adjetivo + apellido célebre). Con `-a` se listan **todos**, incluidos los detenidos — la comprobación del "existir ≠ correr" que la Parte 1 adelantó.

Abriendo `http://localhost:8080` en el navegador: **`Hola desde Docker! TACS 2026`** — la app, respondiendo desde adentro del container.

**El choque de puertos — y su resolución.** ¿Qué pasa si se levanta un **segundo** container con el mismo binding?

```console
$ docker run -d -p 8080:8080 mi-app:1.0
fd187cff35a4...
docker: Error response from daemon: failed to set up container networking: ...
Bind for :::8080 failed: port is already allocated
```

Falló — pero leé **quién** chocó: no los containers, sino **el puente**. El 8080 de TU máquina ya estaba tomado por el primer binding. Los containers, adentro, no tienen conflicto alguno. La solución es otra puerta del lado del host:

```console
$ docker run -d -p 8081:8080 mi-app:1.0
dcb399b02aad...

$ docker ps
CONTAINER ID   IMAGE        COMMAND            STATUS         PORTS                    NAMES
dcb399b02aad   mi-app:1.0   "python3 app.py"   Up 15 seconds  0.0.0.0:8081->8080/tcp   wonderful_hermann
67ccc01743de   mi-app:1.0   "python3 app.py"   Up a minute    0.0.0.0:8080->8080/tcp   determined_snyder
```

**Los dos containers creen estar en el 8080 — y los dos tienen razón**, cada uno en su interfaz de red privada. Solo difieren los puentes (8080 y 8081 del host). `http://localhost:8081` responde lo mismo que el 8080. Es el namespace de red de la Parte 1, resolviendo en vivo el conflicto de puertos de los cinco microservicios.

**Modo interactivo:**

```console
$ docker run -it ubuntu bash
root@e7d25caad8b0:/#
```

- **`-it`** (interactivo + terminal): te **atacha la terminal** al container — queda casi como un SSH a una máquina virtual. Acá: un Ubuntu con `bash` como proceso principal, y vos adentro.

**Logs:**

```console
$ docker logs -f determined_snyder
```

Muestra el **output** del proceso principal — lo que el container "dice por pantalla" (los access logs de un web server, por ejemplo: cada request que atiende). **`-f`** (*follow*): queda mirando en vivo; `Ctrl+C` para salir.

**Ejecutar comandos adentro de un container corriendo:**

```console
$ docker exec -it dcb399b02aad bash
root@dcb399b02aad:/app#
```

`exec` **crea un proceso adicional en el espacio de trabajo del container** — dicho con precisión: no "entra a una máquina", ejecuta un comando dentro de los namespaces de ese container. Con `bash` y `-it`, ese proceso es una shell atachada a tu terminal: estás parado en `/app` (el WORKDIR de la ficha, Parte 2). Exploremos — y prepará el reconocimiento:

```console
root@dcb399b02aad:/app# ls
app.py                              # ← la capa del COPY, vista desde adentro

root@dcb399b02aad:/app# curl
bash: curl: command not found       # ← la imagen slim no trae herramientas de más

root@dcb399b02aad:/app# apt-get install curl
E: Unable to locate package curl    # ← ¡falla! ¿por qué? ↓

root@dcb399b02aad:/app# apt-get update
Get:1 http://deb.debian.org/debian trixie InRelease
Fetched 9948 kB in 1s (8162 kB/s)   # ← re-descarga el catálogo de paquetes

root@dcb399b02aad:/app# apt-get install curl
...
0 upgraded, 27 newly installed...   # ← ahora sí
```

El primer `install` falló porque la imagen base **limpió el catálogo de apt durante su build** — exactamente la limpieza `rm -rf /var/lib/apt/lists/*` explicada en la Parte 2, §3, cobrándose su precio diferido: sin catálogo, apt no sabe qué es `curl`. `apt-get update` lo re-descarga, y todo vuelve a funcionar.

⚠️ Trampa vecina: adentro tampoco está `ps`, y `apt-get install ps` falla con `Unable to locate package` — no porque falte el catálogo esta vez, sino porque **el comando `ps` vive en un paquete que se llama `procps`**: pedir "ps" a secas no lo encuentra jamás.

**Salir y mirar desde afuera:**

```console
root@dcb399b02aad:/app# exit
$                                   # ← tu shell del exec se cerró; el container SIGUE corriendo

$ ps aux | grep app.py
alalbiero  19426  ... grep app.py   # ← solo aparece el propio grep
```

El `exit` cierra **tu** proceso del exec — el container ni se entera, porque su proceso principal sigue vivo. Y el grep desde el host merece lectura fina: **apareció solo el grep mismo** — el `python3 app.py` no está. Es la salvedad de la Parte 1, §4.2, en vivo: esta terminal es de una Mac con Docker Desktop, y el host real de los containers es la **VM interna** — el proceso existe con su PID alto, pero para el kernel de la VM, no para la terminal de macOS. En un Linux nativo, ese mismo grep lo mostraría.

**Buen final y limpieza:**

```console
$ docker stop wonderful_hermann && docker rm wonderful_hermann
wonderful_hermann
wonderful_hermann                   # ← imprime el nombre dos veces: uno por cada comando
```

Y la variante para containers descartables: **`docker run --rm -it ubuntu:22.04 bash`** — el `--rm` hace que el container **se elimine solo al terminar**, sin dejar cadáver en `docker ps -a`. Ideal para pruebas de un solo uso.

## 4. 🔴 La capa R/W: compartir la imagen sin pisarse

Múltiples containers pueden compartir la **misma** imagen base. Cada uno solo agrega su propia **Thin R/W layer** (capa fina de lectura-escritura):

```
   ┌───────────────────────────────────────────────┐
   │  Container A — Thin R/W    (All writes go here)│
   ├───────────────────────────────────────────────┤
   │  Container B — Thin R/W                        │
   ├───────────────────────────────────────────────┤
   │  Container C — Thin R/W                        │
   └───────────────────────────────────────────────┘
   ┌───────────────────────────────────────────────┐
   │  91e54dfb1179                            0 B  │
   ├───────────────────────────────────────────────┤
   │  d74508fb6632                       1.895 KB  │   ubuntu:22.04 Image
   ├───────────────────────────────────────────────┤   (READ-ONLY, compartida)
   │  c22013c84729                       194.5 KB  │
   ├───────────────────────────────────────────────┤
   │  d3a1f33e8a5a                       188.1 MB  │
   └───────────────────────────────────────────────┘
```

**La cuenta de la eficiencia:** 10 containers basados en la misma imagen de 200 MB **no** usan 2 GB — usan ~200 MB de imagen compartida + 10 thin layers de pocos KB cada una. Es uno de los motivos por los que Docker es tan eficiente en producción, especialmente en clusters con muchas instancias corriendo la misma imagen.

**La mayor diferencia entre imagen y container es esa capa R/W.** Y al eliminar un container: la capa R/W **se elimina**; la imagen queda **sin cambios**. Verlo en vivo, con un container de Ubuntu "rompiendo" su mundo:

```console
$ docker run -it ubuntu bash
root@e7d25caad8b0:/# ls
bin  boot  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

root@e7d25caad8b0:/# rm -rf var          # ← "borra" /var de su mundo
root@e7d25caad8b0:/# ls
bin  boot  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr
root@e7d25caad8b0:/# exit                #    (sin /var — para ESTE container)
```

El borrado vive en la capa R/W de **ese** container: la imagen `ubuntu` sigue intacta, y cualquier container nuevo nace con su `/var` en su lugar. Nadie puede dañar el original — solo su propia copia de la vista.

## 5. 🔴 Datos efímeros y el patrón stateless

La consecuencia importante, con todas las letras:

> **Los datos escritos dentro del container son efímeros. Si el container se elimina, esos datos se pierden.** Para persistencia se usan **volúmenes** (§8).

| Sin persistencia (default) | Con volumen |
|---|---|
| Datos en la capa R/W | Datos en un volumen **externo** al container |
| Se pierden al hacer `docker rm` | **Sobreviven** al container |
| Ideal para procesos **stateless** | Ideal para **bases de datos, archivos** |

- **Stateless** ("sin estado"): un proceso que no guarda nada que necesite sobrevivirlo — puede morir y ser reemplazado por otro idéntico sin pérdida.

La pregunta que hay que poder responder: **¿qué pasa si corremos una base de datos en Docker sin volumen y alguien hace `docker rm`?** Se pierden **todos** los datos. Suele sorprender a los que recién empiezan — y es la respuesta esperable en un parcial.

**El porqué de fondo es un patrón de diseño de la industria: los containers no deberían guardar estado.** El razonamiento completo: las réplicas de una API se crean y se destruyen **constantemente** — de día se crean muchas para el tráfico, de noche se matan un montón. Eso es lo normal y lo deseable. Las bases de datos, en cambio, no son felices creciendo y decreciendo: agregar un nodo a un cluster de Cassandra implica esperar que se repliquen los datos — tarda un montón; sacarlo, tampoco es feliz. Entonces: si guardaste datos en el container de una API que se destruye a diario, ese estado **se pierde**. Conclusión del patrón: el estado vive afuera (volúmenes, bases dedicadas); los containers quedan libres para nacer y morir.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué pasa con los datos de un container y qué es el patrón stateless?** Todo lo que un container escribe va a su capa R/W, que se elimina con `docker rm`: los datos son efímeros por diseño. Por eso el patrón de la industria es que los containers no guarden estado (stateless): las réplicas se crean y destruyen constantemente según el tráfico, y cualquier estado guardado adentro se perdería. Los datos que deben persistir van en volúmenes — almacenamiento externo al ciclo de vida del container — que es la solución estándar para bases de datos y archivos.

## 6. 🔴 Eficiencia en disco: las dos tecnologías

Docker basa su manejo de imágenes y administración de containers en **2 tecnologías clave**:

1. **Stackable image layers** (capas apilables): las capas se **comparten** entre múltiples containers e imágenes derivadas. Solo se descarga una vez lo que ya existe localmente.
2. **Copy-on-write (CoW)**: los archivos de las capas read-only **no se copian hasta que el container necesita modificarlos**. Minimiza el uso de disco y de memoria.

Juntas hacen que Docker sea mucho más eficiente en disco que las VMs tradicionales. La primera ya se vio (§4 y Parte 2). La segunda merece el mecanismo completo.

### 6.1 🔴 Copy-on-write: el mecanismo

Cuando un archivo **se modifica** en un container, pasan tres cosas en orden:

1. **Busca el archivo a través de las capas** — empieza en la capa superior, la más nueva, y baja capa por capa hasta la base.
2. Con la primera copia encontrada, realiza una operación **"copy-up"**: **copia el archivo a la capa writable** del container.
3. **Modifica la copia** en la capa writable. **La capa original permanece intacta.**

```
   container quiere modificar /etc/config
        │
        ▼ 1. busca de arriba hacia abajo por las capas
        ▼ 2. COPY-UP: copia el archivo ENTERO a su capa R/W
        ▼ 3. modifica LA COPIA — el original, congelado, ni se entera
```

### 6.2 🔴 El costo del copy-up

Esta operación puede **degradar notablemente la performance**, especialmente cuando hay:

- **Grandes archivos** (el copy-up copia el archivo completo, no el cambio),
- **gran cantidad de capas** (la búsqueda del paso 1 es más larga),
- **deep directory trees** (árboles de directorios profundos).

**Buenas noticias:** el copy-up ocurre **solo la primera vez** que un archivo en particular se modifica. Las modificaciones siguientes ya encuentran la copia en la capa writable — sin overhead adicional.

**Recomendaciones para mitigar** (y esto es criterio evaluable en el TP):

| Recomendación | Por qué |
|---|---|
| Datos que cambian frecuentemente → **Docker Volumes** | Los volúmenes montan directorios **fuera del CoW** (§8) |
| **Minimizar la cantidad de capas** (combinar comandos RUN) | Menos capas = búsqueda más corta |
| **Imágenes base ligeras** (alpine) | Reducen los deep directory trees |
| Datos temporales → **tmpfs mounts** | Montajes en memoria, sin tocar disco |

- **tmpfs mount:** un montaje que vive solo en RAM — nada de lo escrito ahí toca ningún disco.

El ejemplo que junta todo: una base de datos corriendo en Docker **sin volúmenes** dispararía un copy-up en la primera modificación de **cada archivo** del motor. **Por eso siempre se usan volúmenes para las bases de datos** — es la misma conclusión de §5, ahora con el argumento de performance además del de persistencia.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué es copy-on-write y qué implicancias tiene?** Es la estrategia de la capa R/W sobre las capas read-only: los archivos de la imagen no se copian hasta que el container necesita modificarlos; en ese momento se busca el archivo de la capa superior hacia abajo, se hace un copy-up (copia del archivo completo a la capa writable) y se modifica la copia, dejando la original intacta. Minimiza disco y memoria, pero el copy-up degrada performance con archivos grandes, muchas capas o árboles profundos (solo la primera modificación de cada archivo). Mitigaciones: volúmenes para datos que cambian seguido (quedan fuera del CoW), combinar RUNs para minimizar capas, bases ligeras como alpine, y tmpfs para temporales. Una DB sin volumen sufriría copy-up por cada archivo del motor: por eso las DBs siempre usan volúmenes.

## 7. 🔴 `docker diff`: inspeccionar la capa writable

El comando que muestra **exactamente qué cambió** en la capa R/W de un container respecto de su imagen. La demo completa:

```console
$ docker run -d --name test-layers ubuntu:22.04 sleep 300
# └─ un container cuyo proceso principal solo duerme 300 segundos — queda vivo
#    para experimentar (el comando final reemplaza al CMD, como vimos en la Parte 2)

$ docker diff test-layers
$                                       # ← vacío: recién nacido, no cambió nada

$ docker exec test-layers bash -c "echo 'hola TACS' > /tmp/archivo.txt"
# └─ exec sin -it: ejecuta y vuelve. El bash -c hace falta porque el ">" 
#    (redirección a archivo) es sintaxis de shell, y exec ejecuta comandos a secas.

$ docker diff test-layers
C /tmp                                  # ← C = Changed: el directorio cambió
A /tmp/archivo.txt                      # ← A = Added: archivo nuevo en la capa R/W

$ docker inspect test-layers | grep -A 10 "GraphDriver"
        "GraphDriver": { "Name": "overlayfs" ... }     # ← el storage driver en acción

$ docker system df -v                   # ← uso de disco desglosado por container

$ docker stop test-layers && docker rm test-layers
# El archivo.txt ya no existe: la capa R/W se eliminó con el container.
```

Las tres letras del diff: **A** = *Added*, **C** = *Changed*, **D** = *Deleted*. Es útil para debugging — y para **entender qué hace realmente un container desconocido**: su capa R/W es la lista completa de sus efectos.

## 8. 🔴 Volúmenes: persistencia fuera del ciclo de vida

> Los datos dentro del container son efímeros. Para persistencia se usan **Docker Volumes** — almacenamiento que existe **fuera del ciclo de vida del container**.

**El repertorio completo:**

```console
$ docker volume create mis-datos               # crear un volumen nombrado
mis-datos

$ docker run -d --name app-con-datos -v mis-datos:/app/data mi-app:1.0
# └─ -v volumen:/ruta → el container ve /app/data como una carpeta más...
#    que en realidad vive AFUERA, en el volumen

$ docker volume ls
DRIVER    VOLUME NAME
local     0af2c73f81c846bf10f6e41ce7369016578c...   # ← volúmenes anónimos (Docker
local     f01d7724898a466b122fc5aff63aac80d408...   #    los creó solo, sin nombre)
local     mis-datos                                 # ← el nuestro

$ docker volume inspect mis-datos
[ { "CreatedAt": "2026-08-18T18:52:55-03:00",
    "Driver": "local",
    "Mountpoint": "/var/lib/docker/volumes/mis-datos/_data",   # ← dónde vive en el host
    "Name": "mis-datos" } ]

$ docker run -d -v $(pwd)/datos:/app/data mi-app:1.0
# └─ BIND MOUNT: en vez de un nombre, una RUTA del host ($(pwd) = carpeta actual).
#    Monta una carpeta TUYA adentro del container.

$ docker volume prune                          # eliminar volúmenes sin uso
```

**Los dos tipos, y cuándo cada uno:**

| | **Named volume** (`docker volume create`) | **Bind mount** (ruta del host) |
|---|---|---|
| Quién lo gestiona | **Docker** (nombre, no ruta; vive en su territorio) | **Vos** (una carpeta tuya, explícita) |
| Mejor para | **Producción** — datos de la app, bases | **Desarrollo** — *hot reload* de código |

- **Hot reload:** montás la carpeta del código fuente en el container; cada guardado en tu editor se ve adentro al instante, sin rebuild.

**Cómo funciona por dentro, y qué habilita.** El `inspect` lo muestra: el volumen es un **punto de montaje** — una referencia con nombre a una carpeta que vive en el host (`/var/lib/docker/volumes/.../_data`). Cuando el container escribe en `/app/data`, escribe **ahí afuera**, esquivando por completo el CoW. Y ese punto de montaje ni siquiera tiene que ser local: podría apuntar a un **NFS** (un file system de red, remoto) — y esa es la mejor analogía para entender el diseño: *el disco está por fuera del host*; si el sistema operativo del host se rompe, no pasa nada — se actualiza, se re-atacha el volumen, y los datos siguen. En cloud existe exactamente eso como servicio: volúmenes que se **atachan y desatachan** de diferentes instancias — la base de datos creció, la instancia quedó chica, se desatacha el volumen y se engancha en una máquina más potente. En eso consiste la **portabilidad de los datos** (con su costo de operación, claro: mover y re-atachar no es gratis).

**Los tres casos de uso que hay que tener:**

1. **Persistencia simple:** el container escribe archivos en su volumen; el container muere; un container nuevo monta el mismo volumen y **arranca donde quedó el anterior** — "no arranques de cero: seguí procesando desde acá".
2. **Upgrade de una base de datos:** la base versión 7 tiene sus datos en un volumen; se mata ese container, se levanta la **versión 8 apuntando al mismo volumen** — motor nuevo, mismos datos.
3. **Configuración externa** (la indicación del TP, Parte 1 §4.1): montar el archivo de config al correr el container, en vez de plancharlo en la imagen.

**Y las tres precauciones**, que separan el uso correcto del accidente:

⚠️ **El efecto de lado existe.** Con volúmenes y bind mounts ya no estás en el mundo aislado del container: **estás trabajando sobre el host** (o sobre un disco de red). Varios containers pueden montar el mismo volumen y **todos ven lo mismo** — lo que uno escribe, los demás lo ven. Para los que solo necesitan leer, existe el candado: montar en **solo lectura** agregando el parámetro `:ro` (*read-only*) — ideal para archivos de configuración. Y para los que escriben, la regla con las bases de datos: **un solo container escribiendo a la vez** — un MySQL asume ser el único dueño de sus archivos, y debe serlo.

⚠️ **El socket de Docker: no montarlo.** En Linux todo es un archivo — incluso el **socket** por el que se le habla al daemon de Docker (el "teléfono" del motor). Técnicamente se puede montar ese archivo dentro de un container… y desde adentro **manipular el Docker del host**: crear, espiar y destruir todo. Se puede hacer, y es **súper peligroso**: le estás permitiendo al container **escaparse de la Matrix**. Riesgo de seguridad mayúsculo — hay que tener mucho cuidado.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué tipos de volúmenes existen y cuándo se usa cada uno?** Named volumes: los crea y gestiona Docker (`docker volume create`), se referencian por nombre y viven en el territorio de Docker en el host; son la opción para producción y datos persistentes (bases de datos), incluyendo casos como retomar el estado con un container nuevo o migrar de versión de motor montando el mismo volumen. Bind mounts: montan una ruta explícita del host dentro del container; son la opción para desarrollo (hot reload del código) y para configuración externa, idealmente con `:ro` cuando el container solo debe leer. Ambos existen fuera del ciclo de vida del container y esquivan el copy-on-write. Precauciones: varios containers pueden compartir un volumen (efecto de lado sobre el host; con bases de datos, un solo escritor), y jamás montar el socket de Docker dentro de un container.

## 9. 🔴 Networking básico

> Cada container tiene su propia interfaz de red (gracias a los network namespaces). Se conectan al host y entre sí mediante **redes virtuales**.

El primer mecanismo ya se usó en §3: el **port mapping** (`-p host:container`), el puente hacia afuera. Lo que falta es el segundo — **cómo se hablan los containers entre sí**:

```console
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
...            bridge    bridge    local     # ← la red default
...            host      host      local
...            none      null      local

$ docker network create mi-red               # crear una red personalizada

$ docker run -d --name db  --network mi-red postgres:15
$ docker run -d --name app --network mi-red mi-app:1.0
# └─ dos containers, ambos conectados a mi-red, cada uno con --name propio

$ docker exec app ping db                    # ← 💥 desde 'app', "db" RESUELVE:
PING db (172.19.0.2): 56 data bytes          #    el NOMBRE del container alcanza
64 bytes from 172.19.0.2 ...

$ docker network inspect mi-red              # ficha de la red: qué containers viven adentro
```

**Containers en la misma red se comunican por NOMBRE.** Docker les provee resolución de nombres automática (un **DNS** interno — el mecanismo que convierte nombres en direcciones, como en internet, pero privado de tu red): `app` encuentra a `db` escribiendo literalmente `db`, sin averiguar ninguna IP.

⚠️ **La trampa que hay que saber:** en la red **bridge default** — donde caen los containers si no se especifica `--network` — los containers **NO se resuelven por nombre**. El DNS automático existe **solo en redes definidas por el usuario** (*user-defined networks*). Es el tropiezo clásico: dos containers corriendo, "¿por qué no se ven por nombre?" — porque están en la default. La respuesta es siempre crear una red propia y conectarlos ahí. *(Docker Compose hace exactamente esto de forma automática → Parte 4.)*

Las reglas de visibilidad, completas: containers en **interfaces de red distintas no se ven** entre sí; en la **misma interfaz, sí** — y ahí sí conviven compartiendo el espacio de puertos de esa red. Eso permite **segmentar**: decidir con redes quién puede hablar con quién.

**El patrón que cierra la unidad:** una aplicación pegándole a su base de datos — un microservicio y una DB, cada uno en su container, **aislados pero conectados** por una red compartida, hablándose por nombre. Es la forma canónica de levantar app + base con Docker, y la semilla de la arquitectura del TP.

Los tres tipos de red que aparecen en el `ls`, para completar el mapa: **bridge** (puenteada — la default y todas las que crees), **host** (el container renuncia a su red privada y usa directamente la del host) y **none** (sin red: aislamiento total).

> 🎓 **Para el parcial, si te preguntan**
> **¿Cómo se comunican dos containers entre sí?** Cada container tiene su propia interfaz de red (network namespace). Para comunicarlos se crea una red definida por el usuario (`docker network create`) y se conectan ambos con `--network`: dentro de una red propia, Docker provee resolución de nombres automática, así que un container alcanza al otro por su nombre (`ping db`). En la red bridge default NO hay resolución por nombre. Containers en redes distintas no se ven (lo que permite segmentar); el port mapping `-p` queda solo para exponer hacia afuera del host. El patrón típico: app y base de datos en la misma red, hablándose por nombre, con solo la app publicada al exterior.

## 10. 🟡 Cheat sheet — comandos esenciales

La referencia completa de la unidad, con las notas de lectura que corresponden:

| Comando | Descripción |
|---|---|
| `docker build -t nombre:tag .` | Construir imagen desde Dockerfile |
| `docker run imagen` | Correr un container (crea un container basado en la imagen) |
| `docker run -d imagen` | En background (*detached*) — devuelve el control |
| `docker run -it imagen bash` | Modo interactivo con terminal — casi un SSH al container |
| `docker run -p 8080:80 imagen` | Port mapping `host:container` |
| `docker run -v host:container imagen` | Montar volumen / bind mount (`host` = ruta o nombre de volumen) |
| `docker run --rm imagen` | Eliminar el container al terminar |
| `docker ps` / `docker ps -a` | Containers activos / **todos** (incluso terminados) |
| `docker stop / rm nombre` | Detener / eliminar container |
| `docker image ls` | Listar imágenes locales |
| `docker pull imagen:tag` | Descargar imagen del registry |
| `docker push imagen:tag` | Subir imagen al registry |
| `docker exec -it cont bash` | Entrar a un container corriendo |
| `docker logs -f cont` | Ver logs (follow) |
| `docker inspect cont/img` | Metadata en JSON |
| `docker diff cont` | Cambios vs la imagen base |
| `docker system prune -a` | Limpiar TODO lo no usado |

---

## ✅ Checkpoint — Parte 3

*Sin mirar el apunte. Las respuestas no están acá a propósito.*

1. Enumerá las cuatro propiedades que definen a un container. ¿Cuál es "la mayor diferencia entre imagen y container"?
2. Dibujá el ciclo de vida con el comando de cada transición. ¿Qué señales manda `docker stop` y en qué orden?
3. ¿Qué borra exactamente `docker rm`? ¿Qué le pasa a la imagen? ¿Y qué pasa con los containers detenidos que nadie borra?
4. `-d`, `-it`, `-p`, `--rm`, `--name`: qué hace cada flag de `docker run`.
5. Dos `docker run -d -p 8080:8080` seguidos: ¿qué falla exactamente, qué NO chocó, y cómo se resuelve? ¿Por qué los dos containers "tienen razón" al creerse en el 8080?
6. ¿Qué hace `docker exec` con precisión ("crea un proceso en…")? ¿Por qué `exit` no mata al container?
7. Adentro del container: ¿por qué el primer `apt-get install curl` falla y el segundo funciona? ¿Qué pieza de la Parte 2 se está cobrando? ¿Cómo se llama el paquete que trae `ps`?
8. El `ps aux | grep` desde la terminal de una Mac no muestra el proceso del container: ¿por qué, y dónde SÍ se vería?
9. La cuenta de la eficiencia: 10 containers sobre una imagen de 200 MB — ¿cuánto disco usan y por qué?
10. Un container hace `rm -rf /var` y muere: ¿qué ve un container nuevo de la misma imagen? ¿Dónde vivía ese borrado?
11. ¿Qué pasa con una base de datos en Docker sin volumen tras un `docker rm`? Contá el patrón stateless completo: por qué las réplicas de API nacen y mueren a diario, por qué las DBs no, y qué conclusión sale de ahí.
12. Copy-on-write: los tres pasos del mecanismo. ¿Cuándo degrada la performance, cuántas veces paga cada archivo el copy-up, y cuáles son las cuatro mitigaciones?
13. ¿Qué muestra `docker diff` y qué significan A, C y D? ¿Para qué sirve frente a un container desconocido?
14. Named volume vs bind mount: quién gestiona cada uno, para qué se usa cada uno, y qué muestra `docker volume inspect`. ¿Por qué el NFS es la mejor analogía del diseño? ¿En qué consiste la portabilidad de los datos en cloud?
15. Las tres precauciones con volúmenes: el efecto de lado y `:ro`, la regla del único escritor, y el socket de Docker. ¿Qué significa "escaparse de la Matrix"?
16. ¿Dónde funciona la resolución por nombre entre containers y dónde no? ¿Qué reglas de visibilidad imponen las redes, y cuál es el patrón app ↔ DB?

---

**FIN DE LA PARTE 3** · *Sigue: Parte 4 — Docker Compose, el caso real de la materia, y el cierre de la unidad.*
