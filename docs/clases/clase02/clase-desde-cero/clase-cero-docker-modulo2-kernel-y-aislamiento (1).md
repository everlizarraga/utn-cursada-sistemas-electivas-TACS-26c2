# 🐳 Clase desde Cero — Docker · Módulo 2
## Aislar sin virtualizar: el kernel compartido

**Serie:** Clase desde Cero — Docker · Módulo 2 de 7 · El mapa completo está en `clase-cero-docker-roadmap.md`

---

### Sobre este documento

**Qué cubre:** qué es el kernel, y de paso dos palabras que todo el mundo usa y casi nadie define — **sistema operativo** y **distribución** · la idea que hace posible el container: compartir el kernel en vez de duplicarlo · las dos piezas del truco, **namespaces** (qué ve un proceso) y **cgroups** (cuánto usa) · la comparación final VM vs container · la historia de cómo un truco de administradores de sistemas (LXC, 2008) se convirtió en el estándar de la industria (Docker, 2013) · y el capítulo del Linux escondido: por qué en Mac y en Windows hay una VM, cómo es por dentro en cada uno (no son iguales), y por qué en Linux no hace falta nada.

**Qué NO cubre:** todavía no aparece la **imagen** (el paquete) ni el Dockerfile — módulo 3. Tampoco los comandos para operar containers — módulo 4. Este módulo es el corazón conceptual de toda la serie: si esto queda firme, el resto son consecuencias.

### De dónde venís

Del módulo 1 traés: los tres choques de correr procesos sin aislamiento (puertos, versiones, mal vecino) · la VM como solución que aísla perfecto pero cobra caro (recursos cautivos, un sistema operativo por servicio, arranque en minutos) · y la "tercera columna" deseada: aislamiento de VM + costo de proceso + paquete portable. Este módulo entrega las dos primeras.

---

## 1. 🔴 El kernel: el único programa que manda

Cuando prendés una computadora, antes que cualquier otra cosa, arranca **el kernel**: el núcleo del sistema operativo. Es el único programa que toca el hardware de verdad. Todo lo demás — tu navegador, tu terminal, tu `node server.js` — son procesos que viven *arriba* del kernel y le piden todo por favor:

```
        ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌──────────┐
        │ navegador │ │ terminal  │ │ node      │ │ mysql    │   PROCESOS
        └─────┬─────┘ └─────┬─────┘ └─────┬─────┘ └────┬─────┘
              │  "dame memoria" · "abrí este archivo"  │
              │  "mandá esto por la red" · "creá un proceso"
        ┌─────▼───────────────▼───────────────▼────────▼─────┐
        │                      KERNEL                        │  ← el único que manda
        └─────┬───────────────┬───────────────┬────────┬─────┘
        ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐ ┌▼─────┐
        │    CPU    │   │    RAM    │   │   disco   │ │ red  │   HARDWARE
        └───────────┘   └───────────┘   └───────────┘ └──────┘
```

Un proceso no puede escribir en el disco "por su cuenta": le pide al kernel. No puede mandar un paquete de red: le pide al kernel. Hasta para saber qué hora es, le pregunta al kernel. A esos pedidos se les dice **llamadas al sistema**, y son el único puente entre los procesos y el mundo físico.

### 1.1 🔴 Kernel, sistema operativo, distribución: tres palabras que no son sinónimos

Estas tres se usan como intercambiables y no lo son. Definirlas ahora te va a ahorrar confusiones durante toda la serie (y reaparecen con rol estelar en el módulo 3):

- **Kernel:** *un* programa — el núcleo. El motor.
- **Sistema operativo (SO):** no es un programa, es un **paquete**: el kernel **más todo el séquito** que lo rodea — la *shell* (el programa que te da la terminal), las herramientas básicas (`ls`, `cp`, `cat`…), las librerías del sistema, el gestor de paquetes, los archivos de configuración, a veces el entorno gráfico. El kernel es el motor; el SO es el auto completo.
- **Distribución (distro):** acá viene la sutileza que casi nadie cuenta — **"Linux", en sentido estricto, es SOLO el kernel.** Nadie usa "Linux pelado": usás Linux más un séquito que alguien eligió, empaquetó y bautizó. Ese paquete-con-nombre es una **distribución**: Ubuntu, Debian, Kali, Alpine. Todas comparten (esencialmente) el mismo kernel Linux; difieren en el ajuar — qué herramientas traen, qué gestor de paquetes usan, qué tan livianas son.

```
   kernel  =  el motor
   SO      =  el auto completo (motor + carrocería + tablero + ruedas)
   distro  =  un auto armado y bautizado por alguien
              (mismo motor Linux; Ubuntu, Debian y Alpine
               difieren en todo lo demás)
```

De acá salen dos consecuencias que explican todo el módulo:

1. **Todo lo que corre en una máquina, corre sobre UN kernel.** El kernel es quien lleva la lista de procesos, quien reparte la memoria, quien sabe qué proceso tiene qué puerto.
2. **Las VMs del módulo 1 eran caras exactamente por esto:** cada VM traía *su propio kernel* con *su SO entero* alrededor. Cinco servicios = cinco kernels haciendo cinco veces el mismo trabajo administrativo, cada uno con su factura de memoria y disco.

## 2. 🔴 La idea que cambia todo: no dupliques el kernel — hacé que mienta

Acá viene el salto conceptual de toda la serie. Leelo despacio:

> ¿Y si en vez de darle a cada aplicación una computadora entera (con su kernel), dejamos que todas sean **procesos comunes del mismo sistema**… pero el kernel le **miente** a cada una, haciéndole creer que tiene una máquina para ella sola?

Eso es un **container**: un proceso normal del host, al que el kernel le construye una **burbuja de mentiras piadosas**. Adentro de la burbuja, el proceso ve su propia lista de procesos, su propia red con su propio puerto 8080, su propio disco. Afuera de la burbuja, es un proceso más en la lista del kernel, al lado de tu navegador.

```
        CON VMs (módulo 1)                 CON CONTAINERS (este módulo)
   ┌───────┐ ┌───────┐ ┌───────┐        ┌───────┐ ┌───────┐ ┌───────┐
   │ app A │ │ app B │ │ app C │        │ app A │ │ app B │ │ app C │
   ├───────┤ ├───────┤ ├───────┤        │ ○     │ │ ○     │ │ ○     │ ← burbuja
   │ SO    │ │ SO    │ │ SO    │        └───┬───┘ └───┬───┘ └───┬───┘
   │ KERNEL│ │ KERNEL│ │ KERNEL│            │  procesos comunes │
   └───────┘ └───────┘ └───────┘        ┌───▼───────────▼───────▼───┐
   ═════════ HIPERVISOR ═════════       │      UN SOLO KERNEL       │
   ───────── hardware ───────────       └───────────────────────────┘
                                        ───────── hardware ─────────
    3 apps = 3 kernels + 3 SO            3 apps = 3 procesos y punto
```

La analogía para guardar: las VMs eran **casas separadas** — cada una con su instalación eléctrica, su gas, sus cañerías, todo duplicado. Los containers son **departamentos de un edificio**: la estructura, las cañerías y la instalación eléctrica son **una y compartida** (el kernel), pero cada departamento tiene su puerta con llave, su medidor y su privacidad. Nadie diría que vivir en un departamento es "compartir la casa con los vecinos" — y sin embargo, construir el edificio costó una fracción de construir treinta casas.

¿Y cómo hace el kernel para mentir así? Con dos mecanismos que vienen incluidos en Linux desde hace años. No son de Docker — son del kernel. Docker (ya vas a ver) es quien los volvió usables.

## 3. 🔴 Las dos piezas del truco

La burbuja tiene dos mitades, y la división es tan limpia que conviene memorizarla como pregunta y respuesta:

| Mecanismo | Responde a | Ejemplos |
|---|---|---|
| **Namespaces** | ¿Qué **VE** el proceso? | su lista de procesos, su red, su disco, su hostname |
| **Cgroups** | ¿Cuánto **USA** el proceso? | tope de memoria, cuota de CPU, límite de disco y red |

> 📌 **Vocabulario oficial de la serie:** de acá en adelante, **burbuja** = el envoltorio aislante completo, namespaces + cgroups. Cuando leas "burbuja", son esas dos piezas trabajando juntas. (Vas a ver más adelante que la mitad-namespaces aparece a veces sola, aislando otras cosas que no son containers — la idea es tan buena que se usa por partes.)

### 3.1 🔴 Namespaces: cada uno ve su propio mundo

- **Namespace** ("espacio de nombres"): un mecanismo del kernel que le da a un grupo de procesos una **vista privada** de algún recurso del sistema. Mismo kernel, vistas distintas.

Hay varios tipos, uno por cada cosa que se puede "privatizar". Los tres que cargan el peso en esta materia:

**Namespace de PID — tu propia lista de procesos.** Adentro del container, el proceso principal se ve a sí mismo como **PID 1** — el primer y único proceso del universo, como si la máquina hubiera booteado solo para él. No ve tu navegador, no ve tus otros containers, no ve *nada* de afuera. Pero es una vista, no una realidad: para el kernel, ese mismo proceso es uno más de la lista general, con un PID cualquiera:

```
    DESDE ADENTRO de la burbuja          DESDE AFUERA (la lista real del kernel)
   ┌───────────────────────────┐         ...
   │  PID 1   tu app  ← "soy   │         PID 48213   tu app   ← EL MISMO proceso,
   │          el primero y     │         PID 48214   (su hijo)   con su nombre real
   │          casi el único"   │         PID 48215   otro container
   │  PID 12  (un hijo tuyo)   │         PID 1002    tu navegador
   └───────────────────────────┘         ... y 300 procesos más que él no ve
```

Un mismo proceso, dos identidades según desde dónde lo mires. Guardá la imagen: en el módulo 4 la vas a comprobar con comandos, y el **PID 1** va a resultar clave para entender cuándo un container vive y cuándo muere.

**Namespace de red — tu propia red.** Cada container recibe su propia interfaz de red virtual — como si tuviera su propia placa de red — con su propia dirección y, atención, **su propia tabla de puertos**. Releé eso con el módulo 1 en la cabeza: el puerto 8080 era un recurso único *por sistema*… y ahora cada container tiene *su* sistema de red privado. Cinco containers pueden creer, todos a la vez y sin pelearse, que están escuchando en el 8080. El choque de puertos del módulo 1 acaba de evaporarse — cómo se conecta ese mundo privado con el exterior es tema del módulo 6 (hilo H6).

**Namespace de mount — tu propio disco.** Cada container ve su propio árbol de directorios: su `/`, su `/home`, sus archivos. No ve el disco del host, ni el de los otros containers. Y esto abre una pregunta enorme que este módulo **no** va a responder: si el container no trae sistema operativo… **¿qué es ese disco que ve?** ¿De dónde salen esas carpetas de "Ubuntu" cuando el host ni siquiera es Ubuntu? Ese es el hilo H3, y es exactamente el tema del módulo 3. Por ahora, solo registrá que la vista del disco es privada.

Los otros tres, para que la lista esté completa (aparecen poco en la práctica de esta materia): **UTS** (cada container tiene su propio *hostname* — el nombre con el que la máquina se presenta), **IPC** (sus propios canales de comunicación entre procesos) y **User** (sus propios usuarios — ser administrador adentro no implica serlo afuera).

### 3.2 🔴 Cgroups: nadie come más de lo que le toca

- **Cgroup** (*control group*): un mecanismo del kernel para ponerle **límites de consumo** a un grupo de procesos: tanta memoria como máximo, tanta CPU, tanto ancho de disco y de red.

Los namespaces resuelven qué ve cada uno; los cgroups resuelven el **mal vecino**. Volvé al desastre del módulo 1 — el servicio de notificaciones con su memory leak comiéndose los 16 GB y volteando a los vecinos — y mirá la misma escena con cgroups:

```
     RAM: 16 GB, ahora con límites por container
   ┌──────────┬──────────┬────────────────┬──────────┐
   │ api-users│ api-activ│ notificaciones │  libre   │
   │ tope 2GB │ tope 2GB │ tope 4GB       │          │
   │ usa 1GB  │ usa 1GB  │ 2→3→4GB 💥     │          │
   └──────────┴──────────┴────────────────┴──────────┘
                              ▲
              el leak choca contra SU techo de 4GB
              y muere SOLO. Los vecinos ni se enteran.
```

El proceso enfermo ya no puede arrastrar a nadie: crece hasta su tope, el kernel lo frena ahí, y si revienta, revienta solo. El edificio tiene medidores por departamento.

🟡 **La letra chica: el overcommit.** Los límites son opcionales, y ahí hay una decisión de negocio real. Sin límites, todos los containers "ven" los 16 GB completos y los usan a demanda — es más barato (aprovechás cada GB al máximo, sin memoria cautiva como en las VMs), pero volvés al riesgo del mal vecino. Con límites estrictos, cada uno tiene su garantía — más previsible, pero quizás pagás por GB que quedan sin usar. A esa práctica de "prometer entre todos más recursos de los que hay, apostando a que no los usen a la vez" se la llama **overcommit**, y las empresas viven ajustando esa perilla: costo contra estabilidad.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué son namespaces y cgroups?** Son los dos mecanismos del kernel de Linux que hacen posible el container. Los namespaces aíslan **lo que un proceso ve**: su propio árbol de procesos (donde se cree PID 1), su propia interfaz de red con sus puertos, su propia vista del file system, su hostname. Los cgroups limitan **lo que un proceso usa**: memoria máxima, cuota de CPU, ancho de disco y red — eliminando el efecto bad neighbor. Un container es un proceso común del host envuelto en namespaces y cgroups.

## 4. 🔴 La cuenta final: VM vs container

Con las dos piezas sobre la mesa, la comparación completa — probablemente la tabla más evaluable de toda la unidad:

| | **Máquina virtual** | **Container** |
|---|---|---|
| Qué virtualiza | El **hardware** (hipervisor) | El **sistema operativo** (namespaces + cgroups) |
| Kernel | **Propio** — uno por VM | **Compartido** — el del host, para todos |
| Qué es, en el fondo | Una computadora completa simulada | Un **proceso** del host con una burbuja |
| Arranque | Minutos (bootea un SO entero) | **Milisegundos** (es lanzar un proceso) |
| Overhead | Un SO completo por VM: cientos de MB antes de que la app haga nada | Casi cero: no hay SO extra corriendo |
| Recursos | **Reservados** al crearla — cautivos aunque no se usen | **Limitados** con cgroups — flexibles, sin cautiverio |
| Densidad en un host | Unas pocas | **Cientos** |
| Aislamiento | Total — a nivel hardware | Fuerte — a nivel kernel |

🟡 Una honestidad sobre la última fila: el aislamiento de la VM sigue siendo *más* fuerte. Una VM es una computadora aparte a todo efecto; un container confía en que el kernel compartido mienta bien — y un kernel compartido es, por definición, una superficie de contacto entre vecinos. Por eso en escenarios donde conviven clientes que no se tienen ninguna confianza, las VMs siguen teniendo su lugar. Para el caso normal — tus servicios, tu equipo, tu empresa — el aislamiento del container sobra, y la diferencia de costo es abismal.

> 🎓 **Para el parcial, si te preguntan**
> **¿Diferencias entre una máquina virtual y un container?** La VM virtualiza hardware: un hipervisor le asigna recursos reservados y adentro corre un sistema operativo completo con su propio kernel — aislamiento total, pero con overhead de un SO por VM, arranque en minutos y recursos cautivos. El container virtualiza a nivel del sistema operativo: es un proceso del host, aislado con namespaces (qué ve) y cgroups (cuánto usa), que **comparte el kernel del host** — arranca en milisegundos, casi no tiene overhead y permite cientos por máquina, a cambio de un aislamiento fuerte pero no absoluto (el kernel es compartido).

## 5. 🟡 La historia: quince años de un truco buscando su forma

Las piezas del kernel existían y alguien las tenía que empaquetar. La historia tiene dos fechas y una moraleja.

**2008 — LXC (Linux Containers).** El primer empaquetado serio de namespaces + cgroups en una herramienta usable: nace la **virtualización a nivel de sistema operativo**, por contraste con la virtualización a nivel de hardware del hipervisor. ¿El motor? El de siempre: **plata**. Era la era de la nube — pagar por recursos alquilados — y correr diez servicios como containers en una máquina, en vez de diez VMs, era una factura descaradamente más chica.

¿Y por qué no explotó? Porque LXC era **territorio de administradores de sistemas**: exigía conocimiento fino de Linux, configuración de bajo nivel, nada amigable para el desarrollador común. La tecnología estaba; le faltaba la puerta de entrada.

**2013 — Docker.** Una empresa llamada dotCloud libera, como código abierto y escrito en Go, una herramienta que hace lo mismo que LXC pero **pensada para desarrolladores**: comandos simples, y una idea nueva que le va a dar la vuelta al mundo — el empaquetado en *imágenes* (módulo 3, paciencia). En pocos años, "container" y "Docker" se vuelven casi sinónimos.

🟢 **Bajo el capó, para completar el cuadro:** Docker arrancó usando LXC por dentro; después lo reemplazó por su propia pieza (*libcontainer*), y de ahí nació **runC** — el ejecutor de containers que Docker donó como estándar abierto bajo la **OCI** (Open Container Initiative), para que la industria entera comparta el mismo formato en vez de fragmentarse. Si hiciste el setup, ya los viste sin saberlo: tu `docker version` lista `containerd` y `runc` — son estas piezas, corriendo hoy en tu máquina.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué es LXC y en qué se diferencia de la virtualización tradicional?** LXC (Linux Containers, 2008) fue la primera herramienta que empaquetó namespaces y cgroups: inaugura la virtualización **a nivel de sistema operativo** — aislar procesos compartiendo el kernel — frente a la virtualización **a nivel de hardware** del hipervisor, donde cada VM corre su propio SO. Ahorraba enormes recursos (y costo de nube), pero era complejo, de bajo nivel, orientado a sysadmins; Docker (2013) tomó la misma base y la volvió accesible para desarrolladores, sumando el empaquetado en imágenes.

## 6. El Linux escondido: qué hay de verdad en tu máquina

Todo el módulo dice "el kernel" — pero seamos precisos: namespaces y cgroups son mecanismos **del kernel de Linux**. Un container estándar es un proceso *de Linux* con burbuja *de Linux*. Necesita un kernel de Linux abajo, sí o sí. Ahora veamos qué significa eso en cada sistema — porque no es igual en todos, y las diferencias enseñan.

### 6.1 🔴 En Linux nativo: nada que agregar

Si tu máquina corre Ubuntu, Debian o cualquier distro, el kernel correcto **ya es el tuyo**. Docker se instala y corre nativo: tus containers son procesos de tu propio kernel. Un solo kernel en toda la máquina. Fin de la historia — y por eso los servidores del mundo real, donde viven los containers en producción, son Linux.

### 6.2 🔴 En Mac: la VM privada de Docker Desktop

El kernel de macOS se llama **Darwin**, y no tiene namespaces ni cgroups de Linux. Solución: Docker Desktop se trae **su propia VM** — mínima, con un Linux pelado adentro — y todos tus containers viven ahí:

```
   ┌─────────────────────────────────────────────┐
   │                  TU MAC                     │
   │           kernel Darwin ← kernel #1         │
   │                                             │
   │   Docker Desktop (app)                      │
   │   ┌─────────────────────────────────────┐   │
   │   │  SU VM privada, mínima              │   │
   │   │  kernel LINUX ← kernel #2           │   │
   │   │   ┌────────┐ ┌────────┐ ┌────────┐  │   │
   │   │   │ contnr │ │ contnr │ │ contnr │  │   │
   │   │   │   A    │ │   B    │ │   C    │  │   │
   │   │   └────────┘ └────────┘ └────────┘  │   │
   │   └─────────────────────────────────────┘   │
   │    la VM existe SOLO para Docker: la app    │
   │    la prende al abrir y la apaga al salir   │
   └─────────────────────────────────────────────┘
```

Esa VM es **privada de la app**: no la ves, no la administrás, no le podés meter otras cosas — Docker Desktop es dueño de su propio edificio. Para Darwin, toda la VM es (a fines prácticos) un proceso más; para tus containers, el kernel Linux de adentro es "el kernel del host" del que habló todo este módulo. Dos kernels en total: Darwin + Linux.

La **evidencia** ya la tenés del setup: `docker version` muestra `Client: darwin` y `Server: linux` — tu terminal es de macOS, pero quien responde del otro lado es el Linux de la VM.

> 🕳️ **Madriguera — Colima**
> En Mac existe una alternativa open source a Docker Desktop llamada **Colima**: levanta su **propia** VM con Linux (otra, separada — no adentro de la de Docker Desktop) y corre el motor de Docker ahí. Pueden convivir instaladas; el cliente `docker` sabe a cuál hablarle mediante *contextos* — la línea `Context:` que viste en `docker version`. Para esta serie, Docker Desktop alcanza y sobra.
> *Volvé al camino.*

### 6.3 🟡 En Windows: WSL 2, el edificio público

El kernel de Windows se llama **NT** y tampoco sirve. Pero Microsoft tomó otro camino, más interesante — y entenderlo evita una confusión clásica que conviene desactivar de frente: **no hay VM adentro de otra VM.** Vamos por partes.

**Qué es WSL.** *Windows Subsystem for Linux*: el mecanismo oficial de Microsoft para correr Linux dentro de Windows. Vino en dos modos que conviven (por eso la columna `VERSION` cuando listás distros): **WSL 1** no tenía kernel de Linux — era un *traductor*: el kernel NT atendía los pedidos de los programas Linux haciéndose pasar por un kernel Linux; ingenioso, pero imitación incompleta, sin namespaces ni cgroups reales — **Docker no puede correr ahí**. **WSL 2** abandona la imitación: una **VM liviana** administrada por Windows, con un **kernel de Linux real** adentro (compilado por Microsoft) más la plomería mínima. Ojo: esa VM **no es "un Ubuntu"** — es Linux pelado, sin distro. Las distros van aparte, y acá está la pieza clave:

**Qué es una "distro" en WSL — y qué no es.** Recordá la sección 1.1: distro = kernel + ajuar. En WSL, el kernel ya lo pone la VM, uno solo, compartido. Entonces los fabricantes (Canonical para Ubuntu, etc.) empaquetan versiones especiales de sus distros **sin el kernel**: básicamente el disco — el árbol de archivos completo con todo el séquito (`apt`, herramientas, configuración). Cuando "instalás Ubuntu en WSL", descargás ese disco y lo enchufás. Cuando lo abrís, WSL lanza sus procesos **sobre el kernel compartido, cada distro dentro de su propia burbuja** — su vista de archivos, sus procesos, su mundo. Una distro en WSL **no es una VM: es un disco enchufado más una burbuja.**

```
   ┌────────────────────────────────────────────────────────┐
   │                      WINDOWS                           │
   │                 kernel NT ← kernel #1                  │
   │                                                        │
   │   LA VM LIVIANA DE WSL 2 (una sola, del sistema)       │
   │   ┌────────────────────────────────────────────────┐   │
   │   │      kernel LINUX ← kernel #2 (y último)       │   │
   │   │                                                │   │
   │   │  ┌─ burbuja ─┐ ┌─ burbuja ─┐ ┌── burbuja ───┐  │   │
   │   │  │  Ubuntu   │ │  Debian   │ │docker-desktop│  │   │
   │   │  │ tu distro │ │ (si la    │ │ el motor de  │  │   │
   │   │  │ (disco +  │ │  tenés)   │ │ Docker, y    │  │   │
   │   │  │  mundo    │ │           │ │ adentro:     │  │   │
   │   │  │  propio)  │ │           │ │ ┌────┐┌────┐ │  │   │
   │   │  │           │ │           │ │ │ A  ││ B  │ │  │   │
   │   │  │           │ │           │ │ └────┘└────┘ │  │   │
   │   │  │           │ │           │ │  containers  │  │   │
   │   │  └───────────┘ └───────────┘ └──────────────┘  │   │
   │   └────────────────────────────────────────────────┘   │
   └────────────────────────────────────────────────────────┘
```

Leé el diagrama con calma: **una** VM, **un** kernel de Linux, y varias burbujas colgadas de él. La distro `docker-desktop` que Docker Desktop crea al instalarse no es "otra VM adentro" — es *otro disco enchufado al mismo edificio*, una burbuja vecina de tu Ubuntu. Y adentro de esa burbuja, el motor de Docker crea las burbujas de tus containers. **Burbujas anidadas, como carpetas dentro de carpetas:** al kernel no le cuesta nada, porque una burbuja no es una máquina — es contabilidad: anotaciones en los libros del kernel sobre qué le muestra a quién. Anidar VMs sería duplicar motores (carísimo, doble trabajo); anidar burbujas es agregar renglones a un registro.

Total de kernels en tu Windows con Docker andando: **dos** — NT y el Linux de WSL 2. Igual que en Mac: dos y solo dos.

🟢 Una honestidad de alcance: los detalles finos de cómo Microsoft implementa el aislamiento entre distros son cocina interna de ellos (y usa sobre todo la mitad-*vista* de la burbuja: los límites de recursos en WSL se configuran para la VM entera, no por distro). Lo que importa acá es la **lógica** — y es exactamente la de este módulo.

### 6.4 🟡 La asimetría Mac/Windows — y el mapa final

Los dos sistemas resolvieron el mismo problema (no tengo kernel Linux) con filosofías opuestas:

- **En Windows, el sustrato es público.** WSL 2 es una funcionalidad *del sistema*, hecha para que cualquiera use Linux para lo que quiera. Es tuya: la ves, la administrás (`wsl -l -v` y compañía). Docker Desktop es **un inquilino más** que enchufa su distro al edificio que ya existía.
- **En Mac, no existe edificio público.** Cada herramienta se trae **su VM privada**. La de Docker Desktop existe solo para Docker y la administra la app: sin distros, sin lista, sin `wsl` equivalente — la VM corre el motor directamente.

| | **Ubuntu nativo** | **Mac** | **Windows** |
|---|---|---|---|
| Kernels totales | 1 (Linux) | 2 (Darwin + Linux) | 2 (NT + Linux) |
| ¿Dónde está el Linux? | Es el nativo | VM **privada** de Docker Desktop | La VM **única y pública** de WSL 2 |
| ¿El sustrato es tuyo o de la app? | Tuyo | De la app (invisible) | Tuyo (visible: `wsl ...`) |
| ¿Capa "distro"? | — | No: la VM corre el motor directo | Sí: burbujas — Ubuntu, docker-desktop… |
| Tus containers viven en… | Tu propio kernel | La VM de Docker Desktop | La burbuja `docker-desktop` |

**Y el broche, que no es casualidad:** mirá de nuevo cómo WSL 2 organiza sus distros — mundos separados, cada uno creyéndose solo, todos sobre un kernel compartido. Es **el mismo truco de este módulo**, aplicado por Microsoft a otra cosa. Tu notebook con Windows demuestra la tesis de la serie dos veces a la vez: una en cómo WSL aísla distros, otra en cómo Docker aísla containers. La idea "compartir el kernel es barato y alcanza para aislar" no ganó una batalla — se comió a la industria entera, Windows incluido.

**"¿Pará — no era que las VMs eran lo caro?"** Sí, y la cuenta sigue cerrando en los dos sistemas: es **una sola** VM, mínima, compartida por todos tus containers — no una por aplicación. Un edificio entero sobre un único terreno alquilado. 1 VM + 100 containers sigue siendo incomparablemente más barato que 100 VMs.

> 🎓 **Para el parcial, si te preguntan**
> **¿Por qué Docker necesita una VM en Mac y Windows, y en Linux no?** Porque un container comparte el kernel del sistema donde corre, y los containers estándar son de Linux: dependen de namespaces y cgroups, que son mecanismos del kernel de Linux. macOS (kernel Darwin) y Windows (kernel NT) no los tienen, así que Docker Desktop usa una única VM liviana con Linux — privada de la app en Mac, la VM compartida de WSL 2 en Windows — donde corren todos los containers. En Linux el kernel correcto ya está y Docker corre nativo. Sigue siendo eficiente porque es una sola VM para todo, no una por aplicación.

## 7. 🔴 Síntesis: la tercera columna, conseguida a medias

Volvé a la tabla de deseos con la que cerró el módulo 1, y tachá lo conseguido:

| Queríamos… | ¿Lo tenemos? | Gracias a |
|---|---|---|
| Aislamiento (puertos, versiones, recursos, seguridad) | ✅ | Namespaces (qué ve) + cgroups (cuánto usa) |
| Eficiencia (sin SO redundantes ni recursos cautivos) | ✅ | Es un proceso común; el kernel es uno y compartido |
| Arranque instantáneo | ✅ | Lanzar un proceso: milisegundos |
| **"Corre igual en cualquier máquina"** | ❌ **todavía no** | …de esto no dijimos ni una palabra |

El container ya aísla y ya es barato. Pero nada de lo visto explica cómo **empaquetar** tu app con todas sus dependencias para que corra idéntica en las siete máquinas de tu equipo y en el servidor. Y hay una punta suelta que lo delata: el namespace de mount le da al container "su propio disco"… ¿**qué hay** en ese disco? ¿Quién lo llenó? ¿Por qué adentro de un container ves las carpetas de un Ubuntu que tu máquina no tiene?

Las dos preguntas son la misma pregunta, y su respuesta es el invento más importante de Docker: la **imagen**. Módulo 3.

---

## ✅ Checkpoint del Módulo 2

*Sin mirar el material. Las que no salgan marcan qué sección releer. (Las respuestas no están acá a propósito.)*

1. ¿Qué es el kernel y qué cosas administra? ¿Qué es una llamada al sistema?
2. ¿Qué diferencia hay entre kernel, sistema operativo y distribución? ¿Qué es "Linux" en sentido estricto?
3. "Un container comparte el kernel del host." ¿Qué comparte exactamente, y qué NO comparte?
4. Namespaces y cgroups: ¿cuál responde "qué ve" y cuál "cuánto usa"? Dá dos ejemplos de cada uno. ¿Qué llama esta serie "burbuja"?
5. Un proceso dentro de un container se ve como PID 1. ¿Qué ve el kernel del host cuando mira ese mismo proceso?
6. ¿Cómo eliminan los cgroups el efecto bad neighbor? ¿Qué es el overcommit y qué se gana y se arriesga con él?
7. Reconstruí de memoria la tabla VM vs container: qué virtualiza cada uno, kernel, arranque, overhead, recursos, aislamiento.
8. ¿Por qué LXC (2008) no se masificó y Docker (2013) sí?
9. ¿Por qué en Mac y Windows hay una VM y en Ubuntu nativo no? ¿Cuántas VMs hacen falta para cien containers?
10. En Windows con Docker andando: ¿cuántas VMs hay? ¿Cuántos kernels, y cuáles? ¿Qué es la distro `docker-desktop`, y por qué NO es "una VM adentro de otra"?
11. ¿En qué se diferencia la filosofía de Mac (VM privada de la app) de la de Windows (WSL 2 como sustrato público)?
12. ¿En qué se parece la forma en que WSL 2 aísla sus distros a la forma en que Docker aísla containers?

---

## Qué viene en el Módulo 3

La pregunta quedó servida: el container ve "su propio disco" — ¿qué hay adentro y quién lo puso? La respuesta es la **imagen**: un paquete de solo lectura, armado en **capas apiladas**, que contiene el file system completo que el container va a ver — tu app, su runtime, sus librerías, todo. Vas a conocer el **Dockerfile** (la receta que construye imágenes), vas a entender por qué las capas tienen esos códigos raros (hashes), van a volver las **distros** con rol estelar (Ubuntu, Debian y Alpine como puntos de partida de tus imágenes) — y va a cerrar de un golpe el hilo más viejo de la serie: "en mi máquina funciona" deja de existir cuando la máquina viaja con la app.

**FIN DEL MÓDULO 2**
