# 📘 Apunte Maestro — Clase 02: Docker
## Parte 4 — Docker Compose, el caso real y el cierre de la unidad

**Materia:** TACS · 2C 2026 · **Unidad:** clase02 — Docker
**Parte 4 de 4** · Cubre: qué es Docker Compose y las tres cosas que define · el YAML real, anotado · la historia `docker-compose` vs `docker compose` · el caso de la materia (el repo de networking, sus dos arquitecturas y el escalado con service discovery) · hot reload y debugging con Compose · dónde practicar · qué se evalúa en el TP · el resumen de los seis conceptos de la unidad.

**De las Partes 1-3 se asume:** todo — esta parte orquesta lo aprendido. En particular: levantar containers con `run` y sus flags, volúmenes, y la regla de las redes (resolución por nombre **solo** en redes definidas por el usuario).

---

## 1. 🔴 Qué es Docker Compose

La necesidad primero. Con lo aprendido hasta acá, levantar una aplicación real — una base de datos, un web server, una API; o una base, una caché, un *message broker* (un servicio de mensajería entre procesos), un proxy — significa **levantar a mano cada container**: cada uno con su configuración, sus volúmenes, sus networks, en el orden correcto, todos los días. Funciona, y es un suplicio.

**Docker Compose es la herramienta que permite definir y correr aplicaciones Docker multi-container.** La ganancia: **toda esa definición vive en un único archivo YAML** — la arquitectura definida **mediante código** — y se levanta entera con un comando tan sencillo como `docker compose up`.

- **YAML:** formato de texto para datos estructurados; la jerarquía se marca con indentación. El archivo se encuentra como `docker-compose.yaml` o `compose.yaml`.

**El archivo define tres cosas** — la estructura que hay que saber de memoria:

| Sección | Qué define |
|---|---|
| **`services`** | Cada container: su **imagen** (o su *building*), sus **variables de entorno**, sus **puertos**, las **dependencias** entre sí (*links*), sus **volúmenes** |
| **`volumes`** | Los volúmenes de la aplicación |
| **`networks`** | Las redes — es decir, **cómo se comunican los servicios entre sí** |

Es **el estándar para el desarrollo local en cualquier equipo** de la industria.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué es Docker Compose y qué define su archivo?** Es la herramienta para definir y correr aplicaciones Docker multi-container: la arquitectura se define mediante código en un único archivo YAML (`docker-compose.yaml` / `compose.yaml`) y se levanta completa con un solo comando (`docker compose up`), en lugar de levantar a mano cada container con su configuración, volúmenes y redes. El archivo define tres cosas: `services` (cada container con su imagen o build, variables de entorno, puertos, dependencias y volúmenes), `volumes`, y `networks` (cómo se comunican los servicios). Es el estándar del desarrollo local en equipo.

## 2. 🔴 El YAML, anotado

Un fragmento de archivo real (el del caso de §4), línea por línea:

```yaml
version: "3.3"
# └─ ⚠️ Este parámetro está DESACTUALIZADO: ya no se usa. Los Compose modernos
#    lo ignoran. Aparece en muchísimos ejemplos viejos — no lo copies.

services:                              # ═══ sección 1: los containers ═══

  gateway:                             # ← nombre del SERVICIO — lo elegís vos
    container_name: gateway            # ← nombre fijo para el container
    image: nginx/unit:1.23.0-go1.15    # ← la imagen con la que se levanta este servicio
    ports:
     - 80:8080                         # ← el binding de puertos (host:container),
    networks:                          #    igual que el -p de la Parte 3
      - frontend                       # ← a qué red se conecta

  backend-for-frontend:                # ← otro servicio, con su propio nombre...
    container_name: backend-for-frontend
    # ... y probablemente otra imagen, sus puertos, su red
```

Cada clave de `services` es un servicio; cada servicio declara lo que en la Parte 3 se pasaba por flags: `image` (la imagen), `ports` (el `-p`), `networks` (el `--network`), `container_name` (el `--name`)… La traducción mental es directa: **el YAML es la versión declarada de los comandos manuales**.

⚠️ **La nota de historia que evita horas de frustración:** `docker-compose` — **con guion** — era antes un **programa aparte**, que se instalaba por separado y hacía su magia usando Docker por atrás. Los proyectos se **unificaron** (alrededor de 2025): hoy Compose es un subcomando de Docker y se escribe **`docker compose`, con espacio**. Consecuencia práctica: vas a cruzar ejemplos y READMEs desactualizados con `docker-compose ...` que directamente **no van a andar** en instalaciones recientes. No es tu error: es arqueología. Traducí el guion a espacio y seguí.

## 3. 🔴 El caso de la materia: dos arquitecturas, un repositorio

La materia tiene un repositorio hecho para practicar todo esto:

```console
$ git clone https://github.com/tacs-utn/docker-networking.git
```

Adentro hay **cuatro servicios** — `gateway`, `backend-for-frontend`, `backend`, `database` — y **dos archivos compose que difieren en una sola cosa: las redes**. El ejercicio consiste en levantar cada versión y comprobar, con curls entre containers, **quién ve a quién**.

**Ronda 1 — el archivo por defecto (una sola red):**

```console
$ docker compose up -d
# └─ levanta los cuatro servicios definidos en docker-compose.yaml

$ docker exec gateway curl backend-for-frontend:8080     # ← responde ✅
$ docker exec gateway curl backend:8080                  # ← responde ✅
$ docker exec gateway curl database:8080                 # ← responde ✅ ...¡y esto es un problema!
```

Nótense dos cosas. Primero: los curls van **por nombre de servicio** (`backend:8080`) — y resuelven. Compose crea la red del proyecto y conecta los servicios, y esa red es *definida por el usuario*: la resolución por nombre de la Parte 3, §9, viene **de fábrica** en el mundo Compose. Segundo: en esta versión **todos se ven con todos** — incluido el gateway (la pieza expuesta al mundo en el puerto 80) hablándole **directo a la base de datos**. Funciona… y es un edificio sin puertas por dentro.

Para inspeccionar qué redes ve cada container:

```console
$ docker exec gateway ip r                 # ← las rutas de red del container
$ docker exec backend-for-frontend ip r    #    (ip r = ip route: qué redes alcanza)
$ docker exec backend ip r
$ docker exec database ip r
```

**Ronda 2 — el archivo segmentado:**

```console
$ docker compose -f docker-compose-network.yaml up -d
# └─ -f: usar ESTE archivo en vez del default

$ docker exec gateway curl backend-for-frontend:8080     # ← responde ✅ (su vecino de red)
$ docker exec gateway curl backend:8080                  # ← FALLA ❌
$ docker exec gateway curl database:8080                 # ← FALLA ❌: para el gateway,
                                                         #    la base NI EXISTE
```

Mismo stack, mismos cuatro servicios — **otro mapa de redes**. La versión segmentada define varias redes (el fragmento de §2 muestra al gateway en `frontend`) y las reparte en cadena: el gateway comparte red con el BFF, el BFF con el backend, el backend con la base. Repitiendo los `ip r` se ve la diferencia: cada container ahora alcanza **solo** las redes que le tocan. Es la regla de visibilidad de la Parte 3 — *redes distintas no se ven* — usada como **herramienta de arquitectura**: la base quedó a dos redes de distancia de la pieza expuesta, invisible incluso por nombre.

**Ronda 3 — escalar y ver el service discovery:**

```console
$ docker compose -f docker-compose-network.yaml up -d
$ docker compose scale backend-for-frontend=5 backend=3
# └─ ⚠️ sintaxis de la época del guion: en Compose moderno es
#    docker compose up -d --scale backend-for-frontend=5 --scale backend=3
#    (y ojo: un servicio con container_name fijo no puede escalarse — el nombre
#    no puede repetirse 5 veces; para probarlo, quitale el container_name)

# check different IPs on service discovery balancing:
$ docker exec gateway ping backend-for-frontend -c 1     # ← repetir x5
$ docker exec docker-networking_backend-for-frontend_1 ping backend -c 1   # ← repetir x3
```

Con 5 réplicas del BFF y 3 del backend, los pings repetidos al **mismo nombre** devuelven **IPs diferentes**: la resolución de nombres está **repartiendo** entre las réplicas. Eso es **service discovery balancing** — el descubrimiento de servicios balanceando la carga — y es la semilla de cómo escalan los sistemas grandes: N réplicas detrás de un solo nombre. (Fijate también el nombre real de las réplicas: `docker-networking_backend-for-frontend_1` — proyecto_servicio_número, el patrón con que Compose bautiza lo que escala.)

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué demuestra el ejercicio de las dos arquitecturas de red?** Con un solo stack de cuatro servicios (gateway, backend-for-frontend, backend, database) y dos archivos compose que difieren solo en la sección de redes: en la versión de red única todos los servicios se resuelven y alcanzan entre sí por nombre — incluido el gateway hablando directo con la base; en la versión segmentada, cada par de servicios comparte solo la red que necesita, y el gateway ya no puede alcanzar (ni resolver) a la base. Demuestra que las redes de Docker son una herramienta de arquitectura y seguridad: definen quién ve a quién, y permiten aislar las piezas sensibles de las expuestas. Además, escalando un servicio a N réplicas, los pings sucesivos al mismo nombre devuelven IPs distintas: la resolución de nombres balancea entre réplicas (service discovery balancing).

## 4. 🟡 Compose para desarrollar: hot reload y debugging

Un uso de Compose que junta varias piezas de la Parte 3: **desarrollar y debuggear con la app corriendo adentro de containers**. El ejemplo es un proyecto real de referencia — `github.com/alejandropal/docker-typescript-debug` — cuyo compose vale la pena leer anotado:

```yaml
services:
  app1:
    build: .                                # ← no usa "image": CONSTRUYE el Dockerfile
    volumes:                                #    de la carpeta (el build de la Parte 2)
      - ./tsconfig.json:/tsconfig.json:ro   # ← bind mounts :ro del código fuente:
      - ./app1.ts:/app1.ts:ro               #    el container ve TUS archivos, solo lectura
      - ./lib1.ts:/lib1.ts:ro
    ports:
      - 3000:3000                           # ← el puerto de la app
      - 9229:9229                           # ← el puerto del DEBUGGER
    command: yarn nodemon --signal SIGINT --inspect=0.0.0.0:9229 --nolazy app1.ts
    # └─ command reemplaza al CMD de la imagen (Parte 2 §3.2). Acá lanza nodemon:
    #    una herramienta que RE-EJECUTA la app cada vez que detecta cambios en los
    #    archivos — que, gracias a los bind mounts, son los tuyos: guardás en tu
    #    editor y la app adentro del container se recarga sola (hot reload).
    #    El --inspect abre el protocolo de debug de Node hacia afuera.

  app2:
    build: .                                # ← MISMA imagen...
    volumes:
      - ./tsconfig.json:/tsconfig.json:ro
      - ./app2.ts:/app2.ts:ro
      - ./lib1.ts:/lib1.ts:ro
    ports:
      - 3001:3000                           # ← ...otro puerto de host (¡el 3000 ya está tomado!)
      - 9230:9229
    command: yarn nodemon --signal SIGINT --inspect=0.0.0.0:9229 --nolazy app2.ts
    # └─ ...y OTRO command: dos servicios, una imagen, dos apps distintas
```

Con esto, el editor (VS Code) se conecta al puerto de debug y se puede poner un **breakpoint** en el código y frenar la ejecución **de la app que corre adentro del container** — inspección de variables incluida. El resultado, en palabras del propio proyecto: *Docker como la única dependencia necesaria para correr* — ni Node ni las herramientas instaladas en tu máquina; todo adentro.

Las piezas de la unidad que este archivo pone a trabajar juntas: `build` en vez de `image` · bind mounts con `:ro` para el código · el mapeo de puertos evitando el choque en el host · `command` reemplazando al CMD · dos servicios naciendo de la misma imagen con distinto comportamiento.

## 5. 🔴 Práctica, y qué se evalúa en el TP

**Dónde practicar sin instalar nada.** Un aviso primero: el clásico *Play with Docker* (`labs.play-with-docker.com`), que suele figurar en materiales y recursos, **está desactualizado y no se puede usar**. Las alternativas vigentes:

| Opción | Qué ofrece |
|---|---|
| **GitHub Codespaces** | Entorno online con Docker instalado en versión reciente (Compose incluido) y horas gratuitas — ideal si no querés instalar nada o tu máquina es modesta |
| **labex.io** | Ambientes virtuales de práctica |
| **killercoda.com/docker** | Escenarios interactivos de Docker |

Además, la cátedra comparte por la lista de correo un **workshop** de troubleshooting sobre Docker — levantar cosas, bajar otras, probar networking — para seguir practicando por cuenta propia.

**Qué se evalúa — y esto es 🔴 de lo más importante de la unidad:** el uso de Docker **se evalúa explícitamente en el TP**, y no solo el "que funcione":

- **Buenas prácticas** de Docker en general (todo lo visto: orden del Dockerfile, capas, limpieza, volúmenes para los datos, config externalizada).
- **Cómo está arquitecturado el compose YAML** — incluida la decisión de redes: quién ve a quién (§3 no es un ejercicio decorativo: es el criterio con el que van a mirar tu entrega).
- **Qué tipos de imágenes** se usan (bases apropiadas, tamaños razonables — el alpine vs ubuntu de la Parte 2).

La indicación es explícita: no quedarse con lo anecdótico de las demos — **llevar estas prácticas al TP**. La filosofía de evaluación de la materia acompaña: el objetivo no es salir ninja de Docker; es entender los principios, poder participar con criterio en una conversación técnica real, y aplicarlos en la entrega.

**Recursos de referencia:** la **documentación oficial** de Docker y de Docker Compose (docs.docker.com — súper completa, la mejor referencia) y **Docker Hub** (hub.docker.com) para las imágenes oficiales.

## 6. 🟢 Lo que sigue: orquestación

Queda abierta, a propósito, una puerta: existe toda una vuelta de **orquestación de containers** — y ahí entra el concepto de **Kubernetes**, que en la materia se va a ver muy por encima más adelante. Por ahora alcanza con que suene y con entender **por qué existe**: cuando los containers son cientos, repartidos en muchas máquinas, con réplicas que nacen y mueren (el patrón stateless de la Parte 3, a escala industrial), hace falta un sistema que los administre. La documentación de Kubernetes está disponible para el curioso.

## 7. 🔴 Resumen de la unidad: los seis conceptos

El mapa final — si podés explicar estas seis tarjetas, la unidad es tuya:

| | Concepto | En una línea |
|---|---|---|
| 📦 | **Image** | Template read-only de capas hasheadas, construida desde un Dockerfile |
| 🗄️ | **Registry** | Almacén de imágenes. Docker Hub es el público; existen privados en cloud |
| 🚢 | **Container** | Instancia runtime **efímera y aislada**. Namespaces + cgroups = aislamiento |
| 📂 | **Layers** | Stackable + copy-on-write = eficiencia en disco y velocidad de builds |
| 💾 | **Volumes** | Datos persistentes **fuera** del ciclo de vida del container |
| 🌐 | **Networking** | Port mapping hacia afuera; redes virtuales entre containers |

Y los tres primeros — image, registry, container — son **los tres conceptos que hay que poder diferenciar uno de otro**: están súper relacionados, pero en la diferencia radica todo — qué se está ejecutando, qué puede escribir, qué es read-only.

---

## ✅ Checkpoint — Parte 4

*Sin mirar el apunte. Las respuestas no están acá a propósito.*

1. ¿Qué problema resuelve Docker Compose? ¿Qué significa "definir la arquitectura mediante código"?
2. Las tres secciones del archivo compose: ¿qué define cada una? ¿Qué declara un servicio adentro de `services`?
3. En el YAML: ¿a qué flags de `docker run` equivalen `image`, `ports`, `networks` y `container_name`?
4. ¿Qué pasa con el parámetro `version:` en los compose modernos?
5. Contá la historia de `docker-compose` (guion) vs `docker compose` (espacio): qué era cada cosa, qué pasó, y qué consecuencia práctica tiene al copiar ejemplos viejos.
6. El caso de los dos archivos: ¿en qué difieren, y qué responde `docker exec gateway curl database:8080` en cada versión? ¿Por qué?
7. ¿Por qué en el mundo Compose la resolución por nombre funciona "de fábrica", cuando en la Parte 3 hacía falta crear una red propia?
8. ¿Qué muestra `docker exec X ip r` y para qué sirve en el ejercicio?
9. ¿Por qué la versión segmentada es mejor arquitectura, si la de red única "funciona"? ¿Qué pieza queda protegida y de qué?
10. Escalado a 5 réplicas: ¿qué se observa al repetir `ping backend-for-frontend` y cómo se llama el fenómeno? ¿Con qué patrón bautiza Compose a las réplicas? ¿Qué impide escalar un servicio con `container_name`?
11. En el compose de hot reload: ¿qué rol cumplen los bind mounts `:ro`, el `command` con nodemon, y los dos puertos publicados? ¿Cómo logran dos apps distintas desde una misma imagen?
12. ¿Qué tres cosas del uso de Docker se evalúan explícitamente en el TP?
13. ¿Dónde se puede practicar Docker online hoy, y qué recurso clásico ya no está disponible?
14. Reconstruí la tabla de los seis conceptos de la unidad, cada uno en una línea.

---

**FIN DE LA PARTE 4 — Y DEL APUNTE MAESTRO · Clase 02: Docker · TACS 2026**
