# 🐳 Clase desde Cero — Docker · Módulo 1
## El problema: por qué correr varias aplicaciones en una misma máquina duele

**Serie:** Clase desde Cero — Docker · Módulo 1 de 7 · El mapa completo está en `clase-cero-docker-roadmap.md`

---

### Sobre este documento

**Qué cubre:** qué necesita de verdad una aplicación para correr (spoiler: mucho más que su código) · los tres choques que ocurren cuando varias aplicaciones conviven en una misma máquina · la máquina virtual como primera solución seria, y el precio que cobra · los parches que la industria usaba antes de Docker · y la lista exacta de deseos que ninguna de esas soluciones cumplía junta.

**Qué NO cubre:** Docker. En este módulo no vas a ver ni un comando de Docker. Es a propósito: Docker es la respuesta a un problema, y si no sentiste el problema primero, la respuesta parece magia arbitraria. El módulo 2 arranca con la idea que lo hace posible.

### De dónde venís

Se asume que: programaste alguna vez (cualquier lenguaje) · usaste una terminal aunque sea para `git` y `node` · corriste alguna vez un servidor local y lo viste en `http://localhost:algo`.

**No se asume nada más.** Ni sistemas operativos, ni redes, ni infraestructura. Cada término técnico se explica la primera vez que aparece.

---

## 1. 🟡 La escena: una aplicación nunca viaja sola

Imaginate la aplicación del TP de esta materia: una API que organiza actividades al aire libre, consulta un servicio de pronóstico del clima, y guarda actividades y participantes en una base de datos. La escriben siete personas, en siete máquinas distintas, y en algún momento tiene que correr en un servidor en la nube que no es de nadie.

> 🕳️ **Madriguera — "la nube"**
> Alquilar servidores ajenos (de Amazon, Google, etc.) en vez de comprar los propios. Pagás por los recursos que reservás. Con eso alcanza por ahora; el deploy en la nube tiene su propia clase más adelante en la materia.
> *Volvé al camino.*

Cuando pensás "mi aplicación", probablemente pensás en tu carpeta con código. Pero el código, solo, no corre. Mirá lo que la máquina necesita tener para que esa API responda un request:

```
Lo que creés que es tu app          Lo que tu app es de verdad
┌──────────────────┐               ┌────────────────────────────────────┐
│                  │               │  tu código fuente                  │
│   📁 src/        │               │  + el runtime (Node 22, no el 16)  │
│      app.js      │               │  + las librerías que instalaste    │
│      ...         │               │  + librerías del sistema operativo │
│                  │               │  + herramientas (curl, openssl…)   │
│                  │               │  + archivos de configuración       │
│                  │               │  + variables de entorno            │
│                  │               │  + la base de datos, en SU versión │
└──────────────────┘               └────────────────────────────────────┘
```

Vocabulario mínimo para leer ese diagrama:

- **Runtime:** el programa que ejecuta tu código — Node para JavaScript, la JVM para Java, el intérprete de Python para Python. Tu código es texto; el runtime es quien lo hace correr.
- **Librería del sistema:** código compartido que el sistema operativo ofrece a todos los programas (manejo de red, de criptografía, etc.). Tus dependencias de `npm` muchas veces dependen de estas por debajo.
- **Variable de entorno:** un valor con nombre que vive fuera del código y el proceso lee al arrancar (por ejemplo, la contraseña de la base de datos). Se usan justamente para no escribir esos valores dentro del código.
- **Proceso:** un programa *en ejecución*. El archivo `app.js` es un programa; cuando hacés `node app.js`, el sistema operativo crea un proceso: le da memoria, lo anota en su lista con un número único (el **PID**, process ID) y lo pone a correr. Un mismo programa puede tener muchos procesos a la vez.

Todo eso de la columna derecha tiene **versiones**. Y acá empieza el problema.

---

## 2. 🔴 "En mi máquina funciona": la frase más cara del software

La escena clásica, con el equipo del TP:

Vos desarrollás con Node 22. Tu compañero clona el repo, tiene Node 16, y la app explota con un error de sintaxis: usaste una feature que su runtime no conoce. Otro compañero tiene la base de datos en otra versión, y una query que a vos te anda a él le falla. Un cuarto no tiene una herramienta del sistema que tu código invoca, y ni sabía que hacía falta. La app es la misma. El código es el mismo. **El entorno no.**

Y el final del cuento es siempre igual: "en mi máquina funciona". La frase es cierta — y es inútil, porque la app no se entrega en tu máquina. Se entrega en un servidor que tiene *otro* entorno más.

¿Cómo se resolvía esto? Con un documento. Un README de instalación: "instalá Node versión tal, después la base versión cual, configurá esto, exportá aquella variable…". Veinte pasos manuales, ejecutados por humanos, en máquinas todas distintas. Cada paso es una oportunidad de que algo quede diferente, y cuando algo queda diferente, el error aparece **lejos** del paso que lo causó — y nadie sabe por qué a uno le anda y a otro no.

Guardá esta idea, porque es el corazón de todo lo que viene:

> **El problema no es escribir software. Es hacer que el software corra igual en máquinas que no son iguales.**

**Este hilo queda abierto** (H2 en el roadmap): la solución de verdad llega en el módulo 3. Antes hay que ver otro problema, porque este no viene solo.

---

## 3. 🔴 Un servidor, muchas aplicaciones: los tres choques

Cambiemos de escenario. Ahora sos la empresa que hostea. Tenés **un** servidor y querés correr **cinco** servicios en él — la API de usuarios, la de actividades, la de notificaciones, etc. Cada uno es un proceso. ¿Qué puede salir mal?

> 🕳️ **Madriguera — microservicios**
> Partir una aplicación en varios servicios chicos que corren por separado y se hablan por red. Acá solo nos importa que son *varios procesos que hay que hostear*; la arquitectura de microservicios tiene su clase propia más adelante en la materia.
> *Volvé al camino.*

### 3.1 🔴 Primer choque: el puerto es de uno solo

- **Puerto:** el número con el que el sistema operativo distingue a quién va dirigido cada dato que llega por red. La máquina tiene una dirección (su IP); el puerto dice *qué proceso* dentro de esa máquina atiende. Cuando entrás a `http://localhost:8080`, estás diciendo "esta misma máquina (**localhost**), el proceso que atiende el puerto 8080".

La regla del sistema operativo es simple: **un puerto lo puede tomar un solo proceso a la vez**. Es un recurso único, como una silla: si está ocupada, no te podés sentar encima.

Vealo pasar. Un servidor mínimo, en el lenguaje que ya conocés:

```js
// server.js — el servidor HTTP más chico posible
const http = require('http');                 // módulo nativo de Node para hacer servidores (no se instala nada)

const server = http.createServer((req, res) => {   // el servidor: esta función atiende cada request que llega
  res.end('respondió el proceso ' + process.pid);  // contestamos con nuestro PID, para saber QUIÉN respondió
});

server.listen(8080, () => {                   // acá le pedimos al sistema operativo: "reservame el puerto 8080"
  console.log('escuchando en http://localhost:8080 — soy el proceso ' + process.pid);
});

// ¿CÓMO FUNCIONA?
// 1. `node server.js` crea un proceso nuevo; el SO le asigna un PID.
// 2. El proceso le pide al SO el puerto 8080. Si está libre, el SO se lo concede.
// 3. El proceso queda vivo, esperando requests en ese puerto.
// Resultado esperado (terminal 1):
//   escuchando en http://localhost:8080 — soy el proceso 41253   ← tu PID va a ser otro número
```

Ahora, **sin cerrar esa terminal**, abrí una segunda y corré lo mismo:

```console
$ node server.js
Error: listen EADDRINUSE: address already in use :::8080   # ← EADDRINUSE = "address already in use":
    ...                                                    #   el 8080 ya es del primer proceso
```

*(La forma exacta del error varía un poco entre versiones de Node; la línea que importa es la de `EADDRINUSE`.)*

El segundo proceso no puede "compartir" el puerto ni pedir turno: falla y muere. Si tus cinco servicios son servidores web y todos quieren su puerto estándar, cuatro pierden. La salida artesanal es repartir puertos a mano — el 8080, el 8081, el 8082… — y mantener una lista mental de quién es quién. Frágil, y a escala, inmanejable.

**Hilo abierto (H6):** existe un mundo donde los cinco creen tener el 8080 y nadie choca. Módulos 4 y 6.

### 3.2 🟡 Segundo choque: versiones que no conviven

En ese mismo servidor: un servicio está escrito para Java 21. Otro es viejo, nadie lo actualiza, y necesita Java 8. ¿Qué Java instalás *en el servidor*? Se puede tener varios y malabarear cuál usa cada proceso, pero cada malabar es configuración manual que alguien tiene que hacer bien, documentar y no romper. Lo mismo con Node 16 vs 22, con versiones de librerías del sistema, con la herramienta que un servicio necesita en una versión y otro en otra.

El fondo del asunto: **todos los procesos comparten la misma instalación del sistema**. Lo que uno necesita globalmente, lo impone a todos.

### 3.3 🔴 Tercer choque: el mal vecino

Los procesos no solo comparten instalaciones. Comparten los recursos físicos: la memoria RAM, el procesador, el disco, la red. Y no hay ningún límite entre ellos.

- **Memory leak:** bug por el cual un proceso pide memoria y nunca la devuelve, así que su consumo crece sin parar.

Supongamos que el servicio de notificaciones tiene un memory leak:

```
        UN SERVIDOR · 16 GB de RAM · sin ningún aislamiento
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│   api-users      api-activ.     notific.      api-pagos  ...  │
│   ┌──────┐      ┌──────┐      ┌───────────────────────┐       │
│   │ 1 GB │      │ 1 GB │      │ 2GB→5GB→9GB→13GB→ 💥  │       │
│   └──────┘      └──────┘      └───────────────────────┘       │
│                                     memory leak               │
│         la RAM es UNA y es de TODOS a la vez                  │
└───────────────────────────────────────────────────────────────┘
```

El proceso enfermo crece, crece, y en algún momento la memoria del servidor se acaba. ¿Quién falla? **Cualquiera.** El próximo proceso que pida memoria — aunque sea uno sano — recibe el famoso *out of memory* y se cae. Un solo vecino ruidoso tira abajo el edificio entero.

Y no es solo memoria. Un proceso que escribe al disco como loco (por ejemplo, logs sin límite) le roba velocidad de lectura/escritura a los demás — los otros no *ven* sus archivos, pero *sienten* su tráfico: todo se pone lento. Si llena el disco por completo, los procesos centrales de la máquina empiezan a fallar y no se puede escribir más nada. A este fenómeno se le dice **efecto bad neighbor** (mal vecino): un proceso abusivo degrada o voltea a los que tienen la mala suerte de compartir máquina con él.

Cerrando el combo, la **seguridad**: en una máquina sin aislamiento, cada proceso puede ver a todos los demás en la lista de procesos, leer archivos compartidos, y hasta matar procesos ajenos si tiene permisos. Cinco servicios conviviendo son cinco puertas de entrada al mismo living.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué problemas tiene correr múltiples servicios como procesos comunes en un mismo host?** Cuatro: conflicto de puertos (un puerto solo puede tomarlo un proceso), conflicto de versiones (todos comparten la misma instalación de runtimes y librerías), efecto bad neighbor (comparten RAM/CPU/disco/red sin límites, y un proceso abusivo degrada o voltea a los demás) y falta de aislamiento de seguridad (los procesos se ven y pueden interferir entre sí).

---

## 4. 🔴 La primera solución seria: la máquina virtual

La industria ya tenía una respuesta para todo esto, y era buena: **virtualizar**.

- **Virtualización:** hacer que una computadora finja ser varias. Con software se crean máquinas "de mentira" — **máquinas virtuales (VMs)** — que se comportan como computadoras completas e independientes, todas corriendo sobre el mismo hardware físico.
- **Hipervisor:** el software que hace ese truco (VMware, VirtualBox, KVM). Reparte el hardware real entre las VMs y traduce: cuando una VM quiere usar el procesador o el disco, el hipervisor intercede sobre el hardware de verdad.
- **Kernel:** el núcleo del sistema operativo — el programa que administra el hardware, la memoria y los procesos. Todo lo que corre en una máquina, corre *sobre* su kernel. (Este término es el protagonista absoluto del módulo 2; por ahora, esta línea alcanza.)

```
            EL MISMO SERVIDOR, ahora con VMs
┌──────────────────────────────────────────────────────────┐
│  ┌────────────────┐  ┌────────────────┐  ┌─────────────┐ │
│  │      VM 1      │  │      VM 2      │  │    VM 3     │ │
│  │  api-users     │  │  api-activ.    │  │  notific.   │ │
│  │  ──────────    │  │  ──────────    │  │  ─────────  │ │
│  │  SO completo   │  │  SO completo   │  │  SO completo│ │
│  │  (su kernel)   │  │  (su kernel)   │  │  (su kernel)│ │
│  │  RAM: 4 GB ⚓  │  │  RAM: 4 GB ⚓   │  │  RAM: 4 GB ⚓│ │
│  └────────────────┘  └────────────────┘  └─────────────┘ │
│  ════════════════ HIPERVISOR ═══════════════════════════ │
│  ──────────────── hardware físico ────────────────────── │
└──────────────────────────────────────────────────────────┘
                 ⚓ = recursos RESERVADOS, use o no use
```

Ahora sí: cada servicio vive en su propia computadora (de mentira). Cada VM tiene su sistema operativo entero, con su propio kernel. Los tres choques desaparecen: cada uno tiene *su* puerto 8080 (son máquinas distintas), *sus* versiones de todo, *su* memoria asignada que nadie más toca. El aislamiento es total — son, a todo efecto práctico, computadoras separadas.

La analogía que conviene guardar: las VMs son **casas separadas**. Cada una con su instalación eléctrica, su conexión de gas, sus cañerías. Independencia absoluta.

Pero las casas separadas se pagan. Mirá la factura:

| Costo | Por qué |
|---|---|
| **Recursos reservados, use o no use** | Al crear la VM le asignás memoria y procesador fijos. Si le diste 4 GB y usa 1, los otros 3 quedan cautivos: ninguna otra VM puede usarlos. Para cambiar la asignación, en la práctica destruís la VM y la recreás. |
| **Un sistema operativo entero por servicio** | Cinco servicios = cinco sistemas operativos corriendo, cada uno consumiendo cientos de MB de memoria y de disco solo por existir, antes de que tu app haga nada. A ese consumo fijo por existir se le dice **overhead** (el costo de la infraestructura en sí, no del trabajo útil). |
| **Arranque en minutos** | Prender una VM es bootear un sistema operativo completo. Minutos, no instantes. |
| **Mantenimiento multiplicado** | Cinco sistemas operativos son cinco que actualizar, parchear y asegurar. |
| **Traducción constante** | Cada instrucción pasa por el hipervisor hacia el hardware. Funciona muy bien, pero es una capa más entre tu código y el fierro. |

Y el costo tiene una traducción directa que conviene decir sin vueltas: **plata**. En la nube pagás por recurso reservado. Memoria cautiva que nadie usa y sistemas operativos redundantes son facturas más caras de Amazon o Google todos los meses. La eficiencia acá no es elegancia técnica — es dinero.

> 🎓 **Para el parcial, si te preguntan**
> **¿Qué limitaciones tienen las máquinas virtuales?** Reserva estática de recursos (lo asignado queda cautivo aunque no se use), overhead de un sistema operativo completo por VM (cientos de MB de memoria y disco antes de correr la aplicación), arranque lento (minutos, porque bootea un SO entero) y mantenimiento multiplicado (cada VM tiene su propio SO que actualizar). El aislamiento que ofrecen es excelente, pero se paga en recursos — y en la nube, recursos son costo directo.

---

## 5. 🟢 Los parches de esa época: imágenes doradas y recetas

Con VMs por todos lados, el problema del módulo — "que corra igual en todas partes" — se atacaba con dos parches que vale conocer porque explican qué vino después:

**La golden image.** Armabas una VM a mano — sistema operativo, runtime, dependencias, configuración, todo — la dejabas perfecta, y la **congelabas** como una imagen: una foto completa de esa máquina, lista para clonar. ¿Nuevo servidor? Se clona la foto. ¿Nueva versión de la app? Se clona la foto y se reemplaza solo el paquete de la app. Funcionaba, con dos dolores: la foto pesa lo que pesa un sistema operativo entero (cientos de MB o gigas por versión), y *actualizarla* — cambiar una pieza de adentro — era rearmar y recongelar. Incómodo, pesado, lento de distribuir.

**La receta scripteada.** En vez de congelar el resultado, se escribía la *receta*: un script que, corrido sobre una máquina limpia, instalaba y configuraba todo solito. Herramientas como Vagrant hacían exactamente eso. Más liviano de distribuir que una foto de gigas — pero la receta corre pasos sobre máquinas vivas: tarda, y puede fallar por mitades.

> 🕳️ **Madriguera — Vagrant**
> Herramienta para describir y crear VMs desde un archivo de configuración. Fue muy popular para entornos de desarrollo; hoy, para este problema, quedó desplazada por lo que vas a aprender en esta serie. No la vas a necesitar.
> *Volvé al camino.*

Fijate qué dejaron estos parches: la idea de **empaquetar el entorno completo** (golden image) y la idea de **describirlo como código** (la receta). Las dos ideas eran correctas. Lo que faltaba era poder hacerlas sin cargar con un sistema operativo entero cada vez.

---

## 6. 🔴 El deseo imposible

Resumamos el módulo en una tabla. Dos formas de correr cinco servicios, y lo que cada una te da:

| Queremos… | Procesos pelados en un host | Una VM por servicio |
|---|---|---|
| Aislamiento (puertos, versiones, recursos, seguridad) | ❌ nada | ✅ total |
| Eficiencia (sin recursos cautivos ni SO redundantes) | ✅ máxima | ❌ carísima |
| Arranque | ✅ instantáneo | ❌ minutos |
| "Corre igual en cualquier máquina" | ❌ ni cerca | 🟡 con golden images: pesado e incómodo |

Cada columna tiene lo que a la otra le falta. Lo que la industria quería era la columna que no existe:

> Que cada servicio corra **aislado como si tuviera su máquina** (su puerto 8080, sus versiones, sus límites de recursos)…
> …pero que por dentro sea apenas **un proceso más** del sistema: sin SO propio, sin reserva cautiva, arrancando en milisegundos…
> …y que viaje como **un paquete** con todas sus dependencias adentro, que corra idéntico en la máquina de cada uno de tus seis compañeros y en el servidor de la nube.

Esa tercera columna existe. Se llama **container**, y la herramienta que la volvió usable para cualquier desarrollador se llama **Docker**. Pero para entender el truco — cómo se puede estar aislado *sin* tener máquina propia — hay que abrir el sistema operativo y mirar la pieza que las VMs duplicaban y los containers deciden compartir: el kernel. Eso es exactamente el módulo 2.

---

## ✅ Checkpoint del Módulo 1

*Sin mirar el material. Las que no salgan marcan qué sección releer. (Las respuestas no están acá a propósito.)*

1. Una aplicación es "código + otras cosas". Nombrá al menos cinco de esas otras cosas.
2. ¿Por qué "en mi máquina funciona" es un problema y no una solución? ¿Qué es lo que difiere entre máquinas si el código es el mismo?
3. ¿Por qué dos procesos no pueden escuchar en el mismo puerto de la misma máquina?
4. Explicá el efecto *bad neighbor* con un ejemplo de memoria y uno de disco.
5. ¿Qué tres choques aparecen al correr varios servicios como procesos comunes en un mismo servidor?
6. Las VMs resuelven el aislamiento por completo. ¿Qué cuatro costos cobran a cambio?
7. ¿Por qué "recursos reservados que no se usan" es un problema de plata y no solo de elegancia técnica?
8. ¿Qué era una golden image, qué problema atacaba y cuáles eran sus dos dolores?
9. Describí con tus palabras la "tercera columna": las tres propiedades que queremos combinar y que ni el proceso pelado ni la VM dan juntas.

---

## Qué viene en el Módulo 2

La pregunta queda armada: ¿cómo puede un proceso estar aislado como una máquina sin *ser* una máquina? La respuesta vive dentro de Linux y tiene dos piezas con nombre propio — **namespaces** y **cgroups** — más una decisión clave: compartir el kernel en vez de duplicarlo. De paso vas a entender por qué Docker en tu Mac (y en Windows) esconde una máquina virtual chiquita, y qué historia conecta un invento de 2008 llamado LXC con la herramienta que usás hoy.

**FIN DEL MÓDULO 1**
