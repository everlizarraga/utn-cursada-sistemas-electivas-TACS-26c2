# TACS — Clase 03 · Arquitectura Web y Frontend
## Parte 2 — Los materiales de la página: HTML, CSS, JavaScript y las Web APIs

> **Sobre esta parte.** Abre los cuatro cuadraditos de arriba de la pila (Parte 1, sección 5): **HTML** como estructura, **CSS** como apariencia, **JavaScript** como comportamiento, y las **Web APIs** — la pieza que casi nadie nombra y que conecta a las otras tres con el browser. Es la parte de los *materiales*: con qué se construye una página. **No cubre** cómo el browser ejecuta JavaScript por dentro (Parte 3) ni cómo convierte estos materiales en pixels (Parte 4).
>
> **De dónde venís.** Parte 1: el viaje de la URL, la pila de la web, y la idea de que el browser es un cliente HTTP + render + estándares. De la clase 01 se asume HTTP y sus requests/responses. Hereda la convención `👁️ → Parte N` declarada en la Parte 1.

---

## 1. 🟡 Tres archivos y un browser

Al final de la Parte 1 quedó una response HTTP con un documento HTML adentro. La pregunta ahora es qué hace el browser con eso — y la respuesta corta es: **el HTML es el punto de entrada, y desde ahí se tira del resto**.

El browser interpreta archivos en formato HTML, que a su vez pueden solicitar la carga e interpretación de archivos CSS y JavaScript. La página mínima que muestra el trío completo:

```html
<!DOCTYPE html>                                  <!-- Le declara al browser: esto es HTML moderno -->
<html>                                           <!-- Raíz del documento: todo vive adentro -->
  <head>                                         <!-- Metadatos y recursos: nada de esto "se ve" -->
    <link rel="stylesheet" href="helloworld.css">  <!-- "Browser: andá a buscar este CSS y aplicalo" -->
    <script src="./helloworld.js"></script>        <!-- "Browser: andá a buscar este JS y ejecutalo" -->
  </head>
  <body>HOLA</body>                              <!-- El contenido visible de la página -->
</html>
```

```
// ¿CÓMO FUNCIONA?
// 1. Llega el HTML (paso ⑤ del viaje de la Parte 1).
// 2. El browser lo lee de arriba a abajo. Al encontrar <link>, dispara
//    OTRA request HTTP para traer helloworld.css. Al encontrar <script>,
//    otra más para helloworld.js. Una página = varias requests.
// 3. Con las tres piezas, renderiza: estructura (HTML) + apariencia (CSS)
//    + comportamiento (JS).
// Resultado esperado: una página que dice HOLA, estilada por el CSS,
// con el JS ya ejecutado.
```

Un detalle para dejar sembrado: *dónde* pongas ese `<script>` dentro del documento tiene consecuencias de performance reales — el browser lee secuencialmente y un script puede frenarle la lectura. Eso se desarrolla con nombre y apellido en la Parte 4.

**Tu herramienta para ver todo esto en vivo son las DevTools del browser** (clic derecho sobre cualquier página → *Inspeccionar*, o F12). Dos pestañas van a acompañar toda la parte: **Elements**, que muestra el árbol HTML *vivo* de la página tal como el browser lo tiene en memoria, y su panel **Styles**, que muestra qué reglas CSS están aplicadas a cada elemento — incluidas las que perdieron y quedaron tachadas. Vale abrirlas sobre cualquier sitio mientras leés: todo lo que sigue se puede tocar ahí.

---

## 2. 🟡 HTML — la estructura que habla

HTML es un lenguaje ~~de programación~~ **de markup** (de *marcado*: texto anotado con etiquetas que describen su estructura), primo del XML*, con el que se definen dos cosas: la **estructura** de la página y la **semántica** del contenido.

*\* XML: formato genérico de texto estructurado con etiquetas — lo viste serializando datos en la clase 01; HTML es de la misma familia, especializado en documentos web.*

No es un lenguaje de programación — no tiene variables, ni condicionales, ni lógica. Es simple a propósito. Pero que sea simple no lo hace menor: sobre su forma se construye literalmente todo lo demás.

**Primera propiedad: forma un árbol.** Las etiquetas abren y cierran, y lo que abre adentro de otra cosa es su hijo. Padres e hijos → un árbol:

```html
<ul>                                        <!-- ul = unordered list: una lista sin numerar -->
  <li>                                      <!-- li = list item: un ítem de esa lista -->
    <a href="/wiki/Main_Page">Main page</a> <!-- a = anchor: un link hacia otra página -->
  </li>
  <li>
    <a href="/wiki/Portal:Contents">Contents</a>
  </li>
  <li>
    <a href="/wiki/Portal:Current_events">Current events</a>
  </li>
</ul>
```

```
            ul                ← la lista (padre)
       ┌────┼──────┐
      li    li     li         ← los ítems (hijos)
       │    │      │
       a    a      a          ← los links (nietos)
```

Este árbol no es un detalle estético: es **la** estructura de datos sobre la que va a trabajar todo lo que viene — JavaScript lo recorre (sección 4), el browser lo construye pieza por pieza para renderizar (Parte 4), y React existe para no tener que tocarlo a mano (Parte 5).

**Segunda propiedad: los tags son semánticos.** Las etiquetas no dicen cómo se ve algo — dicen **qué es**. Leyendo el código de arriba, sin ver ningún render, ya sabés que hay una lista, que tiene tres ítems, y que cada ítem es un link de navegación. El contenido se describe a sí mismo. Ese es el diseño de HTML: un menú es `<ul>` con `<li>`, un título es `<h1>`, la navegación es `<nav>` — y esa información semántica la aprovechan los lectores de pantalla, los buscadores y cualquier programa que necesite *entender* la página en vez de solo dibujarla.

> 🕳️ **Madriguera — Accesibilidad**
> HTML trae un arsenal para que las páginas sean usables con lectores de pantalla y otras tecnologías asistivas: etiquetas semánticas, atributos como `title`, `alt` y `accesskey`, y toda la familia ARIA. La semántica de la que hablamos acá es su cimiento. Hay una electiva entera de la carrera (Experiencia de Usuario y Accesibilidad) que vive en este tema.
> *Volvé al camino — esto se profundiza aparte, otro día.*

> **Para el parcial, si te preguntan** — *¿Qué es HTML y qué significa que sea semántico?*
> HTML es un lenguaje de markup (no de programación) con el que se define la estructura de una página y la semántica de su contenido. Sus etiquetas forman un árbol de padres e hijos, y son semánticas porque describen **qué es** cada contenido (lista, ítem, link, título) y no cómo se ve — lo que permite entender el documento sin renderizarlo.

---

## 3. 🟡 CSS — la apariencia y la regla de especificidad

CSS es el lenguaje con el que se define la **apariencia**: permite aplicar estilos **de manera selectiva** a elementos del documento HTML. Colores, fuentes, tamaños, posiciones — todo se declara acá. Se conecta a la página con el `<link>` que viste en la sección 1.

La palabra clave es *selectiva*: una regla CSS tiene dos mitades — un **selector** que dice *a quiénes* les aplica, y un bloque de propiedades que dice *qué* les aplica. Y los selectores tienen una jerarquía que hay que saber leer:

```css
/* Selector por TAG: aplica a TODOS los <a> de la página */
a {
    text-decoration: none;    /* saca el subrayado de los links */
    color: #0645ad;           /* azul */
    background: none;
}

/* Selector compuesto: solo los <a> que estén adentro de un <li>,
   adentro de algo con class="body", adentro de algo con class="portal",
   adentro DEL elemento con id="mw-panel" */
#mw-panel .portal .body li a {
    color: #0645ad;
}
```

La escalera de selectores, de lo más general a lo más específico:

```
  MENOS específico ─────────────────────────────► MÁS específico

     a                    .portal                   #mw-panel
  (tag pelado)           (. = clase)                (# = id)
  todos los <a>       los elementos con         EL elemento con
  de la página        class="portal"            id="mw-panel"
                      (puede haber muchos)      (único en la página)
```

- `#nombre` selecciona **por id** — un id identifica a un único elemento.
- `.nombre` selecciona **por clase** — una clase agrupa a varios elementos.
- `nombre` pelado selecciona **por tag** — todos los elementos de ese tipo. Y esto último es **lo más peligroso y lo menos performante que podés hacer**: peligroso porque una regla sobre `a` o `div` pega en toda la página, incluidos lugares que no tenías en mente; menos performante porque el browser tiene que evaluarla contra muchísimos más elementos.

**¿Y si dos reglas chocan?** Como en el ejemplo: las dos definen `color` para los mismos links. Ahí entra la regla madre de CSS — la **especificidad**: los estilos se aplican **de lo general a lo particular, siempre**. La regla más específica gana. `#mw-panel .portal .body li a` le gana a `a` pelado, y el `color` de la regla genérica queda **pisado**. Las DevTools te lo muestran literal: en el panel Styles, la propiedad perdedora aparece tachada. Cuando "el CSS no anda", el 90% de las veces la respuesta está en ese panel: otra regla más específica está ganando.

Última foto de situación, que va a reaparecer: **no existe una única forma establecida de usar CSS** en la industria. Hay quien lo escribe inline, quien lo separa en archivos, y un ecosistema entero de libraries, preprocesadores y compiladores que generan CSS por vos. Ese zoológico tiene su sección propia en la Parte 6 — por ahora alcanza con saber que existe y que la especificidad rige en todos.

> **Para el parcial, si te preguntan** — *¿Qué significan `#`, `.` y un nombre pelado en un selector CSS? ¿Cuál conviene evitar?*
> `#` selecciona por id (un único elemento), `.` por clase (un grupo de elementos) y el nombre pelado por tag (todos los elementos de ese tipo). El selector por tag pelado es el más riesgoso y el menos performante: afecta a toda la página y obliga al browser a evaluar la regla contra muchos más elementos. Ante conflicto entre reglas, gana la más específica (especificidad: de lo general a lo particular).

---

## 4. 🟡 JavaScript — el comportamiento (y el árbol que toca)

JavaScript es **un lenguaje de programación que maneja el browser**. Nació para "agregarle comportamiento" a páginas que eran documentos estáticos; hoy puede manipular la página **íntegramente**. Es lo que le da **interactividad** a la web: que algo pase cuando clickeás, que el contenido cambie sin recargar, que la página reaccione.

¿Y sobre qué actúa ese comportamiento? Sobre el árbol de la sección 2. El browser toma tu HTML y construye en memoria una representación viva de ese árbol, llamada **DOM** (*Document Object Model*): el documento convertido en objetos a los que se les puede pedir y ordenar cosas. El DOM es la maqueta viva de la página — tocás la maqueta, cambia la página.

Ejemplo completo, para pegar en un archivo `demo.html` y abrir con el browser:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Demo DOM</title>
  </head>
  <body>
    <h1>Título original</h1>

    <script>
      // Busca en el DOM el primer elemento que matchee el selector 'h1'.
      // document.querySelector recibe UN SELECTOR CSS — exactamente los de la
      // sección 3: 'h1' (tag), '.clase', '#id' — y devuelve ese elemento.
      const miTitulo = document.querySelector('h1');

      // Cambia el texto del elemento. El DOM se modifica → la página, también.
      miTitulo.textContent = '¡Hola mundo!';

      // Resultado esperado: al abrir la página ves "¡Hola mundo!".
      // El h1 decía "Título original", pero el script corrió y lo
      // reescribió antes de que llegues a verlo.
    </script>
  </body>
</html>
```

```
// ¿CÓMO FUNCIONA?
// 1. El browser parsea el HTML y arma el DOM: html → body → h1 ("Título original").
// 2. Encuentra el <script> y ejecuta el JS.
// 3. querySelector('h1') recorre el árbol y devuelve el nodo del h1.
// 4. textContent = '...' modifica ese nodo EN el árbol.
// 5. El browser refleja el cambio en pantalla. Nunca se tocó el archivo HTML:
//    se tocó su representación viva en memoria.
```

Fijate el puente que se acaba de tender: **querySelector usa selectores CSS**. Las tres tecnologías no son mundos separados — JS encuentra elementos usando el idioma de CSS para operar sobre el árbol que definió HTML.

**Ahora, la advertencia.** ¿Se puede construir una aplicación entera así, a puro `querySelector` y modificación manual (lo que se llama *vanilla JS* — JavaScript sin frameworks)? Se puede, y para páginas chicas o para aprender está perfecto. Pero en una aplicación real, **manipular el DOM a mano es considerado un antipatrón**: con decenas de componentes y estados que cambian, sincronizar a mano qué parte del árbol tocar en cada momento se vuelve inmanejable y frágil. Pasa, pero la idea es que no pase. Guardá este dolor — es exactamente el que los frameworks vienen a resolver, y la razón de existir de React (Parte 5).

---

## 5. 🔴 Web APIs — el cuadradito que faltaba

Queda la cuarta pieza de la pila, la que quedó debiendo la Parte 1. Y conviene arrancar por lo que acabás de hacer sin darte cuenta: **`document.querySelector` es, literalmente, una Web API.** Ya usaste una.

Cuando se dice "la web es HTML, CSS y JavaScript" se está omitiendo la pieza que hace que eso funcione como sistema. El browser no agarra el HTML y ahí termina todo: **está interactuando todo el tiempo a través de APIs** — con el código, con la red, con la página. Si no entendés qué es una Web API, es difícil que entiendas todo lo que pasa después en esta clase. De ahí el 🔴.

**¿Qué son?** No hay una definición de libro, pero la idea es esta: las Web APIs son como **libraries que trae el browser**, las interfaces por las que tu código JavaScript le pide cosas al browser y a las demás piezas de la página. ¿Querés tocar el HTML? No lo tocás "directo": le hablás a la web API del DOM. ¿Querés hacer una request HTTP desde JS? Web API de Fetch. El browser tiene el control real de todo (la página, la red, los timers); las Web APIs son las ventanillas por las que se lo pedís.

```
        TU CÓDIGO JavaScript
              │
              │  llama a…
              ▼
   ┌────────────────────────────┐
   │        WEB APIs            │   ← la INTERFAZ: especificada por
   │   DOM · Fetch · Crypto ·…  │     estándares, igual en todo browser
   └────────────────────────────┘
              │
              ▼
        EL BROWSER hace
        el trabajo real            ← la IMPLEMENTACIÓN: cada browser
        (tocar la página,            la resuelve a su manera
         salir a la red, …)
```

**¿Por qué "API" y por qué "web"?** Son *APIs* en el sentido pleno: definen una **interfaz** (qué funciones existen, qué reciben, qué devuelven) que después **cada browser implementa por su cuenta** — Chrome, Firefox y Safari implementan cada uno su `querySelector`, pero todos respetan el mismo contrato, y por eso tu código corre en cualquiera. Y son "web" simplemente porque son las APIs del mundo web; "API" a secas es un término demasiado genérico (una API REST de la clase 01 también es una API, y no tiene nada que ver con estas).

Las que van a aparecer en esta clase:

| Web API | Qué te deja hacer | Dónde reaparece |
|---|---|---|
| **DOM** | Leer y modificar el árbol de la página desde JS | Partes 4 y 5 |
| **Fetch** | Hacer requests HTTP desde JS, sin recargar la página | Parte 5 (es el corazón de Ajax) |
| **Timers** (`setTimeout` y cía.) | Programar código para que corra más tarde | Parte 3 (protagonista del event loop) |
| **Crypto** | Operaciones criptográficas en el browser | — |
| **WASM** (WebAssembly) | Ejecutar código compilado, casi a velocidad nativa, en el browser | — |

La lista completa es larga y se encuentra buscando "browser web APIs" — hay APIs para audio, geolocalización, notificaciones, almacenamiento y más. No hace falta conocerlas todas; hace falta reconocer el patrón: **capacidad del browser expuesta a tu código = Web API**.

Dos observaciones para cerrar la idea:

**Los frameworks no las reemplazan — las envuelven.** Cuando llegues a React (Parte 5) vas a escribir código que jamás menciona al DOM, y sin embargo, abajo de todo, React está llamando a estas mismas APIs. Los frameworks te abstraen el laburo; la maquinaria es esta.

**El eco del lado del servidor.** Si alguna vez usaste JavaScript fuera del browser — Node.js — habrás notado que también hay `fetch`, también hay `setTimeout`, también hay asincronismo. No son "el browser": son implementaciones equivalentes de esas mismas interfaces, del lado del servidor. La interfaz trasciende al browser; por eso saber Web APIs es saber JavaScript-en-el-mundo-real, de los dos lados del cable.

> 🕳️ **Madriguera — WebAssembly (WASM)**
> WASM permite compilar lenguajes como C++ o Rust a un formato binario que el browser ejecuta casi a velocidad nativa — es lo que hace posible correr juegos, editores de video o AutoCAD adentro de una pestaña. Convive con JavaScript, no lo reemplaza.
> *Volvé al camino — esto se profundiza aparte, otro día.*

> **Para el parcial, si te preguntan** — *¿Qué es una Web API? Dé ejemplos.*
> Las Web APIs son las interfaces que el browser le expone al código JavaScript para interactuar con sus capacidades: manipular la página (DOM), hacer requests HTTP (Fetch), programar timers (`setTimeout`), criptografía (Crypto), ejecutar código compilado (WebAssembly). Son "APIs" porque definen una interfaz estándar que cada browser implementa por su cuenta, garantizando que el mismo código corra en todos. Sin ellas, JavaScript no podría tocar ni la página ni la red.

---

## Checkpoint — Parte 2

*(Sin respuestas a propósito: recuperación activa. Las respuestas van al complemento.)*

1. ¿Cómo se relacionan los tres archivos de una página web? ¿Cuál es el punto de entrada y cómo se cargan los otros dos?
2. ¿Por qué HTML no es un lenguaje de programación? ¿Qué tipo de lenguaje es?
3. ¿Qué significa que los tags de HTML son semánticos? Dá un ejemplo donde la semántica te permita entender el contenido sin ver el render.
4. ¿Qué estructura de datos forma un documento HTML y por qué es importante para todo lo que viene después?
5. En CSS: ¿qué selecciona `#panel`, qué selecciona `.panel` y qué selecciona `panel` a secas? ¿Cuál de los tres es el más peligroso y por qué?
6. Dos reglas CSS definen colores distintos para el mismo elemento. ¿Cuál gana y cómo se llama esa regla? ¿Dónde lo verificás en las DevTools?
7. ¿Qué es el DOM y qué relación tiene con el archivo HTML?
8. `document.querySelector('h1')` — ¿qué recibe esa función, qué devuelve, y qué conexión muestra entre JS y CSS?
9. ¿Por qué manipular el DOM a mano se considera un antipatrón en aplicaciones reales, si técnicamente funciona?
10. ¿Qué es una Web API? ¿Por qué se llaman "API" y por qué "web"?
11. Nombrá tres Web APIs y qué permite hacer cada una.

---

## Lo que viene — Parte 3

Ya está el qué (los materiales) — falta el cómo. En la sección 4 el browser "ejecutó el JS", así nomás, como si fuera gratis. No lo es: las computadoras no entienden JavaScript. La Parte 3 se mete adentro del browser a ver la maquinaria que lo hace posible — el **engine** que traduce tu código a código máquina, la diferencia entre interpretar y compilar (y el truco de hacer las dos cosas a la vez), y el **runtime** con su pieza más famosa: el **event loop**, la respuesta a cómo un lenguaje que solo puede hacer una cosa a la vez sostiene toda la interactividad de la web. Los timers de la tabla de Web APIs son los protagonistas.

**FIN DE LA PARTE 2**
