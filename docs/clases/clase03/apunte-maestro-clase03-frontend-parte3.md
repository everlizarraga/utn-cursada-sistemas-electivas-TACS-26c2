# TACS — Clase 03 · Arquitectura Web y Frontend
## Parte 3 — Adentro del browser: el motor, el runtime y el event loop

> **Sobre esta parte.** En la Parte 2 el browser "ejecutó el JavaScript" como si fuera gratis. Esta parte abre el capot: el **engine** que traduce tu código a algo que la máquina entienda, la diferencia entre **interpretar y compilar** (y el truco de hacer las dos cosas a la vez), el **call stack** y el **memory heap**, y la joya de la corona: el **runtime** con su **event loop** — la respuesta a cómo un lenguaje que ejecuta una sola cosa a la vez sostiene toda la interactividad de la web. Cierra con el escalado y con los **service workers** y las **PWAs**. **No cubre** cómo el browser convierte HTML y CSS en pixels — eso es la Parte 4.
>
> **De dónde venís.** Parte 1 (la pila, el viaje de la URL), Parte 2 (JavaScript, el DOM y las Web APIs — sobre todo la idea de Web API y la fila de "Timers" de aquella tabla, que acá se vuelve protagonista). De la clase 02 se asume Docker: contenedores y levantar varias instancias de un servicio. Hereda la convención `👁️ → Parte N`.

---

## 1. 🟡 El problema: la máquina no habla JavaScript

Arranquemos por la incomodidad fundacional: **las computadoras no entienden JavaScript**. Entienden código máquina — instrucciones binarias del procesador. Y los browsers, en su origen, solo entendían HTML y CSS. Entonces, ¿cómo hace tu `querySelector` de la Parte 2 para ejecutarse?

Necesitás un traductor. Ese traductor es el **JavaScript Engine**: el motor que corre cada browser, que recibe código JavaScript y lo convierte en código máquina que el procesador puede ejecutar.

No hay un único engine — cada vendor construyó el suyo (fiel al patrón de la Parte 2: interfaz estándar, implementaciones múltiples):

- **V8** — desarrollado por Google, escrito en C++. Es el motor de Chrome… y de **Node.js**: cuando corrés JavaScript "en el servidor", estás corriendo el mismo motor, sacado del browser y puesto a trabajar solo. Este dato explica el "eco del lado del servidor" de la Parte 2.
- **SpiderMonkey** — el motor de Firefox, de los primeros que existieron, con el creador del propio JavaScript involucrado en su desarrollo.
- Y varios más (JavaScriptCore en Safari, etc.).

Cada motor implementa el mismo lenguaje — definido por un estándar llamado **ECMAScript**, que dice cómo debe comportarse JavaScript (quién lo escribe y qué pasa cuando los motores difieren igual: Parte 4). Lo que importa acá es la arquitectura: **un programa adentro del browser cuyo trabajo es traducir**.

---

## 2. 🔴 De tu código a código máquina

¿Y cómo traduce? En etapas. Mirá el mismo renglón en tres idiomas:

```
// JavaScript (para humanos)      // Bytecode de V8 (intermedio)      // Código máquina x86_64
let result = 1 + obj.x;           LdaSmi [1]                          movl rbx,[rax+0x1b]
                                  Star r0                             REX.W movq r10,0x100000000
                                  LdaNamedProperty a0, [0], [4]       REX.W cmpq r10,rbx
                                  Add r0, [6]                         ...

◄──────────────────────────────────────────────────────────────────────────────────────►
   Mejor para humanos                                              Mejor para máquinas
```

- **Bytecode**: un formato intermedio — instrucciones simples y compactas que ya no son "para leer" pero todavía no son de un procesador concreto.
- **Código máquina**: las instrucciones binarias reales del procesador de tu computadora.

El pipeline completo dentro del motor:

```
         ┌───────────────────── JAVASCRIPT ENGINE ─────────────────────┐
 [JS] ──►│  Parser ──► AST ──► Interpreter ────────► Bytecode ──┐      │
         │               ▲          ┆                           ├──► ⚙ ejecuta
         │               │ (observa)┆                           │      │
         │            Profiler ◄┄┄┄┄┘                           │      │
         │               ┆                                      │      │
         │            Compiler ──► Código optimizado ───────────┘      │
         └─────────────────────────────────────────────────────────────┘
```

Paso a paso, la rama de arriba (la del Profiler y el Compiler la vemos en la sección 4):

1. **Análisis léxico**: el motor "splitea" tu código en **tokens** — las piezas mínimas con significado (`let`, `result`, `=`, `1`, `+`…) — para identificar qué está intentando hacer.
2. **Parser → AST**: con esos tokens arma el **AST** (*Abstract Syntax Tree*, árbol de sintaxis abstracta): tu código convertido en árbol, donde cada nodo clasifica una construcción — "esto es una declaración de variable", "esto es una función", "esto es una suma".
3. **Interpreter → Bytecode**: recorriendo ese árbol, el intérprete genera y ejecuta bytecode.
4. De ahí, a **código máquina**, y el procesador hace lo suyo.

¿Otra vez un árbol? Sí — tercer árbol de la clase: HTML forma un árbol (Parte 2), el DOM es un árbol, y ahora tu **código** también se vuelve árbol antes de ejecutarse. Convertir texto en árbol es *el* movimiento estándar de todo lo que interpreta lenguajes.

El AST se puede ver en vivo: existe una herramienta web, **AST Explorer** (astexplorer.net), donde pegás JavaScript en un panel y en el otro aparece el árbol. Un `let tips = [...]` se convierte en un nodo `VariableDeclaration` con su `VariableDeclarator` adentro, cuyo `Identifier` tiene `name: "tips"`; una `function printTips() {...}` aparece como `FunctionDeclaration`. Cada pedazo de tu código, clasificado y colgado del árbol. Si cursaste Sintaxis y Semántica, esto te va a sonar: es la misma teoría, corriendo adentro de tu browser millones de veces por día.

---

## 3. 🔴 Intérprete vs. compilador

El pipeline dijo "Interpreter" — y ahí hay una decisión de diseño con nombre propio. Hay dos grandes estrategias para ejecutar un lenguaje:

- Un **compilador** traduce **antes de tiempo**: agarra todo tu programa, lo convierte a código de bajo nivel, y deja un **ejecutable** como archivo de salida. Después, ejecutás eso. Ejemplo clásico: C.
- Un **intérprete** traduce **sobre la marcha**: lee y ejecuta el código línea por línea, **en runtime**, sin archivo de salida. Ejemplo clásico: JavaScript (en su origen).

Cada estrategia tiene su trade-off, y el ejemplo de la clase lo deja desnudo:

```javascript
// Una función que suma dos números.
function someCalculation(x, y) {
  return x + y;
}

// La llamamos MIL veces… siempre con los mismos argumentos.
for (let i = 0; i < 1000; i++) {
  someCalculation(5, 4);   // Resultado esperado: 9. Las mil veces. Siempre 9.
}
```

- El **intérprete** ejecuta la función las mil veces, de punta a punta, aunque el resultado sea siempre el mismo. No puede saberlo: va línea por línea, sin mirar el panorama.
- Un **compilador**, analizando el programa entero antes de ejecutar, puede darse cuenta de que esa llamada siempre retorna 9 — y optimizar: mil veces "9", cero veces "ejecutar la suma".

**¿Por qué la web eligió intérprete, entonces?** Por el arranque. Un intérprete **empieza a correr ya**: el browser recibe el archivo JS del servidor (el fetch de la Parte 2) y lo ejecuta al instante, sin esperar ninguna compilación previa. Para una página web, donde el usuario está mirando una pantalla en blanco, ese arranque inmediato vale oro. El precio: se pierde la optimización.

**Una aclaración de paso, porque el terreno se presta a confusión:** interpretado/compilado es **independiente** del sistema de tipos. Que un lenguaje sea estática o dinámicamente tipado, fuerte o débilmente tipado, o que tenga *inferencia de tipos* (no escribís el tipo, pero el compilador lo deduce por cómo usás la variable) — son características **del lenguaje**, no de su estrategia de ejecución. Podés tener un lenguaje 100% interpretado y fuertemente tipado. Una cosa no implica la otra.

> **Para el parcial, si te preguntan** — *¿Qué diferencia hay entre un lenguaje interpretado y uno compilado?*
> El compilador traduce todo el programa antes de ejecutarlo y produce un ejecutable (traducción anticipada); el intérprete traduce y ejecuta línea por línea en runtime, sin archivo de salida. El intérprete arranca más rápido (clave en la web); el compilador puede optimizar porque ve el programa completo. JavaScript nació interpretado, C es el ejemplo típico de compilado.

---

## 4. 🔴 JIT: quedarse con las dos cosas

¿Y si no hubiera que elegir? Eso es exactamente lo que hace el JavaScript moderno. **JavaScript ya no es un lenguaje puramente interpretado**: comenzó siéndolo, y evolucionó a ser interpretado **y** compilado a la vez, mediante lo que se llama **JIT** — *Just-in-Time compiler*, el compilador "justo a tiempo".

Acá entra la rama de abajo del pipeline de la sección 2:

- El **Interpreter** arranca al instante, como siempre — nadie espera.
- En paralelo, el **Profiler** (un programa que observa a otro programa mientras corre) mira por dónde pasa tu código: qué funciones se ejecutan mucho, qué loops se repiten.
- Cuando detecta código "caliente" — como `someCalculation` llamada mil veces — se lo pasa al **Compiler**, que compila **esa porción** a **código optimizado**.
- El motor termina ejecutando una mezcla: bytecode interpretado para el código frío, código compilado y optimizado para el caliente. Arranque inmediato **y** optimización. Las dos cosas.

Y esto no es un invento de JavaScript ni algo limitado a la web — es un principio general. **Java hace exactamente lo mismo**: compilás tu programa a bytecode, la JVM lo levanta, y adentro tiene su propio JIT con una pieza llamada *hotspot*: un profiler mira tu código en ejecución, y después de un millón de loops por el mismo lugar, el compilador en runtime agarra ese "punto caliente" y lo optimiza al vuelo. Mismo concepto, otra marca.

> **Para el parcial, si te preguntan** — *¿JavaScript es interpretado o compilado?*
> Hoy, las dos cosas: nació interpretado, pero los motores modernos usan compilación JIT (Just-in-Time). El intérprete ejecuta de inmediato mientras un profiler observa el código en runtime; las porciones que se ejecutan mucho ("calientes") se compilan a código optimizado sobre la marcha. Así se combina el arranque instantáneo del intérprete con la performance del compilador. Java hace lo mismo en la JVM (hotspot).

---

## 5. 🟡 Traducciones antes del motor

El motor no es el único traductor de la historia. Hay una familia de compiladores que corren **antes**, en tu máquina de desarrollo, y traducen **de código fuente a código fuente**:

- **Babel**: un compilador de JavaScript… a JavaScript. Agarra código con sintaxis moderna y lo traduce a sintaxis vieja, para que corra en browsers que todavía no soportan lo nuevo.
- **TypeScript**: otro lenguaje — un superset de JavaScript con tipos — cuyo compilador lo traduce a JavaScript plano. A ese proceso, de un lenguaje fuente a otro lenguaje fuente, se lo llama **transpilación**.

El detalle clave para no marearse: al browser **nunca** le llega TypeScript ni sintaxis exótica. Le llega JavaScript, y el motor de las secciones anteriores hace su trabajo de siempre. Estas herramientas viven en la etapa de *build* de tu proyecto — la Parte 6 les da su lugar en la cadena (bundlers), y la Parte 4 retoma a TypeScript para responder una pregunta más filosa: ¿te salva de las rarezas de JavaScript, o no?

---

## 6. 🟡 Call stack y memory heap

Dos estructuras más viven dentro del motor, y las necesitamos con nombre antes del plato fuerte. Te van a sonar de Sistemas Operativos — acá, en su versión JavaScript:

- El **call stack** (pila de llamadas) mantiene el rastreo de **qué se está ejecutando**: corre el código en orden y, cada vez que se llama una función, la apila arriba; cuando la función termina, la desapila. La última función llamada está siempre en el tope. Es lo que le permite al motor saber "estoy adentro de `baz`, que fue llamada por `bar`, que fue llamada por `foo`".
- El **memory heap** es donde se **guarda la información**: los objetos, las variables, los datos que tu código va creando.

```
   ┌──────────── JAVASCRIPT ENGINE ────────────┐
   │   CALL STACK          MEMORY HEAP         │
   │  ┌───────────┐      ┌─────────────────┐   │
   │  │  baz()    │ ◄tope│  { obj }  "str" │   │
   │  │  bar()    │      │  [array]  42    │   │
   │  │  foo()    │      │  …objetos y     │   │
   │  └───────────┘      │   variables     │   │
   │   qué se ejecuta     └────────────────┘   │
   └───────────────────────────────────────────┘
```

Esto no es teoría invisible: abrí las DevTools, poné un breakpoint en cualquier script, y el panel te muestra el call stack real — podés avanzar paso a paso y ver cómo las funciones se apilan y desapilan.

Guardá una propiedad del call stack, porque es la bisagra de todo lo que sigue: **hay uno solo, y ejecuta una cosa a la vez.**

---

## 7. 🔴 El runtime y el event loop

### 7.1 El problema

JavaScript es **single thread** — mono-hilo: un *thread* (hilo) es una secuencia de ejecución, y JavaScript tiene **una**. Ejecuta una instrucción a la vez, una función a la vez, en su único call stack. No hace dos cosas en paralelo. Nunca.

Frená un segundo en lo absurdo de eso. La página de la Parte 2 espera clicks, puede tener un timer andando, puede estar esperando una respuesta de la red — todo "al mismo tiempo"… ¿con un solo hilo? ¿Cómo puede ser que mientras espera los 3 segundos de un timer la página no quede congelada?

La respuesta no está en el motor. Está en lo que lo rodea.

### 7.2 El runtime: el motor no está solo

El motor (call stack + memory heap) vive adentro de una estructura mayor: el **JavaScript Runtime** — el entorno de ejecución completo, que suma las piezas que hacen posible la asincronía:

```
┌────────────────────────── JAVASCRIPT RUNTIME ──────────────────────────┐
│                                                                        │
│  ┌────────── ENGINE ──────────┐        ┌────────────────────────┐      │
│  │  CALL STACK   MEMORY HEAP  │ ─────► │        WEB APIs        │      │
│  │ ┌──────────┐               │ delega │   DOM                  │      │
│  │ │          │  (objetos,    │ tareas │   fetch()              │      │
│  │ │          │   variables)  │ async  │   setTimeout() ⏱       │      │
│  │ └──────────┘               │        └───────────┬────────────┘      │
│  └──────▲─────────────────────┘            al completarse, encola      │
│         │                                          ▼                   │
│         │  pasa el callback SOLO         ┌────────────────────────┐    │
│         │  cuando el stack está vacío    │     CALLBACK QUEUE     │    │
│         └───────── ⟳ EVENT LOOP ◄────────│   [ callback ] [ … ]   │    │
│                                          └────────────────────────┘    │
└────────────────────────────────────────────────────────────────────────┘
```

Las piezas nuevas:

- Las **Web APIs** de la Parte 2, ahora en su rol estelar: son las manos extra del browser. Cuando el código pide algo asíncrono, el motor **se lo delega a la Web API correspondiente** y sigue con lo suyo.
- La **callback queue**: una cola de espera. Un *callback* es una función que dejás lista para que se ejecute más tarde, cuando algo termine ("cuando pasen 3 segundos, ejecutá esto"). Cuando una Web API completa su tarea, deposita el callback acá.
- El **event loop**: un loop infinito — imaginate un `for` gigante girando — que coordina: mira si hay algo en las colas y, **cuando el call stack está vacío**, le pasa el próximo callback para que se ejecute.

### 7.3 La demo: verlo girar

El caso concreto. Cuatro funciones, un timer:

```javascript
// Imprime un saludo en la consola.
function printHello() {
    console.log('Hello from baz');
}

// setTimeout es la Web API de timers: "dentro de 3000 ms,
// ejecutá la función que te paso" (printHello es el callback).
function baz() {
    setTimeout(printHello, 3000);
}

function bar() {
    baz();     // bar solo llama a baz
}

function foo() {
    bar();     // foo solo llama a bar
}

foo();         // ← acá arranca todo

// Resultado esperado: la consola queda en silencio ~3 segundos,
// y recién entonces aparece:  Hello from baz
// Mientras tanto, la página NUNCA estuvo congelada.
```

```
// ¿CÓMO FUNCIONA? — el viaje completo, cuadro por cuadro:
//
// ① foo() se apila. Llama a bar() → se apila. Llama a baz() → se apila.
//
//      CALL STACK: [ foo, bar, baz ]
//
// ② baz ejecuta setTimeout(printHello, 3000). setTimeout es una Web API:
//    el motor le ENTREGA el timer al browser ("avisame en 3 segundos")
//    y sigue al instante. NO espera.
//
//      CALL STACK: [ foo, bar, baz ]     WEB API: ⏱ 3000ms → printHello
//
// ③ baz termina → se desapila. bar termina → se desapila. foo → ídem.
//
//      CALL STACK: [ vacío ]             WEB API: ⏱ contando…
//
// ④ Pasan los 3 segundos. La Web API deposita printHello en la cola.
//
//      CALLBACK QUEUE: [ printHello ]
//
// ⑤ El event loop ve: cola con tarea + stack vacío → pasa printHello
//    al stack. Se ejecuta. Consola: "Hello from baz".
```

La demo interactiva de esto existe y es oro: el sitio **Loupe** (latentflip.com/loupe) te deja pegar exactamente este código y ver el stack, la Web API y la cola animándose en tiempo real. Hay también una charla clásica del autor explicando el event loop con esta herramienta — si un video vale la pena de toda la clase, es ese.

Dos consecuencias del paso ⑤ que tienen que quedar grabadas:

**El callback espera al stack vacío.** El event loop le comunica la tarea al call stack **recién cuando todas las tareas síncronas terminaron**. Por eso un `setTimeout(f, 3000)` garantiza "no antes de 3 segundos", pero no "exactamente a los 3 segundos": si el stack está ocupado, el callback espera su turno.

**Promises y async/await viajan por el mismo circuito.** Una *promise* es un objeto que representa un resultado que va a llegar más tarde (una respuesta de la red, por ejemplo); `async/await` es sintaxis cómoda sobre promises; y `fetch` devuelve una promise. Para el runtime, todos son lo mismo que el timer: proceso asíncrono → se delega → el callback vuelve por una cola. Son la misma primitiva con distintos trajes.

### 7.4 Las colas finas: microtasks primero

Dije "las colas", en plural, a propósito. El event loop real no tiene una única cola — tiene varias, con **prioridades**. La distinción que importa:

- **Microtasks**: los callbacks de promises (`.then`, lo que sigue a un `await`). Tienen **prioridad**: apenas el stack se vacía, el event loop drena las microtasks **antes** de tocar cualquier otra cosa.
- **Macrotasks**: los timers (`setTimeout`, `setInterval`) y demás callbacks "gruesos". Van después.

O sea: si una promise resuelta y un `setTimeout(f, 0)` están esperando a la vez, **gana la promise**. El loop, además, tiene más estaciones — una función de muy bajo nivel llamada `nextTick`, colas de I/O, manejadores de cierre — y gira pasando por todas en un orden fijo.

> 🕳️ **Madriguera — El orden exacto del loop**
> ¿Qué corre primero entre un `nextTick`, una promise y un timeout encolados juntos? Hay una respuesta precisa, fase por fase, y es un clásico de los quizzes raros de entrevista. No hace falta saberla de memoria — hace falta entender el mecanismo de colas con prioridades; el orden fino se estudia si la entrevista lo pide.
> *Volvé al camino — esto se profundiza aparte, otro día.*

### 7.5 Don't block the main thread

Todo el edificio se sostiene sobre una regla de supervivencia que vas a escuchar en cualquier lugar serio: **no bloquees el hilo principal**. Si tu código síncrono se queda masticando un cálculo eterno, el stack nunca se vacía → el event loop nunca entrega nada → los clicks no responden, los timers no disparan, la página está **muerta**. En un lenguaje mono-hilo, bloquear el único hilo es apagar el mundo. Por eso todo lo lento (red, timers, I/O) se delega a las Web APIs y vuelve por las colas: para que el hilo principal esté siempre libre, siempre respondiendo.

> **Para el parcial, si te preguntan** — *Si JavaScript es single thread, ¿cómo hace varias cosas "a la vez"?*
> No las hace: el hilo de JavaScript ejecuta una cosa por vez en su call stack. La concurrencia la aporta el runtime: las operaciones asíncronas (timers, requests) se delegan a las Web APIs del browser, que trabajan por fuera del hilo; al completarse, encolan sus callbacks en la callback queue, y el event loop se los pasa al call stack cuando este queda vacío (con prioridad para las microtasks de promises sobre las macrotasks como setTimeout). El hilo nunca espera: delega y sigue.

---

## 8. 🟡 ¿Y si mi máquina tiene 16 cores?

Pregunta inevitable: tengo un procesador monstruoso, muchos núcleos — ¿JavaScript corre más "ancho"? ¿El event loop gira más veces?

**No.** Hay **un event loop por proceso**, siempre. JavaScript va a usar su único hilo igual, tengas los cores que tengas. Lo que sí podés hacer:

- **Workers** (escalar vertical, con pinzas): una API que permite levantar un hilo aparte y dedicarle un laburo puntual pesado. Existe, sirve para casos específicos, pero JavaScript **famosamente no escala bien vertical** — no es aconsejable armar castillos de workers.
- **Muchas instancias** (escalar horizontal — la joda de JavaScript): en lugar de un proceso gigante, levantás **varias instancias de tu programa**, cada una con su propio event loop y su propio call stack, repartiéndose el trabajo. ¿Y cómo se levantan N copias idénticas de un servicio, aisladas y baratas? Exacto: **containers** — la clase 02 entera era la preparación de esta jugada.

```
   Escalar VERTICAL (con límites)        Escalar HORIZONTAL (el camino JS)
   ┌───────────────────────┐            ┌─────────┐ ┌─────────┐ ┌─────────┐
   │  proceso JS           │            │ app ⟳   │ │ app ⟳   │ │ app ⟳   │
   │  event loop ⟳         │            │(container│ │(container│ │(container│
   │  + worker + worker…   │            └─────────┘ └─────────┘ └─────────┘
   └───────────────────────┘              cada instancia: su event loop
```

> 🕳️ **Madriguera — La fauna de "workers"**
> "Worker" es un apellido compartido por varias cosas distintas: los worker threads para cómputo pesado (los de esta sección) y los **service workers** de la sección siguiente, que juegan otro juego. Mismo nombre, roles distintos — no los mezcles.
> *Volvé al camino — esto se profundiza aparte, otro día.*

---

## 9. 🟡 Service workers y PWAs

Una vuelta de tuerca final al "single thread": el **navegador** puede levantar más de un motor de JavaScript, independientes entre sí. Un **service worker** es exactamente eso — código JavaScript tuyo que el browser ejecuta **en otro hilo, en background**, separado del hilo de tu página. JavaScript sigue siendo mono-hilo… pero pueden convivir dos "JavaScripts".

¿Para qué sirve un hilo tuyo corriendo en background?

- **Push notifications**: recibir notificaciones **incluso sin el browser abierto**. Cuando un sitio de turismo o un e-commerce te manda una notificación al escritorio sin que tengas ninguna pestaña suya, eso es un service worker vivo en background.
- **Caché offline**: interceptar requests y servir páginas guardadas. Poné el modo avión y volvé a entrar: en vez del dinosaurio de "sin conexión", aparece la página cacheada (con sus errores de red después, claro — pero la experiencia se sostiene).

Esas dos capacidades son los músculos de las **PWA** (*Progressive Web Apps*): aplicaciones web que buscan **comportarse como una app mobile**. Se pueden "agregar al inicio" del teléfono — te queda un ícono que abre la app a pantalla completa — pero abajo sigue siendo un browser embebido mostrando tu web, con los superpoderes que le dan los service workers: notificaciones, funcionamiento offline, el *look and feel* de app nativa. Sin pasar por ninguna app store.

Una aclaración de arquitectura que suele confundir: **la PWA es frontend, y es independiente de quién la sirva**. No hay un "deploy especial de PWA": tu servidor — sea un monolito, una CDN (una red de servidores que entrega archivos estáticos desde el punto más cercano al usuario) o lo que fuere — entrega el HTML y el JavaScript como siempre; una vez que llegaron al browser, el que muestra y ejecuta el front es **el cliente**. Si la app necesita más datos, hace otra request. Las distintas maneras de servir un frontend — que acá quedaron apenas insinuadas — son un tema con todas las letras en la Parte 6.

> **Para el parcial, si te preguntan** — *¿Qué es un service worker y qué relación tiene con las PWA?*
> Un service worker es código JavaScript que el browser ejecuta en un hilo separado, en background, independiente de la página. Habilita push notifications (incluso con el browser cerrado) y caché offline (interceptar requests y servir contenido guardado). Las PWA (Progressive Web Apps) son aplicaciones web que usan esas capacidades para comportarse como apps mobile: ícono en el inicio, pantalla completa, notificaciones y funcionamiento sin conexión — siendo, por debajo, una web en un browser embebido.

---

## Checkpoint — Parte 3

*(Sin respuestas a propósito: recuperación activa. Las respuestas van al complemento.)*

1. ¿Qué problema resuelve el JavaScript Engine? Nombrá dos engines y dónde corre cada uno.
2. Ordená y explicá: AST, tokens, bytecode, código máquina.
3. ¿Qué es el AST y qué otras dos estructuras de la clase comparten su forma? ¿Por qué será?
4. Un intérprete y un compilador ejecutan el mismo `for` de mil llamadas idénticas. ¿Qué hace distinto cada uno y qué trade-off ilustra?
5. ¿Por qué a la web le convenía un lenguaje interpretado?
6. ¿"Interpretado" implica "sin tipos fuertes"? Justificá.
7. ¿Qué es el JIT y qué rol juega el profiler? ¿Qué tiene que ver Java en esta historia?
8. ¿Qué es la transpilación? ¿Qué le llega finalmente al browser cuando el proyecto está escrito en TypeScript?
9. ¿Qué guarda el call stack y qué guarda el memory heap?
10. Explicá el recorrido completo de un `setTimeout(f, 3000)`: por qué la página no se congela y por qué los 3000 ms son un mínimo, no una promesa exacta.
11. ¿Qué diferencia hay entre microtasks y macrotasks, y quién gana si compiten?
12. ¿Qué significa "don't block the main thread" y qué pasa si lo bloqueás?
13. Tenés que escalar una app Node en una máquina de 32 cores. ¿Cuál es el camino recomendado y qué pieza de la clase 02 lo hace práctico?
14. ¿Qué es un service worker, en qué se diferencia del hilo de tu página, y qué dos capacidades le da a una PWA?

---

## Lo que viene — Parte 4

El motor ya ejecuta; falta **ver**. La Parte 4 sigue el camino del texto al pixel: cómo el HTML y el CSS de la Parte 2 se convierten en los árboles **DOM** y **CSSOM**, cómo se combinan en el *render tree* (y qué se cae en el camino), qué pasa cuando cada browser interpreta la spec a su manera, el famoso meme de la igualdad de JavaScript resuelto con el spec en la mano — y la parte con plata: cuánto tarda tu página, cómo se mide (web vitals, TTFB, LCP), por qué un `<script>` mal puesto te bloquea el render que acá quedó sembrado desde la Parte 2, la regla de los 14 KB, y la herramienta que audita todo eso: Lighthouse.

**FIN DE LA PARTE 3**
