# 🐳 Clase desde Cero — Docker · Módulo 5
## El costo de las capas: la caché de build y el copy-on-write

**Serie:** Clase desde Cero — Docker · Módulo 5 de 7 · El mapa completo está en `clase-cero-docker-roadmap.md`

---

### Sobre este documento

**Qué cubre:** los dos fenómenos económicos de las capas. Del lado del **build**: la caché de capas, la cascada de invalidación, y por qué **el orden del Dockerfile es la diferencia entre builds de un segundo y builds de minutos** — con plata de por medio. Del lado del **runtime**: el **copy-on-write** (qué pasa de verdad cuando un container toca un archivo de la imagen congelada), el espía `docker diff`, los peligros de la capa read-write, y el medidor y la escoba para que Docker no se coma tu disco.

**Qué NO cubre:** la cura definitiva para los datos efímeros (volúmenes — módulo 6). Este módulo te hace *sentir* el problema; el que viene lo resuelve.

**Módulo de hacer:** como el 4, todos los experimentos traen su salida esperada — leíble de punta a punta, ejecutable para fijar. Cierra el hilo **H4** (abierto en el módulo 3) y deja el **H5** doliendo a propósito.

**Nota para el TP de la materia:** de toda la serie, este es el módulo que la cátedra más mira en tu entrega — el orden del Dockerfile y las buenas prácticas de capas se evalúan explícitamente.

### De dónde venís

Del módulo 4 traés: la imagen viva como container (pila congelada + capa read-write encima), el build con `docker build -t`, y la carpeta con `app.py` + `Dockerfile` ya construida como `mi-app:1.0`. Este módulo trabaja sobre esa misma carpeta.

---

## 1. 🔴 Experimento 1: el rebuild gratis

Sin tocar nada — mismos archivos, misma receta — volvé a cocinar:

```console
$ docker build -t mi-app:1.0 .
[+] Building 0.4s (9/9) FINISHED
 => [internal] load build definition from Dockerfile
 => [1/3] FROM docker.io/library/python:3.11-slim@sha256:…
 => CACHED [2/3] WORKDIR /app           # ← no lo hizo: lo RECORDÓ
 => CACHED [3/3] COPY app.py .          # ← ídem
 => exporting to image
```

**0.4 segundos.** El primer build tardó lo suyo; este fue gratis. La palabra clave es `CACHED`: Docker no re-ejecutó nada — reconoció que ya tenía cada capa cocinada y la reutilizó. *(En Dockers viejos la marca es `---> Using cache` por paso; misma esencia.)*

¿Cómo decide? Con los hashes del módulo 3, capa por capa y **en orden**:

- Para un **`RUN`**: "¿el texto de la instrucción es idéntico al de la vez pasada, y todas las capas anteriores son las mismas? → reuso la capa congelada, no ejecuto nada."
- Para un **`COPY`**: el texto no alcanza (dice `COPY app.py .` siempre) — Docker calcula el **hash del contenido del archivo**. ¿Mismo hash que la vez pasada? → reuso. ¿Un byte distinto? → esta capa hay que re-cocinarla.

La caché es literalmente la aplicación de "mismo contenido → mismo hash → misma capa": si nada cambió, ¿para qué cocinar de nuevo lo que ya está congelado?

## 2. 🔴 Experimento 2: una letra, y la cascada

Ahora tocá el código: abrí `app.py` y cambiale el saludo —

```python
        self.wfile.write("Hola desde Docker! v2".encode())   # ← agregaste " v2"
```

Rebuild:

```console
$ docker build -t mi-app:1.1 .
[+] Building 1.2s (9/9) FINISHED
 => [1/3] FROM docker.io/library/python:3.11-slim@sha256:…
 => CACHED [2/3] WORKDIR /app           # ← hasta acá, la caché sobrevive
 => [3/3] COPY app.py .                 # ← SIN "CACHED": el hash del archivo cambió →
 => exporting to image                  #    esta capa se re-cocinó
```

El `COPY` detectó el cambio (hash distinto) y se re-ejecutó. Trivial con una capa de 2 kB… pero mirá la regla general que se esconde acá, porque es LA regla del módulo:

> **Una capa invalidada invalida TODAS las capas que le siguen.** La caché se corta en el primer cambio y no vuelve: todo lo que está después en la receta se re-cocina, haya cambiado o no.

```
   LA CASCADA DE INVALIDACIÓN
                                        el archivo copiado cambió
   Dockerfile          1er build             2do build
   ─────────────       ──────────           ──────────
   FROM base           (base)                (base)      ✅ caché
   RUN instalar X      cocina 60s            CACHED      ✅ caché
   RUN instalar Y      cocina 30s            CACHED      ✅ caché
   COPY codigo  ◀──────── acá cambió ──────▶ RE-COCINA   💥 y de acá
   RUN preparar Z      cocina 20s            RE-COCINA   💥 para abajo,
   CMD ...             (anota)               (re-anota)  💥 TODO de nuevo
```

¿Por qué tan drástico? Porque cada capa se cocinó **parada sobre** las anteriores: si una cambia, Docker no puede garantizar que las siguientes darían el mismo resultado — así que las rehace todas. La caché no es mágica: es una fila de dominós, y el primero que cae voltea el resto.

## 3. 🔴 La consecuencia: el orden del Dockerfile es plata

Ahora juntá las piezas: la caché se corta en el primer cambio… y no todo cambia al mismo ritmo. Tu **código** cambia veinte veces por día. Tus **dependencias** cambian una vez por semana, con suerte. La **base**, cada tanto. Entonces la pregunta del millón es: *¿qué ponés primero en la receta?*

Compará los dos mundos con la misma app:

```dockerfile
# ❌ ORDEN MALO: lo volátil primero          # ✅ ORDEN BUENO: lo estable primero
FROM python:3.11-slim                        FROM python:3.11-slim
WORKDIR /app                                 WORKDIR /app
COPY . .                # el código, arriba  COPY requirements.txt .
RUN pip install -r requirements.txt          RUN pip install -r requirements.txt
                                             COPY . .                # el código, al final
CMD ["python3", "app.py"]                    CMD ["python3", "app.py"]

# Cambiás UNA letra del código y rebuildeás:
# ❌ el COPY se invalida → y el pip install   # ✅ requirements.txt no cambió → su COPY
#    que le sigue SE RE-EJECUTA ENTERO:       #    y el pip install quedan CACHED.
#    minutos de descarga e instalación,       #    Solo se re-copia el código: ~1 segundo.
#    POR CADA cambio de código.
```

El truco fino del orden bueno: se copia **primero el archivo que declara las dependencias** (`requirements.txt` en Python, `package.json` en Node — ¿te acordás de la "jugada maestra" prometida en el módulo 3? era esto) y **recién después el resto del código**. Así la capa pesada de instalación solo se invalida cuando cambian *las dependencias declaradas*, no cuando cambia cualquier archivo tuyo.

**Comprobalo vos (experimento 3).** Sumale a la receta una dependencia de mentira para que haya algo que instalar:

```dockerfile
# Dockerfile — versión con dependencia, ORDEN BUENO
FROM python:3.11-slim
WORKDIR /app
RUN pip install requests        # ← simula tu capa de dependencias (unos segundos)
COPY app.py .
CMD ["python3", "app.py"]
```

Build dos veces (la segunda: todo CACHED). Cambiá el saludo de `app.py`, rebuild: el `pip install` dice **CACHED** y solo el COPY se re-cocina — un segundo. Ahora invertí las dos líneas (COPY antes del RUN), cambiá el saludo otra vez, rebuild: el `pip install` **se re-ejecuta entero**. Acabás de medir con tus manos la diferencia entre los dos mundos. (En esta app de juguete son segundos; en una app real, la capa de dependencias son minutos.)

**Y la escala, que es donde esto se vuelve plata en serio:** una empresa con cientos de servicios, cada uno rebuildeándose decenas de veces por día en su sistema de integración continua (los servidores que construyen y prueban cada cambio automáticamente). Un Dockerfile mal ordenado ahí no es "un build lento": son **horas de máquinas alquiladas re-cocinando capas idénticas**, todos los días, multiplicado por todo. El orden del Dockerfile es de las pocas líneas de "configuración" que se traducen directo a factura.

🟡 Dos equilibrios para no irse al otro extremo: primero, las capas también **las lee gente** — a veces conviene un `RUN` de más por legibilidad, y está bien: optimizá donde duele (las capas pesadas), no fanáticamente en todas. Segundo, tampoco conviene atomizar en cincuenta capas: la lectura del overlay busca archivo por archivo de arriba hacia abajo — una pila kilométrica hace ese viaje más largo. Criterio, no dogma.

> 🎓 **Para el parcial, si te preguntan**
> **¿Cómo funciona la caché de build y por qué importa el orden del Dockerfile?** Docker reutiliza capas ya construidas: un RUN se reusa si su instrucción y todas las capas previas no cambiaron; un COPY compara el hash del contenido de los archivos. Una capa invalidada invalida en cascada todas las siguientes. Por eso el orden óptimo pone lo estable primero (instalación de dependencias, copiando antes solo el archivo que las declara) y lo volátil al final (el código fuente): así un cambio de código re-cocina solo la capa liviana del código, y la capa pesada de dependencias queda cacheada. Mal ordenado, cada cambio de código re-ejecuta la instalación completa — un costo que a escala de CI y muchos servicios se traduce directamente en tiempo y dinero.

## 4. 🔴 Copy-on-write: cuando el container toca lo congelado

Cambio de escenario: del build al **runtime**. El módulo 4 te mostró que todo lo que el container escribe cae en su capa read-write. Falta el detalle mecánico — y tiene un costo escondido. Tres operaciones posibles, tres destinos:

| El container quiere… | Qué pasa por debajo | Costo |
|---|---|---|
| **Crear** un archivo nuevo | Se escribe directo en la capa read-write | Barato: pesa lo que pesa el archivo |
| **Modificar** un archivo de la imagen | ⚠️ **Copy-up**: el archivo se copia **ENTERO** de la capa congelada a la read-write, y recién ahí se modifica la copia | Caro: pesa el archivo COMPLETO, aunque hayas cambiado un byte |
| **Borrar** un archivo de la imagen | Se anota una **marca de borrado** (*whiteout*) en la capa read-write que lo tapa | El archivo sigue abajo, congelado, ocupando lo mismo — solo dejó de verse |

A esta política se la llama **copy-on-write** ("copiar al escribir"): no copies nada por adelantado — recién cuando alguien quiera escribir, copiá. Es lo que hace baratísimo *arrancar* un container (la capa nace vacía: cero copias) y lo que esconde la trampa: **el copy-up no copia el cambio — copia el archivo.** Un carácter agregado a un archivo de 100 MB congelado = 100 MB nuevos en tu capa read-write, con la versión vieja intacta abajo. Si venís de git, el contraste ilumina: git guarda *deltas* (la diferencia, unas líneas); el overlay copia *el archivo entero*, sea texto o binario. Son herramientas para problemas distintos — pero el instinto de "cambié poquito, ocupa poquito" acá **no aplica**.

**Sentilo en los dedos (experimento 4)** — el clásico: un archivo grande congelado, y un solo punto agregado. Carpeta nueva, Dockerfile de dos líneas:

```dockerfile
FROM python:3.11-slim
RUN dd if=/dev/urandom of=/grande.bin bs=1M count=100
# └─ dd: herramienta que copia bytes en crudo. Acá fabrica /grande.bin con
#    100 MB de bytes aleatorios (/dev/urandom es el "grifo" de azar de Linux).
#    Queda CONGELADO en una capa de la imagen.
```

```console
$ docker build -t experimento-cow .
$ docker run -d --name cow experimento-cow sleep 600
# └─ sleep 600: un PID 1 que solo duerme 10 minutos — mantiene vivo el
#    container mientras experimentamos (¡override del CMD, módulo 4!)

$ docker ps -s
CONTAINER ID   IMAGE            ...   NAMES   SIZE
9c1f...        experimento-cow  ...   cow     0B (virtual 235MB)
# └─ -s = size: la columna SIZE es TU CAPA READ-WRITE. Ahora: 0B — recién nacida,
#    vacía. Lo "virtual" es imagen + capa (todo lo que el container ve).

$ docker exec cow sh -c 'echo "." >> /grande.bin'      # ← UN punto al final del archivo
# └─ Dos notas de forma: `exec` sin -it ejecuta el comando y vuelve — la shell
#    interactiva del módulo 4 solo hace falta para quedarse adentro. ¿Y el `sh -c`?
#    El `>>` ("agregá al final de") es sintaxis DE shell: exec ejecuta comandos a
#    secas, sin shell — así que acá la invocamos NOSOTROS, a propósito y explícita.
#    (La misma shell que el módulo 4 §5.1 te enseñó a NO dejar que se cuele sola.)

$ docker ps -s
CONTAINER ID   IMAGE            ...   NAMES   SIZE
9c1f...        experimento-cow  ...   cow     ≈105MB (virtual 235MB)
# └─ 💥 UN BYTE de cambio → ≈100MB en la capa read-write: el copy-up
#    copió el archivo entero. Y la versión original sigue congelada abajo.
```

Un punto. Cien megas. Esa es la lección — y su moraleja operativa la vas a usar en el TP: los archivos que un container va a *modificar* (datos, uploads, bases) **no deberían vivir en capas congeladas**. ¿Dónde entonces? Módulo 6.

## 5. 🔴 `docker diff`: el espía de la capa read-write

Hay un comando hecho a medida de este módulo: te muestra **exactamente qué difiere** entre un container y su imagen — o sea, el contenido de la capa read-write, clasificado:

```console
$ docker diff cow
C /grande.bin        # ← C = Changed: el copy-up del experimento, en el registro
```

Tres letras posibles: **A** (*Added*: archivo nuevo en la capa), **C** (*Changed*: copy-up hecho), **D** (*Deleted*: marca de borrado). Vale la pena ver la D en acción, junto con la demostración final de que la imagen es intocable:

```console
$ docker exec cow rm -rf /var        # ← el container "borra" /var de su mundo
$ docker exec cow ls /
bin  boot  dev  etc  grande.bin  home  ...  usr        # ← sin /var: para ÉL, no existe más

$ docker diff cow
C /grande.bin
D /var                               # ← ahí está: la marca de borrado
D /var/cache
D /var/lib
...
```

¿Rompimos algo? Solo el mundo privado de `cow`. Ahora el remate — borrá ese container y parí uno nuevo de la **misma** imagen:

```console
$ docker rm -f cow                   # (-f: fuerza stop+rm en un paso, para vagos con apuro)
$ docker run --rm experimento-cow ls /
bin  boot  dev  etc  grande.bin  home  ...  var        # ← /var VOLVIÓ
```

No "volvió": **nunca se fue.** La imagen estaba congelada; el borrado vivía en la capa read-write de `cow`, y murió con él. Container nuevo = capa nueva y vacía = la imagen impecable otra vez. El rollo de película no se raya por lo que pase en una sala.

## 6. 🔴 Los peligros de la capa read-write

Todo el módulo converge en dos advertencias serias — una de espacio, una de datos.

**Peligro 1: el container que se come el disco del host.** La capa read-write no tiene límite propio, y vive (convención de la serie: siempre decimos dónde) en `/var/lib/docker` — el disco de la VM en Mac/Windows, tu disco directo en Linux. Un container que escribe sin control — el caso clásico: **una app logueando a archivos internos, creciendo para siempre** — infla su capa, que infla `/var/lib/docker`, que llena el disco… y cuando el disco del host se llena, **se llena para todos**: los demás containers no pueden escribir, la máquina entera sufre. ¿Te suena? Es el **bad neighbor** del módulo 1, renacido a nivel disco. Y acá cierra con todas las letras la buena práctica plantada en el módulo 4: **una app en un container loguea a stdout** — no solo para que `docker logs` la vea: para que los logs *no vivan en la capa read-write* engordando sin techo. (En sistemas serios, esos logs de stdout se evacúan a un sistema central — tema de más adelante en la materia.)

**Peligro 2: tus datos son efímeros — y ahora lo sabés con precisión mecánica.** Junta las piezas de los módulos 4 y 5: los datos viven en la capa read-write → la capa muere con `docker rm` → **los datos mueren con ella.** El escenario del frío en la espalda, completo: tu container corre la base de datos del TP; cada alta de usuario es una escritura en la capa read-write; un día alguien recrea el container — un `rm` de limpieza, un redeploy, un update de la imagen — y la base **nace vacía**. Sin backup, sin undo, sin drama en los logs: simplemente no está más. Esto no es un bug de Docker: es el diseño — los containers son ganado descartable, no mascotas con memoria. Los datos que deben sobrevivir al container **no pueden vivir en la capa read-write**. Dónde sí: **volúmenes**, la primera mitad del módulo 6. El hilo H5 queda oficialmente al rojo vivo.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué es copy-on-write y qué implicancias tiene?** Es la política de la capa read-write del container sobre la imagen congelada: los archivos nuevos se escriben directo en la capa; modificar un archivo de la imagen dispara un copy-up (se copia el archivo COMPLETO a la capa read-write y se modifica la copia — aunque el cambio sea un byte, y la versión original sigue ocupando espacio abajo); borrar solo agrega una marca de borrado que lo oculta sin liberar espacio. Implicancias: arrancar containers es barato (la capa nace vacía), pero modificar archivos grandes congelados es caro, la capa read-write puede crecer hasta llenar el disco del host (por eso se loguea a stdout y no a archivos), y todo lo escrito es efímero — muere con `docker rm` — por lo que los datos persistentes requieren volúmenes.

## 7. 🟡 El medidor y la escoba

Después de cinco módulos de pulls, builds y experimentos, tu disco acumuló capas, imágenes viejas y containers muertos. Las dos herramientas de higiene:

**El medidor:**

```console
$ docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          4         2         ≈450MB    ≈220MB (48%)     # ← lo recuperable: imágenes sin uso
Containers      3         2         ≈105MB    ≈105MB           # ← capas read-write de los muertos
Local Volumes   0         0         0B        0B               # ← protagonistas del módulo 6
Build Cache     12        0         ≈130MB    ≈130MB           # ← las capas cacheadas de la sección 1
```

Una radiografía de cuánto te está comiendo Docker y cuánto es rescatable. Con `-v` (*verbose*) te lo desglosa ítem por ítem. *(Los números tuyos serán otros — lo que importa es leer las columnas.)*

**La escoba, en tres tamaños:**

```console
$ docker container prune      # barre los containers Exited (te pide confirmación)
$ docker image prune          # barre imágenes "colgadas" (capas huérfanas de rebuilds viejos)
$ docker system prune -a      # ⚠️ LA ESCOBA GRANDE: containers muertos, redes sin uso,
                              #    caché de build, Y TODA IMAGEN sin un container vivo usándola.
                              #    Poderosa y sin remordimientos: lo borrado se re-descarga
                              #    o re-buildea después, sí — pero pagando de nuevo la espera.
```

Regla práctica: `system df` primero, escoba después, y la `-a` solo cuando sabés lo que estás resignando. Para tus experimentos de esta serie: `docker container prune` + `docker image prune` de vez en cuando alcanza y sobra.

## 8. 🔴 Síntesis — hilos, y lo que le importa a la cátedra

| Hilo | Estado |
|---|---|
| **H4** — ¿el build por qué tarda / por qué vuela? | ✅ **Cerrado**: la caché de capas + la cascada + el orden del Dockerfile |
| **H5** — ¿dónde quedan mis datos? | 🔥 **Al rojo**: efímeros en la capa read-write, muertos con el `rm`. Cura: módulo 6 |

Y la traducción directa a tu TP, porque este módulo ES la parte evaluada: cuando la cátedra abra tu Dockerfile va a mirar exactamente esto — ¿el archivo de dependencias se copia antes que el código? ¿la instalación queda cacheada entre builds? ¿la limpieza va encadenada en su capa? ¿los datos y logs viven fuera de las capas? Todo lo que hiciste con las manos en este módulo es la respuesta.

---

## ✅ Checkpoint del Módulo 5

*Sin mirar el material. Las que no salgan marcan qué sección releer. (Las respuestas no están acá a propósito.)*

1. ¿Cómo decide Docker si reutiliza una capa en el build? ¿Qué compara para un `RUN` y qué para un `COPY`?
2. Enunciá la regla de la cascada de invalidación. ¿Por qué una capa cambiada obliga a re-cocinar TODAS las siguientes?
3. ¿Qué va primero en un Dockerfile bien ordenado: el código o la instalación de dependencias? ¿Por qué?
4. ¿Cuál es el truco de copiar `requirements.txt` / `package.json` ANTES que el resto del código?
5. ¿Por qué el orden del Dockerfile "es plata"? Contá el argumento de escala (CI, servicios, builds por día).
6. ¿Qué dos equilibrios evitan el fanatismo del orden perfecto?
7. Copy-on-write: ¿qué pasa por debajo al crear, modificar y borrar un archivo desde el container? ¿Cuál de las tres operaciones esconde el costo grande?
8. Un container agrega un byte a un archivo congelado de 100 MB. ¿Cuánto crece su capa read-write y por qué? ¿En qué se diferencia esto de cómo guarda cambios git?
9. ¿Qué muestra `docker diff` y qué significan A, C y D?
10. Un container borra `/var` y después lo borrás a él. ¿Qué ve un container nuevo de la misma imagen, y por qué?
11. ¿Por qué loguear a archivos dentro del container es doblemente mala práctica? ¿Qué cadena de desastre puede disparar? ¿Qué viejo conocido del módulo 1 reaparece?
12. Reconstruí el argumento completo de por qué una base de datos NO puede vivir en la capa read-write.
13. ¿Qué muestra `docker system df`? ¿Qué diferencia hay entre `docker image prune` y `docker system prune -a`, y por qué el segundo lleva ⚠️?

---

## Qué viene en el Módulo 6

La cura y la conexión. Primero, la cura del H5: los **volúmenes** — los tres tipos de montaje (named volumes, bind mounts y tmpfs), cuál usar para la base de datos del TP, cuál para desarrollar con el código recargándose en vivo, y por dónde entra la configuración que el módulo 3 dejó deliberadamente fuera de la imagen. Después, la conexión: **redes** — cómo dos containers se encuentran y se hablan entre sí *sin* pasar por los puentes de puertos, cómo tu app va a encontrar a su base de datos llamándola por su nombre, y el cierre definitivo del hilo H6.

**FIN DEL MÓDULO 5**
