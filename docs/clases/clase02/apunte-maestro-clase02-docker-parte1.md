# 📘 Apunte Maestro — Clase 02: Docker
## Parte 1 — Del problema al modelo: virtualización, containers y qué es Docker

**Materia:** TACS · 2C 2026 · **Unidad:** clase02 — Docker
**Parte 1 de 4** · Cubre: agenda de la unidad · el problema que motiva containers · virtualización y VMs · LXC y su historia · namespaces y cgroups · qué es Docker · los tres conceptos · primer container.

**Cómo leer los bloques de consola (vale para las cuatro partes).** Hay dos tipos, y distinguirlos evita malentendidos: las **demos de la parte** se explican donde aparecen y se pueden ejecutar siguiendo el texto (en esta parte: las de la sección 7). Las **vistas adelantadas**, marcadas con `👁️ → Parte N`, muestran la salida de un comando que se enseña más adelante — están como **evidencia del concepto**, no como instrucción: todavía no son para tirar.

---

## 1. 🔴 El problema: cinco microservicios en un servidor

Antes de cualquier definición, el escenario que motiva todo lo que sigue.

Tenés que desplegar **cinco microservicios**. Con máquinas virtuales, el camino natural es una VM por servicio. Eso significa:

- **Reserva estática de recursos:** al crear cada VM le asignás de antemano su memoria y sus cores — 4 GB, 16 GB, lo que sea. Los use o no, quedan reservados para ella. Si le diste 4 GB y usa 1, los otros 3 están inmovilizados. Cambiar esa asignación implica, en la práctica, destruir la VM y volver a crearla.
- **Cinco sistemas operativos completos:** cada VM necesita su kernel y su SO enteros para arrancar. Con ~500 MB de footprint por SO, estás pagando ese peso cinco veces **antes** de que corra una sola línea de tu código.

Conclusión intermedia: aislar sale carísimo. Entonces surge la tentación del otro extremo — **juntar los cinco en un solo servidor**. Total, son APIs chicas, optimizadas, con poco tráfico: un servidor de 4 GB y adentro los cinco ejecutables. Se ahorra el overhead de cinco SO. ¿Qué se pierde?

| Problema del "todo junto" | Qué pasa |
|---|---|
| **Disponibilidad** | Se cae el servidor y se caen los cinco servicios a la vez. Un solo punto de falla para todo. |
| **Ciclo de vida acoplado** | Actualizar, mantener o reiniciar uno obliga a coordinar con los otros cuatro. |
| **Conflicto de dependencias** | Un microservicio corre en Java 21 y otro quedó en Java 8 porque nadie lo actualizó. ¿Qué versión instalás en el servidor? No hay forma de aislarlas. |
| **Conflicto de puertos** | Los cinco escuchan en el 8080 por defecto. En un mismo SO, **solo uno puede tener el 8080**. |
| **Bad neighbor** | No hay pared entre ellos: si uno empieza a consumir disco, memoria o CPU sin control, los demás se enteran — y sufren. |

Ese último merece su nombre propio porque va a volver: **efecto bad neighbor** (mal vecino) — procesos que comparten recursos sin límites, donde uno que se descontrola perjudica a todos los que tiene al lado. La memoria es finita: si alguien consume y consume, en algún momento otro empieza a fallar con la clásica excepción de *out of memory*, y si el que se queda sin recursos es el host, **vuelan todos los procesos vecinos con él**.

Tenemos entonces dos extremos malos: **aislamiento caro** (VMs) o **convivencia peligrosa** (todo junto). Lo que sigue es la tecnología que rompe ese dilema.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué es el efecto bad neighbor?** Es la degradación que sufre un proceso porque otro, con el que comparte recursos del mismo host sin límites impuestos, los consume de forma descontrolada (memoria, CPU, disco, red). Al ser recursos finitos, el consumo excesivo de uno provoca fallas en los demás — típicamente out of memory — y puede llegar a tumbar el host completo con todos sus procesos. Se mitiga con cgroups, que imponen cuotas por container.

## 2. 🔴 Virtualización: los dos enfoques

**Virtualización** es la capacidad de ejecutar **múltiples sistemas o entornos aislados sobre un mismo hardware físico**. Hay dos enfoques, y la diferencia entre ellos es el eje de toda la unidad.

| | **Máquina Virtual (VM)** | **Container** |
|---|---|---|
| **Aislamiento** | Un SO completo por VM | Proceso aislado dentro del host |
| **Kernel** | **Propio** por cada VM | **Comparte el kernel del host** |
| **Overhead** | Alto — GB de RAM por VM | Bajo — MB |
| **Inicio** | Minutos (1 a 5) | Milisegundos (< 1 segundo) |
| **Tamaño de imagen** | GB | MB |
| **CPU overhead** | Alto | Casi nulo |
| **Memoria** | Reserva fija, estática | Comparte con el host, dinámica |
| **Portabilidad** | Media — depende del hipervisor | Muy alta — corre en cualquier lado |
| **Aislamiento (fuerza)** | Muy fuerte | Fuerte (namespaces) |
| **Se gestiona con** | Hipervisor: VMware, VirtualBox, KVM | Docker Engine, containerd |

- **Hipervisor:** el software que crea y administra máquinas virtuales, repartiendo el hardware real entre ellas.

**La analogía canónica:** las VMs son **casas separadas** — cada una con sus propias cañerías, su electricidad, sus cimientos. Los containers son **departamentos en un edificio**: comparten la infraestructura del edificio, pero cada uno es independiente y nadie entra al de al lado.

Y la formulación técnica de esa diferencia:

```
   MÁQUINA VIRTUAL                        CONTAINER
   ┌─────────┬─────────┐                  ┌─────────┬─────────┐
   │  App A  │  App B  │                  │  App A  │  App B  │
   ├─────────┼─────────┤                  ├─────────┴─────────┤
   │ SO+kernel│SO+kernel│  ← se duplica    │  (sin SO propio)  │
   ├─────────┴─────────┤     el SO         ├───────────────────┤
   │    HIPERVISOR     │  ← virtualiza    │   DOCKER ENGINE   │
   ├───────────────────┤     HARDWARE      ├───────────────────┤
   │   SO del host     │                   │  SO del host      │  ← UN solo kernel,
   ├───────────────────┤                   ├───────────────────┤     compartido
   │     HARDWARE      │                   │     HARDWARE      │
   └───────────────────┘                   └───────────────────┘
   virtualización a          vs           virtualización a
   nivel HARDWARE                          nivel SISTEMA OPERATIVO
```

⚠️ **La distinción precisa, porque es pregunta de parcial:** el hipervisor provee **virtualización a nivel de hardware** — simula máquinas completas, y encima de cada una corre un SO entero con su propio kernel. El container provee **virtualización a nivel de sistema operativo** — no simula hardware ni fabrica un kernel: usa mecanismos del kernel *ya existente* para darle a un proceso la ilusión de tener su propia máquina. Por eso el container no puede tener un SO distinto al del host: **el kernel es uno solo y es compartido**.

Consecuencia económica, que es la que importa en la industria: si me ahorro cinco kernels y cinco reservas fijas de memoria, me ahorro **cores y RAM**, y eso se traduce directo en **menos plata pagada al proveedor de nube** por hostear los mismos servicios. La ventaja no es solamente que arranca más rápido: es la factura.

🟡 **No son excluyentes.** En producción se combinan: containers **dentro** de VMs, cuando se quiere el aislamiento fuerte de la VM y la eficiencia del container. En los proveedores de nube, esa es la disposición habitual: cada VM corre sus propios containers.

## 3. 🔴 LXC: de dónde viene todo esto

Docker no apareció por generación espontánea. Toma su concepto de **LXC (Linux Containers)**.

**La historia, en orden:**

**2008 — nace LXC.** Surge como alternativa liviana para generar entornos virtuales. La virtualización ya existía, pero era pesada y poco performante: máquinas virtuales con reserva anticipada de recursos y un kernel completo por instancia. LXC ofrece otra cosa — **un entorno virtual que comparte el kernel del sistema operativo host**, con su propio espacio de procesos y de red.

Definición formal: **LXC es una tecnología de virtualización a nivel de sistema operativo para Linux**, que permite a un servidor ejecutar múltiples instancias de sistemas operativos aislados — llamados **VPS** (Servidores Privados Virtuales) o **VE** (Entornos Virtuales).

> **El punto clave de LXC:** *no provee una máquina virtual.* Provee un **entorno virtual con su propio espacio de procesos y de redes**. Todos los containers comparten el kernel del host.

**¿Y por qué hoy nadie habla de LXC?** Porque tenía un problema grande: **no era sencillo**. Requería conocimiento de bajo nivel — era una herramienta más cercana a los *sysadmins* (administradores de sistemas) que a los desarrolladores. La comunidad amó el concepto, pero la barrera de entrada lo dejó en un nicho.

**2013 — aparece Docker**, escrito en Go, open source, lanzado por dotCloud (empresa que después se renombró Docker Inc.). Su propuesta: la misma idea, pero **amigable y más poderosa**. Y su recorrido técnico es una lección en sí misma:

```
   2008          2013                 años siguientes            hoy
   LXC    →   Docker usa LXC   →   implementación propia   →   runC  +  OCI
                                                                (estándar abierto)
```

Docker arrancó **usando LXC** por debajo. Después desarrolló su propia implementación, que con el tiempo derivó en **runC** — un runtime de containers con **especificación propia bajo la OCI (Open Container Initiative)**: un **estándar abierto** para el runtime de containers.

- **Runtime de containers:** el componente que efectivamente crea y ejecuta el container a partir de una imagen, hablando con el kernel.

El desenlace es la parte que conviene retener: **Docker no solo adaptó una tecnología que ya existía — la convirtió en un estándar para toda la industria.** Y su éxito se mide en adopción: hoy es difícil que un desarrollador con años en la industria no conozca Docker, al menos lo suficiente para entender de qué va y levantar algo sencillo.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué es LXC y qué relación tiene con Docker?** LXC (Linux Containers, 2008) es una tecnología de virtualización a nivel de sistema operativo para Linux: permite correr múltiples entornos aislados (VPS/VE) que **comparten el kernel del host**, cada uno con su propio espacio de procesos y de red — no es una máquina virtual. Docker (2013) toma ese concepto y lo hace accesible para desarrolladores; internamente usó LXC en sus inicios, luego pasó a una implementación propia que derivó en runC, con especificación bajo la OCI, convirtiendo la tecnología en un estándar abierto de la industria.

## 4. 🔴 Namespaces y cgroups: cómo se logra el aislamiento

Acá está el corazón técnico. Un container es **un proceso común del host**, con dos mecanismos del kernel Linux aplicados encima. Docker no inventó ninguno de los dos: es una **capa de abstracción sobre estas primitivas del kernel** (las mismas que usa LXC).

### 4.1 Namespaces — aislamiento de recursos

Los **namespaces** ("espacios de nombres") proveen **aislamiento de los recursos del sistema operativo**: al proceso se le crea un espacio virtual propio, y dentro de él solo ve lo suyo.

| Namespace | Qué le da al container |
|---|---|
| **PID** | Su propio **árbol de procesos** |
| **Network** | Sus propias **interfaces de red** |
| **Mount** | Sus propios **puntos de montaje** de file system |
| **UTS** | Su propio **hostname** |
| **IPC** | Comunicación entre procesos aislada |
| **User** | Sus propios **UID/GID** (identificadores de usuario y grupo) |

Dos de ellos resuelven problemas planteados en la sección 1:

**El namespace de red mata el conflicto de puertos.** Como cada container tiene **su propia interfaz de red**, dos web servers pueden escuchar en el 8080 al mismo tiempo, cada uno en la suya, sin pisarse. Aquel "solo uno puede tener el 8080" era una limitación del SO compartido, no del universo.

**El namespace de mount habilita montar carpetas del host.** Se le puede montar al container una carpeta del host en, por ejemplo, `/tmp` o `/data`, en modo solo lectura o lectura-escritura. Esto es muy útil para **externalizar archivos de configuración** — y tiene una recomendación explícita para el TP:

> 🔴 **Indicación para el TP.** Conviene usar montajes para la configuración: si hay que cambiar una variable, o poner una **API key** (clave de acceso a un servicio externo), que **no quede planchada dentro de la imagen** — se le monta al container en el momento de ejecutarlo. Es la excepción declarada al empaquetado: "todo lo que funciona en mi máquina debería funcionar igual en producción" vale para todo… **menos** el string de conexión a la base de datos y las API keys, porque desde tu máquina no te vas a conectar a la base productiva. Esas cosas se externalizan, en un archivo de configuración o en variables de entorno. *(Cómo se hace, con sus comandos: Parte 3.)*

### 4.2 El PID namespace, en detalle

Es el más ilustrativo, y el que más se pregunta. La regla: **adentro del container, el proceso principal siempre se ve a sí mismo como PID 1** — aunque en el host tenga un PID completamente distinto. Y como cada container tiene SU propio namespace, esto vale para todos a la vez — dos containers corriendo, dos procesos convencidos de ser "el 1":

```
┌──────────────────────────────────────────────────────────────────────┐
│  root pid namespace — EL HOST (quien tiene el kernel)                │
│                                                                      │
│   pid 1 (init) · pid 2 (systemd) · ...los procesos normales...       │
│   pid 3  ───▶  es el pid 1 del container A                           │
│   pid 4  ───▶  es el pid 2 del container A                           │
│   pid 58 ───▶  es el pid 1 del container B                           │
│                                                                      │
│   ┌─ pid namespace (container A) ──┐  ┌─ pid namespace (container B)┐│
│   │  pid 1 → el proceso principal  │  │  pid 1 → el proceso         ││
│   │  pid 2 → un proceso hijo       │  │          principal          ││
│   └────────────────────────────────┘  └─────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────┘
```

El mismo proceso tiene **dos identidades simultáneas**: es el PID 1 adentro y el PID 3 (o 58, o 3847) afuera. Ninguna es falsa. ¿Quién tiene la verdad? **El host**: es quien tiene el kernel y, por lo tanto, quien gestiona realmente los procesos — conoce el valor original de cada uno. Por eso, desde el host, el proceso del container **se ve**, con su PID distinto. Pero está ejecutando en el entorno del container.

**Cómo se ve esto en una consola** — dos vistas adelantadas (👁️ los comandos se enseñan en la Parte 3; por ahora son evidencia del concepto, no instrucción). Adentro de un container, el censo de procesos muestra un universo diminuto:

```console
# 👁️ vista adelantada → los comandos docker exec y ps aux se enseñan en la Parte 3
root@dcb399b02aad:/app# ps aux
USER   PID  ...  COMMAND
root     1  ...  bash             # ← el comando del container: PID 1
root    36  ...  ps aux           # ← este mismo comando
```

*(`ps aux` es el censo de procesos de Linux — `a`: de todos los usuarios, `u`: formato detallado, `x`: incluso los sin terminal. No confundir con `docker ps`, que lista containers.)*

Dos procesos, nada más. Y si se siguen lanzando comandos, se van generando PIDs hijos incrementales **dentro de ese container**. Del lado del host, en cambio, el mismo proceso aparece con su PID real:

```console
# 👁️ vista adelantada → y OJO: solo funciona en ciertos entornos (ver ⚠️ abajo)
$ ps aux | grep app.py
usuario  58234  ...  python3 app.py    # ← el mismo proceso, con su PID del host
```

⚠️ **Esta segunda vista depende de dónde corra Docker, y conviene saberlo ANTES de intentarla.** El "host" que ve los PIDs reales es la máquina que tiene el kernel Linux. En un **Linux nativo**, esa máquina es la tuya: el comando funciona tal cual desde tu terminal. En **Mac y Windows con Docker Desktop**, el host de los containers es una **VM interna y sellada** — no existe una terminal de esa VM a la que entrar, así que este comando, tirado en TU terminal, no muestra nada. No es que el mapeo de PIDs no exista: es que el host que lo ve no sos vos. La comprobación directa es un lujo de Linux nativo.

> 🕳️ **Madriguera — espiar la VM igual**
> Existe un truco (`--pid=host`) para ver los procesos de la VM a través de un container especial. Pura curiosidad de comprobación — no hace falta para la materia.
> *Volvé al camino.*

> 🎓 **Para el parcial, si te preguntan**
> **Si corro un proceso dentro de un container, ¿corre dentro del host?** Corre en el entorno del container: con sus namespaces y cgroups, en un ambiente controlado y aislado, y con un PID propio que dentro del container típicamente es 1. Pero el host es quien tiene el kernel y gestiona los procesos, así que ese mismo proceso existe también para él, con un PID distinto y visible desde afuera. Es un proceso del host al que se le aplicó aislamiento — no un proceso "dentro de otra máquina".

### 4.3 cgroups — límite de recursos

Los **cgroups** (*control groups*) hacen lo que los namespaces no: **limitan y contabilizan el uso de recursos del sistema**.

| Recurso | Qué se controla |
|---|---|
| **CPU** | Cuotas de procesador |
| **Memory** | Límite de RAM |
| **I/O** | Ancho de banda de disco |
| **Network** | Ancho de banda de red |

Su razón de ser es exactamente el problema de la sección 1: **mitigan el efecto bad neighbor**. Sin cgroups, un container que empieza a instanciar cosas descontroladamente compite por la memoria con todos los demás, el kernel tiene que repartir un recurso finito, y en algún momento alguien falla con *out of memory* — y si el host se queda sin memoria, se lleva puestos a todos los containers vecinos. Con cgroups, cada container tiene su techo: **no puede acaparar los recursos del host**.

La división de tareas, para no confundirlos nunca:

```
   NAMESPACES  →  aislamiento de la VISTA    →  "¿qué ve el proceso?"
   CGROUPS     →  límite de los RECURSOS     →  "¿cuánto puede consumir?"
```

Y el porqué de fondo, que es la idea potente: **el host gestiona los procesos, y desde ahí limita lo que cada uno puede hacer.** Ambiente controlado y aislado, sin duplicar sistemas operativos.

> 🎓 **Para el parcial, si te preguntan**
> **¿Diferencia entre namespaces y cgroups?** Ambos son mecanismos del kernel Linux sobre los que Docker construye el container. Los **namespaces** proveen aislamiento de recursos del SO: le dan al proceso su propio árbol de procesos (PID), interfaces de red, puntos de montaje (mount), hostname (UTS), IPC y UID/GID — controlan **qué ve**. Los **cgroups** (control groups) limitan y contabilizan el consumo de recursos —CPU, memoria, I/O de disco y red— controlando **cuánto puede usar**, lo que mitiga el efecto bad neighbor. Docker es una capa de abstracción sobre ambas primitivas, las mismas que usa LXC.

## 5. 🔴 Qué es Docker

Con los mecanismos ya explicados, la definición cierra sola:

> **Docker posibilita empaquetar una aplicación con todas sus dependencias en una unidad estandarizada de desarrollo de software.**

Qué entra en ese paquete:

- Código fuente
- **Runtime** (Node.js, Python, la JVM, etc. — el motor que ejecuta el lenguaje)
- **System tools** (las herramientas del sistema)
- **System libraries** (las librerías del sistema)
- Cualquier otra cosa que se pueda instalar en un servidor

Y el resultado: **garantiza que el software siempre correrá igual, independientemente de su ambiente.**

**El problema que resuelve** tiene nombre propio y es un clásico de la industria: *"works on my machine"* — funciona en mi máquina, y explota en producción. Antes de Docker, la alternativa era documentar largos pasos de instalación y rezar para que la máquina destino terminara idéntica. Con Docker, **la misma imagen corre igual en la laptop de cualquier dev, en CI/CD, en staging y en producción.** Fin del "en mi máquina funciona".

- **CI/CD:** *Continuous Integration / Continuous Delivery* — los sistemas automáticos que construyen, prueban y despliegan el código en cada cambio. **Staging:** el ambiente de prueba que imita producción antes de publicar.

⚠️ **La única excepción al empaquetado**, ya adelantada en §4.1 y que vale marcar de nuevo porque es criterio evaluado: la configuración que **debe** cambiar entre ambientes — el string de conexión a la base de datos, las API keys — no viaja adentro de la imagen. Se externaliza en un archivo montado o en variables de entorno.

## 6. 🔴 Los tres conceptos

Toda la unidad se para sobre tres pilares. Diferenciarlos es lo primero que se espera:

| Concepto | Definición | Analogía OOP |
|---|---|---|
| **`image`** | Template **read-only** del sistema de archivos | La **clase** |
| **`registry`** | Almacén de imágenes (como GitHub, pero para imágenes) | El repositorio |
| **`container`** | **Instancia en ejecución** de una imagen | El **objeto** (instancia) |

La analogía con programación orientada a objetos es la que más rinde: **imagen = clase, container = instancia**. De una imagen se instancian N containers, todos idénticos al nacer, cada uno con su vida propia después.

Cada uno se desarrolla en su lugar: **imágenes y registry** en la Parte 2, **containers** en la Parte 3.

## 7. 🟡 Primer container: `hello-world`

La verificación de la instalación y el primer container, con su salida real. **Estos comandos son la demo de esta parte** — cada uno se explica acá mismo, y son los únicos que esta parte propone ejecutar.

```console
$ docker --version
Docker version 28.5.0, build 887030fbe8

$ docker info
Client: Docker Engine - Community
 Version:    28.5.0
 Context:    colima                    # ← qué motor está atendiendo (varía por instalación)
Server:                                # ← si aparece "Server", el motor responde: está OK
 Containers: 3
  Running: 1
  Paused: 0
  Stopped: 2                           # ← existir ≠ estar corriendo (Parte 3)
 Images: 22
 Server Version: 28.4.0
 Storage Driver: overlayfs             # ← el driver de capas apiladas (Parte 2)
 Cgroup Driver: cgroupfs
 Cgroup Version: 2                     # ← los cgroups de §4.3, acá abajo
 ...
 Kernel Version: 6.8.0-64-generic      # ← el kernel COMPARTIDO por todos los containers
 Operating System: Ubuntu 24.04.2 LTS
 Docker Root Dir: /var/lib/docker      # ← dónde Docker guarda todo, en disco
```

`docker --version` muestra la versión del cliente; `docker info` es la ficha completa del motor: cuántos containers y cuántas imágenes hay, qué storage driver usa, y — la línea que conviene mirar con lo aprendido en §2 — **cuál es el kernel**, uno solo, que todos los containers van a compartir. *(Los números concretos varían según cada instalación — lo que importa es saber leer las columnas.)*

```console
$ docker run hello-world
Unable to find image 'hello-world:latest' locally      # ← 1. no está en el cache local
latest: Pulling from library/hello-world               # ← 2. la descarga de Docker Hub
58dee6a49ef1: Pull complete
Digest: sha256:5dd0d3e6e255913fc30f90b9f2b1d359cc2cbdb48090cc4b65f1676e203243cc
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.
```

`docker run hello-world` corre un container desde la imagen `hello-world` — una imagen mínima de prueba pensada exactamente para esto. **Qué hizo Docker internamente**, paso por paso:

1. Buscó la imagen `hello-world` en el **cache local**.
2. No la encontró → la **descargó de Docker Hub** (el registry público).
3. Creó un **container** a partir de la imagen.
4. **Ejecutó** el container: imprimió el mensaje.
5. El container **terminó y se detuvo**.

Y el propio mensaje revela la arquitectura de Docker, que conviene leer con atención porque explica el punto 5: hay un **cliente** (el comando `docker` que tipeás) y un **daemon** (el motor que hace el trabajo). El cliente contacta al daemon, el daemon baja la imagen, crea el container, lo corre, y le devuelve la salida al cliente para que la muestre en tu terminal.

- **Daemon:** un programa que corre permanentemente de fondo, esperando pedidos. El daemon de Docker es quien realmente administra imágenes y containers; el comando `docker` solo le habla.

Después de esto, el container quedó **detenido**: existe, ya no corre. Se puede comprobar con `docker ps -a` (👁️ → se enseña en la Parte 3, junto con toda la distinción existir ≠ correr).

## 8. 🟢 Lo que viene en la unidad

La agenda completa de la clase, para ubicarse:

1. ¿Qué es la virtualización? VMs vs Containers ✅ *(Parte 1)*
2. Linux Containers (LXC) — fundamentos ✅ *(Parte 1)*
3. ¿Qué es Docker? El problema que resuelve ✅ *(Parte 1)*
4. Conceptos clave: image · registry · container ✅ *(Parte 1)*
5. Docker Images, Dockerfile y capas de filesystem → *Parte 2*
6. Docker Registry y gestión de imágenes → *Parte 2*
7. Docker Containers: namespaces y cgroups ✅ *(Parte 1 — la teoría; los comandos, Parte 3)*
8. Eficiencia en disco: stackable layers + copy-on-write → *Parte 3*
9. Volúmenes y Networking básico → *Parte 3*
10. Cheat sheet y demos → *Partes 3 y 4*

---

## ✅ Checkpoint — Parte 1

*Sin mirar el apunte. Las respuestas no están acá a propósito.*

1. Contá el dilema de los cinco microservicios: qué se paga con una VM por servicio, y qué cinco problemas aparecen si se los junta a todos en un solo servidor.
2. ¿Qué es el efecto bad neighbor y hasta dónde puede escalar el daño?
3. Reconstruí la tabla VM vs container: kernel, overhead, arranque, tamaño, memoria, portabilidad, aislamiento.
4. ¿Qué diferencia hay entre virtualización a nivel de hardware y a nivel de sistema operativo? ¿Por qué un container no puede correr un SO distinto al del host?
5. ¿Por qué la eficiencia de los containers es un argumento económico y no solo técnico?
6. ¿Qué es LXC, en qué año surge y cuál es su "punto clave"? ¿Por qué no se popularizó?
7. Contá el recorrido LXC → Docker → runC/OCI. ¿Cuál es el aporte de Docker que trasciende a la herramienta?
8. Enumerá los seis namespaces y qué aísla cada uno. ¿Cuál explica que dos containers escuchen en el 8080 sin chocar?
9. ¿Qué controlan los cgroups y qué problema mitigan?
10. Explicá la doble identidad del PID con el diagrama de los dos containers: ¿qué ve cada proceso adentro, qué ve el host, y por qué el host tiene la verdad? ¿En qué entornos se puede comprobar directamente desde tu terminal, y en cuáles no — y por qué?
11. ¿Qué empaqueta Docker exactamente, y qué garantiza ese empaquetado? ¿Cuál es la única excepción y por qué?
12. Definí image, registry y container, con la analogía de OOP.
13. Enumerá los cinco pasos que ejecuta Docker al correr `docker run hello-world` con la imagen ausente. ¿Qué rol cumplen el cliente y el daemon?

---

**FIN DE LA PARTE 1** · *Sigue: Parte 2 — Imágenes: capas, Dockerfile, build y el registry.*
