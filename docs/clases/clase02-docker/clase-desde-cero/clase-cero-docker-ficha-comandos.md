# 🐳 Clase desde Cero — Docker · Ficha de comandos

**Qué es esto:** la referencia operativa de toda la serie — propósito en una línea, el comando, sus flags. **Nada más.** Los conceptos viven en los módulos; esta es la hoja que tenés al costado mientras practicás. Buscá con Ctrl/Cmd + F.

**Convenciones:** `...` = recorte, no se tipea · `X` = nombre o ID de un container · `[ ]` = opcional. Los comandos de administración de WSL (Windows) viven en su setup, no acá.

---

## Setup — Verificación

**¿El cliente ve al servidor? (si aparece la sección Server, la VM responde)**
```console
$ docker version
```

**Test de punta a punta: baja imagen, crea container, corre, saluda**
```console
$ docker run hello-world
```

---

## Módulo 3 · §7 — Inspeccionar imágenes

**Descargar una imagen del registry (se ve bajar capa por capa)**
```console
$ docker pull python:3.11-slim
```

**Listar imágenes locales: nombre, tag, ID, tamaño**
```console
$ docker image ls
```

**Ver la pila de una imagen — la receta al revés (0B = metadata de la ficha)**
```console
$ docker history python:3.11-slim
```

---

## Módulo 4 · §2 — Build y run

**Cocinar la receta de la carpeta actual → imagen**
```console
$ docker build -t mi-app:1.0 .
```
- `-t nombre:tag` → bautiza la imagen
- `.` → build context: dónde están el Dockerfile y lo que se COPY-a

**Crear y arrancar un container (el comando estrella)**
```console
$ docker run -d --name mi-servidor -p 8080:8080 mi-app:1.0
```
- `-d` → de fondo (*detached*); sin él, la terminal queda pegada al container
- `--name X` → nombre propio (sin esto, Docker inventa uno)
- `-p HOST:CONTAINER` → puente de puertos: puerta de calle → puerto de la burbuja
- `--rm` → container descartable: se auto-borra al terminar
- `comando al final` → reemplaza el CMD de la ficha (el PID 1 pasa a ser ese comando)
- *(más flags de run: `-e` en M6 §5 · `-v`/`--tmpfs` en M6 §2-4 · `--network` en M6 §6)*

**Probarlo (o abrir `http://localhost:8080` en el navegador)**
```console
$ curl localhost:8080
```

---

## Módulo 4 · §3 — Censo y logs

**Containers corriendo**
```console
$ docker ps
```
- `-a` → *all*: también los `Exited` (existir ≠ correr)
- `-s` → columna SIZE: cuánto pesa la capa read-write (ver M5)

**Lo que el PID 1 imprimió por stdout**
```console
$ docker logs mi-servidor
```
- `-f` → *follow*: en vivo (Ctrl+C para salir)

---

## Módulo 4 · §4 — Entrar a la burbuja

**Ejecutar un comando extra dentro de un container que ya corre**
```console
$ docker exec -it mi-servidor bash
```
- `-it` → interactivo + terminal: para quedarte adentro con una shell
- sin `-it` → ejecuta el comando, imprime, vuelve
- adentro: `exit` cierra TU shell — el container sigue (su PID 1 ni se enteró)

**Censo de procesos de Linux — adentro del container (≠ `docker ps`)**
```console
$ ps aux
```
- en imágenes slim no viene: `apt-get update && apt-get install -y procps` primero

---

## Módulo 4 · §6 — Ciclo de vida

**Armar sin arrancar (raro de usar; existe para entender que run = create + start)**
```console
$ docker create --name X ... imagen
```

**Arrancar un container armado — o REVIVIR un `Exited` (misma capa read-write)**
```console
$ docker start X
```

**Buen final: señal de terminación al PID 1 + segundos de gracia. Datos a salvo**
```console
$ docker stop X
```

**Congelar / descongelar los procesos en el aire (nicho)**
```console
$ docker pause X
$ docker unpause X
```

**Entierro: borra capa read-write y ficha — lo escrito por el container muere**
```console
$ docker rm X
```
- `-f` → fuerza stop + rm en un paso

---

## Módulo 5 · §4-5 — Espiar la capa read-write

**Tamaño de la capa read-write de cada container vivo**
```console
$ docker ps -s
```

**Qué difiere entre el container y su imagen (A=agregado, C=modificado, D=borrado)**
```console
$ docker diff X
```

---

## Módulo 5 · §7 — El medidor y la escoba

**Radiografía de espacio: imágenes, containers, volúmenes, caché de build**
```console
$ docker system df
```
- `-v` → desglose ítem por ítem

**Barrer containers `Exited`**
```console
$ docker container prune
```

**Barrer imágenes colgadas (capas huérfanas de rebuilds viejos)**
```console
$ docker image prune
```

**⚠️ La escoba grande: además borra TODA imagen sin un container vivo usándola**
```console
$ docker system prune -a
```

---

## Módulo 6 · §2 — Named volumes

**El ciclo completo de un volumen**
```console
$ docker volume create datos-tacs
$ docker volume ls
$ docker volume inspect datos-tacs
$ docker volume rm datos-tacs        # solo si nadie lo usa
$ docker volume prune                # barre los que ningún container referencia
```

**Montarlo en un container (datos que sobreviven al rm del container)**
```console
$ docker run -v datos-tacs:/data ...
```
- `nombre:/ruta` → volumen administrado por Docker, montado en esa ruta

---

## Módulo 6 · §3-4 — Bind mounts y tmpfs

**Montar una carpeta TUYA adentro del container (portal en vivo, ambos sentidos)**
```console
$ docker run -v "$(pwd)/carpeta":/ruta ...
```
- `$(pwd)` → la ruta absoluta de donde estás parado (el bind exige rutas absolutas)
- `:ro` al final → solo lectura (el candado para montar configuración)

**Carpeta solo en RAM: no toca ningún disco, se esfuma con el container**
```console
$ docker run --tmpfs /ruta ...
```

---

## Módulo 6 · §5 — Config por afuera

**Variables de entorno al crear el container (se fijan ahí: cambiar = recrear)**
```console
$ docker run -e CLAVE=valor ...
```
- `--env-file archivo.env` → muchas de una: un archivo con `CLAVE=valor` por línea (no va al repo)

---

## Módulo 6 · §6 — Redes

**El ciclo completo de una red**
```console
$ docker network create red-tacs
$ docker network ls
$ docker network inspect red-tacs    # qué containers viven adentro
$ docker network connect red-tacs X  # sumar un container YA corriendo
$ docker network rm red-tacs         # solo sin containers adentro
```

**Meter un container en una red al nacer (adentro: resolución por nombre)**
```console
$ docker run --network red-tacs --name api ...
```

---

## Módulo 7 · §4-6 — Compose

**Levantar el proyecto entero: red, volúmenes, containers, en orden**
```console
$ docker compose up -d
```
- `-d` → de fondo
- `--build` → re-cocina las imágenes propias (las de `build:`)
- `--scale api=3` → N réplicas de un servicio (sin `container_name` ni `ports:` fijo)
- `-f archivo.yaml` → usar otro archivo que el default

**Operar el proyecto**
```console
$ docker compose ps                  # censo, solo de este proyecto
$ docker compose logs -f app         # logs de un servicio (o de todos, sin nombre)
$ docker compose stop                # detiene; datos y containers quedan
$ docker compose down                # borra containers y red; volúmenes SOBREVIVEN
```
- `down -v` → ⚠️ borra TAMBIÉN los volúmenes del proyecto (la base incluida)

---

## Módulo 7 · §8 — Registry

**Publicar tu imagen en Docker Hub, de punta a punta**
```console
$ docker login                                   # una vez, con tu Docker ID
$ docker tag mi-app:1.0 tuusuario/mi-app:1.0     # alias: dos nombres, MISMO IMAGE ID
$ docker push tuusuario/mi-app:1.0               # sube las capas que el Hub no tenga
$ docker pull tuusuario/mi-app:1.0               # desde cualquier máquina del mundo
```

---

**FIN DE LA FICHA — Clase desde Cero · Docker**
