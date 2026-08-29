# 🐳 Clase desde Cero — Docker · Módulo 7
## Orquestar y distribuir: Compose, el registry y los tags

**Serie:** Clase desde Cero — Docker · Módulo 7 de 7 — el final · El mapa completo está en `clase-cero-docker-roadmap.md`

---

### Sobre este documento

**Qué cubre:** la cura del hilo H7 — **Docker Compose**: tu arquitectura entera declarada en un YAML y levantada con un comando · la traducción exacta de todo lo manual (módulos 4-6) a líneas del archivo · la red automática con resolución por nombre · escalar servicios a N réplicas · **el caso real de la materia** para practicarlo todo · y la pieza final del trío del módulo 3: el **registry** a fondo, con el naming completo de las imágenes, el flujo `push`/`pull`, y la verdad sobre los **tags** y el traicionero `latest`.

**Qué NO cubre:** nada — es el último. Lo que la serie deja abierto a propósito (Kubernetes, CI/CD, observabilidad) pertenece a clases futuras de la materia, y vas a llegar a ellas con este piso.

**Módulo de hacer**, con tres niveles: un ejemplo anotado para leer, una demo mínima para correr, y el repositorio real de la cátedra como práctica completa.

### De dónde venís

De los módulos 4-6 traés todas las piezas sueltas: build, run con sus flags, volúmenes, variables de entorno, redes con resolución por nombre. Y traés el dolor final: levantar el ambiente son seis comandos largos, en orden, cada vez, por cada compañero, todos los días.

---

## 1. 🔴 El infierno de los seis comandos — visto de frente

Esto es levantar "a mano" el stack del TP con lo aprendido hasta ayer:

```console
$ docker network create red-tacs                      # 1. la red
$ docker volume create datos-db                       # 2. el volumen
$ docker run -d --name db --network red-tacs \        # 3. la base: su volumen,
    -v datos-db:/var/lib/mysql \                      #    su password, SIN -p
    -e MYSQL_ROOT_PASSWORD=secreta imagen-de-mysql
$ docker build -t mi-app:1.0 .                        # 4. cocinar la app
$ docker run -d --name app --network red-tacs \       # 5. la app: su puente,
    -p 8080:8080 -e DB_HOST=db mi-app:1.0             #    su config, su red
$ # 6. ...y rezar haber tipeado todo igual que ayer
```

Funciona — lo probaste pieza por pieza. Pero es un **ritual oral**: vive en tu memoria (o en un README que alguien desactualiza), se tipea con errores, y cada compañero lo ejecuta *ligeramente distinto*. ¿No te suena de algún lado? Es el **README de instalación del módulo 1**, reencarnado un nivel más arriba: ya no "instalá Node y la base a mano", ahora "tirá estos seis comandos a mano". Y la solución es la misma que aquella vez: **dejar de contar la receta y escribirla** — que la arquitectura sea un archivo, versionado en el repo, idéntico para todos.

Ese archivo existe, se llama **Docker Compose**, y su lema cabe en una línea: *definí tu aplicación multi-container en un YAML; levantala entera con un comando.*

## 2. 🔴 El archivo: anatomía de `compose.yaml`

- **YAML:** un formato de texto para datos estructurados, primo de JSON pero sin llaves ni comillas obligatorias — la jerarquía se marca con **indentación** (⚠️ espacios, no tabs; dos espacios por nivel es la convención). Lo vas a leer de corrido en un minuto.

El archivo se llama `compose.yaml` (o `docker-compose.yml` — Docker acepta ambos; el primero es el nombre moderno) y vive en la raíz del proyecto. Acá va el stack del TP completo, línea por línea — **cada renglón es algo que ya sabés hacer a mano**:

```yaml
# compose.yaml — el stack del TP, declarado

services:                        # ═══ SECCIÓN 1: los containers de tu app ═══

  app:                          # ← nombre del servicio. Elegís vos. Este nombre
    build: .                    #    ES el nombre de red (¡resolución por nombre!)
    #     └─ "cociná el Dockerfile de esta carpeta" — el docker build -t de siempre
    ports:
      - "8080:8080"             # ← el -p del módulo 4: puente host:container
    environment:                # ← los -e del módulo 6, puerta 1
      - DB_HOST=db              #    fijate el VALOR: "db" — ¡el nombre del otro servicio!
      - DB_PASSWORD=${DB_PASSWORD}
    #                └─ ${...} = "leelo del archivo .env de esta carpeta":
    #                   los secretos NO se escriben acá (el compose.yaml SÍ va al repo;
    #                   el .env NO va — la separación del módulo 6, automatizada)
    depends_on:
      - db                      # ← orden de arranque: primero db, después app
    networks:
      - backend                 # ← el --network del módulo 6

  db:                           # ← el segundo container
    image: mysql:8.4            # ← sin build: usa una imagen ya hecha. Versión FIJADA
    #                                (nada de latest — sección 9 explica por qué)
    environment:
      - MYSQL_ROOT_PASSWORD=${DB_PASSWORD}
    volumes:
      - datos-db:/var/lib/mysql # ← el -v del módulo 6: el volumen, montado donde
    networks:                   #    el motor guarda los datos
      - backend
    # 👀 sin "ports:": la base NO tiene puente a la calle — invisible desde afuera,
    #    visible para sus vecinos de red. El patrón de seguridad del módulo 6, gratis.

volumes:                         # ═══ SECCIÓN 2: los volúmenes con nombre ═══
  datos-db:                     # ← el docker volume create, declarado

networks:                        # ═══ SECCIÓN 3: las redes ═══
  backend:                      # ← el docker network create, declarado
```

⚠️ **Dos avisos de arqueología**, porque el mundo está lleno de ejemplos viejos y vas a cruzarlos seguro: (1) muchos YAML arrancan con `version: "3"` — ese campo **está obsoleto**: los Compose modernos lo ignoran; no lo copies. (2) Vas a ver el comando escrito `docker-compose` **con guion** — era un programa aparte; hoy Compose viene integrado y se escribe `docker compose`, con espacio. Si un tutorial usa el guion, es viejo: traducí y seguí.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué es Docker Compose y cómo se estructura su archivo?** Es la herramienta para definir y correr aplicaciones multi-container: la arquitectura se declara como código en un YAML (`compose.yaml`) y se levanta entera con `docker compose up`. El archivo tiene tres secciones: `services` (cada container: su imagen o su build, puertos, variables de entorno, volúmenes montados, redes, dependencias de arranque), `volumes` (los volúmenes con nombre) y `networks` (las redes). Ventajas: la arquitectura queda versionada en el repo, idéntica y reproducible para todo el equipo, reemplazando una secuencia de comandos manuales por un archivo y un comando.

## 3. 🔴 La tabla de traducción — tres módulos en un archivo

Todo lo que aprendiste a mano tiene su renglón en el YAML. Esta tabla es el resumen de medio curso:

| A mano (módulos 4-6) | En el compose.yaml |
|---|---|
| `docker build -t ...` | `build: .` |
| elegir imagen en el `run` | `image: mysql:8.4` |
| `--name` | el **nombre del servicio** (la clave del YAML) |
| `-p 8080:8080` | `ports:` |
| `-e CLAVE=valor` | `environment:` (+ `.env` para secretos) |
| `-v volumen:/ruta` | `volumes:` del servicio |
| `docker volume create` | sección `volumes:` de abajo |
| `--network red` | `networks:` del servicio |
| `docker network create` | sección `networks:` de abajo |
| pasarle un comando al `run` (override del CMD) | `command:` |
| **los seis comandos en orden** | **`docker compose up`** |

Nada nuevo bajo el sol: Compose no inventa capacidades — **declara las que ya tenías**. Si algún día un YAML te resulte críptico, la pregunta que lo destraba es siempre "¿qué comando manual es esta línea?".

## 4. 🔴 Operarlo: up, down, y la familia

```console
$ docker compose up -d        # levanta TODO: crea red, volúmenes, containers — en orden
$ docker compose ps           # censo, pero solo de ESTE proyecto
$ docker compose logs -f app  # los logs de un servicio (o de todos, sin nombre)
$ docker compose stop         # detiene los containers (los datos quedan — es un stop)
$ docker compose up -d        # ...y los vuelve a levantar
$ docker compose down         # detiene Y BORRA containers y red del proyecto
```

Detalles que importan:

- **`up` es idempotente e inteligente:** si ya está todo arriba, no hace nada; si cambiaste el YAML o el código (con `build:`), recrea *solo lo que cambió* (`docker compose up -d --build` fuerza la re-cocción de las imágenes propias).
- **`down` ≠ `stop`, y es la versión proyecto-entero del módulo 4:** `stop` detiene (reversible, datos de las capas intactos); `down` **borra** los containers y la red. ¿Y los volúmenes? **Sobreviven al `down`** — la virtud del módulo 6 se respeta: tu base de datos aguanta el ciclo down/up sin perder nada.
- ⚠️ **`docker compose down -v` es la excepción atómica:** el `-v` borra TAMBIÉN los volúmenes del proyecto. Existe para "quiero empezar de cero de verdad" — y es exactamente el comando que no querés tirar por reflejo un domingo a la noche con la base cargada.
- 🟡 Sobre `depends_on`, la verdad completa, porque es LA trampa del primer `up` de todo el mundo: garantiza el **orden de arranque** (primero db, después app)… pero "arrancó" no es "está lista". Un motor de base tarda unos segundos entre que su proceso existe y que acepta conexiones — y en ese hueco tu app, que arrancó obedientemente segunda, intenta conectarse, recibe el rechazo, **explota y queda `Exited`**… mientras la base termina de despertarse, ya sin nadie que le hable. El síntoma clásico: "el compose levanta todo pero la app se muere sola al arrancar, y si la arranco a mano después, anda". La salida recomendada — y la que tu app del TP debería tener — no es de Docker sino **de tu código: reintentar la conexión** (varios intentos con espera entre ellos) en vez de morir al primer rechazo. No es un parche: es resiliencia que sirve siempre — la base puede reiniciarse en cualquier momento de la vida, no solo en el arranque.

> 🕳️ **Madriguera — healthchecks**
> La solución fina existe: definirle al servicio de la base un *healthcheck* (una prueba periódica de "¿estás sana?") y que el `depends_on` espere la condición de saludable, no el mero arranque. Es configuración adicional en el compose, está bien documentada, y queda fuera de esta serie: con los reintentos en la app te alcanza y te sobra.
> *Volvé al camino.*

## 5. 🔴 La red automática — la trampa del módulo 6, desactivada de fábrica

Releé el YAML de la sección 2 y notá lo que **no** hice: la sección `networks` es opcional. Si la omitís, Compose **crea sola una red para el proyecto** y mete a todos los servicios adentro. Y acá viene el regalo: esa red automática es una red *definida por el usuario* — **con guía telefónica incluida**. Los servicios se resuelven entre sí **por su nombre de servicio**, de entrada, sin configurar nada. La trampa clásica del módulo 6 (la red default de Docker sin DNS) no existe en el mundo Compose: `DB_HOST=db` funciona porque `db` es el nombre del servicio, y punto.

**Sentilo en dos archivos (demo mínima)** — en tu carpeta de `mi-app`, creá:

```yaml
# compose.yaml
services:
  api:
    build: .
    ports:
      - "8080:8080"
  visitante:
    image: mi-app:1.0
    command: >
      python3 -c "from urllib.request import urlopen; import time; time.sleep(2);
      print(urlopen('http://api:8080').read().decode())"
    # └─ el command: (override del CMD, módulo 4) espera 2 segundos y llama
    #    a http://api:8080 — "api" por NOMBRE DE SERVICIO
```

```console
$ docker compose up
[+] Running 3/3
 ✔ Network miapp_default        Created     # ← la red automática del proyecto
 ✔ Container miapp-api-1        Created     # ← nombres: proyecto-servicio-réplica
 ✔ Container miapp-visitante-1  Created
api-1        | (arranca y espera)
visitante-1  | Hola desde Docker!            # ← 💥 encontró a "api" por nombre, de fábrica
```

Un archivo, un comando, dos containers, una red con DNS — y la prueba de vida impresa. Cortá con `Ctrl+C` y limpiá con `docker compose down`.

## 6. 🟡 Escalar: el mismo servicio, N veces

Compose puede levantar **varias réplicas** de un servicio:

```console
$ docker compose up -d --scale api=3
```

Tres containers de `api`, todos en la red, todos respondiendo al mismo nombre — y cuando alguien llama a `http://api:8080`, la guía telefónica **reparte** entre las réplicas (llamados sucesivos pueden caer en containers distintos: eso es *service discovery* con balanceo, la semilla de cómo escalan los sistemas grandes). Dos letras chicas: (1) un servicio escalado no puede tener `ports:` fijo (tres containers no comparten un puente — el módulo 4 te explicó por qué) ni **`container_name`** — un nombre fijo no puede repetirse tres veces; por eso en YAMLs bien diseñados solo llevan `container_name` los servicios que jamás escalan. (2) En material viejo el comando figura como `docker-compose scale api=3` — arqueología, ya sabés traducir.

## 7. 🔴 El caso real de la materia: dos YAML, una lección

La cátedra tiene un repositorio público hecho exactamente para practicar esto — cuatro servicios y **dos archivos compose que difieren en una sola cosa: las redes**. Es el patrón de segmentación del módulo 6, servido para tocar:

```console
$ git clone https://github.com/tacs-utn/docker-networking.git
$ cd docker-networking
```

⚠️ El repo tiene sus años: su README usa `docker-compose` con guion y sus YAML traen `version:` — arqueología de la sección 2, que ahora leés sin pestañar.

**Ronda 1 — todos en la misma red** (`docker-compose.yaml`, el archivo por defecto):

```console
$ docker compose up -d
[+] Running 5/5   (red + gateway, backend-for-frontend, backend, database)

$ docker exec gateway curl -s backend-for-frontend:8080 | head -3
<!doctype html><html lang="en">...            # ← el gateway alcanza al BFF: responde
$ docker exec gateway curl -s database:8080 | head -3
<!doctype html><html lang="en">...            # ← ¡y TAMBIÉN alcanza a la base! 😨
```

*(Los servicios responden una página de bienvenida — el contenido da igual: la señal es que **responden**.)* Todo se ve con todo: funciona… y es el edificio sin puertas por dentro. El gateway — la pieza expuesta al mundo — tiene línea directa a la base de datos.

**Ronda 2 — la arquitectura segmentada** (`docker-compose-network.yaml` — el mismo stack, tres redes: el gateway con el BFF, el BFF con el backend, el backend con la base):

```console
$ docker compose down
$ docker compose -f docker-compose-network.yaml up -d
#                └─ -f: "usá ESTE archivo" en vez del default

$ docker exec gateway curl -s backend-for-frontend:8080 | head -3
<!doctype html>...                             # ← su vecino de red: responde ✅
$ docker exec gateway curl -s database:8080
curl: (6) Could not resolve host: database     # ← 💥 la base NO EXISTE para el gateway:
                                               #    ni siquiera figura en su guía telefónica
```

Mismo stack, misma imagen, mismos containers — **otro mapa de redes, otro universo de quién ve a quién**. La base quedó a dos redes de distancia de la pieza expuesta: si alguien compromete el gateway, la base ni aparece en su horizonte. Esta decisión — *dibujar quién habla con quién* — es de las cosas que la cátedra evalúa explícitamente en el compose de tu TP. Yapas para jugar: `docker exec gateway ip r` te muestra las redes que cada container ve (compará entre servicios), y `docker compose -f docker-compose-network.yaml up -d --scale backend-for-frontend=3` + varios curls repetidos te dejan ver el reparto entre réplicas de la sección 6 (fijate en el YAML: gateway y database tienen `container_name` — por eso justamente *ellos* no se pueden escalar). Al terminar: `docker compose -f docker-compose-network.yaml down`.

## 8. 🔴 El registry: la pieza final del trío

El módulo 3 presentó el trío — imagen, container, **registry** — y desarrolló dos. Cerremos el tercero: el registry es **el servidor donde las imágenes viven y viajan** — un depósito con API: le subís imágenes (*push*), le bajás imágenes (*pull*), las versiona por tags. GitHub, pero de imágenes: viene alimentando la serie desde las sombras — cada `Unable to find image locally → Pulling` de los módulos anteriores fue un viaje al registry.

**El naming completo** — ahora sí, el nombre verdadero de una imagen:

```
     docker.io / library / ubuntu : 22.04
     └────┬───┘ └───┬───┘ └──┬───┘ └──┬──┘
      registry    usuario   imagen   tag
      (el server) (la cuenta)        (la versión)
```

Con los defaults que explican todo lo que viviste: sin registry → **`docker.io`** (Docker Hub); `library` → la cuenta de las **imágenes oficiales** (por eso tus pulls decían `Pulling from library/python`); sin tag → **`latest`** (sección 9). Otros registries se nombran por su URL: `ghcr.io/usuario/imagen` (GitHub), o las URLs largas de los registries de la nube.

**El flujo completo para publicar tu imagen** (con el Docker ID del setup):

```console
$ docker login                                  # una vez: tus credenciales del Hub
$ docker tag mi-app:1.0 tuusuario/mi-app:1.0    # "bautizala TAMBIÉN con tu espacio adelante"
$ docker image ls
REPOSITORY          TAG    IMAGE ID       ...
mi-app              1.0    91c2f4a8b3d0   ...   # ← ¡MISMO IMAGE ID!
tuusuario/mi-app    1.0    91c2f4a8b3d0   ...   # ← docker tag no copió nada:
                                                #    son DOS nombres para UN artefacto
$ docker push tuusuario/mi-app:1.0              # sube las capas (las que el Hub no tenga)
$ docker pull tuusuario/mi-app:1.0              # ...y desde cualquier máquina del mundo
```

Ese `IMAGE ID` repetido es la revelación de la sección: **los nombres son etiquetas pegadas a un hash** — el artefacto es uno, inmutable, identificado por su contenido; los nombres son alias. Guardala: la sección 9 la lleva al límite.

🟡 **Públicos y privados.** El Hub es el registry público; las empresas suelen tener el suyo **privado**: por confidencialidad (tu imagen contiene tu código), por control y por cumplimiento normativo (un banco no sube sus artefactos a un depósito público). Dos sabores: *administrado por el cloud* (el de AWS, por ejemplo, guarda sobre su almacenamiento y **cobra por transferencia** — imágenes de 2 GB versionadas a diario se facturan solas: otra razón económica para el Alpine del módulo 3 y el orden del módulo 5) o *auto-hosteado* (con software como Harbor — más control, más responsabilidad). Y la responsabilidad es seria: el registry es un **punto único de falla** — si se cae, nadie puede desplegar ni escalar (crear una réplica nueva empieza con un pull). Tu app puede seguir corriendo; tu capacidad de *cambiarla* quedó rehén.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué es un registry y cómo se nombra una imagen?** El registry es el repositorio donde se almacenan y distribuyen imágenes vía push/pull (Docker Hub es el público por defecto). El nombre completo de una imagen es `[registry]/[usuario]/[imagen]:[tag]` — con defaults: sin registry se asume docker.io, `library` es la cuenta de las imágenes oficiales, y sin tag se asume `latest`. Las empresas usan registries privados (auto-hosteados, como Harbor, o administrados por el cloud) por confidencialidad, control y compliance — asumiendo su disponibilidad como crítica: un registry caído impide desplegar y escalar (todo despliegue empieza con un pull), y su costo de almacenamiento y transferencia hace que el tamaño de las imágenes sea también una variable económica.

## 9. 🔴 Los tags — y la verdad sobre `latest`

Última pieza de toda la serie, y una que desarma un malentendido casi universal.

Un **tag** es una etiqueta móvil pegada a un hash. De la sección 8 ya sabés que varios nombres pueden apuntar al mismo artefacto — ahora mirá el sistema completo en acción, con la historia clásica del **versionado semántico** (el esquema `mayor.menor.parche` que ya conocés de npm):

```
   Publicás la 1.0.1:                     Sale un hotfix, la 1.0.2:
   mi-app:1.0.1 ──┐                       mi-app:1.0.1 ──▶ (hash A)   ← quieto
   mi-app:1.0   ──┼──▶ (hash A)           mi-app:1.0.2 ──┐
   mi-app:latest──┘                       mi-app:1.0   ──┼──▶ (hash B)  ← SE MOVIERON
                                          mi-app:latest──┘
```

Tres etiquetas, un artefacto — y cuando publicás el parche, **las etiquetas gordas se mueven al hash nuevo** mientras las precisas se quedan clavadas. El sistema es un contrato entre quien publica y quien consume: **el tag que elegís al hacer pull (o en tu `FROM`) es tu política de riesgo**:

| Pull de… | Recibís | Política |
|---|---|---|
| `mi-app:1.0.1` | Ese hash exacto, para siempre | Máxima reproducibilidad — cero sorpresas |
| `mi-app:1.0` | El último parche de la 1.0 | Hotfixes automáticos, sin cambios grandes |
| `mi-app:latest` | Lo último que se haya pusheado, sea lo que sea | La ruleta |

Y acá el malentendido a fusilar: ⚠️ **`latest` no significa "estable", ni "recomendada", ni siquiera garantiza "la más nueva" — es solo el tag por defecto**: el que se aplica cuando alguien pushea sin especificar. Es una etiqueta como cualquiera, que apunta adonde el último push descuidado la haya dejado. Un `FROM python:latest` significa "mi build de mañana puede usar otro Python que el de hoy" — el fantasma de "en mi máquina funciona" volviendo por la puerta del tiempo: misma receta, resultado distinto según el día. Por eso la práctica que tu TP debe mostrar (y que venís haciendo desde el módulo 3 sin saberlo): **versiones fijadas, siempre** — `python:3.11-slim`, `mysql:8.4`, jamás `latest` en un `FROM` ni en un compose.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué es el tag `latest` y qué problema tiene?** Es simplemente el tag por defecto — el que se asume al hacer pull, push o build sin especificar versión. No garantiza que la imagen sea la más nueva ni la más estable: es una etiqueta móvil que apunta al último push que no especificó tag. Usarlo (en un FROM o un compose) rompe la reproducibilidad: el mismo build puede dar resultados distintos según el día. La buena práctica es fijar versiones explícitas (`python:3.11-slim`), reservando la decisión de cuánta actualización aceptar a la elección del tag: exacto (máxima reproducibilidad), menor (recibir parches) o latest (sin garantías).

## 10. 🔴 Fin de la serie: mirá desde dónde venís

Los siete hilos, todos saldados:

| Hilo | Abierto en | Cerrado en | Con |
|---|---|---|---|
| H1 — Aislar cuesta carísimo | M1 | M2 | Namespaces + cgroups: la burbuja |
| H2 — "En mi máquina funciona" | M1 | M3 | La imagen: la máquina viaja con la app |
| H3 — ¿Qué disco ve el container? | M2 | M3 | La imagen, proyectada por el mount |
| H4 — ¿El build por qué tarda/vuela? | M3 | M5 | La caché de capas y el orden de la receta |
| H5 — ¿Dónde quedan mis datos? | M4 | M6 | Volúmenes: fuera de las capas |
| H6 — ¿Cómo se hablan? | M1 | M6 | Redes propias, resolución por nombre |
| H7 — Levantar todo a mano | M6 | **M7** | Compose: un YAML, un comando |

En el módulo 1, "correr dos servidores en una máquina" era un choque de puertos sin solución. Hoy tenés: el modelo mental completo (proceso + burbuja + kernel compartido), el paquete que mata el "en mi máquina funciona", la receta ordenada para que los builds vuelen, los datos a salvo de los containers descartables, la arquitectura segmentada hablándose por nombre, todo declarado en un archivo del repo, y tu imagen lista para viajar por el mundo con nombre y versión. **Eso es exactamente el esqueleto de infraestructura que tu TP necesita** — lo que queda es aplicarlo a tu app real, y ese trabajo ya no es de esta serie: es tuyo, y estás equipado.

Después de la práctica llega el **apunte maestro** de la unidad — que vas a leer como repaso, comprobando cuánto de él ya es tuyo. Los checkpoints de los 7 módulos siguen ahí para cuando el parcial asome.

---

## ✅ Checkpoint del Módulo 7

*Sin mirar el material. Las que no salgan marcan qué sección releer. (Las respuestas no están acá a propósito.)*

1. ¿Qué problema resuelve Compose, y por qué es "el README de instalación del módulo 1, resuelto un nivel más arriba"?
2. Las tres secciones del compose.yaml: ¿qué declara cada una? ¿Qué comandos manuales reemplazan `ports:`, `environment:`, `volumes:` (del servicio) y `build:`?
3. ¿Por qué los secretos van en `.env` y no en el compose.yaml? ¿Cuál de los dos archivos va al repo?
4. Los dos avisos de arqueología: ¿qué pasa con `version:` y con `docker-compose` (con guion)?
5. `up`, `stop`, `down`, `down -v`: ¿qué hace cada uno, qué sobrevive a cada uno, y cuál lleva ⚠️?
6. ¿Qué garantiza `depends_on` — y qué NO garantiza?
7. La red automática de Compose: ¿en qué se diferencia de la red default de Docker del módulo 6? ¿Por qué `DB_HOST=db` funciona sin configurar nada?
8. ¿Qué hace `--scale api=3`? ¿Por qué un servicio escalado no puede tener `container_name` ni `ports:` fijo? ¿Qué es el reparto por service discovery?
9. En el caso de la cátedra: ¿qué cambia entre los dos YAML, y qué le pasa al `curl` del gateway hacia la base en cada uno? ¿Qué gana la arquitectura segmentada?
10. El naming completo de una imagen: las cuatro partes y los tres defaults.
11. El flujo para publicar: los cuatro comandos, en orden. ¿Qué demuestra que `docker tag` no copia nada?
12. ¿Por qué las empresas usan registries privados? ¿Qué significa que el registry sea un punto único de falla, y qué costo económico tienen las imágenes pesadas?
13. ¿Qué es un tag? Contá la historia del hotfix 1.0.2: ¿qué etiquetas se mueven y cuáles quedan clavadas? ¿Qué política de riesgo expresa cada forma de pull?
14. ¿Qué es `latest` en realidad, qué NO garantiza, y por qué jamás va en un `FROM`?

---

**FIN DEL MÓDULO 7 — Y DE LA SERIE. Clase desde Cero · Docker · TACS**
