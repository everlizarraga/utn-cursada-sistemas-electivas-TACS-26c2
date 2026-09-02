# TACS — Clase 03 · Arquitectura Web y Frontend
## Parte 4 — Del texto al pixel: render, estándares y performance

> **Sobre esta parte.** El motor ya ejecuta (Parte 3); ahora hay que **ver**. Esta parte sigue el camino del texto al pixel: los árboles **DOM** y **CSSOM** y su combinación en el *render tree*, qué pasa cuando cada browser interpreta el estándar a su manera (y quiénes ponen orden), el famoso meme de la igualdad de JavaScript resuelto con el spec en la mano, y el bloque con plata: **performance** — cómo se mide (web vitals, TTFB, LCP), dónde se pierde el tiempo (navigation timing, camino crítico, scripts bloqueantes, los 14 KB del primer viaje) y con qué se audita (**Lighthouse**). **No cubre** Ajax ni frameworks — Parte 5.
>
> **De dónde venís.** Parte 1 (el viaje de la URL — acá se le ponen relojes), Parte 2 (HTML como árbol, especificidad de CSS, el `<script>` cuya posición quedó sembrada como problema), Parte 3 (el hilo principal que no hay que bloquear, la transpilación). Hereda la convención `👁️ → Parte N`.

---

## 1. 🔴 Del texto al árbol: DOM + CSSOM → Render Tree

La response de la Parte 1 trae **texto**. La pantalla muestra **pixels**. Todo lo que pasa en el medio empieza acá:

```
 [ Characters ] ─► [ Tokens ] ─► [ Nodes ] ─► [ DOM ]
    el texto        piezas con     objetos      el árbol
    crudo           significado    del árbol    completo
```

El DOM — la Web API estrella de la Parte 2 — **se forma a partir de texto**: el browser toma los caracteres del HTML, los **tokeniza** (¿te suena? es el mismo movimiento que hizo el motor con tu código en la Parte 3), convierte los tokens en **nodos**, y con los nodos arma el árbol. Un documento como este:

```html
<html>
  <head>
    <meta charset="UTF-8">
    <link rel="stylesheet" href="style.css">
  </head>
  <body>
    <p>Hello, <span>web performance</span> students</p>
    <div><img src="awesome-photo.jpg"></div>
  </body>
</html>
```

…se convierte en este **DOM**:

```
                    html
                 ┌───┴────┐
               head      body
              ┌──┴──┐   ┌──┴─────────┐
            meta   link  p           div
                      ┌──┼─────────┐   │
                "Hello,"  span  "students"  img
                           │
                  "web performance"
```

**Pero el DOM solo no alcanza para pintar** — dice qué hay, no cómo se ve. En paralelo, el browser parsea el CSS y arma un segundo árbol: el **CSSOM** (*CSS Object Model*). Ojo con la forma de este árbol: no refleja cómo escribiste el CSS — refleja la **especificidad** de la Parte 2. Lo general arriba, afectando en cascada a los hijos:

```
      body { font-size: 16px }               ← lo general arriba…
       ├── p    { font-size: 16px; font-weight: bold }
       │    └── span { font-size: 16px; display: none }   ← …baja en cascada
       ├── span { font-size: 16px; color: red }
       └── img  { font-size: 16px; float: right }
```

(Fijate cómo el `font-size: 16px` del `body` se derrama hacia todos los descendientes: eso **es** la cascada, dibujada.)

Y ahora, la combinación. DOM (qué hay) + CSSOM (cómo se ve) = **Render Tree**, el árbol de lo que efectivamente se va a pintar:

```
      body { font-size: 16px }
       ├── p { font-size: 16px; font-weight: bold }
       │     ├── "Hello"
       │     └── "students"
       └── div
             └── img { font-size: 16px; float: right }
```

Mirá con lupa qué falta: **el `span` con `display: none` no está**. Ni él ni su texto. El render tree contiene **solo lo visible**: `head`, `meta`, `link` tampoco aparecen (nunca se ven), y todo lo que el CSS declara oculto queda afuera del árbol antes de pintar un solo pixel. El render tree no es "el DOM con estilos" — es la intersección visible de los dos árboles.

> **Para el parcial, si te preguntan** — *¿Qué es el render tree y cómo se forma?*
> Es el árbol de lo que efectivamente se va a renderizar, resultado de combinar el DOM (la estructura del documento, construida tokenizando el HTML) con el CSSOM (los estilos organizados por especificidad, construido parseando el CSS). Contiene solo los nodos visibles con sus estilos calculados: lo que no se ve — `head`, metadatos, elementos con `display: none` — queda excluido.

---

## 2. 🟡 Cada browser decide (los quirks)

Ahora, una verdad incómoda sobre todo lo anterior. HTML y CSS son **specs**: definen **qué** hacer, no **cómo** hacerlo. Son la interfaz de una implementación — y como toda spec, tienen partes incompletas y ambigüedades. ¿Quién implementa? Cada browser, por su cuenta. ¿Resultado? **Quirks**: el mismo código, comportamientos distintos según dónde corra.

No es teoría — es de lo más fácil de demostrar de toda la clase:

- **En HTML:** un `<fieldset disabled>` (el elemento que agrupa controles de formulario, deshabilitado) debería apagar todo su contenido. Poné adentro un botón, un link y un checkbox: el botón queda muerto en todos lados… pero según el browser, **el link sigue disparando eventos**, y el texto del checkbox responde distinto. La spec dejó grietas; cada implementación las rellenó a su gusto.
- **En CSS:** existe un div ya legendario — mismas propiedades, exactamente el mismo código — que renderiza **cinco formas distintas en cinco browsers**: dos cuadrados superpuestos en Firefox, un cuadrado chico en Edge, una forma irregular en Chrome, un cuadrado con la esquina cortada en Safari, y en Internet Explorer un cuadrado con un agujero en el medio. Cinco dibujos diferentes del mismo CSS.
- **En JavaScript:** métodos que existen en Chrome y no en Firefox, comportamientos de copiado distintos, features nuevas que un browser ya soporta y otro no. Pasa cada vez menos — pero pasa.

La consecuencia práctica es directa: si querés una plataforma web de uso masivo, es **mucho muy importante** verificar que funcione correctamente en los browsers más populares. Y para eso hay una herramienta que tiene que estar en tu barra de favoritos: **caniuse.com** — una matriz gigante de features × browsers × versiones. ¿Querés usar un `if` de CSS recién salido? Caniuse te dice qué browsers lo soportan y desde qué versión.

Pero el dato de caniuse es media respuesta. La otra media es **conocer a tus usuarios**: ¿qué browsers usan? ¿qué versiones? ¿qué porcentaje quedaría afuera si usás tal feature? La decisión nunca es "¿existe la feature?" sino "¿la puede renderizar la mayoría de **mi** tráfico?". Ese cruce — features disponibles × parque real de browsers de tus usuarios — es el pan de cada día del frontend serio.

---

## 3. 🟡 Organismos contra el caos

Si cada browser decide, ¿por qué la web no se atomizó en "esta página funciona solo en Chrome"? Porque hay organizaciones dedicadas a **revisar propuestas y definir estándares**, para que los desarrolladores no tengamos que hacer una página por navegador:

- **WHATWG** — hace los estándares vivos de la web: DOM, HTML, las Web APIs. (Dato de color con moraleja: nació como un *fork* de W3C — hasta los organismos de estándares se bifurcan.)
- **W3C** — revisa y aprueba los cambios.
- **ECMA** — es la casa de **ECMAScript**: la especificación de un lenguaje que se comporta de cierta manera. JavaScript **es** una implementación *ECMAScript-compliant* — y no es la única: hay otros lenguajes que cumplen la misma spec.

La relación clave para fijar: **ECMAScript es el contrato; JavaScript, el producto**. Cuando en la Parte 3 dijimos que V8 y SpiderMonkey implementan "el mismo lenguaje", el árbitro de ese "mismo" es la spec de ECMA. Y esto no es un dato de trivia — es una herramienta de trabajo, como muestra la próxima sección.

---

## 4. 🟡 JavaScript siendo JavaScript (el meme y el spec)

El material de burla favorito de internet:

```
0   == "0"    →  true
0   == []     →  true

…entonces, por transitividad de la igualdad, "0" == [] debería ser true, ¿no?

"0" == []     →  false      ¿¿ ??
```

La igualdad de JavaScript **no es transitiva**. Y la reacción estándar es "JavaScript es una porquería, miren las cosas que hace". Si te quedás con el meme — sí, parece un lenguaje roto. Pero si vas un paso más allá, la historia cambia: **nada de esto "se le escapó" a nadie. Está especificado.**

En la spec de ECMAScript conviven dos operadores de igualdad: `==` (*loosely equal*, igualdad "suelta") y `===` (estricta). Y para `==`, el algoritmo — se llama, literalmente, *IsLooselyEqual* — dicta reglas de **conversión** paso a paso. La que resuelve la primera línea: *si `x` es un número y `y` es un string, convertí `y` a número y comparé de nuevo.* Entonces:

- `0 == "0"` → `"0"` se convierte al número `0` → `0 == 0` → **true**. Dice exactamente lo que el spec manda.
- `0 == []` → el array se convierte a primitivo: `[]` se vuelve el string `""`, y `""` convertido a número es `0` → `0 == 0` → **true**. Otra regla del mismo algoritmo.
- `"0" == []` → acá **ambos se vuelven strings comparables**: `[]` → `""`… y `"0"` no es `""` → **false**. Ninguna regla convierte los dos lados al mismo `0`: los caminos de conversión son distintos, y por eso la transitividad se rompe.

¿Es raro? Sí. ¿Es un bug? No: es un **comportamiento especificado**, consultable, con su algoritmo numerado. Esa es la lección transferible: **cuando no entendés por qué JavaScript hace lo que hace, la respuesta final vive en el spec de ECMAScript**. Nadie te pide leerlo entero — pero saber que existe, y que se puede navegar hasta el algoritmo exacto, te saca del terreno del folklore. (Lección práctica inmediata: usá `===`. La igualdad estricta no convierte nada y el meme desaparece.)

### ¿Y TypeScript no me salva de esto?

Pregunta natural, respuesta precisa: **no — y no es su trabajo.** TypeScript no es "una versión arreglada de JavaScript": es **otro lenguaje**, un *superset* de JavaScript (lo incluye entero) que **responde al mismo spec**. Al compilar, se **transpila** a JavaScript plano (Parte 3, sección 5) — así que en runtime, `==` sigue siendo el `IsLooselyEqual` de siempre.

Lo que TypeScript sí te da es otra cosa, y es valiosa: **forzar tipos en desarrollo**. El ejemplo que lo aterriza: un controller que recibe un query param. No controlás lo que te mandan — puede venir cualquier cosa. Si en TypeScript declarás que ese parámetro es un `string`, el lenguaje te ayuda a que lo sea: no va a convertir implícitamente a escondidas — te va a marcar el error, y las verificaciones que genere por detrás (un `typeof`, un error si no da) son JavaScript, porque **por detrás es JavaScript**. En síntesis: el spec no cambia; cambia lo que el lenguaje te ayuda a garantizar mientras escribís.

> **Para el parcial, si te preguntan** — *¿Por qué `0 == "0"` es `true` pero `"0" == []` es `false`? ¿TypeScript lo soluciona?*
> Porque `==` aplica el algoritmo de conversión *IsLooselyEqual* del spec de ECMAScript: número vs. string convierte el string a número (`0 == "0"` → true), pero cada par de tipos sigue su propio camino de conversión, y esos caminos no son transitivos (`[]` se vuelve `""`, que no es igual al string `"0"`). No es un bug: está especificado. TypeScript no lo cambia — es un superset que transpila a JavaScript y responde al mismo spec; lo que aporta es chequeo de tipos en tiempo de desarrollo. La solución práctica en runtime es `===`, que compara sin conversiones.

---

## 5. 🟢 ¿Mi página genera valor? (la observabilidad del frontend)

Antes de entrar a performance, la pregunta que la justifica: ¿cómo sé que mi página **genera valor** — plata — y que lo hace en todos los browsers? Respuesta: instrumentándola. Existe toda una categoría de herramientas — pensalas como *la observabilidad del frontend* — que funcionan como scripts que inyectás en tu página y te devuelven datos de cómo la gente la usa:

- **A/B testing**: mostrarle a una parte de tus usuarios la variante A y a otra la B, y medir cuál convierte mejor. Decisiones por evidencia, no por opinión.
- **Heatmaps** (mapas de calor): dónde clickea la gente, hasta dónde scrollea, por dónde pasa.
- **Métricas de comportamiento**: cuánto dura una visita, qué recorridos se hacen, qué se abre y qué se ignora.

Tres nombres del rubro para tener oídos: **Google Analytics** y **Amplitude** (analítica de usuarios) y **BrowserStack** (testing cross-browser: tu página corriendo en decenas de combinaciones browser/dispositivo reales — la respuesta industrial al problema de la sección 2).

---

## 6. 🔴 Performance: ¿cuánto tiempo es aceptable?

Y acá aparece la plata en serio. La Parte 1 cerró con "el frontend es donde el valor se cobra"; esta sección le pone números.

El vínculo entre velocidad y negocio es directo y está medido: el tiempo de carga se traduce en **bounce rate** — el porcentaje de usuarios que entran y se van sin interactuar. Mientras más tarda la página, más gente se harta, se va, y **busca a la competencia** (te pasa a vos también, seamos honestos):

| Tiempo de carga (seg) | Bounce rate |
|---|---|
| 1 | 7 % |
| 2 | 6 % |
| 3 | 11 % |
| 4 | 24 % |
| 5 | 38 % |
| 6 | 46 % |

De 3 a 5 segundos el abandono **se triplica**. Cada segundo es plata que se va por la puerta.

Bien — hay que medir. ¿Medir *qué*? Las **web vitals**: el conjunto de métricas estandarizadas de experiencia de carga. Hay muchas; las tres para tener claras acá:

- **TTFB — Time To First Byte**: el tiempo desde que comienza la request hasta que el browser recibe **el primer byte** de la response. Mide todo el tramo de ida: DNS, conexión, procesamiento del servidor. Ideal: **P75 < 0,8 segundos** (*P75 = percentil 75: el 75% de tus usuarios por debajo de ese valor — se mide sobre tráfico real, no sobre tu máquina*).
- **LCP — Largest Contentful Paint**: el tiempo hasta renderizar **el elemento más grande visible** en el viewport — la imagen o el bloque de texto principal. Es el mejor proxy de "el usuario ya ve lo que vino a ver". Ideal: **P75 < 2,5 segundos**.
- ~~**FMP — First Meaningful Paint**~~: medía el momento en que la respuesta a "¿esto ya es útil?" se volvía sí (layout + contenido principal + fuentes cargadas). **Deprecada** — era difícil de definir con precisión, y hoy LCP ocupa ese rol. Queda el aprendizaje: las métricas también tienen ciclo de vida.

---

## 7. 🟡 El ciclo de vida de una carga (ponerle relojes al viaje)

¿Y dónde se pierde el tiempo? Para responder eso, el browser expone **el viaje de la Parte 1 convertido en línea de tiempo medible**, con un evento en cada frontera. Es el diagrama de *navigation timing*:

```
             ┌┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄ Resource Timing ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┐
 startTime   ┆                                               ┆
    │  redirectStart/End  fetchStart  domainLookupStart/End  ┆  domInteractive
    ▼        ▼         ▼      ▼          ▼   connectStart…   ┆     ▼  domContentLoaded…  domComplete
 ┌──────────┐ ┌──────────┐ ┌─────┐ ┌─────┐ ┌─────────┬──────────┐ ┌────────────┐ ┌──────┐
 │ Redirect │ │  Cache   │ │ DNS │ │ TCP │ │ Request │ Response │ │ Processing │ │ Load │
 └──────────┘ └──────────┘ └─────┘ └─────┘ └─────────┴──────────┘ └────────────┘ └──────┘
                                           ▲          ▲          ▲                      ▲
                                     requestStart  responseStart  responseEnd     loadEventEnd

                          TTFB = tiempo(startTime, responseStart)
```

Leelo con la Parte 1 en la mano: **Redirect** (si la URL redirige a otra), **Cache** (¿lo tengo guardado? — el cacheo de la madriguera de la Parte 1), **DNS**, **TCP**, **Request/Response** — es exactamente DNS → TCP → HTTP → HTML, el viaje de siempre. Y siguen dos etapas nuevas del lado del browser: **Processing** (parsear y construir los árboles de la sección 1) y **Load** (terminar de cargar todo). Cuando estudiamos los protocolos no estábamos hinchando: **este diagrama es ilegible si no sabés qué es cada etapa** — y este diagrama es como se diagnostica una página lenta.

Porque cada frontera tiene su evento con nombre (`domainLookupStart`, `connectStart`, `responseStart`, `domInteractive`, `domComplete`…), y vos — o tus herramientas — se pueden **colgar de esos eventos** para saber exactamente qué tarda. El razonamiento diagnóstico, con las métricas de la sección 6:

- **TTFB alto** → el problema está en la franja izquierda: redirects de más, DNS lento, servidor que tarda en responder. Fijate la fórmula en el diagrama: TTFB va de `startTime` a `responseStart`.
- **TTFB bajo pero LCP alto** → la red vino rápida; el problema está **del otro lado**: Processing, render, recursos pesados. El cuello es tuyo, no del viaje.

Dos métricas, un diagrama, y ya sabés en qué mitad del mundo buscar. Eso es medir.

---

## 8. 🔴 El camino crítico (y cómo dejar de pisarlo)

Zoom a la etapa **Processing** — porque ahí adentro vive el concepto que ata esta parte entera: el **critical rendering path**, el camino crítico del render. Es la secuencia mínima de pasos que el browser **no puede evitar** antes de mostrar algo:

```
 ┌────────────┐
 │ Parse HTML │──┐
 └────────────┘  ├──► [ Render Tree ] ──► [ Layout ] ──► [ Paint ]
 ┌────────────┐  │      (sección 1)        ¿dónde va      pintar
 │ Parse CSS  │──┘                          cada cosa?     pixels
 └────────────┘
```

Parsear el HTML, parsear el CSS, combinar en el render tree (sección 1), calcular el **layout** (la geometría: dónde va y cuánto mide cada elemento) y **pintar**. El render inicial ocurre recién cuando el HTML y el CSS iniciales están cargados y procesados — **siempre** hay un HTML de entrada y un CSS que lo estila, y son la puerta de todo. Conclusión operativa: esos dos archivos tienen que ser **mínimos**, y servirlos tiene que estar **tuneadísimo**.

### El enemigo público: el script en el medio

Ahora sí, a cobrar la semilla plantada en la Parte 2 ("dónde pongas el `<script>` tiene consecuencias"). El browser lee el documento con un escáner **secuencial**: va procesando y tokenizando el texto a medida que llega, para pintar. Si en el medio de ese texto le clavás un `<script>`, el browser — que es secuencial y obediente — **frena todo**: descarga el script completo, lo parsea, lo ejecuta (con el motor de la Parte 3)… y recién después sigue leyendo tu HTML. Le bloqueaste el **main thread** — sí, el mismo de "don't block the main thread": la regla de la Parte 3 tiene su caso más famoso acá, en el render.

Para eso existen dos atributos del tag `<script>`, que son una **pista** que le das al browser ("no le des bola a esto ahora — lo que quiero mostrar es el HTML"):

```
 <script>          ████████░░░░░░░░░████████████     █ = parseando HTML
                           ▒▒▒▒▒▒▒▒▒▓▓▓              ░ = parseo PAUSADO
                           descarga  ejecuta          ▒ = descargando script
                                                      ▓ = ejecutando script
 <script async>    ████████████████░░░██████████
                           ▒▒▒▒▒▒▒▒▒▓▓▓
                           descarga en paralelo; solo pausa al ejecutar

 <script defer>    ████████████████████████████ ▓▓▓
                           ▒▒▒▒▒▒▒▒▒
                           descarga en paralelo; ejecuta AL FINAL
```

- **Sin atributo**: el parseo se pausa durante la descarga **y** la ejecución. El peor caso.
- **`async`**: descarga en paralelo mientras el HTML se sigue parseando; pausa **solo** para ejecutar, apenas termina de bajar.
- **`defer`**: descarga en paralelo y **difiere** la ejecución hasta que el parseo terminó. El HTML nunca espera.

### La lista de tuneo

Con el camino crítico entendido, las optimizaciones dejan de ser recetas mágicas y se vuelven obvias:

- **Scripts con `async`/`defer`** — no bloquear el escáner, recién visto.
- **Minificar y comprimir HTML y CSS** — *minificar*: eliminar espacios, saltos y nombres largos del código (bytes que el browser no necesita); *comprimir*: enviarlo comprimido por la red (gzip y familia). Menos bytes en el camino crítico.
- **¿CSS inline o en archivos?** — Respuesta de la casa: **inline lo crítico** (los estilos de lo que se ve al entrar, adentro del propio HTML, para que lleguen en el primer viaje) **y async el resto**. Ni todo adentro ni todo afuera: lo que pinta primero, primero.
- **Fonts y assets: async** — que las fuentes y recursos no frenen el primer render.
- **La regla de los 14 KB.** Esta merece su párrafo. Cuando se abre la conexión TCP (el handshake de la Parte 1), el protocolo no arranca a fondo: usa un mecanismo llamado **slow start** — el primer round trip transporta una ventana de **~14 KB**, y en cada ida y vuelta la ventana **se duplica**, negociándose más grande, hasta que entra todo. Consecuencia: lo que entre en esos primeros 14 KB llega un round trip entero antes que el resto. Ideal de manual: HTML + CSS crítico en ≤ 14 KB. En la práctica, por el crecimiento exponencial de la ventana, hasta ~100 KB se considera razonable — son pocos round trips igual. La idea de fondo no es el número exacto: es **pensar en round trips**, porque cada ida y vuelta con el servidor se paga completa. Y ojo TP: cómo armes el *bundle* de tu aplicación (el paquete de archivos final que servís — Parte 6) define cuántos KB le tirás al usuario en la primera request; si es todo de una y no mirás esto, vas a tener un problema de carga que no vas a saber diagnosticar.
- **Declarar el charset al principio.** El `<meta charset="UTF-8">` tiene que aparecer **en el primer KB** del HTML: le dice al parser qué codificación usar; si no está, el browser tiene que adivinarla y el parseo se enlentece. (Y Lighthouse — ya llegamos — te lo marca como error.)

> **Para el parcial, si te preguntan** — *¿Qué es el critical rendering path y cómo se optimiza?*
> Es la secuencia mínima que el browser recorre para el primer render: parsear HTML y CSS, combinar en el render tree, calcular layout y pintar. Se optimiza sacando del camino todo lo que lo bloquea o engorda: scripts con `async`/`defer` para no frenar el parseo, HTML/CSS minificados y comprimidos, CSS crítico inline y el resto async, fonts y assets asíncronos, contenido crítico dentro de la ventana inicial de TCP slow start (~14 KB por round trip, duplicándose), y el charset declarado en el primer KB.

---

## 9. 🟡 Lighthouse: la auditoría en un click

Todo lo anterior — métricas, camino crítico, optimizaciones — converge en una herramienta que lo audita de una: **Lighthouse**, de Google, incluida en las DevTools de Chrome (pestaña *Lighthouse* del inspector). Le pasás una página; la herramienta **la renderiza por vos** y escupe un reporte con puntajes de 0 a 100 por categoría — verde de 90 a 100, naranja de 50 a 89, rojo abajo de 50:

```
  Performance   Accessibility   Best Practices    SEO
      45             56               93           100
     (rojo)       (naranja)         (verde)      (verde)
```

Por cada categoría, el detalle. En Performance: las métricas medidas (First Contentful Paint 2,6 s, Speed Index 6,4 s, Time to Interactive 10,1 s…), una tira de capturas de cómo se fue pintando la página, y — la parte de oro — **Opportunities**: oportunidades concretas de mejora con su ahorro estimado ("diferí las imágenes fuera de pantalla: te ahorrás 2,85 s"). Es configurable: mobile o desktop, y qué auditar — performance, accesibilidad, SEO (*Search Engine Optimization*, qué tan encontrable sos en buscadores — se desarrolla en serio en la Parte 5), buenas prácticas.

El punto pedagógico, para cerrar el círculo de la parte: Lighthouse te *soluciona* el diagnóstico… **si sabés de qué te está hablando**. "Defer offscreen images", "reduce render-blocking resources", "declare charset early" — cada oportunidad del reporte es una de las secciones de esta parte con otro sombrero. Primero se entiende el camino crítico; después la herramienta multiplica. Al revés, es cargo cult.

---

## Checkpoint — Parte 4

*(Sin respuestas a propósito: recuperación activa. Las respuestas van al complemento.)*

1. ¿Cómo se construye el DOM a partir del texto HTML? ¿Qué proceso de la Parte 3 se le parece?
2. ¿Por qué el CSSOM es un árbol, si el CSS no se escribe como árbol?
3. ¿Qué contiene el render tree y qué queda afuera? ¿Dónde terminó el `span` con `display: none`?
4. "HTML y CSS son specs." ¿Qué significa eso y qué consecuencia práctica tiene? Dá un ejemplo de quirk.
5. ¿Qué te responde caniuse, y qué otra información necesitás cruzar para tomar la decisión completa?
6. ¿Qué roles cumplen WHATWG, W3C y ECMA? ¿Qué relación hay entre ECMAScript y JavaScript?
7. ¿Por qué `0 == "0"` da `true` y `"0" == []` da `false`? ¿Dónde está la respuesta definitiva a este tipo de preguntas?
8. ¿TypeScript "arregla" la igualdad de JavaScript? ¿Qué aporta entonces?
9. ¿Qué es el bounce rate y qué relación medible tiene con el tiempo de carga?
10. Definí TTFB y LCP, con sus valores ideales. ¿Qué significa el P75?
11. Tu TTFB es bajo pero tu LCP es alto. ¿En qué mitad del ciclo de carga buscás el problema, y por qué?
12. Enumerá las etapas del critical rendering path.
13. ¿Qué le hace al parseo un `<script>` sin atributos? ¿Qué cambian `async` y `defer`, exactamente?
14. Explicá la regla de los 14 KB: de dónde sale, cómo crece la ventana, y qué decisión de diseño te impone.
15. ¿Por qué el charset va en el primer KB del HTML?
16. ¿Qué es Lighthouse, dónde vive, y por qué "usarla sin entender el camino crítico" es cargo cult?

---

## Lo que viene — Parte 5

Hasta acá, el modelo es el de la Parte 1: pedís un documento, llega, se renderiza. Ahora hacete esta pregunta: ¿y si querés **cambiar algo**? ¿Un dato, un contador, un post-it nuevo? Con este modelo, la respuesta es humillante: pedís **todo el documento de nuevo**. La Parte 5 cuenta cómo la web se sacó ese corset de encima — **Ajax** y el asincronismo de la Parte 3 puestos a pedir pedacitos, el salto al cliente pesado, y el framework que ordenó el quilombo resultante: **React**, con su virtual DOM (que ahora vas a entender de una: es un render tree de mentira para no tocar el de verdad), sus componentes, sus props, su estado — y la factura que todo eso le pasó al SEO.

**FIN DE LA PARTE 4**
