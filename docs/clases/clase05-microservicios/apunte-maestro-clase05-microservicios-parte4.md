# 📘 Apunte maestro — Clase 05 — Microservicios
## Parte 4 de 6 — Errores, backpressure y asincronismo

> **Unidad:** `clase05` · TACS 2C 2026 · Clase del 08/09/2026 · Apunte maestro, Parte 4 de 6 · Leyenda e índice completo en la Parte 1.

**Qué cubre esta parte.** Qué pasa cuando una caja del mapa se cae o se pone lenta, y por qué sin herramientas el daño se propaga hasta el usuario. Las herramientas: timeouts, rate limiting, circuit breaker, reintentos, colas y buffers. El caso completo de backpressure paso a paso. Y la otra forma de comunicar servicios: colas de mensajes y asincronismo, con RabbitMQ y Kafka, la dead letter queue, y el criterio para saber cuándo suma y cuándo complica.

**Qué NO cubre.** Las desventajas estructurales de microservicios ni las malas prácticas: Parte 5.

**De dónde venís.** Parte 3: el mapa del e-commerce y el hecho de que Sesión y usuario, y en general cualquier caja muy llamada, es un punto del que dependen todas las demás. Parte 1, §5: colas de mensajes e idempotencia como herramientas para "las dos o ninguna".

---

## 1. El problema: manejar errores entre aplicaciones 🔴

En el monolito, un error era un asunto **a nivel aplicativo**: una excepción, quizás una library que te ayudaba, y listo. Las llamadas eran funciones y los errores eran bien conocidos.

Con ocho cajas hablando por red, las preguntas cambian:

- **¿Cómo manejo errores entre aplicaciones?** Apps caídas, apps lentas, apps que responden mal.
- **¿Qué límites de carga establezco?** Por ejemplo, un máximo de requests aceptadas.
- **¿Hay timeouts entre cajas?** Muchas veces nos olvidamos, y cuando hay un problema, explota.
- **¿Reintentos?** ¿Y qué herramientas de arquitectura me ayudan (colas, bases de datos)?
- **Backpressure**: si una app está caída y **sus clientes no reaccionan**, los errores **se propagan a través del stack**.

Ahora necesito herramientas que sirvan a nivel aplicativo **y** a nivel arquitectura, porque cuando una aplicación se cae, el resto tiene que saber qué hacer: propagar el error de forma controlada, o **degradarse *gracefully*** (caerse de a poco, caerse "bien", entre comillas), en vez de arrastrar a todos. El término **backpressure** (contrapresión) nombra ese fenómeno: la presión que una pieza lenta o caída ejerce hacia atrás sobre todas las que dependen de ella. Se ve en detalle en la sección 3; antes, las herramientas.

---

## 2. Las herramientas 🔴

No hay que usarlas todas, ni una es mejor que otra: son **consideraciones** que aparecen cuando trabajás con servicios.

### 2.1 Timeouts

**¿Cuánto esperás antes de decir "listo, no responde"?** Yo te llamo; si no respondés en un segundo, asumo que no vas a responder, y en vez de colgarme, corto. Eso es un timeout.

Regla: **timeouts entre aplicaciones, y de aplicaciones a base de datos, siempre.** Si no lo definís, una llamada se puede quedar **dos minutos colgada**, y con ella todos los que esperaban esa respuesta. Es lo que más se olvida y lo que más explota.

### 2.2 Rate limit

**¿Cuántos pedidos acepto en una franja de tiempo?** Si me pegás miles de veces en un minuto, algo está pasando: te bloqueo. Es un límite del lado del que **recibe**, para protegerse de un cliente que se desbocó (por un bug, por un ataque, por un usuario apretando F5, ver 3.3).

### 2.3 Circuit breaker

También existe el límite **al revés**: del lado del que **envía**. Yo te pego, pero vos estás tardando demasiado; entonces, en vez de comerme el timeout cada vez, **te anulo como servidor por un tiempo**. Eso es un **circuit breaker** ("disyuntor": el mismo que tenés en la casa, que corta cuando hay un problema para que no se queme todo).

El ejemplo, con números:

1. Le pego a un servicio y me da **timeout todo el tiempo**. Supongamos que el timeout es de **5 segundos**.
2. Si me quedo esperando 5 segundos en cada llamada, es ineficiente: cada request mía tarda 5 segundos para nada.
3. Entonces **anulo a ese servidor por 10 minutos**: ni siquiera espero el timeout; asumo que está abajo y respondo sin él.
4. A los 10 minutos **le pregunto: ¿estás arriba?** Si responde, todo vuelve a la normalidad. Si no, lo sigo anulando otros 10.

```
   ┌──────────────┐  timeouts seguidos  ┌────────────────┐  pasan 10 min  ┌──────────────┐
   │   NORMAL     │────────────────────►│    ANULADO     │───────────────►│  ¿ESTÁS      │
   │ llamo y      │                     │ no llamo,      │                │   ARRIBA?    │
   │ espero       │◄────────────────────│ respondo sin   │◄───────────────│ una llamada  │
   └──────────────┘   respondió bien    │ el servicio    │  sigue caído   │ de prueba    │
                                        └────────────────┘                └──────────────┘
```

Es la misma analogía del disyuntor: **yo, como cliente, penalizo a los servidores que no responden.** Se hace internamente entre servicios, y se hace mucho con proveedores.

**Caso real de un sitio de viajes (y de bancos).** Para armar una búsqueda le pego a Expedia, a Booking y a Airbnb para ver qué ofertas hay. Uno de los tres está respondiendo lento. Yo quiero responder rápido, y **mi tiempo de respuesta es el máximo de mis proveedores**: si uno tarda 8 segundos, yo tardo 8. Entonces lo penalizo un rato: lo aborto, y me pierdo sus ofertas, pero mi sistema **en general** responde más rápido. Si tengo un proveedor muy lento, me conviene sacarlo. Todas estas estrategias existen para que la aplicación **vista como un todo**, la empresa, responda bien.

> **Para el parcial, si te preguntan: ¿qué es un circuit breaker y en qué se diferencia de un timeout?**
> El timeout define cuánto espero una respuesta antes de cortar una llamada. El circuit breaker actúa cuando los timeouts se repiten: el cliente "anula" al servidor por un tiempo y deja de llamarlo (responde sin él), sin esperar el timeout en cada request; pasado ese tiempo prueba una llamada y, si el servidor responde, vuelve a la normalidad. Como un disyuntor: corta para que el problema de una pieza no se lleve al resto.

### 2.4 Reintentos y herramientas de arquitectura

Reintentar una llamada fallida es válido **si la operación es idempotente** (Parte 1, §5.2); si no, reintentar duplica. Y hay herramientas de arquitectura que absorben fallas: una **cola de mensajes** (el mensaje espera hasta que el consumidor pueda), o una base de datos como intermediaria.

### 2.5 Colas y buffers: las formas de hacer backpressure

Además de contener al que falla, puedo **regular** la velocidad entre productor y consumidor. Tres formas:

- **Con una cola de mensajes**: lo que llega se encola y se consume al ritmo del consumidor. (Sección 4.)
- **Con un buffer en memoria**: yo tengo en memoria las cosas que me llegan y las voy procesando.
- **Negociando la velocidad**: yo, cliente, le pregunto al servidor **qué tan listo está** y en base a eso le pego más rápido o más lento, como un semáforo. Frameworks como Spring lo automatizan con módulos dedicados.

⚠️ El asincronismo entre servicios es un **arma de doble filo**: es una buena herramienta de backpressure y, a la vez, un flujo que depende mucho de respuestas asíncronas y no está bien pensado se vuelve muy difícil de mantener. Se desarrolla en 4.4.

---

## 3. Backpressure: el caso, paso a paso 🔴

Ahora el fenómeno completo, sobre el mapa de la Parte 3. Cada paso es un estado de la falla; seguilos en orden.

### 3.1 Problema original: Datos se pone lento

**Datos** es la capa que llama a los proveedores externos de catálogo. Un día, esos proveedores empiezan a responder mal: tardan **30 segundos, un minuto**. Datos se pone rojo.

```
   Políticas Comerciales ───► [ DATOS ] ───► Servicios externos  (tardan 30 s, 1 min)
                                🔴
```

### 3.2 Se propaga: se agotan los recursos

Si no tengo **nada** de lo de la sección 2, **Políticas Comerciales** (que llama a Datos) se va a quedar **esperando**. Y esperar consume recursos: si tengo hilos, me quedo sin hilos; si tengo sockets, me quedo sin sockets. **Cualquier recurso finito que tenga se agota en algún momento.** Cuando Políticas Comerciales se queda sin hilos, ya no puede atender a nadie, aunque la pregunta no tuviera nada que ver con Datos.

Y eso **fluye por el resto de la aplicación**: Destacados y Facetado llaman a Políticas Comerciales y se quedan esperando; el Front llama a los dos y se queda esperando. Llega al front. **El usuario queda durísimo.**

```
   usuario ──► [ FRONT ] ──┬──► [ DESTACADOS ] ──┐
                 🔴        │         🔴          ├──► [ POL. COM. ] ──► [ DATOS ] ──► externos
                           └──► [ FACETADO  ] ───┘         🔴              🔴          (lentos)
                                     🔴
             ◄─────────── la espera se propaga hacia atrás, caja por caja ───────────
```

### 3.3 Problema autogenerado: F5, F5, F5

¿Qué hace un usuario cuando la página no carga? La respuesta buena: **se va**, no prueba más. La respuesta real, y peor: **le vuelve a dar**. F5, F5, F5, F5, F5, F5.

```
    o     F5 …
   /|\    F5 …
   / \    F5 …      ──► [ FRONT ] ──► ... cada F5 es una request nueva encima de la anterior
  usuario F5 …
          F5 …
```

Acá hay un detalle de HTTP que lo hace grave: en la mayoría de las versiones del protocolo, **las requests no son cancelables**. Una vez que la tiraste, por más que la canceles de tu lado (o cierres el browser), **el servidor la recibe y la procesa igual.** Entonces cada F5 no reemplaza la request anterior: **la suma**. Me creo un problema sobre el problema. Es el **problema autogenerado**: **cantidad excesiva de requests**, generadas por la propia falla.

Y cuando hay **starvation** (inanición: los recursos están todos ocupados y las requests nuevas no consiguen ninguno) y una **cola gigante en memoria** de requests esperando, pasan dos cosas, las dos malas: o **estalla todo por los aires**, o **aguanta**, que quizás es peor, porque después tiene que ir liberando de a poco todos los recursos que acumuló.

### 3.4 La recuperación también es en cadena

Datos se va a restablecer en algún momento: se avivaron, lo reiniciaron, anda bien. Datos vuelve a verde. **Pero el resto no vuelve a la vez.** Políticas Comerciales todavía puede estar trabada, con su cola de requests acumuladas y sus hilos ocupados. Y Front, Destacados y Facetado siguen esperando a Políticas Comerciales. Si no hiciste nada de la sección 2, **va a tardar** hasta que todo el resto funcione bien: se restablece en cadena, como se cayó.

```
   usuario ──► [ FRONT ] ──┬──► [ DESTACADOS ] ──┐
                 🔴        │         🔴          ├──► [ POL. COM. ] ──► [ DATOS ] ──► externos
                           └──► [ FACETADO  ] ───┘         🔴              🟢 ya volvió
                                     🔴               todavía trabada
```

### 3.5 Control de backpressure

Frente a los dos problemas, el original y el autogenerado, el **control de backpressure** son las herramientas de la sección 2 aplicadas en las cajas intermedias: **circuit breaker** (Políticas Comerciales anula a Datos y responde sin catálogo, en vez de quedarse sin hilos), **blacklist** (bloquear a los clientes que se desbocaron, como el rate limit al usuario del F5), timeouts, colas. Con requests no cancelables es peor, pero **en cualquier escenario tenés que usar alguna de estas técnicas para que esto no te pase.**

> **Para el parcial, si te preguntan: ¿qué es backpressure y cómo se controla?**
> Es la propagación hacia atrás de una falla: cuando una aplicación se cae o se pone lenta y sus clientes no reaccionan, esos clientes se quedan esperando, agotan sus recursos (hilos, sockets, memoria) y arrastran a sus propios clientes, hasta llegar al usuario. Se agrava por un problema autogenerado: los reintentos (el F5) suman requests que no son cancelables. Se controla con timeouts, circuit breaker, rate limit o blacklist de clientes, colas y buffers, para que cada caja se degrade de forma controlada en vez de propagar la espera.

---

## 4. Asincronismo y colas de mensajes 🔴

Todo lo anterior asume comunicación **síncrona**: te llamo y espero la respuesta. Hay otra forma.

### 4.1 Síncrono vs asíncrono

- **Flujo síncrono:** una request y una response. Eso es todo el flujo. Vos esperás, el otro responde.
- **Flujo asíncrono:** mandás una request, recibís una response **a veces** (en general un ACK: "recibido"), y **no te enterás de qué hace el consumidor** con eso, salvo que hagas *polling* (preguntar cada tanto "¿ya está?") o algo parecido.

El asincronismo sirve para **desacoplar al que envía del que recibe**: el que envía no depende de que el que recibe esté disponible ni sea rápido en ese momento. Esa es su relación con backpressure: es una de las formas de regular la velocidad (2.5).

### 4.2 Cómo funciona una cola de mensajes

En microservicios (o entre servicios en general), el asincronismo se hace con **colas de mensajes**:

1. Vos **enviás un mensaje** a la cola ("hay que notificar al usuario 42").
2. Probablemente recibís un **ACK**: la cola confirma que lo tiene.
3. Del otro lado hay **uno o N consumidores** del mensaje, que lo leen y hacen cosas.
4. Cuando el consumidor termina, hace **su** ACK ("ya está"), y ese mensaje **se destruye**.

```
   Productor ──"notificar a 42"──► ┌─────────────┐ ──► Consumidor 1 ─ ACK ─┐
                ◄───── ACK ─────── │    COLA     │                        ├─► el mensaje se borra
                                   │ [m1][m2][m3]│ ──► Consumidor 2 ─ ACK ─┘
                                   └─────────────┘
```

Es lo que viste en la Parte 1 (§5.2) como herramienta para "las dos o ninguna": el mensaje persiste hasta que todos los consumidores confirmaron.

### 4.3 No todas las colas son iguales

- **Colas más simples, pensadas como buffer** (típicamente en memoria): **RabbitMQ**, **ActiveMQ**. Se van almacenando mensajes, se consumen, y cuando hay ACK del consumidor el mensaje se destruye. "Simples" es una forma de decir; son muy usadas.
- **Brokers más complejos y muy potentes**: **Kafka**. Un broker es el intermediario que recibe y distribuye mensajes; Kafka es el de referencia cuando el volumen es grande. Soporta:
  - **modos de entrega**: que un mensaje se lea **exactamente una vez**, **al menos una vez**, o **a lo sumo una vez**. Es análogo a los niveles de consistencia que viste en NoSQL (clase 04): cada modo es un trade-off entre garantía y costo;
  - **transacciones**;
  - **escritura a disco** (los mensajes persisten aunque se caiga el broker);
  - **particiones** y **tópicos** (canales con nombre a los que se publica y de los que se consume), y muchas cosas que otras colas no dan.

  Eso da **mucha complejidad y, a la vez, mucho desacoplamiento**.
- **Servicios managed**: plataformas ya hechas (en la nube) donde creás un servicio y te da un **pub/sub** (*publish/subscribe*: publicadores que emiten a un tópico y suscriptores que reciben; es un broker con otro nombre) sin operarlo vos.

> 🕳️ **Madriguera — la arquitectura de Kafka**
> Tópicos partidos en particiones distribuidas entre brokers, consumidores agrupados en *consumer groups*, offsets por consumidor, replicación entre brokers. Es una pieza **complejísima** de entender y de operar; la clase la nombra como referencia, no la enseña.
> *Volvé al camino — esto se profundiza aparte, otro día.*

### 4.4 El arma de doble filo

Se ha visto (dos veces, y era polémico) que **toda aplicación tuviera una cola de mensajes asociada**: levantabas una aplicación y había una cola al lado, con **la misma API que la aplicación pero asíncrona**. Le podías pegar a cualquier aplicación de forma asíncrona. Al principio suena bien.

El problema: **muchos flujos son más cómodos de hacer asíncronos cuando los estás desarrollando, y después son muy difíciles de mantener.** Si tengo cuatro interacciones entre microservicios y todos están esperando la respuesta de forma asíncrona:

- el código es **más inmantenible**;
- tengo **más piezas que se mueven**;
- tengo **más problemas para dar una respuesta rápido**.

El síntoma canónico: el **GET asíncrono**. "Te hago un GET, pero es asíncrono." Pero es un GET: **dame la información ahora.** "No, es asíncrono." Eso es una desventaja muy grande **si no modelo bien el caso de uso**. No quiere decir que esté mal; quiere decir que te puede pasar.

### 4.5 Cuándo suma: notificaciones y la dead letter queue

El caso donde el asincronismo es claramente mejor: **tengo que notificar a N usuarios.** En vez de hacer N envíos síncronos dentro de la request, **mando un evento asíncrono**, y un consumidor lo lee y manda la notificación. Es mucho mejor que hacerlo todo síncrono, por dos motivos:

1. **Desacopla**: la request que dispara la notificación termina rápido; el envío ocurre después, a su ritmo.
2. **Los errores no se pierden.** Los brokers y colas te dan algo llamado **dead letter queue** (cola de "cartas muertas"): cuando hay un error procesando un mensaje, ese mensaje **se almacena en una cola de errores**. Los errores **se persisten**; no se pierden como en un flujo síncrono, donde una excepción que nadie atrapó desaparece. Después podés revisarlos y reprocesarlos.

```
   Actividad ──evento──► [ COLA ] ──► Notificaciones: envía mail/SMS ─ ✔ ACK ─► se borra
                                                                     ─ ✘ error ─► [ DEAD LETTER QUEUE ]
                                                                                   (el error queda guardado)
```

Un **microservicio de notificaciones que consume de RabbitMQ** es exactamente este patrón, y es el ejemplo canónico de buen uso. Si tu TP lo tiene, este es el argumento para justificarlo: no es "usamos una cola porque es moderno", es "el envío no bloquea la request y los fallos de envío quedan persistidos para reintentar". El uso en un TP puede quedar un poco didáctico, y está bien.

> **Para el parcial, si te preguntan: ¿cuándo conviene comunicación asíncrona entre servicios y cuándo no?**
> Conviene cuando el que envía no necesita la respuesta para continuar y el trabajo puede hacerse después, a otro ritmo: notificar a N usuarios, por ejemplo. Desacopla emisor de receptor, regula la carga (backpressure) y, con una dead letter queue, persiste los errores en vez de perderlos. No conviene cuando el flujo necesita la respuesta ahora (un GET asíncrono no tiene sentido) ni cuando muchas interacciones asíncronas encadenadas vuelven el sistema inmantenible: hay que modelar bien el caso de uso.

> **Para el parcial, si te preguntan: ¿qué diferencia hay entre RabbitMQ y Kafka?**
> RabbitMQ (como ActiveMQ) es una cola más simple, que funciona como un buffer: los mensajes se encolan, se consumen y se destruyen con el ACK del consumidor. Kafka es un broker mucho más potente y complejo: escribe a disco, tiene tópicos y particiones, transacciones, y modos de entrega configurables (exactamente una vez, al menos una vez, a lo sumo una vez). Más desacoplamiento y garantías, a cambio de mucha más complejidad.

---

## Checkpoint — Parte 4

*(Sin respuestas: van al complemento.)*

1. ¿Por qué el manejo de errores cambia de naturaleza al pasar de un monolito a servicios que se hablan por red?
2. ¿Qué pasa si una llamada entre dos servicios no tiene timeout definido? ¿Y entre un servicio y su base?
3. Rate limit y circuit breaker: ¿de qué lado de la llamada actúa cada uno y contra qué protege?
4. Reproducí el ejemplo del circuit breaker con timeout de 5 segundos y anulación de 10 minutos. ¿Qué gana el cliente?
5. "Mi tiempo de respuesta es el máximo de mis proveedores." Explicá qué decisión justifica esa frase.
6. Contá la cascada de backpressure del e-commerce desde los proveedores externos hasta el usuario. ¿Qué recurso se agota en cada caja?
7. ¿Por qué el F5 del usuario es un "problema autogenerado" y qué propiedad de HTTP lo agrava?
8. ¿Por qué la recuperación también es en cadena, aunque la caja original ya haya vuelto?
9. Explicá el ciclo de vida de un mensaje en una cola: envío, ACK, consumidores, destrucción.
10. ¿Qué es un GET asíncrono y por qué es una señal de caso de uso mal modelado?
11. ¿Qué es una dead letter queue y qué ventaja concreta da frente a un flujo síncrono?

---

## Qué viene en la Parte 5

La lista honesta de lo que microservicios empeora: responsabilidades que no se sabe en qué caja van, overhead de red, tareas redundantes, un sistema más complejo aunque cada aplicación sea más simple, la coordinación entre equipos y ambientes, y la complejidad de management (el two-pizza team y cómo la IA está cambiando el tamaño de los equipos). Los servicios anémicos y los picoservicios, el caso de Amazon que volvió a un monolito, la estandarización, y la lista de malas prácticas: repo compartido, SDK en vez de API, mensajería sin versionar, servicios olvidados.

---

**FIN DE LA PARTE 4 — Apunte maestro clase05 — Microservicios**
