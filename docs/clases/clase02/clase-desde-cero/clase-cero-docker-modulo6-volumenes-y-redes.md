# 🐳 Clase desde Cero — Docker · Módulo 6
## Persistir y comunicar: volúmenes, configuración y redes

**Serie:** Clase desde Cero — Docker · Módulo 6 de 7 · El mapa completo está en `clase-cero-docker-roadmap.md`

---

### Sobre este documento

**Qué cubre:** la cura del hilo H5 — los **tres tipos de montaje** (named volumes, bind mounts, tmpfs) y cuál usar para qué · por dónde entra la **configuración** que el módulo 3 dejó deliberadamente fuera de la imagen (variables de entorno y archivos montados) · y el cierre del hilo H6: las **redes** — cómo dos containers se encuentran y se hablan **por nombre**, sin puentes de puertos, y cómo se arma el patrón app ↔ base de datos que tu TP necesita.

**Qué NO cubre:** levantar todo esto con un solo comando en vez de a mano — eso es Compose, módulo 7.

**Módulo de hacer**, como el 4 y el 5: demos cortas con salida esperada, sobre la misma `mi-app:1.0` (sin descargas nuevas).

### De dónde venís

Del módulo 5 traés el frío en la espalda: los datos del container viven en la capa read-write → la capa muere con `docker rm` → una base de datos en un container "normal" pierde todo en un descuido. Y del módulo 4, el puente `-p` como única forma de entrarle a un container — que servía para vos, pero no resuelve cómo se hablan los containers *entre sí*.

---

## 1. 🔴 La idea que cura el H5: montar

El problema, en tres líneas: la capa read-write es efímera por diseño — los containers son descartables, y eso es una *virtud* (recrearlos, actualizarlos, escalarlos sin miedo). Pero algunos datos deben sobrevivir al container. Conclusión inevitable: **esos datos no pueden vivir en las capas — tienen que vivir AFUERA, en un lugar que el container vea pero que no muera con él.**

El mecanismo se llama **montar** (*mount*): enchufar una carpeta que vive fuera del container **dentro del árbol de archivos que el container ve**. ¿Te suena el verbo? Es el mismo namespace de mount del módulo 2 — el proyector de cine — haciendo un truco extra: además de proyectar la imagen y la capa read-write, proyecta *una carpeta externa* en una ruta elegida:

```
        EL CONTAINER, ahora con un montaje
       ┌─────────────────────────────────────────────┐
  ✏️   │  capa read-write            (efímera)       │
  🔒   │  capas de la imagen         (congeladas)    │
       │                                             │
       │   /data  ◀━━━━━━ MONTAJE ━━━━━━━━━━━━━━━━━━━┿━━▶ 📦 carpeta EXTERNA
       │   lo que el container ve como una carpeta   │     (sobrevive al container)
       │   más… en realidad atraviesa la pared       │
       └─────────────────────────────────────────────┘
        escribir en /data NO toca la capa read-write:
        cae directo en la carpeta externa
```

Todo lo que el container escriba en la ruta montada **esquiva las capas por completo**: ni copy-up, ni whiteout, ni muerte con el `rm`. Y hay **tres tipos** de montaje según *dónde* viva esa carpeta externa. Los tres, uno por uno.

## 2. 🔴 Tipo 1 — Named volumes: la caja fuerte administrada por Docker

Un **volumen con nombre** es una carpeta que Docker crea, administra y guarda en su territorio (`/var/lib/docker/volumes` — convención de la serie: en el disco de la VM en Mac/Windows, tu disco en Linux). Vos la conocés por su nombre, no por su ruta.

**Vealo sobrevivir (demo 1):**

```console
$ docker volume create datos-tacs
datos-tacs

$ docker run --rm -v datos-tacs:/data mi-app:1.0 sh -c 'echo "sobreviví" > /data/nota.txt'
# └─ -v nombre:/ruta → monta el volumen "datos-tacs" en /data del container.
#    Este container escribe la nota... y --rm lo entierra al terminar.

$ docker run --rm -v datos-tacs:/data mi-app:1.0 cat /data/nota.txt
sobreviví        # ← OTRO container, nacido después del entierro del primero,
                 #    lee el archivo. El dato vivió FUERA de ambos.
```

Dos containers distintos, los dos nacidos y enterrados (`--rm`), y el archivo sigue ahí. **El H5, curado ante tus ojos**: el dato no estaba en ninguna capa — estaba en el volumen.

El caso de uso estrella es el que te heló la espalda: **la base de datos**. El patrón universal es montar un volumen exactamente donde el motor de la base guarda sus datos (cada motor documenta su ruta — MySQL usa `/var/lib/mysql`, por ejemplo):

```console
$ docker run -d --name mi-db -v datos-db:/var/lib/mysql  ... imagen-de-mysql
```

Y de regalo, la historia que muestra el poder completo: tenés esa base corriendo en la versión 5.7 del motor y querés pasar a la 8. Con el volumen, el upgrade es: `stop` al container viejo, `rm` sin miedo (¡los datos no están en él!), y un `run` de la imagen nueva **montando el mismo volumen** — el motor 8 arranca, encuentra los datos ahí, y sigue. El container era descartable; los datos, sagrados; el volumen es la frontera entre ambos.

La caja de comandos:

```console
$ docker volume ls                # censo de volúmenes
$ docker volume inspect datos-tacs   # ficha: dónde vive, quién lo usa
$ docker volume rm datos-tacs     # borrarlo (recién cuando NADIE lo usa)
$ docker volume prune             # barrer los volúmenes que ningún container referencia
```

> ⚠️ **`docker rm` del container NO borra sus volúmenes — y es a propósito.** Es la gracia del invento: los datos sobreviven al descuido. La contracara: los volúmenes huérfanos se acumulan en silencio (el `docker system df` del módulo 5 los lista en su propia fila). Se limpian a conciencia con `volume rm` / `volume prune` — nunca por accidente.

## 3. 🔴 Tipo 2 — Bind mounts: una carpeta TUYA, adentro del container

El **bind mount** (*bind* = atar) monta **una carpeta de tu máquina, la que vos elijas**, dentro del container. En vez de un nombre, pasás una ruta:

```console
$ mkdir compartida && echo "hola desde el host" > compartida/saludo.txt

$ docker run --rm -v "$(pwd)/compartida":/mirador mi-app:1.0 cat /mirador/saludo.txt
hola desde el host                    # ← el container LEE tu carpeta real
# └─ $(pwd) = "la carpeta donde estoy parado"; el bind mount exige rutas absolutas,
#    y este truco te la escribe solo.

$ docker run --rm -v "$(pwd)/compartida":/mirador mi-app:1.0 \
    sh -c 'echo "hola desde el container" >> /mirador/saludo.txt'

$ cat compartida/saludo.txt
hola desde el host
hola desde el container               # ← 💥 el container escribió EN TU CARPETA
```

Esa última línea es la potencia y el peligro en una: **no hay pared**. La carpeta es la misma de los dos lados, en vivo — lo cual habilita el caso de uso estrella del desarrollo: montás la carpeta de tu **código fuente** dentro del container, editás en tu editor de siempre, y el container ve cada guardado *al instante*, sin rebuild (con herramientas que recargan al detectar cambios — el *hot reload* que ya conocés de tu mundo Node). Y también significa esto:

> ⚠️ **Borrar adentro es borrar en el host.** Un `rm` dentro de la ruta montada elimina TUS archivos reales — sin capa read-write que amortigüe, sin whiteout, sin resurrección. El bind mount es un portal, no una copia.

Por eso existe el candado: agregando `:ro` (*read-only*) al final — `-v "$(pwd)/config":/config:ro` — el container puede leer pero no tocar. Para montar configuración, casi siempre lo que querés.

🖥️ **Según tu sistema:** el bind mount es EL lugar donde las trampas de terminal muerden. En **Git Bash** (Windows), `$(pwd)` más rutas tipo `/mirador` disparan la traducción de rutas del setup (§2) — en la terminal de Ubuntu, cero drama. Y una perla de rendimiento para Windows: conviene que tus proyectos vivan **dentro del mundo de Ubuntu** (`~/proyectos/...`) y no en el disco de Windows vía `/mnt/c/...` — los bind mounts que cruzan esa frontera son notablemente más lentos.

## 4. 🟡 Tipo 3 — tmpfs: la carpeta que vive en la RAM

El tercer tipo es el raro de la familia: **tmpfs** monta una carpeta que existe **solo en memoria** — no toca el disco del container *ni el del host*, jamás. Al morir el container, se esfuma sin dejar rastro físico:

```console
$ docker run --rm --tmpfs /secretos mi-app:1.0 sh -c 'echo "token123" > /secretos/s.txt && cat /secretos/s.txt'
token123        # ← existió solo en RAM, y ya no existe en ningún lado
```

¿Para qué? Datos **sensibles o puramente temporales** que no deben quedar escritos en ningún disco: tokens, material criptográfico, archivos intermedios de un cálculo. Nicho, pero cuando hace falta, nada lo reemplaza.

**La tabla de decisión de los tres** — probablemente lo más citable del módulo:

| | **Named volume** | **Bind mount** | **tmpfs** |
|---|---|---|---|
| ¿Dónde viven los datos? | Territorio de Docker (`/var/lib/docker/volumes`) | **Una carpeta tuya**, la que elijas | **RAM** — en ningún disco |
| ¿Sobreviven al container? | ✅ Sí | ✅ Sí (son tus archivos) | ❌ No — se esfuman |
| ¿Quién lo administra? | Docker (nombre, no ruta) | Vos (ruta explícita) | Nadie: es humo |
| Caso estrella | **Datos de la base** del TP | **Código fuente** en desarrollo (hot reload) · config con `:ro` | Secretos y temporales que no deben tocar disco |
| Sintaxis | `-v nombre:/ruta` | `-v /ruta/host:/ruta` (+`:ro` opcional) | `--tmpfs /ruta` |

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué tipos de montaje existen y cuándo se usa cada uno?** Tres. Named volumes: carpetas creadas y administradas por Docker en su territorio, referenciadas por nombre; sobreviven al `docker rm` y son el estándar para datos persistentes (bases de datos) — otro container puede adoptar el mismo volumen (p. ej., para migrar de versión de motor sin perder datos). Bind mounts: montan una carpeta arbitraria del host dentro del container, en vivo y en ambos sentidos; ideales para código fuente en desarrollo (hot reload) y, con `:ro`, para configuración — con el riesgo de que borrar adentro borra en el host. Tmpfs: carpeta solo en memoria, sin persistencia en ningún disco; para datos sensibles o temporales. Los tres esquivan las capas: lo escrito ahí no pasa por la capa read-write.

## 5. 🔴 La configuración entra por afuera — el asterisco del módulo 3, cobrado

El módulo 3 cerró el hilo H2 con un asterisco: *casi* todo viaja adentro de la imagen — menos lo que cambia entre ambientes a propósito (contraseñas, claves de APIs, URLs de servicios). Hornearlo en la imagen sería regalarlo a cualquiera que la descargue, y clavaría un solo ambiente para siempre. Entonces, ¿por dónde entra? Dos puertas, las dos recién aprendidas o a un paso:

**Puerta 1 — variables de entorno** (las conocés del módulo 1: valores con nombre que el proceso lee al arrancar). El flag es `-e`:

```console
$ docker run --rm -e SALUDO="Hola config" mi-app:1.0 sh -c 'echo $SALUDO'
Hola config        # ← el valor entró desde afuera; la imagen no lo contiene
```

La MISMA imagen corre en tu máquina con la base de prueba y en producción con la real — solo cambian los `-e` del `run`. Para no tipear veinte `-e`, existe `--env-file archivo.env` (un archivo con `CLAVE=valor` por línea — que por supuesto **no se sube al repo**). Este mecanismo es exactamente el que tu TP va a usar para las claves de las APIs externas.

**Puerta 2 — un archivo de configuración montado**, con el candado puesto:

```console
$ docker run -d -v "$(pwd)/config.json":/app/config.json:ro mi-app:1.0
# └─ el container lee SU /app/config.json… que en realidad es TU archivo, intocable (:ro)
```

Criterio rápido: valores sueltos → variables de entorno; configuración con estructura (un JSON, un YAML entero) → archivo montado `:ro`. Las dos puertas conviven sin drama.

## 6. 🔴 Redes: el cierre del hilo H6 — hablarse por nombre

Cambio de tema, mismo espíritu. Estado del H6 hasta hoy: cada container tiene su red privada (módulo 2), y el único acceso era el puente `-p` hacia TU máquina (módulo 4). Pero el TP no necesita que *vos* le entres a la base — necesita que **la app le hable a la base**, container a container. ¿Los conectamos con puentes? ¿La app sale a la calle por un `-p` para volver a entrar por otro? Feo, frágil, y expone la base al mundo. La solución de verdad:

**Redes definidas por el usuario.** Creás una red con nombre, metés containers en ella, y adentro pasa la magia:

```console
$ docker network create red-tacs
f0a3c1...

$ docker run -d --name api --network red-tacs mi-app:1.0
# └─ OJO: sin -p. Este container NO tiene puente a tu máquina. Ya vas a ver por qué.

$ docker run --rm --network red-tacs mi-app:1.0 \
    python3 -c "from urllib.request import urlopen; print(urlopen('http://api:8080').read().decode())"
Hola desde Docker!
# └─ 💥 Leelo de nuevo: un segundo container llamó a  http://api:8080  —
#    "api" A SECAS, el NOMBRE del container, como si fuera un dominio de internet.
#    Y le respondió. Sin -p, sin puentes, sin saber ninguna dirección.
```

*(Sin `curl` en la imagen slim, la demo usa dos líneas de Python para hacer el request — el mensajero da igual; el milagro es el nombre.)*

Lo que acaba de pasar se llama **resolución de nombres**: dentro de una red creada por vos, Docker mantiene una **guía telefónica automática** (un DNS interno — el mismo mecanismo que convierte `google.com` en una dirección, pero privado de tu red) donde **cada container figura por su nombre**. `api` no es una dirección que averiguaste: es el `--name` del container, y con eso alcanza.

⚠️ **Y la trampa que te ahorro:** esto funciona **solo en redes definidas por el usuario**. Si largás dos containers a secas (sin `--network`), caen en la red default de Docker… donde comparten cable pero **la guía telefónica no existe**: `http://api:8080` muere con un `Name or service not known`. Es el tropiezo clásico de todo principiante — "¡pero si están los dos corriendo!" — y la respuesta es siempre la misma: crear tu red y meterlos adentro. Regla práctica: **para containers que deben hablarse, siempre una red propia.**

Ahora armá el patrón completo del TP con lo que ya sabés:

```
                        red-tacs  (tu red)
        ┌──────────────────────────────────────────┐
        │   ┌────────┐   http://db:5432  ┌──────┐  │
        │   │  app   │ ─────────────────▶│  db  │  │   · la app encuentra a "db"
        │   └───┬────┘   (por NOMBRE)    └──────┘  │     por nombre, sin averiguar nada
        └───────┼──────────────────────────────────┘   · la db NO tiene -p:
                │  -p 8080:8080                          ES INVISIBLE desde tu máquina
                ▼                                        y desde internet. Solo sus
         tu navegador / el mundo                         vecinos de red la ven.
```

Fijate la elegancia de seguridad que salió gratis: la base **no tiene puente a la calle** — nadie de afuera puede siquiera intentar conectársele. El único expuesto es quien debe estarlo: la app, por su `-p`. Y las redes también **aíslan hacia afuera**: containers en redes distintas no se ven entre sí — podés segmentar (frontend por acá, backend por allá, la base al fondo) y decidir con precisión quién ve a quién. Ese diseño de "quién habla con quién" es literalmente una de las cosas que la cátedra evalúa de tu compose — y el módulo 7 lo automatiza.

**Hilo H6: cerrado.** Los containers se hablan directo, por nombre, dentro de redes tuyas — los puentes `-p` quedan solo para la frontera con el mundo exterior.

La caja de comandos:

```console
$ docker network ls                    # censo de redes (vas a ver las default + las tuyas)
$ docker network inspect red-tacs      # ficha: qué containers viven adentro
$ docker network connect red-tacs X    # sumar un container YA corriendo a una red
$ docker network rm red-tacs           # borrar una red (sin containers adentro)
```

🟡 Para completar el cuadro (aparecen en `network ls` y en la bibliografía): además de las redes que creás — que son del tipo **bridge**, puenteadas — existen dos modos especiales: **host** (el container renuncia a su red privada y usa directamente la de la máquina — adiós burbuja de red, adiós conflictos resueltos; casos muy puntuales) y **none** (sin red en absoluto: container ermitaño, máximo aislamiento). Saber que existen alcanza.

> 🎓 **Para el parcial, si te preguntan**
> **¿Cómo se comunican dos containers entre sí?** Creando una red definida por el usuario (`docker network create`) y conectando ambos containers a ella (`--network`). Dentro de una red propia, Docker provee resolución de nombres automática (DNS interno): cada container es alcanzable por su nombre (`http://api:8080`), sin exponer puertos con `-p` — que solo hace falta para acceder desde fuera de Docker. En la red bridge por defecto los containers NO se resuelven por nombre, por lo que la práctica estándar es siempre una red propia. Containers en redes distintas están aislados entre sí, lo que permite segmentar la arquitectura (p. ej., la base de datos sin puertos publicados: alcanzable por la app, invisible desde el exterior).

## 7. ⚠️ Una puerta que existe y no hay que abrir

Ya que sabés montar cosas, una advertencia de adulto: en internet vas a cruzar tutoriales que montan `/var/run/docker.sock` dentro de un container. Ese archivo es **el teléfono directo al Docker del host** — un container con eso montado puede crear, espiar y destruir *todos* los demás containers, montar lo que quiera, tomar la máquina: **escaparse de la Matrix**. Tiene usos legítimos muy específicos (herramientas de administración), pero la regla para vos es simple: no lo hagas, y desconfiá por reflejo de cualquier receta que lo pida sin explicar exactamente por qué.

## 8. 🔴 Síntesis — dos hilos cerrados y el TP a un módulo

| Hilo | Estado |
|---|---|
| **H5** — ¿dónde quedan mis datos? | ✅ **Cerrado**: fuera de las capas — named volumes (datos), bind mounts (código y config), tmpfs (secretos) |
| **H6** — ¿cómo conviven y se hablan? | ✅ **Cerrado**: redes propias + resolución por nombre; `-p` solo para la frontera exterior |

Y el inventario nuevo:

| Comando | Qué hace |
|---|---|
| `docker volume create / ls / inspect / rm / prune` | El ciclo completo de los volúmenes |
| `-v nombre:/ruta` · `-v /host:/cont` (+`:ro`) · `--tmpfs /ruta` | Los tres montajes |
| `-e CLAVE=valor` · `--env-file x.env` | La config que entra por afuera |
| `docker network create / ls / inspect / connect / rm` | El ciclo completo de las redes |
| `--network mi-red` | Meter un container en una red al nacer |

Mirá tu mano de cartas: una app en una imagen bien ordenada (M3-M5), una base con su volumen (M6), las dos en una red hablándose por nombre (M6), la config entrando por afuera (M6)… El problema es que levantar todo eso son **seis comandos largos, en orden, cada vez** — crear la red, crear el volumen, levantar la db con sus `-e` y su `-v`, levantar la app con su `--network` y su `-p`… Multiplicalo por siete compañeros levantando el ambiente todos los días. El hilo **H7** — "levantar todo a mano es un infierno" — queda oficialmente abierto y sangrando. Su cura tiene nombre de archivo YAML, un solo comando, y es el módulo final.

---

## ✅ Checkpoint del Módulo 6

*Sin mirar el material. Las que no salgan marcan qué sección releer. (Las respuestas no están acá a propósito.)*

1. ¿Por qué los datos persistentes no pueden vivir en la capa read-write, y qué significa "montar" como solución? ¿Qué pieza del módulo 2 hace el trabajo?
2. ¿Qué pasa con lo escrito en una ruta montada — pasa por la capa read-write, sufre copy-up, muere con el `rm`?
3. Named volume: ¿quién lo administra, dónde vive, y cómo demostrarías con dos containers `--rm` que los datos lo sobreviven?
4. Contá el upgrade de una base de datos (motor 5.7 → 8) usando un volumen. ¿Qué se descarta y qué se conserva?
5. ¿Por qué `docker rm` NO borra los volúmenes del container, y por qué eso es una virtud con contracara? ¿Cómo se limpian?
6. Bind mount: ¿qué monta, cuál es su caso estrella en desarrollo, y cuál es su peligro simétrico? ¿Qué agrega `:ro`?
7. En Windows: ¿dónde conviene que vivan tus proyectos para que los bind mounts vuelen, y por qué?
8. ¿Qué es tmpfs, dónde viven sus datos, y para qué tipo de datos es la única opción correcta?
9. Las dos puertas de la configuración: ¿cuándo variables de entorno y cuándo archivo montado? ¿Por qué nada de eso puede hornearse en la imagen?
10. ¿Qué es la resolución de nombres en una red definida por el usuario? ¿Qué pasa si intentás lo mismo en la red default — y cuál es la regla práctica que evita ese tropiezo?
11. Dibujá el patrón app ↔ db del TP: ¿quién tiene `-p` y quién no, y qué gana en seguridad la que no lo tiene?
12. ¿Para qué segmentarías containers en redes distintas? ¿Qué evalúa la cátedra de ese diseño?
13. ¿Qué es montar el socket de Docker y por qué la regla es no hacerlo?

---

## Qué viene en el Módulo 7 — el final

Todo lo que sabés hacer a mano, declarado en un archivo: **Docker Compose** — el YAML con sus tres secciones (`services`, `volumes`, `networks`), el `docker compose up` que levanta tu arquitectura entera con un comando, cómo escalar un servicio a cinco réplicas, y el caso real de la materia para practicarlo. Más la pieza final del trío del módulo 3: el **registry** a fondo — el naming completo de las imágenes, los tags, el `latest` y por qué es traicionero, y el flujo `push`/`pull` con el que tu imagen viaja al mundo. Con eso, la serie desemboca exactamente donde tu TP la necesita.

**FIN DEL MÓDULO 6**
