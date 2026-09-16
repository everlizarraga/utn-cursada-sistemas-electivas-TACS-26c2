# 📘 Apunte maestro — Clase 05 — Microservicios
## Parte 1 de 6 — El monolito: de dónde venimos

> **Unidad:** `clase05` · TACS 2C 2026 · Clase del 08/09/2026 · Apunte maestro, Parte 1 de 6

**Qué cubre esta parte.** Qué es una aplicación monolítica, por qué en pequeña escala es la mejor opción, y el catálogo completo de problemas que aparecen cuando crece: en el código, en la gente, en la infraestructura y sobre todo en el deploy. En el medio, un desvío que vale la pena tomar a fondo: qué pasa cuando una operación tiene que tocar dos datastores a la vez y cómo se resuelve sin transacción atómica. Cierra con el "approach intermedio": un hack con el balanceador que alivia sin migrar.

**Qué NO cubre.** La migración a microservicios, la arquitectura resultante y sus problemas propios. Eso arranca en la Parte 2.

**Índice de la unidad.**

| Parte | Tema |
|---|---|
| **1** | **El monolito: ventajas, problemas, dos datastores y transacciones, deploys, hack del balanceador** ← estás acá |
| 2 | Del monolito a microservicios: el e-commerce, el elefante, la primera extracción, refactors disruptivos |
| 3 | La arquitectura refactorizada: el mapa completo, contratos, versionado de APIs, autenticación entre servicios |
| 4 | Errores, backpressure y asincronismo: timeouts, rate limit, circuit breaker, colas de mensajes |
| 5 | No todo es bueno: desventajas, complejidad de management, malas prácticas |
| 6 | Requisitos de la organización (escalabilidad, monitoreo, ciclo de vida, testing, disponibilidad) y cierre |

**Leyenda.** 🔴 central, evaluable · 🟡 secundario · 🟢 mencionado al pasar · 🕳️ madriguera (tangente cerrada a propósito) · ⚠️ advertencia. Esta unidad no tiene código ni comandos: el peso visual va en diagramas.

**Cómo se lee esta unidad.** Es la clase más "blanda" de la materia hasta ahora: no hay demo ni código, hay criterio. Lo que se evalúa es que puedas **explicar el trade-off** de cada decisión con un ejemplo concreto, no recitar una lista. Cada sección de esta parte cierra con un problema que microservicios promete resolver, y varias de las siguientes partes muestran a qué precio.

---

## 0. Información operativa 🟢

- **Entrega 2 del TP:** no hace falta esperar la corrección de la Entrega 1 para arrancar. Se trabaja sobre lo que hay. Si hay un blocker (algo de la Entrega 1 que frena la 2), se manda un mail al ayudante preguntando por ese blocker puntual.
- **Microservicios en el TP:** a lo largo de los años hubo grupos que hicieron el TP con microservicios y otros con un solo servicio. Ambas son válidas: en esta materia la elección es una cuestión didáctica, no un requisito. Lo que sí se espera es que puedas justificar la que tomaste, y esta clase es el material para eso.

---

## 1. Antes de arrancar: diseño de sistemas 🟢

Esta clase se pisa con Diseño de Sistemas y con Desarrollo de Software, pero desde otro lado: no cómo se programa un servicio, sino **cómo se decide** cuántos servicios hay, cómo se hablan y quién los mantiene. Ese tipo de discusión te la vas a encontrar en tres lugares de la vida laboral: **entrevistas** (la ronda de "system design"), **plannings** y **RFCs**.

**RFC** (*Request for Comments*): documento donde se define una spec, un estándar o una forma de trabajar; dentro de una empresa, es el escrito con el que un equipo propone un cambio (por ejemplo, de arquitectura) y el resto lo comenta antes de implementarlo. Muchas cosas que usás todos los días están escritas en RFCs públicas: **HTTP** está definido en RFCs; **Paxos** (un mecanismo de elección de líder y consenso, para que un conjunto de máquinas se ponga de acuerdo en un valor aunque algunas fallen) también. Una RFC es, salvando distancias, el paper del desarrollo de software: sin la rigurosidad académica, pero con el mismo rol. Leer una entera, en el tema que elijas, te sube el nivel de golpe.

Las nociones que se dan por básicas en cualquiera de esos tres contextos, y que la materia va tocando: replicación de base de datos y sharding (clase 04), microservicios (hoy), cómo servir un front (clase 03), macro vs micro framework, usar ORM o no, balanceador de carga (clase 01). No hace falta dominar todo: hace falta tener la noción y saber dónde profundizar.

- **ORM** (*Object-Relational Mapper*): library que mapea objetos de tu lenguaje a tablas de una base relacional, para que trabajes con objetos y no con SQL a mano. Vuelve en la sección 5.

> 🕳️ **Madriguera — micro vs macro framework (Javalin vs Spring Boot)**
> Un **micro framework** (Javalin, Spark) te da lo mínimo, rutas HTTP y poco más, y vos elegís cada pieza que le enchufás (parser de JSON, templates, persistencia). Un **macro framework** (Spring Boot) te da el paquete completo, con todas las decisiones tomadas. Trade-off: control y liviandad contra velocidad de arranque y convenciones. Discusión más amplia que esta clase.
> *Volvé al camino — esto se profundiza aparte, otro día.*

> 🕳️ **Madriguera — Scrum, Agile y la IA**
> Scrum es una forma de organizar el trabajo en iteraciones cortas con ceremonias (planning, daily, retro) que surgió como respuesta al modelo en cascada. Dos cosas seguras: los requisitos cambian todo el tiempo, así que trabajar en iteraciones va a seguir; y el desarrollo dirigido por especificaciones con IA (**Spec Driven Development**, clase 08) está cambiando el peso de esas ceremonias y quizás de roles como PO o Scrum Master. Está bien saberlo; no vale la pena hacer foco.
> *Volvé al camino — esto se profundiza aparte, otro día.*

---

## 2. Qué es una aplicación monolítica 🔴

Arrancá con la escena, que es la historia de casi todo sistema.

Una empresa arranca con una aplicación que resuelve **toda** la solución: catálogo, ventas, usuarios, reportes, todo en el mismo proyecto. En ese momento es la decisión correcta: el contexto es chico y una sola aplicación alcanza. Pasa el tiempo, la empresa crece, y cada feature nueva se suma **al mismo lugar**, porque ahí está todo a mano. La aplicación crece en volumen y en responsabilidades. Nadie decidió "vamos a hacer un monolito": simplemente pasó. **Cuando nos percatamos, por lo general, ya tenemos el problema entre manos.**

Eso es una **aplicación monolítica**: una aplicación concebida para ser toda la solución, que se compila y se despliega como **un único artefacto** (un solo binario, un solo proceso). Adentro puede haber "módulos" (la palabra es polémica, porque depende de cómo esté programado que esos módulos existan de verdad o sean solo carpetas), pero hacia afuera es una sola pieza.

- **Artefacto deployable**: el resultado del build que se lleva a producción; en Java, un `.jar` o `.war`. "Único artefacto" quiere decir que hay uno solo por versión de toda la aplicación.

---

## 3. Ventajas del monolito 🔴

Antes de listar problemas, dejá claro por qué **todo el mundo arranca así**, y por qué en pequeña escala sigue siendo la mejor opción.

| Ventaja | Por qué |
|---|---|
| **Simpleza** | Lidiás con una sola tecnología, un solo stack. Agregar algo es agregarle algo al monolito. |
| **Baja latencia** | Todo vive en el mismo proceso: menos saltos de red, menos colas, menos indirecciones. Llamar a una función es más rápido que llamar por la red. |
| **Único artefacto deployable** | Versión nueva = buildear una cosa y deployar esa cosa. |
| **En pequeña escala, aprovecha los recursos** | Un solo proceso paga un solo overhead. |
| **Es fácil agregar funcionalidad que use lo que ya hay** | La base de datos, las librerías y los servicios ya implementados están a mano, en el mismo código. |

Dos de estas merecen una vuelta más.

**Baja latencia, con pinzas.** Si tengo que hacer todo dentro de un solo artefacto, no tengo que autenticar a nadie: no hay "otros servicios" a los que hablar. Si necesito la base de datos, tengo el driver en una library al lado mío; no tengo que ir a ningún lado. El "con pinzas" es porque la latencia de una aplicación no depende solo de cuántos saltos de red hay; en la Parte 5 vas a ver que en microservicios ese overhead existe pero se mitiga.

**Aprovechar los recursos = menos overhead.** Overhead: el costo fijo que paga cada pieza por el solo hecho de existir, antes de hacer trabajo útil. Imaginate un montón de servicios y que cada uno levanta su balanceador, reserva su memoria, tiene su espacio propio para lo que sea que el lenguaje necesite. Un monolito paga eso una sola vez.

> **Para el parcial, si te preguntan: ¿qué es una aplicación monolítica y qué ventajas tiene?**
> Es una aplicación concebida para ser toda la solución, que se despliega como un único artefacto. Sus ventajas: simpleza (un solo stack), baja latencia (todo en el mismo proceso, sin saltos de red), un único artefacto deployable, aprovechamiento de recursos en pequeña escala (un solo overhead), y facilidad para agregar funcionalidad que reutilice base de datos, librerías y servicios ya implementados.

---

## 4. Problemas del monolito 🔴

Llega un punto donde ocurren cosas incómodas. No es una lista de defectos técnicos: son problemas que se **viven**, de código, de gente, de infraestructura, de deploy. Pensá cada uno con la escena de la sección 2, ya con decenas de personas trabajando adentro.

### 4.1 Deploys muy grandes → resta agilidad

Tengo un artefacto solo. Cuando hago deploy tengo que buildear, compilar, pasar el **CI** (integración continua: el pipeline automático que compila y corre los tests en cada cambio), correr los tests de integración (que tienen que levantar una cosa gigante, con sus data sources, las conexiones a bases de datos que la app necesita para arrancar), todo el proceso para un aplicativo enorme, e idealmente hacerlo **una sola vez**. Y estás limitado por la capacidad de la máquina donde eso corre. Cada deploy es un evento.

### 4.2 No escala en gente

Diez o más personas tocando el mismo binario es un problema, y no solo porque el código se vuelve un lío. Tenés más gente en el mismo código, haciendo **scopes distintos en el mismo lugar**, aun cuando todavía no haya equipos formales.

El síntoma más real: **el monolito no se adapta a la estructura de la organización.** Dependés de gente con la que no tenés contacto. Si sos del área X y necesitás pushear algo, tenés que esperar a que otra área, que no tiene nada que ver con lo tuyo, resuelva su problema, porque comparten el artefacto y el deploy. Este es el problema que empuja a microservicios más que cualquier otro, y se retoma en la Parte 3 cuando aparezca el ciclo de vida propio por equipo.

> **Para el parcial, si te preguntan: ¿por qué se dice que un monolito "no escala en gente"?**
> Porque muchas personas con scopes distintos tocan el mismo binario y comparten un único deploy: el trabajo de un área queda bloqueado por el de otra que no tiene relación con ella. El monolito no se adapta a la estructura de la organización.

### 4.3 Tiempos altos: test, build, release, deploy

Va de la mano del testing. Supongamos que la aplicación tiene **4.000 tests** (no es un número inventado: se ve). Cada vez que hay que correr los tests, corren los 4.000, y eso tarda. Y en 4.000 tests siempre hay alguno **mal hecho**: no determinístico, un **flaky test**, un test que a veces pasa y a veces falla sin que cambie el código, por depender del reloj, del orden de ejecución o de un recurso externo. Sumale tests de integración gigantes y de punta a punta. Cada ciclo test → build → release → deploy es largo, y el flaky te lo hace repetir.

También te pasa en el día a día: vos estás haciendo algo, otro está haciendo algo arriba, y capaz te lo movió. En un monolito el testing de tu parte está contaminado por los cambios de todos.

### 4.4 Código legacy conviviendo con código nuevo → difícil de evolucionar

Lo obvio: lo viejo y lo nuevo en el mismo codebase. Lo menos obvio: la **trampa de las dependencias**.

Caso Java con **Maven** (el gestor de build y dependencias estándar de Java: declara qué libraries usa el proyecto y las descarga). Tu equipo necesita la library L en la versión 2, porque usa una feature nueva. Otro equipo, en el mismo monolito, necesita L en la versión 1, porque la 2 les rompe algo. Un proyecto Java con Maven **no puede tener dos versiones de la misma library**: Maven tiene un algoritmo de resolución que, ante el conflicto, elige **una** de las dos, por más que estén declaradas en módulos distintos. El `.jar` final sale con una sola versión de L, y alguno de los dos equipos se queda sin lo que necesitaba. Con .NET pasa lo mismo.

```
      monolito.jar
        ├── módulo-ventas   ──► L v2  ─┐
        │                              ├─► Maven elige UNA → el .jar sale con L v1 o L v2
        └── módulo-reportes ──► L v1  ─┘      → un equipo se queda sin lo que necesita
```

> 🕳️ **Madriguera — cómo elige Maven**
> La regla se llama *nearest wins*: gana la versión declarada más cerca de la raíz en el árbol de dependencias; a igual distancia, la primera declarada. Se puede forzar una versión con `<dependencyManagement>`. Detalle de herramienta, no de esta clase.
> *Volvé al camino — esto se profundiza aparte, otro día.*

### 4.5 Un problema en la aplicación puede arrastrar todas las funcionalidades

Si se cae el servidor, perdiste **todas** las funcionalidades. Pero no hace falta que se caiga la máquina: alcanza con que **una** feature se porte mal, porque todas viven en el mismo proceso y compiten por los mismos recursos.

Qué tipo de problemas puede meter una feature y pagar el resto:

- **GC storm**: el *garbage collector* (el mecanismo de la JVM que libera la memoria que ya no se usa) entra en un ciclo de recolecciones seguidas y la aplicación entera se queda frenada mientras tanto.
- **Leak de memoria**: una feature reserva memoria y nunca la libera; el proceso entero se queda sin memoria.
- **Leak de hilos / hilos bloqueados**: hilos (*threads*, las unidades de ejecución concurrente dentro del proceso) que quedan colgados esperando algo y no se devuelven al pool.
- **Uso incorrecto o intensivo de CPU.**

Y de dónde salen: una feature *long running* (de larga duración) que larga un job pesado; requests que no se cortan; sockets abiertos (socket: la conexión de red que un proceso mantiene abierta); acceso a distintos protocolos: uno tira un **FTP** (protocolo de transferencia de archivos, viejo pero vivo), otro llama a un servicio de Amazon, otro usa una **SDK** propietaria (*Software Development Kit*: library que te da un proveedor para integrarte con su servicio). Todo eso vive en el mismo codebase y en la misma máquina, y cualquiera puede ser el que tire abajo a los demás. Es mitigable, pero no deja de ser un problema.

### 4.6 Fronteras entre módulos no del todo claras

"Es más difícil encontrar dónde está el error" es la versión visible. La de fondo es que en un monolito las fronteras entre módulos son borrosas, y eso permite dependencias que después no se pueden desarmar.

Analogía con el **problema del diamante** de la herencia múltiple: B y C heredan de A, y D hereda de B y de C; ¿qué versión de lo heredado de A recibe D? Con módulos pasa algo parecido, en la forma más común de **llamada circular**: A necesita llamar a B y B necesita llamar a A. ¿Cuál incluye a cuál primero? Tengo un bucle que **en un monolito estoy obligado a resolver**. Microservicios te lo deja pasar bastante más; en la Parte 3 vas a ver que ahí el mismo bucle sigue siendo un problema, solo que ya no te lo frena el compilador.

```
   Diamante (herencia)              Llamada circular (módulos)
          A                           ┌──────┐   llama    ┌──────┐
         / \                          │  A   │───────────►│  B   │
        B   C                         │      │◄───────────│      │
         \ /                          └──────┘   llama    └──────┘
          D   ¿qué A recibe D?         ¿cuál se construye primero?
```

### 4.7 Funcionalidades diferentes pueden requerir infraestructura diferente

Una parte es **CPU intensive**, otra necesita mucha memoria, otra mucho disco. En un monolito **escalar es escalar todo**: si una sola ruta necesita más memoria, tengo que darle más memoria a la máquina entera (o a todas las réplicas). Se puede mitigar deployando distintas partes en distintos lugares, que es el hack de la sección 7, pero es eso, una mitigación.

### 4.8 Funcionalidades diferentes pueden requerir configuraciones diferentes

Un proceso tiene **una** configuración. Lo que se choca:

- **JVM args**: los argumentos con que arranca la máquina virtual de Java (memoria máxima, qué garbage collector usar, etc.). Ejemplo concreto: una feature necesita mucho *throughput* (cantidad de trabajo procesado por unidad de tiempo) y otra necesita baja latencia; el GC que le conviene a una no le conviene a la otra, y **no puedo tener dos configuraciones de GC en un mismo proceso**. Elijo una y alguien pierde.
- Características del sistema operativo.
- Versión de la **JRE** (*Java Runtime Environment*, el runtime que ejecuta la aplicación).
- Librerías externas nativas, por ejemplo **DLLs** en Windows.
- Y, de vuelta, el problema de Maven de 4.4.

### 4.9 Toda la aplicación está hecha con tecnologías similares

Verdad a medias. Dentro de un mismo stack puede haber algo de mezcla: en la JVM conviven Java, Groovy, Scala y Kotlin (lenguajes distintos que compilan al mismo bytecode y corren en la misma máquina virtual), aunque esa interoperabilidad no siempre es 100% amigable. Lo que **no** puedo es mezclar PHP con Java, Ruby y .NET en un solo proceso. Puede ser ventaja (simpleza) o desventaja (la herramienta ideal para una parte no es la del resto); depende de cómo lo mires.

### 4.10 ¿Y la base de datos?

El mismo problema, en la capa de persistencia: una sola tecnología de base para toda la aplicación, cuando distintas partes querrían distintos modelos (los que viste en la clase 04). Y acá se abre el desvío que sigue.

---

## 5. El desvío: dos datastores y las transacciones 🔴

Empezá con una pregunta: **¿alguna vez trabajaste con una aplicación conectada a dos bases de datos al mismo tiempo?** No es raro: una app con MongoDB y SQL Server, por ejemplo. Datastore: cualquier lugar donde persistís datos, sea una base relacional, una documental o un servicio externo.

Con dos conexiones y sin ORM, el caso simple anda: cada query elige explícitamente contra qué conexión va. Si una transacción toca tres o cuatro tablas de la **misma** base, abrís, hacés todo, y si anduvo bien, commit. Es ACID como lo viste en la clase 04: o pasa todo o no pasa nada.

**La pregunta clásica:** ¿y si una transacción tiene que involucrar a **ambas** conexiones?

### 5.1 Por qué es un problema (y por qué el ORM lo empeora)

El problema en su forma general: tengo un proceso que hace un `INSERT` en mi base **y** le hace una request a otro sistema. La condición del negocio es **se hacen las dos o no se hace ninguna**. No puede pasar que una falle y la otra no.

```
   ┌──────────────┐   INSERT     ┌────────────┐
   │              │─────────────►│   Base A   │  ✔ ok, ya escribió
   │   Proceso    │              └────────────┘
   │              │   request    ┌────────────┐
   │              │─────────────►│ Sistema B  │  ✘ falla
   └──────────────┘              └────────────┘
        ¿y ahora? Base A ya tiene el dato. No hay COMMIT que abarque a los dos.
```

Con una sola base relacional, eso es una transacción y listo. Con dos datastores (dos bases, o una base y una API) **deja de ser atómico**: no hay un `COMMIT` que abarque a los dos.

Con ORM es peor. En Java, **JPA** (*Java Persistence API*: el estándar de annotations que hace la "magia" de mapear objetos a tablas relacionales) hace un montón de cosas por abajo: **flushea** datos (manda a la base lo que tenías pendiente en memoria), **commitea** operaciones que no viste, y algunas operaciones **inician y commitean la transacción automáticamente**. Con `@Transactional` (la annotation que envuelve un método en una transacción) y dos datastores tenés que indicarle cuál abre la transacción, cuál la cierra, o si abrís dos y cómo se coordinan. Ya con una base es delicado; con dos es un quilombo.

### 5.2 Cómo se resuelve: no atómicamente, con cuidado

Se puede resolver, en un servicio o en varios, pero **no atómicamente**. Tres herramientas:

1. **Idempotencia a favor.** Si la operación es idempotente (hacerla una vez o N veces deja el mismo resultado; lo viste con los verbos HTTP en la clase 01), lo que salió mal se puede **reintentar** sin miedo a duplicar. Reintentar hasta que las dos pasen es una forma de llegar a "las dos o ninguna".
2. **Indirección con una cola de mensajes.** En vez de hacer la tarea, dejo un mensaje que dice "hay que hacer X". Del otro lado hay uno o más **consumidores** que leen el mensaje y hacen su parte; cuando **todos** los consumidores confirmaron con un **ACK** (*acknowledgement*, acuse de recibo), el mensaje se borra. Si alguno no confirmó, el mensaje sigue ahí y se vuelve a intentar. **RabbitMQ** es una cola de este tipo. (La mecánica completa de colas está en la Parte 4.)
3. **Compensación.** Mando las dos operaciones; si una falla, llamo al **rollback de la que salió bien**, aunque viva en otro servicio. Cada operación necesita su "deshacer" explícito.

Ninguna discrimina por arquitectura: valen en un monolito con dos bases y valen entre servicios. Ninguna es amigable.

### 5.3 Transacción distribuida y patrón Saga 🟡

En microservicios el problema tiene nombre: **transacción distribuida**. Tengo que coordinar más de un servicio donde **las bases de datos no se ven entre sí; se ven las APIs**. Lo resuelvo con lo mismo de arriba: un **pub/sub** (broker de mensajes con publicadores y suscriptores; **Kafka** es el ejemplo de referencia, y una **MQ**, *message queue*, es la versión más simple; Parte 4), o con el **patrón Saga**: una forma de coordinar distintos microservicios, parecida al pub/sub pero hablándose entre servicios, donde **todos commitean o algunos hacen rollback**. Y "rollback" acá quiere decir llevar todo a un **estado consistente anterior**: el que tenías antes de arrancar, cuando todo estaba en orden.

⚠️ Es mucho más macanudo hacer una transacción SQL y olvidarte de todo. Los microservicios **no resuelven este problema**; lo empeoran (Parte 5). Cuando diseñes, preguntate si de verdad necesitás dos datastores en la misma operación.

> **Para el parcial, si te preguntan: una operación tiene que escribir en dos datastores distintos, "las dos o ninguna". ¿Cómo lo resolvés?**
> No hay transacción atómica que abarque a los dos, así que se resuelve con cuidado y no atómicamente: usando idempotencia para poder reintentar lo que falló, indireccionando por una cola de mensajes (el mensaje se borra recién cuando todos los consumidores hicieron ACK), o compensando (si una operación falla, ejecuto el rollback de la que salió bien). Entre servicios esto se llama transacción distribuida y se coordina con un pub/sub o con el patrón Saga.

> **Para el parcial, si te preguntan: ¿qué es el patrón Saga?**
> Un patrón para coordinar una transacción distribuida entre varios microservicios cuyas bases no se ven entre sí: cada servicio hace su parte y, si alguno falla, los demás ejecutan compensaciones (rollback) para volver al estado consistente anterior. Todos commitean o algunos hacen rollback.

---

## 6. Deploys del monolito 🔴

Pensá cómo deployarías un monolito. Las respuestas rápidas: "corro el binario", "lo subo a un VPS" (clase 02), "lo conecto a Render y listo" (clase 03). Todas válidas: mecánicamente, deployar un monolito puede ser fácil. **El punto no es la mecánica: es qué hago con las features.**

### 6.1 Las features se acumulan y las probabilidades se multiplican

Un monolito **no se puede deployar todos los días**, por todo lo de la sección 4. Entonces las features se acumulan hasta el próximo deploy. Cada feature tiene su propia probabilidad de fallar. Y la probabilidad de que **todo** salga bien es el **producto** de las individuales:

```
    1 feature  con 95 % de éxito → 0,95      = 95 %  de que el deploy salga bien
    5 features con 95 % de éxito → 0,95⁵     ≈ 77 %
   20 features con 95 % de éxito → 0,95²⁰    ≈ 36 %
```

Cuanto más grande el deploy, más probable que algo falle. Y como no soy ágil deployando, no puedo achicar el lote.

### 6.2 El día del deploy

Al momento del deploy tiene que haber **un técnico de cada parte** (de cada story, de cada feature, de cada equipo): mirando los logs, testeando la funcionalidad, y sabiendo **qué hacer** si aparece un problema. Caso real de la industria, no tan viejo: un deploy **cada dos meses, a las cuatro de la mañana, con seis personas**, una por cada equipo. Estás un poco a la buena de que todo ande bien.

### 6.3 Rollback total y hotfix caro

Si algo sale mal, hago **rollback**, y el rollback es **de todo**: las veinte features vuelven atrás, incluidas las diecinueve que andaban. Rollback caro.

Si me doy cuenta de un error y sé cómo arreglarlo (un **hotfix**: corrección urgente y puntual en producción), tengo que buildear todo de vuelta, hacer otro deploy completo, y verificar que ese deploy no se cruce con ningún módulo de otro equipo.

### 6.4 Features disruptivas

¿Y si una feature del lote es **disruptiva**, en el sentido de que **no tiene retrocompatibilidad** (la versión nueva no puede convivir con la anterior: cambia el formato de los datos, la API, el esquema)? Entonces no la puedo desplegar de a poco: paso a deployar y "muero en el deploy". Hay estrategias (migrar datos, levantar más de un servicio con el mismo código), pero se empiezan a complicar las cosas. Esto se retoma en la Parte 2, con los refactors disruptivos.

### 6.5 Hoy: release trains, nightly, canary. Y el límite

Antes se buildeaba y deployaba mucho menos seguido. Hoy, incluso con monolitos, hay formas de acelerar:

- **Release train**: calendario fijo de releases (uno por semana, uno por día); lo que está listo sube en ese tren, lo que no, espera al siguiente. Desacopla "cuándo sale" de "cuándo terminó cada feature".
- **Nightly**: build automático todas las noches con lo que hay.
- **Canary**: deployar la versión nueva a una porción chica del tráfico (o a un solo nodo) y mirar cómo se comporta antes de extenderla a todos. Una nightly deployada como canary todos los días es una práctica real con monolitos.

Pero incluso así **pierdo agilidad**: el ciclo de vida de cada feature pasa a estar atado a un montón de cosas que no dependen de ella. Y las preguntas de fondo siguen siendo incómodas: ¿cada cuánto son los deploys? ¿son disruptivos o no? ¿puedo hacer un deploy fácil? ¿en qué momento del día puedo hacerlo? En la Parte 6 (§4) vas a ver estas preguntas respondidas desde microservicios.

> **Para el parcial, si te preguntan: ¿por qué el deploy de un monolito es riesgoso?**
> Porque el deploy es grande y poco frecuente: se acumulan muchas features y la probabilidad de que todo salga bien es el producto de las probabilidades de cada una, así que cae con cada feature agregada. Además el rollback es de toda la aplicación, un hotfix exige rebuildear y redeployar todo, y hace falta un técnico de cada parte presente el día del deploy.

---

## 7. Approach intermedio: el hack del balanceador 🟡

Antes de partir el monolito hay un alivio que **se puede hacer tranquilamente** y que no es una solución de largo plazo. Aprovechás el balanceador de carga (clase 01), que en este caso no balancea *round robin* (repartir requests por turno entre nodos idénticos) sino **por rutas**, para mandar distintas rutas a distinta infraestructura:

```
                               /ServicioA     ┌───────────┐┐┐
                         ┌───────────────────►│ Artefacto │││  Cluster 1
   ╭──────────────╮      │                    └───────────┘┘┘  nodos con MUCHA memoria,
   │ Balanceador  │──────┤                                     tuneados para A
   ╰──────────────╯      │     /ServicioB     ┌───────────┐┐
                         └───────────────────►│ Artefacto ││   Cluster 2
                                              └───────────┘┘   otra configuración
```

Cómo funciona: dividís la aplicación **lógicamente** en el servicio A y el servicio B, sin partir el código. La ruta de A requiere instancias con muchísima memoria: le abrís un cluster con nodos configurados y tuneados para eso, y cuando llega algo a `/ServicioA`, va por ahí. Lo que llega a `/ServicioB` va a otro cluster, con otra configuración. Puede ser **el mismo deployable** en los dos clusters, o no; y puede haber una red privada distinta a cada cluster.

Tres aclaraciones que ordenan la idea:

- **No es un deploy: es un estado permanente de tu aplicación.** No lo hacés "mientras subís una versión"; así queda armada la topología.
- Como los clusters son independientes, **puedo deployar en el cluster 1 y no en el 2**, y tener **momentáneamente versiones distintas** en cada uno. Eso alivia los deploys grandes de la sección 6.
- **Es un load balancer, no un API Gateway.** Lo que hace el ruteo es un Apache o un NGINX (servidores web que reciben requests y las redirigen según reglas) con rutas configuradas. Un API Gateway (una pieza que centraliza la entrada a varios servicios y suma lógica encima: autenticación, límites, transformación) es otra cosa; aparece en la Parte 2, con el Strangler Fig.

**Por qué no es de largo plazo.** A medida que la aplicación crece, esto se vuelve menos amigable: sigue siendo un solo codebase, un cambio de un lado puede impactar del otro, y encima es **muy rígido** en un mundo cloud. Si vivís con **pods** (la unidad mínima que despliega Kubernetes: uno o más containers que se levantan y mueren juntos; Kubernetes lo viste de pasada en la clase 02) y escalado automático, tener configuraciones distintas para nodos distintos, atadas a versiones distintas, va contra la corriente. Ahí es donde decís: **me tengo que meter con servicios más chiquitos.** Ese es el tema de la Parte 2.

### 7.1 No confundir con blue/green 🟢

Otra cosa que se puede hacer con un balanceador: tener la versión nueva deployada en otro nodo y, cuando está lista, **cambiar el balanceador para que apunte a esa versión nueva**. Eso es **blue/green**: dos entornos completos (el azul con la versión actual, el verde con la nueva) y un switch entre ambos. Es una técnica de **deploy**, no un estado permanente como el hack de arriba. Se puede hacer, y no es lo que estamos viendo acá.

> **Para el parcial, si te preguntan: ¿qué es el "approach intermedio" para aliviar un monolito antes de migrarlo?**
> Usar un balanceador de carga con ruteo por rutas para mandar distintas partes de la aplicación (/ServicioA, /ServicioB) a clusters distintos, cada uno con la infraestructura y configuración que esa parte necesita, sin partir el código. Es un estado permanente de la topología, permite deployar y tener versiones distintas por cluster, pero no es solución de largo plazo: la aplicación sigue siendo un solo codebase y el esquema es rígido en entornos cloud con escalado automático.

---

## Checkpoint — Parte 1

*(Sin respuestas: van al complemento.)*

1. ¿Por qué "nadie decide hacer un monolito" y sin embargo la mayoría de los sistemas terminan siéndolo?
2. La ventaja "baja latencia" del monolito va con pinzas. ¿Qué la justifica y qué la relativiza?
3. ¿Qué significa que el monolito "no se adapta a la estructura de la organización"? Da un ejemplo con dos áreas.
4. ¿Qué es un flaky test y por qué en un monolito con miles de tests duele más?
5. Dos equipos del mismo monolito Java necesitan versiones distintas de una misma library. ¿Qué pasa al buildear y por qué?
6. Una feature necesita un garbage collector tuneado para throughput y otra uno para baja latencia. ¿Por qué el monolito no puede darles lo que piden?
7. ¿Por qué el ORM (JPA) complica una operación que toca dos datastores, en vez de simplificarla?
8. Explicá con tus palabras las tres herramientas para lograr "las dos o ninguna" sin transacción atómica: idempotencia, cola de mensajes, compensación.
9. ¿Por qué la probabilidad de que un deploy salga bien cae con cada feature que se suma al lote?
10. ¿En qué se diferencia el hack del balanceador por rutas de un blue/green? ¿Y de un API Gateway?

---

## Qué viene en la Parte 2

Con los problemas del monolito claros, la Parte 2 arranca la migración: un e-commerce monolítico con sus capas, un equipo de treinta personas y una subida cada dos o tres meses. Cómo se come un elefante, cómo se elige el primer pedazo que se extrae, cómo se integra la caja nueva sin borrar la vieja, cómo se hace el rollout de a poco, y qué pasa cuando el refactor es disruptivo y hay que migrar datos.

---

**FIN DE LA PARTE 1 — Apunte maestro clase05 — Microservicios**
