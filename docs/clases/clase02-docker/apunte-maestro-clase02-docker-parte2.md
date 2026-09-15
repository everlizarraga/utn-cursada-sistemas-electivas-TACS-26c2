# 📘 Apunte Maestro — Clase 02: Docker
## Parte 2 — Imágenes: capas, Dockerfile, build y registry

**Materia:** TACS · 2C 2026 · **Unidad:** clase02 — Docker
**Parte 2 de 4** · Cubre: qué es una imagen · capas de filesystem y hashes · el Dockerfile completo instrucción por instrucción · RUN vs CMD vs ENTRYPOINT · build y caché · exploración de imágenes (`history`, `inspect`) · el registry, su naming, tags, push/pull · registries privados, disponibilidad y costos.

**De la Parte 1 se asume:** container = proceso aislado con namespaces + cgroups · kernel compartido · los tres conceptos (image, registry, container) · imagen = clase, container = instancia · la convención de lectura: los bloques marcados `👁️ → Parte N` son **vistas adelantadas** — evidencia de un comando que se enseña más adelante, no instrucción para ejecutar.

---

## 1. 🔴 Qué es una imagen

Definición operativa, con sus cinco propiedades:

- **Templates read-only** desde los que se crean containers.
- Derivadas de una **imagen base** (BusyBox, Alpine, Ubuntu, etc.).
- Armadas por un **conjunto de capas** que, combinadas, conforman el filesystem.
- **Construidas** a partir de un **Dockerfile**.
- Cada capa es **hasheada** y **cacheada**.

Dicho con precisión: cada imagen Docker referencia una **lista de capas read-only que representan *deltas* de filesystem** — diferencias, no copias completas. Esas capas se apilan una sobre otra y conforman el **sistema de archivos raíz** del container.

⚠️ **Las imágenes son inmutables.** Nadie puede modificar una imagen en uso. Eso es precisamente lo que garantiza la **reproducibilidad**: la misma imagen, en cualquier máquina, es idéntica.

Un ejemplo mínimo para ver el mecanismo en acción:

```dockerfile
FROM ubuntu:16.10

RUN apt-get update && apt-get install -y apache2

ENTRYPOINT [ "apache2ctl" ]
```

Este Dockerfile crea **3 capas**: `FROM` (imagen base), `RUN` (los cambios sobre esa base) y `ENTRYPOINT` (metadata de la imagen).

## 2. 🔴 Capas del filesystem

**Cada instrucción del Dockerfile genera una capa del sistema de archivos.** Así se ve una imagen real, con sus capas identificadas por hash, la instrucción que las creó y cuánto pesa cada una:

```
┌──────────────────────────────────────────────────────────────┐
│ Container layer (R/W) — Random UUID       All writes go here │  ← del CONTAINER
└──────────────────────────────────────────────────────────────┘
                  Image layers (READ-ONLY)
┌──────────────────────────────────────────────────────────────┐
│ 91e54dfb1179 — COPY app.py                              13 B │
├──────────────────────────────────────────────────────────────┤
│ d74508fb6632 — RUN pip install                      194.5 KB │
├──────────────────────────────────────────────────────────────┤
│ c22013c84729 — RUN apt-get                          1.895 KB │
├──────────────────────────────────────────────────────────────┤
│ d3a1f33e8a5a — FROM ubuntu:22.04                    188.1 MB │
└──────────────────────────────────────────────────────────────┘
```

Tres lecturas del diagrama:

1. **Correspondencia instrucción ↔ capa:** cada `FROM`, `RUN`, `COPY` dejó su capa, con su peso propio. La base pesa 188 MB; el `COPY` del código, 13 bytes.
2. **Todo lo de abajo es READ-ONLY.** Las capas de la imagen están selladas.
3. **Arriba hay una capa distinta:** la **container layer (R/W)**, identificada por un UUID aleatorio, donde van **todas las escrituras**. No pertenece a la imagen sino al container, y es el tema central de la Parte 3.

**El rol del hash.** Las capas están **hasheadas criptográficamente por su contenido**. La consecuencia es directa: **si el contenido no cambia, el hash no cambia, y Docker reutiliza la capa cacheada** en vez de reconstruirla. Eso acelera dramáticamente los builds — y se ve en vivo en §5.

- **Hash criptográfico:** una huella digital calculada a partir del contenido. Mismo contenido → mismo hash siempre; un byte distinto → hash completamente distinto.

De ahí sale la regla de oro de escritura de Dockerfiles: **ordenar las instrucciones de menos a más cambiante, para maximizar los cache hits — el `COPY` del código, siempre al final.** Volvemos sobre esto en §5 con la evidencia.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué es una imagen Docker y cómo está compuesta?** Es un template read-only del filesystem desde el que se crean containers, construido a partir de un Dockerfile y derivado de una imagen base. Internamente es una lista de capas read-only que representan deltas de filesystem: cada instrucción del Dockerfile genera una capa, que se apila sobre las anteriores hasta conformar el filesystem raíz del container. Cada capa se hashea criptográficamente por su contenido, lo que permite cachearlas y compartirlas entre imágenes. Las imágenes son inmutables: no se modifican, garantizando reproducibilidad.

## 3. 🔴 El Dockerfile: anatomía completa

El archivo de texto con la receta de construcción, instrucción por instrucción:

```dockerfile
# Imagen base
FROM ubuntu:22.04
# └─ Punto de partida: sobre qué imagen se construye la nuestra. Toda receta arranca acá.

# Metadatos
LABEL maintainer="tacs@ejemplo.com"
# └─ Etiqueta informativa en la metadata de la imagen (quién la mantiene).

# Ejecuta en tiempo de BUILD (crea capa)
RUN apt-get update && apt-get install -y \
    python3 python3-pip \
    && rm -rf /var/lib/apt/lists/*
# └─ Ejecuta un comando DENTRO de la imagen durante el build y congela el
#    resultado como capa. Acá van tres cosas encadenadas con &&:
#    · apt-get update → descarga el catálogo de paquetes de la distribución
#      (qué paquetes existen, en qué versión, de dónde bajarlos) a /var/lib/apt/lists/
#    · apt-get install -y → instala Python usando ese catálogo (-y: no preguntar)
#    · rm -rf /var/lib/apt/lists/* → borra el catálogo, que ya no hace falta
#    El encadenado con && NO es cosmético: las tres operaciones ocurren dentro
#    de UNA capa, así el catálogo nace y muere antes de que la capa se selle.
#    En instrucciones RUN separadas, la capa del install ya habría quedado
#    congelada CON el catálogo adentro, y esos MB viajarían para siempre.
#    (La barra \ al final es continuación de línea: la instrucción sigue abajo.)

# Copia archivos del host al container
COPY app.py /app/app.py
# └─ Lleva archivos desde la carpeta del proyecto hacia la imagen. Genera capa.

# Directorio de trabajo
WORKDIR /app
# └─ Fija el directorio de trabajo para las instrucciones siguientes y para
#    el container al arrancar (lo crea si no existe).

# Puerto que expone el container (documentación)
EXPOSE 8080
# └─ Documenta qué puerto escucha la app. NO abre ni publica nada:
#    la publicación real se hace con -p al correr el container (Parte 3).

# Comando que corre al iniciar el container
CMD ["python3", "app.py"]
# └─ Metadata: qué se ejecuta cuando el container arranca. No corre durante el build.
```

### 3.1 🔴 RUN vs CMD — los dos tiempos

La distinción más importante del Dockerfile, porque separa los dos momentos de la vida de una imagen:

| | **RUN** | **CMD** |
|---|---|---|
| **Cuándo se ejecuta** | En **build time** (al construir la imagen) | En **runtime** (cuando el container arranca) |
| **Qué produce** | **Crea una capa** con el resultado | No crea capa: es metadata |
| **Cuántas veces** | Una sola, al buildear | Cada vez que nace un container |

```
   BUILD (docker build)                    RUN (docker run)
   ─────────────────────                   ──────────────────────
   se ejecutan los RUN                     NO se ejecuta ningún RUN
   se congelan las capas                   se lee la metadata
   el CMD solo se ANOTA          ───▶      se ejecuta el CMD → PID 1
```

Por eso el container arranca en milisegundos: al correrlo **no se instala nada** — todo se instaló durante el build, una sola vez.

### 3.2 🔴 CMD vs ENTRYPOINT

Ambos definen qué se ejecuta al arrancar el container. La diferencia:

- **CMD**: puede **sobrescribirse** — si el usuario pasa un comando en `docker run`, reemplaza al CMD por completo.
- **ENTRYPOINT**: es el **ejecutable principal**, más difícil de sobrescribir — lo que el usuario pase se agrega como argumento.

| En el Dockerfile | `docker run imagen` | `docker run imagen X` |
|---|---|---|
| `CMD ["python3","app.py"]` | corre `python3 app.py` | corre `X` — el CMD se reemplaza entero |
| `ENTRYPOINT ["python3"]` | corre `python3` | corre `python3 X` — X pasa como argumento |
| `ENTRYPOINT ["python3"]` + `CMD ["app.py"]` | corre `python3 app.py` | corre `python3 X` — el ejecutable queda fijo |

Cuando están los dos, **no se ejecutan dos cosas**: de ambas anotaciones se ensambla **una sola línea de comando** — ENTRYPOINT aporta el ejecutable fijo y CMD, los argumentos por defecto reemplazables.

> 🎓 **Para el parcial, si te preguntan**
> **¿Diferencia entre RUN, CMD y ENTRYPOINT?** RUN se ejecuta en build time y crea una capa en la imagen con el resultado. CMD y ENTRYPOINT no se ejecutan en el build: son metadata que define el comando de arranque del container. CMD es el comando por defecto y se sobrescribe por completo si el usuario pasa un comando en `docker run`; ENTRYPOINT fija el ejecutable principal y lo que el usuario pasa se agrega como argumentos. Combinados, ENTRYPOINT define el ejecutable y CMD sus argumentos por defecto.

## 4. 🔴 Construir una imagen propia

El flujo completo, de cero a container corriendo. Primero los dos archivos del proyecto:

```console
$ mkdir demo-docker && cd demo-docker
```

```python
# app.py — un servidor HTTP mínimo
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):                                        # atiende cada GET
        self.send_response(200)                              # responde 200 OK
        self.end_headers()
        self.wfile.write(b"Hola desde Docker! TACS 2026")    # el cuerpo de la respuesta

HTTPServer(("", 8080), Handler).serve_forever()              # escucha en el 8080, siempre
```

```dockerfile
# Dockerfile
FROM python:3.11-slim     # base con Python ya instalado (slim = recortada)
WORKDIR /app
COPY app.py .
EXPOSE 8080
CMD ["python3", "app.py"]
```

Y la construcción:

```console
$ docker build -t mi-app:1.0 .
```

- **`-t mi-app:1.0`** → nombre y tag de la imagen resultante.
- **`.`** (el punto final) → el **build context**: la carpeta desde donde Docker lee el Dockerfile y los archivos a copiar.

Salida del build, paso por paso:

```console
Sending build context to Docker daemon  3.072kB
Step 1/5 : FROM python:3.11-slim
 ---> 9c900dea9e8f                    # ← ID de la capa base
Step 2/5 : WORKDIR /app
 ---> Using cache
 ---> e972bbdc2ac2
Step 3/5 : COPY app.py .
 ---> 1c98ca19e4a2                    # ← sin "Using cache": esta capa se construyó
Step 4/5 : EXPOSE 8080
 ---> Running in a53003175026         # ← container intermedio para ejecutar el paso
 ---> Removed intermediate container a53003175026
 ---> d2755f5cc662
Step 5/5 : CMD ["python3", "app.py"]
 ---> Running in fcb16294c333
 ---> Removed intermediate container fcb16294c333
 ---> 5288008eab18
Successfully built 5288008eab18       # ← ID de la imagen final
Successfully tagged mi-app:1.0        # ← y su nombre
```

Cada `Step N/5` es una instrucción del Dockerfile, y cada una deja su ID. Fijate también los **containers intermedios**: para ejecutar cada paso, Docker crea un container temporal, aplica la instrucción, guarda el resultado como capa y lo elimina (`Removed intermediate container`). La imagen se construye, literalmente, corriendo containers descartables.

Después del build:

```console
$ docker image ls mi-app
REPOSITORY   TAG    IMAGE ID       CREATED         SIZE
mi-app       1.0    130c19dd7397   2 hours ago     212MB

$ docker run -d -p 8080:8080 mi-app:1.0    # 👁️ → los flags de run se enseñan en la Parte 3
# Abrir: http://localhost:8080
```

*(👁️ vista adelantada: la línea cierra el flujo build → run para ver la app viva, pero `docker run` con sus flags — `-d`, `-p` — es tema de la Parte 3.)*

## 5. 🔴 La caché de capas, en vivo

Volvé a buildear **sin tocar nada**:

```console
$ docker build -t mi-app:1.0 .
Step 1/5 : FROM python:3.11-slim
 ---> 9c900dea9e8f
Step 2/5 : WORKDIR /app
 ---> Using cache                     # ← no lo ejecutó: reusó la capa
 ---> e972bbdc2ac2
Step 3/5 : COPY app.py .
 ---> Using cache
 ---> 6a69bdf40ea7
Step 4/5 : EXPOSE 8080
 ---> Using cache
 ---> 131af7c875f3
Step 5/5 : CMD ["python3", "app.py"]
 ---> Using cache
 ---> 130c19dd7397
Successfully built 130c19dd7397
```

**Todo cacheado.** Ningún paso se re-ejecutó: el hash de cada capa coincide con el de la vez anterior, así que Docker las reutiliza.

Ahora **editá `app.py`** y volvé a buildear:

```console
$ docker build -t mi-app:1.0 .
Step 2/5 : WORKDIR /app
 ---> Using cache                     # ← hasta acá, la caché sobrevive
Step 3/5 : COPY app.py .
 ---> 1c98ca19e4a2                    # ← 💥 el contenido cambió → capa nueva
Step 4/5 : EXPOSE 8080
 ---> Running in a53003175026         # ← y de acá para abajo, TODO se re-ejecuta
Step 5/5 : CMD ["python3", "app.py"]
 ---> Running in fcb16294c333
Successfully built 5288008eab18       # ← nuevo ID de imagen
```

Ahí está el fenómeno completo: el `COPY` detectó el cambio de contenido (hash distinto) y se rehizo, y **todas las instrucciones posteriores perdieron su caché también**, aunque no hubieran cambiado. Es la **invalidación en cascada**: cada capa se construyó apoyada en las anteriores, así que si una cambia, Docker no puede garantizar que las siguientes den el mismo resultado — las rehace todas.

Y de ahí la regla de §2, ahora con evidencia: **de menos a más cambiante, el código al final.** Si el `COPY` del código estuviera antes de la instalación de dependencias, cada cambio de una línea de código re-ejecutaría la instalación completa; con el orden correcto, la capa pesada queda cacheada y solo se rehace la capa liviana del código.

> 🎓 **Para el parcial, si te preguntan**
> **¿Cómo funciona la caché de build y por qué importa el orden de las instrucciones?** Cada capa se identifica por el hash de su contenido: si la instrucción y las capas previas no cambiaron, Docker reutiliza la capa cacheada en vez de reconstruirla. Cuando una capa se invalida, se invalidan en cascada todas las siguientes. Por eso el Dockerfile se ordena de instrucciones menos cambiantes a más cambiantes —dependencias primero, código fuente al final—: así un cambio de código solo reconstruye la capa liviana del código y la capa pesada de dependencias permanece cacheada.

## 6. 🟡 Explorar imágenes: `history` e `inspect`

Tres comandos para auditar cualquier imagen:

```console
$ docker image ls                        # imágenes locales
$ docker history nginx                   # las capas de una imagen
$ docker inspect nginx                   # la metadata completa, en JSON
$ docker history --no-trunc nginx        # ídem history, con los hashes completos
```

**El listado local:**

```console
$ docker image ls
REPOSITORY        TAG             IMAGE ID       CREATED         SIZE
mi-app            1.0             130c19dd7397   2 hours ago     212MB
python            3.11-slim       9c900dea9e8f   5 days ago      212MB
postgres          15              5f72c7b5bd61   5 days ago      649MB
ubuntu            22.04           2edbbc5dc405   8 days ago      107MB
nginx             latest          8541484afbc9   13 days ago     256MB
hello-world       latest          5dd0d3e6e255   4 months ago    17.1kB
<none>            <none>          f2f0dabfe996   10 months ago   77.5MB     # ← ver abajo
node              22              2bb201f33898   10 months ago   1.62GB
nginx             stable-alpine   8f2bcf97c473   16 months ago   75.8MB     # ← ver abajo
```

Dos cosas para leer acá. Primero, **`nginx:latest` pesa 256 MB y `nginx:stable-alpine` 75.8 MB** — la misma aplicación sobre una base mínima ocupa una fracción; por eso las imágenes basadas en **Alpine** (una distribución Linux minimalista pensada para containers) son la opción habitual cuando el tamaño importa. Segundo, las filas **`<none>`**: son imágenes que quedaron sin nombre porque un rebuild reasignó su tag a una imagen nueva — basura recuperable, que se limpia con `docker image prune` (§8).

⚠️ La columna `CREATED` indica cuándo la imagen fue **construida por sus autores**, no cuándo la descargaste.

**Las capas de una imagen real:**

```console
$ docker history nginx
IMAGE          CREATED       CREATED BY                                    SIZE
8541484afbc9   13 days ago   CMD ["nginx" "-g" "daemon off;"]              0B      # ← metadata
<missing>      13 days ago   STOPSIGNAL SIGQUIT                            0B
<missing>      13 days ago   EXPOSE map[80/tcp:{}]                         0B
<missing>      13 days ago   ENTRYPOINT ["/docker-entrypoint.sh"]          0B
<missing>      13 days ago   COPY 30-tune-worker-processes.sh /docker-…    16.4kB
<missing>      13 days ago   COPY 20-envsubst-on-templates.sh /docker-…    12.3kB
<missing>      13 days ago   COPY docker-entrypoint.sh /                   8.19kB
<missing>      13 days ago   RUN /bin/sh -c set -x && groupadd --syst…     84.9MB  # ← la pesada
<missing>      13 days ago   ENV NGINX_VERSION=1.31.3                      0B
<missing>      13 days ago   LABEL maintainer=NGINX Docker Maintainers…    0B
<missing>      2 weeks ago   # debian.sh --arch 'arm64' out/ 'trixie'…     109MB   # ← la base
```

Tres observaciones que valen para cualquier imagen:

1. **Cada línea es una instrucción del Dockerfile**, y `history` las muestra **de arriba hacia abajo en orden inverso**: la primera línea es la última instrucción de la receta.
2. **Las capas de 0B son metadata** (CMD, ENV, EXPOSE, LABEL, ENTRYPOINT, STOPSIGNAL): anotan, no agregan archivos. **Las capas de `RUN` son las más pesadas** — acá, 84.9 MB.
3. **`<missing>`** en la columna IMAGE no es un error: en imágenes descargadas, los IDs de los pasos intermedios quedaron en la máquina donde se construyó la imagen y no viajan; solo el ID de la imagen final existe localmente.

**La metadata en detalle:**

```console
$ docker inspect nginx
        "Architecture": "arm64",
        "Os": "linux",
        "Size": 61431587,
        "GraphDriver": { "Name": "overlayfs" },          # ← el driver de capas apiladas
        "RootFS": {
            "Type": "layers",
            "Layers": [                                   # ← la lista de capas, por digest
                "sha256:c01c35a040a25a51cd473910e3212a46d85fb700a6467c687f231d7edd47cbc1",
                "sha256:7cef4eace808164866bf71de504fff28925561e57b71beb36288982151a87046",
                ...
            ]
        },
        "Descriptor": {
            "digest": "sha256:8541484afbc9c8a5a8a99b379568ebbc957f658583ec9448fc43104229c03cf8",
        },
        "Config": { "Cmd": ["nginx", "-g", ...] }         # ← el comando de arranque
```

`inspect` devuelve el JSON completo: arquitectura, sistema operativo, tamaño, el **digest SHA256** de la imagen, la lista de **hashes de todas sus capas**, y su configuración de arranque. Es la ficha técnica que la imagen lleva adentro.

## 7. 🔴 El registry

Definición, con las dos formulaciones que conviene tener:

- **Formal:** almacena imágenes Docker de forma centralizada. Similar a un repositorio Git, pero para imágenes.
- **Operativa:** es un **web server con file system** donde se suben archivos de imágenes y se les ponen tags. Las que buildeás las subís ahí, y otro se las baja.

| Tipo | Cuál | Para qué |
|---|---|---|
| **Público** | **Docker Hub** (hub.docker.com) — el registry por defecto | Imágenes oficiales y de la comunidad |
| **Privado** | AWS ECR, GCP Artifact Registry, Azure ACR, GitHub Container Registry, **Harbor** (on-premise) | Imágenes internas de una empresa |

Es **open source**: se puede levantar uno propio con la imagen `registry:2` — de hecho, corre como un container más y no tiene mayor ciencia.

### 7.1 🔴 Naming convention

El nombre completo de una imagen tiene cuatro partes:

```bash
# Formato completo
[registry]/[usuario]/[imagen]:[tag]

# Ejemplos
docker.io/library/ubuntu:22.04                            # Docker Hub oficial
ghcr.io/mi-org/mi-app:v1.2.3                              # GitHub Container Registry
123456.dkr.ecr.us-east-1.amazonaws.com/mi-app:latest      # AWS ECR
```

⚠️ **Los defaults explican todo lo que se ve en la práctica.** Cuando escribís `docker pull nginx` no estás olvidándote nada: sin registry explícito, **el default es `hub.docker.com`**; `library` es la cuenta de las imágenes **oficiales**; y sin tag, se asume `latest`. Pero si tu imagen vive en un registry propio, **hay que poner la URL de ese registry** — de lo contrario Docker va a buscarla al Hub y no la va a encontrar. Docker resuelve por detrás si estás autenticado contra ese registry y pasa el token correspondiente.

### 7.2 🔴 Tags: alias sobre el mismo artefacto

Un **tag es solo un alias — un puntero al digest SHA256 de la imagen.** De ahí se desprende todo:

```console
$ docker build -t mi-usuario/mi-app:1.0 .     # construyo con un tag
$ docker tag mi-usuario/mi-app:1.0 mi-usuario/mi-app:1.0.1     # otro alias, MISMA imagen
$ docker tag mi-usuario/mi-app:1.0.1 mi-usuario/mi-app:latest  # alias de un alias
```

Las tres etiquetas **apuntan al mismo artefacto**. Y eso habilita el esquema de **versionado semántico**, que es el contrato entre quien publica y quien consume:

```
   Publico la 1.0.1                        Sale un hotfix: publico la 1.0.2
   ─────────────────                       ────────────────────────────────
   mi-app:1.0.1 ──┐                        mi-app:1.0.1 ──▶ (artefacto A)   ← QUIETO
   mi-app:1.0   ──┼──▶ (artefacto A)       mi-app:1.0.2 ──┐
   mi-app:latest──┘                        mi-app:1.0   ──┼──▶ (artefacto B) ← SE MUEVEN
                                           mi-app:latest──┘
```

Quien hace el `pull` **decide cuánto update quiere versus cuánta estabilidad**: si apunta a `1.0.1` se queda clavado en ese artefacto para siempre; si apunta a `1.0`, recibe automáticamente el hotfix `1.0.2` cuando el publicador mueva la etiqueta.

⚠️ **Sobre `latest`:** es simplemente el **tag por defecto** — un alias que apunta al último build. `docker pull nginx` equivale exactamente a `docker pull nginx:latest`. **No está recomendado en producción**: mejor usar tags semánticos, porque `latest` no garantiza qué artefacto vas a recibir y rompe la reproducibilidad.

### 7.3 🟡 Pull, tag, push: el flujo completo

```console
$ docker search nginx                          # buscar imágenes en Docker Hub
$ docker pull nginx:1.25-alpine                # pull con tag específico
$ docker pull nginx:latest                     # equivale a "docker pull nginx"
$ docker image ls                              # ver lo descargado

$ docker tag mi-app:1.0 mi-usuario/mi-app:1.0  # preparar la imagen para el push
$ docker login                                 # autenticarse en el registry
$ docker push mi-usuario/mi-app:1.0            # subirla
```

**`pull` baja, `push` sube.** Con una cuenta de Docker se pueden subir imágenes y dejarlas abiertas para que cualquiera las descargue. Sobre autenticación: para el `pull` de una imagen **pública** no hace falta login; sí lo requieren las imágenes **privadas** (Docker Hub permite marcar una imagen como privada) y cualquier registry privado. Para el `push`, login siempre — y ojo con el default: sin registry explícito, lo que pushees termina en el registry **público**.

### 7.4 🔴 Registry en producción: disponibilidad, compliance y costos

Acá está lo que separa el uso de laboratorio del uso profesional, y es material de criterio evaluable.

**El registry es un single point of failure.** Pensalo: cualquier máquina que necesite crear un container va a hacer **pull** de ese registry. Si el registry se cae, no podés **crear nuevas réplicas** — no es que no podés publicar versiones nuevas: no podés **escalar**. Si eso pasa con un Black Friday a las puertas, es un problema serio. Por eso, si lo levantás vos, tenés que resolver su alta disponibilidad y su capacidad de concurrencia.

**Por eso los clouds ya lo proveen.** El registry administrado del proveedor de nube suele tener costo casi nulo de base: lo que se cobra es el **almacenamiento** y las **lecturas** — cada pull. (El de Amazon, por ejemplo, guarda sobre su servicio de almacenamiento de objetos.) A cambio se puede **optimizar costos**: definir cuánto tiempo vive cada imagen, mandarla a almacenamiento frío, etc. Y la disponibilidad ya está resuelta por el proveedor: es una *capability* sobre la que uno se apoya.

**Cuándo conviene igual levantar el propio.** Cuando hay una razón que lo justifique: **compliance** o seguridad. Una empresa muy celosa de su código, o un banco con normativa que le impide alojar sus imágenes fuera, va a levantar un registry **self-hosted** — con proyectos open source como **Harbor** (que está dentro de la CNCF, la fundación que agrupa los proyectos del ecosistema cloud native). El trade-off es explícito: más control y más *fine tuning*, a cambio de hacerse cargo de la disponibilidad, la concurrencia y el mantenimiento. Si no hay una razón de peso, levantar el propio **tiene pocas ganancias**.

En resumen: la decisión depende del tipo de empresa, de cuánto se use el registry, y de qué normativa haya que cumplir. **Tecnología como medio, no como fin.**

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué es un registry y qué hay que considerar al usarlo en producción?** Es el almacén centralizado de imágenes Docker —un web server con filesystem donde se suben imágenes con sus tags—, análogo a un repositorio Git pero para imágenes; Docker Hub es el público por defecto y existen privados (AWS ECR, GCP Artifact Registry, Azure ACR, GitHub Container Registry, Harbor self-hosted). En producción es un **single point of failure**: todo despliegue y todo escalado empieza con un pull, así que un registry caído impide crear nuevas réplicas. Por eso suele usarse el registry administrado del cloud, que resuelve disponibilidad y cobra por almacenamiento y lecturas; se justifica levantar uno propio principalmente por razones de compliance o seguridad, asumiendo el costo de garantizar su disponibilidad.

## 8. 🟡 Limpieza

```console
$ docker image prune       # borra imágenes sin usar (las <none> del listado)
$ docker system prune -a   # ⚠️ borra containers, imágenes y todo lo no usado
```

`docker system prune -a` **libera gigas** de una: containers detenidos, imágenes sin container asociado, redes y caché. Lo borrado se puede volver a bajar o buildear — pagando de nuevo la espera. Útil para recuperar espacio; peligroso si no sabés qué estás resignando.

---

## ✅ Checkpoint — Parte 2

*Sin mirar el apunte. Las respuestas no están acá a propósito.*

1. Enumerá las cinco propiedades de una imagen Docker. ¿Qué significa que las capas sean "deltas de filesystem"?
2. ¿Por qué las imágenes son inmutables y qué garantiza esa inmutabilidad?
3. ¿Qué relación hay entre instrucciones del Dockerfile y capas? ¿Qué capa aparece por encima de todas y a quién pertenece?
4. ¿Qué se hashea exactamente, y qué consecuencia tiene eso sobre los builds?
5. ¿Cuál es la regla de ordenamiento de instrucciones y por qué el `COPY` del código va al final?
6. Recorré el Dockerfile completo: qué hace cada instrucción, cuáles crean capa y cuáles no. ¿Por qué las tres operaciones de apt van encadenadas con `&&` en un mismo RUN?
7. RUN vs CMD: ¿en qué momento se ejecuta cada uno y qué produce cada uno?
8. CMD vs ENTRYPOINT: ¿qué pasa con cada uno si el usuario pasa un comando en `docker run`? Cuando están los dos, ¿cuántos procesos nacen?
9. En `docker build -t mi-app:1.0 .` — ¿qué hace el `-t` y qué es el punto final?
10. ¿Qué son los "intermediate containers" que aparecen y desaparecen durante el build?
11. Se cambia una línea de `app.py` y se rebuildea: ¿qué pasos aparecen como `Using cache` y cuáles no? ¿Por qué los posteriores también se rehacen?
12. En `docker history`: ¿en qué orden aparecen las instrucciones? ¿Qué significan las capas de 0B, cuáles son las más pesadas, y qué es `<missing>`?
13. ¿Qué información devuelve `docker inspect` sobre una imagen?
14. ¿Qué significan las filas `<none>` de `docker image ls` y cómo se limpian? ¿Por qué `nginx:stable-alpine` pesa una fracción de `nginx:latest`?
15. Escribí el formato completo del nombre de una imagen y explicá los tres defaults.
16. ¿Qué es un tag realmente? Contá el escenario del hotfix 1.0.2: qué etiquetas se mueven, cuáles quedan fijas, y qué decide el que hace pull.
17. ¿Qué es `latest` y por qué no se recomienda en producción?
18. ¿Cuándo hace falta `docker login` y cuándo no?
19. ¿Por qué el registry es un single point of failure y qué se rompe concretamente si se cae?
20. ¿Qué se paga en un registry administrado por el cloud, y qué razones justifican levantar uno self-hosted?

---

**FIN DE LA PARTE 2** · *Sigue: Parte 3 — El container en ejecución: ciclo de vida, capa R/W, copy-on-write, volúmenes y redes.*
