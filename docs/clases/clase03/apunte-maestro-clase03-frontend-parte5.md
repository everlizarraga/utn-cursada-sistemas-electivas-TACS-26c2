# TACS — Clase 03 · Arquitectura Web y Frontend
## Parte 5 — Ajax, React y la SPA: la web deja de ser un documento

> **Sobre esta parte.** El modelo de la Parte 1 — pedir un documento, recibirlo, pintarlo — tiene un defecto estructural, y esta parte cuenta la escalada que lo resolvió: **Ajax** y el arte de pedir pedacitos, el salto al **cliente pesado**, los **frameworks** que ordenaron el quilombo, y el ganador de esa guerra: **React** — virtual DOM, componentes, props, estado, hooks y routing — junto con el precio de todo esto: la **SPA** y su pelea con el **SEO**. **No cubre** las estrategias para servir la aplicación (SSR, SSG y compañía) ni el empaquetado — Parte 6.
>
> **De dónde venís.** Parte 2 (el DOM, y aquella advertencia de que tocarlo a mano es antipatrón — acá se cobra), Parte 3 (callbacks, promises, async/await y el event loop: todo Ajax viaja por ese circuito), Parte 4 (el render tree — el virtual DOM es de la familia — y la glosa de SEO, que acá se desarrolla). Hereda la convención `👁️ → Parte N`.

---

## 1. 🔴 El defecto del modelo documento

Recapitulemos el modelo que venimos usando desde la Parte 1, ahora con ojo crítico. Aplicación clásica, monolítica: el browser pide, el servidor **cocina un HTML completo** y lo manda, el browser lo pinta. ¿Defectos?

Que es **todo o nada**. Te llega un esqueleto grande, lleno de contenido, todo junto, ya renderizado — lo pegás y funciona. Pero ahora querés cambiar **una cosa**: un dato, un contador, el estado de un botón. ¿Qué opción tenés? Pedirle al servidor **todo el documento de nuevo**. El trabajo que ya habías hecho — descargar, parsear, construir los árboles de la Parte 4 — lo tirás y lo repetís de cero. No es modular: no podés pedir "solo la partecita que cambió". Y cada actualización paga el **round trip completo** (y ya sabés de la Parte 4 lo que cuesta cada ida y vuelta).

Peor todavía del lado del usuario: sin esto que viene, un `POST` significa quedarte **mirando la página congelada** mientras el servidor resuelve y te redirige a un documento nuevo. Cada interacción, una recarga. Cada recarga, una espera.

El diagnóstico es claro: tenemos un **cliente liviano** — el browser como terminal boba que solo sabe pintar lo que le mandan — desperdiciando todo lo que las Partes 2 y 3 mostraron que el browser puede hacer. La solución va a ser exactamente la opuesta: darle trabajo al cliente. Usar el browser como lo que es — **un cliente HTTP con todas las de la ley** — y hablarle a APIs en vez de pedir documentos.

---

## 2. 🔴 Ajax: pedir pedacitos

La pieza que cambió la historia se llama **Ajax** (*Asynchronous JavaScript + XML* — el nombre envejeció mal: hoy nadie usa XML, pero la sigla quedó). La idea: que **JavaScript**, desde adentro de la página, haga requests HTTP **sin recargar el documento**, espere la respuesta **asíncronamente** (el circuito exacto de la Parte 3: se delega, sigue el hilo, vuelve por la cola), y use el DOM para insertar el resultado donde corresponda.

### 2.1 La evolución de las formas

Antes de Ajax: cualquier update → **full-page refresh**. A partir de ahí, la comunidad fue puliendo la manera de escribir lo mismo — porque por detrás, todas estas formas **son lo mismo**: asincronismo sobre el runtime de la Parte 3.

**Primera generación: XMLHttpRequest** — el Ajax original, y un monumento a la incomodidad:

```javascript
// Se crea el objeto de request "a mano".
var xhr = new XMLHttpRequest();

// Se configura: verbo, URL, y true = asíncrono.
xhr.open("get", "/ajax?name=value", true);

// Un CALLBACK (Parte 3) que se dispara con cada cambio de estado del request:
xhr.onreadystatechange = function() {
  if (xhr.readyState == 4) {                            // 4 = "terminó"
    if (xhr.status == 200 || xhr.status == 304) {       // ¿status code OK?
      // La data viene en XML y hay que escarbarla nodo por nodo…
      var statusNode = xhr.responseXML.getElementsByTagName("status")[0],
        dataNode = xhr.responseXML.getElementsByTagName("data")[0];

      if (statusNode.firstChild.nodeValue == "ok") {
        handleSuccess(processData(dataNode))            // recién acá, el éxito
      } else {
        handleFailure()
      }
    } else {
      handleFailure()
    }
  }
};

// Y al final, se dispara. Todo lo anterior era preparación.
xhr.send(null);
```

Funciona — y es horrible: un evento con una función, adentro chequeás el estado, chequeás el status code, escarbás la respuesta, anidás los casos. Con dos o tres requests encadenadas, esto se vuelve una pirámide ilegible.

**Las generaciones siguientes**, cada una una abstracción sobre la anterior: **jQuery** (una library histórica que envolvió esto en callbacks más amables), **Promises** (el objeto-resultado-futuro de la Parte 3), y **async/await** — que es *syntactic sugar* sobre Promises (azúcar sintáctico: otra forma de escribir lo mismo, más rica). El mismo flujo de arriba, hoy:

```javascript
// La misma idea: pedir datos y manejar éxito/fracaso. Comparen la legibilidad.
async function handleSubmit () {
  try {
    // await: "esperá este resultado asíncrono" — sin bloquear el hilo (Parte 3):
    // mientras la respuesta no llega, el event loop sigue atendiendo la página.
    const user = await getUser("tylermcginnis");
    const weather = await getWeather(user.location);   // y encadenar es trivial

    handleSuccess({ user, weather });                  // éxito
  } catch (e) {
    handleFailure(e);                                  // cualquier fallo cae acá
  }
}

// Resultado esperado: mismo comportamiento que el XHR de arriba,
// en una fracción del código, legible de arriba a abajo.
```

### 2.2 Qué habilita Ajax (las dos puertas)

**Puerta 1 — modularizar el front.** Pensá el front dividido en cuadraditos. Ahora podés traerte **un cuadradito** — un pedazo de HTML — y con el DOM insertarlo donde va: un modal, un cartel, una publicidad dinámica. Ya no pedís la página; pedís piezas.

**Puerta 2 — la interesante: hablar JSON con una API.** En vez de pedir HTML, JavaScript le pega a una **API REST** (clase 01) que devuelve **JSON** — y el browser tiene herramientas nativas para manipular JSON. Así trabaja la enorme mayoría de los fronts hoy: se pide **un** HTML inicial, y de ahí en más todo lo que haya que traer, crear o modificar viaja como JSON contra la API. **La lógica de cómo pintar esos datos queda del lado del cliente.** Eso te empuja derecho al *cliente pesado* — y, como vas a ver, no hay ningún problema con eso.

Y una tercera consecuencia, más sutil y más poderosa: **el estado se vuelve dinámico**. Ejemplo: una app de post-its. El usuario crea uno. Vos, en el front, podés *mentirle*: mostrarle el post-it como creado **ya**, y encolar la request al servidor por atrás. ¿Sos temporalmente inconsistente con el backend? Sí. ¿Al usuario le importa? No — su post-it está ahí, la app se siente instantánea. Mientras el hilo principal siga interactivo (Parte 3), podés jugar con el estado como quieras: pedirlo de a partes, adelantarlo, corregirlo después. El estado dejó de ser algo que se pide todo-o-nada — y esa palabra, **estado**, se va a convertir en la protagonista del resto de la parte.

> **Para el parcial, si te preguntan** — *¿Qué es Ajax y qué cambió?*
> Ajax es la técnica de hacer requests HTTP desde JavaScript, asíncronamente, sin recargar el documento: la página pide datos (típicamente JSON a una API REST) y actualiza solo lo necesario vía DOM. Reemplazó el modelo de full-page refresh por actualizaciones parciales, habilitó mover la lógica de presentación al cliente y volvió el estado dinámico. Evolución de las formas: XMLHttpRequest → jQuery → Promises → async/await (todas, la misma asincronía por detrás).

---

## 3. 🟡 Los frameworks JS (y la guerra)

Ajax abrió la puerta y por esa puerta entró un problema. Si querés páginas interactivas de verdad, tenés que manipular el DOM todo el tiempo — y **usar el DOM a mano es un dolor** (la deuda de la Parte 2, cobrándose): cada cambio de datos te obliga a acordarte qué parte del árbol tocar, en qué orden, sin pisar lo demás. A escala de aplicación, inmanejable.

Necesitábamos ayuda. Y la ayuda llegó en forma de **frameworks JavaScript**, con una promesa doble: mejorar la **experiencia de usuario** — transiciones entre pantallas hechas en JS, sin volver a cargar documentos — y facilitar el **desarrollo** — menos tiempo de llegada al mercado, menos costo de mantenimiento. Traducido al idioma de la Parte 1: **$$$**, por las dos vías.

La promesa era tan jugosa que se desató una guerra por el mercado: **AngularJS** (hoy poco usado), **Vue**, más tarde **Svelte**, y decenas más — cada uno con su filosofía, peleando por el *market share* del frontend. ¿Cómo terminó? Con un ganador claro: **React le ganó al mercado**, y el resto viene atrás. Por eso lo que sigue se explica con React: no porque sea "el mejor" en abstracto, sino porque es **el adoptado por la industria** — el que te vas a cruzar laburando. (Y la aclaración honesta de rigor: entender React *a fondo* da para una cursada entera; el objetivo acá es que cuando abras su documentación, cada palabra te suene y sepas qué estás mirando.)

---

## 4. 🔴 React: la UI es una función del estado

### 4.1 Qué es (y qué es ese código raro)

React nace en Facebook, de un dolor real: no podían mantener sus UIs — demasiadas interacciones, demasiado estado, un desastre. Su respuesta fue un movimiento audaz: **meter el JavaScript en el HTML**. Así se veía React hace ~10 años:

```jsx
// Sintaxis VIEJA (clases) — hoy se escribe distinto, pero muestra la idea original:
class HelloMessage extends React.Component {
  render() {
    return (
      <div>
        Hello {this.props.name}    {/* ← ¡¿una variable ADENTRO del HTML?! */}
      </div>
    );
  }
}
```

Mirá todo lo que pasa ahí: hay un `<div>` — parece HTML — y adentro, entre llaves, **una variable**. Dinámica: la puedo modificar, puedo meter un contador, y lo que se ve cambia. Eso no es HTML ni es JavaScript puro: es **JSX**, la sintaxis de React que mezcla los dos. (Hoy React es orientado a funciones, no a clases — lo vas a ver en la sección 7 — pero el concepto es este desde el día uno.)

Pregunta obligada: **¿cómo hace el browser para renderizar esto?** No lo hace. **El browser no entiende React** — al día de la fecha, no sabe qué es esto. El browser recibe solamente HTML, CSS y JavaScript. Lo que pasa es lo mismo que con TypeScript en la Parte 3: cuando codeás en React, antes de servirlo hay un paso de **compilación** que convierte toda esta magia en HTML + CSS + JS planos. Se le manda todo eso al cliente una vez, el cliente lo interpreta con su maquinaria de siempre — y de ahí en adelante, hay *updates*.

### 4.2 El virtual DOM

¿Y cómo maneja esos updates? Acá está la pieza técnica central. React genera una estructura interna llamada **Virtual DOM**: una copia liviana del DOM, que se construye rápidamente **a partir del estado de los componentes**. El ciclo, a alto nivel:

```
   estado cambia
        │
        ▼
  [ Virtual DOM nuevo ]──┐
                         ├──► DIFF ──► "cambió SOLO este nodo" ──► toca el DOM real
  [ Virtual DOM actual ]─┘   (comparación)                         (vía Web APIs)
                                                                        │
                                                    DOM real:   ● ← rojo: se actualiza
                                                               ╱ ╲
                                                              ●   ●  ← verdes: ni se tocan
```

Cuando algo cambia en algún componente, React genera un virtual DOM nuevo, hace un **diff** contra el actual (compara los dos árboles y encuentra las diferencias), y actualiza **únicamente los componentes necesarios** del DOM real — usando, abajo de todo, las Web APIs de la Parte 2. ¿Por qué tomarse el trabajo de la copia? Porque tocar el DOM real es **caro** (cada cambio puede disparar el camino crítico de la Parte 4: layout, paint); comparar objetos JavaScript en memoria es barato. Mejor calcular en el barato y tocar el caro lo mínimo indispensable.

Para vos como desarrollador, el resultado es que los updates son *naturales*: cambiás el estado, y React se encarga de todo lo demás.

### 4.3 La filosofía: declarativo

Y eso decanta en la frase que resume a React — anotala, subrayala:

> **La UI es una función del estado.**   `UI = f(estado)`

Si cambio el estado, cambia la UI. Punto. No describo *cómo* actualizar la pantalla — describo *qué* debe verse para cada estado, y React resuelve el cómo. Eso es programar **declarativamente**, y el contraste con el mundo anterior lo muestra mejor que nada este duelo:

```javascript
// IMPERATIVO (jQuery, la vieja escuela): le decís al DOM QUÉ HACER, paso a paso.
var duel = $("#Duel");                 // agarrá este elemento…
duel.css("color", "red");              // …pintalo de rojo…
duel.html("<h1>Dark side</h1>");       // …y metele este HTML adentro.
```

```javascript
// DECLARATIVO (acá con Vue, mismo espíritu que React): declarás el ESTADO…
var duel = new Vue({
  el: '#Duel',
  data: {
    color: 'green',        // …y la vista REACCIONA sola a estos datos.
    title: 'Light side'    // Cambiás data → cambia lo que se ve. Nunca tocás el DOM.
  }
})
```

El lado oscuro y el lado luminoso, literalmente. En el mundo declarativo no tenés que saber qué es el DOM ni cómo se toca: te bajás React, laburás con estado y props, y *el tipo se encarga de todo*. Se vuelve muy atractivo, muy rápido.

> **Para el parcial, si te preguntan** — *¿Qué es el virtual DOM y para qué sirve?*
> Es una representación liviana del DOM que React mantiene en memoria, generada a partir del estado de los componentes. Ante un cambio de estado, React construye un virtual DOM nuevo, lo compara con el anterior (diff) y aplica al DOM real únicamente las diferencias. Así minimiza las manipulaciones del DOM real — que son caras porque disparan re-render — y permite programar declarativamente: la UI es una función del estado.

---

## 5. 🔴 La SPA: qué ganás, qué pagás

Una aplicación construida así — un solo HTML inicial y de ahí en más JavaScript manejando todo — se llama **SPA**: *Single Page Application*, aplicación de una sola página. Compará los dos ciclos de vida:

```
 CICLO TRADICIONAL                          CICLO SPA
 ┌────────┐  GET inicial      ┌────────┐   ┌────────┐  GET inicial (pesado) ┌────────┐
 │        │ ────────────────► │        │   │        │ ────────────────────► │        │
 │ Client │ ◄──── [doc] ───── │ Server │   │ Client │ ◄──── [doc React] ─── │ Server │
 │        │  Form POST        │        │   │        │       Ajax            │        │
 │        │ ────────────────► │        │   │        │ ────────────────────► │        │
 │        │ ◄──── [doc] ───── │        │   │        │ ◄──── [JSON] ──────── │        │
 └────────┘  otro documento   └────────┘   └────────┘  solo datos; el       └────────┘
             ENTERO, siempre                           cliente pinta (virtual DOM)
```

En el modelo tradicional, cada interacción devuelve un documento entero. En la SPA, hay **un** GET inicial más pesado — se lleva la aplicación completa — y después cada update es Ajax contra la API REST: viaja JSON, el cliente lo pinta con su virtual DOM, y listo.

**Ventajas** (bien aplicada y en el escenario correcto):
- **Transiciones fluidas** — navegar no recarga nada; *luce como una app*, no como una secuencia de documentos.
- **Requests livianas** dentro de la app — JSON en vez de documentos enteros.
- Navegación *seamless*, sin pantallas en blanco esperando cargas → **mejora la UX** (y la Parte 4 ya te enseñó a qué se traduce la UX: bounce rate, conversión, plata).

**Desventajas** — porque nada es gratis:
- **Se lleva pésimo con el SEO** (ya mismo lo desarmamos).
- **Manejo de History / Routing** — el botón "atrás" del browser deja de funcionar solo (sección 10).
- **Alta complejidad del lado del cliente** — el cliente quedó pesado, con muchísima lógica; eso hay que construirlo y mantenerlo.
- **La request inicial es pesada** — carga gran parte de la app: la library, la lógica, todo. Y la Parte 4 te contó exactamente qué significa una primera carga gorda (14 KB, round trips, LCP).

### El caso SEO: por qué la SPA es invisible

Primero, qué es **SEO** — *Search Engine Optimization*: el arte de que tu página **la encuentren por Google sin pagar**, "de manera orgánica". El mecanismo: Google tiene **robots** que recorren la web *scrapeando* (leyendo automáticamente) páginas para **indexarlas** y asignarles prioridad. Si tu página está "bien hecha" según sus criterios, te bonifica posiciones; si no, te penaliza — y nadie te encuentra. Y como posicionar se traduce directo en ventas, hay una industria entera dedicada a gamificar el algoritmo… y Google respondiendo con cambios de algoritmo constantes (con la IA y los LLMs scrapeando todo, más que nunca).

Ahora, el choque. Antes, el robot visitaba tu URL y recibía **HTML puro, estático, ya escrito**: contenido de sobra para indexar. ¿Qué recibe cuando visita una SPA?

```html
<!DOCTYPE html>
<html>
  <head><meta charset="UTF-8" /></head>
  <body>
    <div id="app"></div>          <!-- …¿un div vacío? -->
    <script src="app.js"></script> <!-- "la página está adentro de este JS, jurado" -->
  </body>
</html>
```

Un cachito de HTML **chiquitito** — un par de tags, un div vacío — y la referencia a un `app.js` gigante donde vive absolutamente toda la página. Los robots, que históricamente entienden HTML, se encuentran con… nada que indexar. Tu contenido existe, pero solo *después* de ejecutar JavaScript. Resultado: SEO por el piso. (¿Tiene solución? Varias — y son exactamente el tema que abre la Parte 6.)

> **Para el parcial, si te preguntan** — *Ventajas y desventajas de una SPA.*
> Ventajas: transiciones fluidas sin recarga (experiencia de app), requests internas livianas (JSON contra una API en vez de documentos), mejor UX en el escenario correcto. Desventajas: mala relación con el SEO (el HTML inicial es casi vacío y el contenido vive en JS, que los robots indexan mal), el manejo de historia/routing no funciona out-of-the-box, alta complejidad del lado del cliente, y una request inicial pesada que carga gran parte de la aplicación.

---

## 6. 🔴 Componentes: piezas que se componen

Volvamos adentro de React, que hasta ahora es una caja negra "declarativa". Su unidad de construcción es el **componente**: tu propia pieza de UI, con su **encapsulamiento** y sus **responsabilidades**, hecha para **reutilizarse**. ¿Ves una lista de resultados? Probablemente sea *un* componente renderizado N veces. ¿Un bloque destacado? Otro componente. La pantalla entera se piensa como piezas:

```
   Piezas (componentes)              Pantalla armada
   ① barra superior          ┌──────────────────────┐
   ② card de producto        │ ①①①①①①①①①①①①①①①①  │
   ③ botón de acción     ──► │ ②②   ②②   ②②        │
   ④ bloque de texto         │ ④④④  ③   ④④④  ③    │
                             └──────────────────────┘
```

¿Y cómo se organizan las piezas? Adiviná: **en árbol** (el cuarto de la clase — HTML, DOM, AST, y ahora este):

```
                <LoginPage />
              ┌───────┼─────────┐
        <Header />  <LoginForm />  <Footer />
         ┌────┴────┐
     <Logo />   <Menu />
```

Cada componente puede contener otros; la página es la raíz. Dos datos de evolución: **antes los componentes eran clases** (como el `HelloMessage` de la sección 4), **hoy son funciones** — React se volvió mucho más funcional con los años (y también más *full stack*, pero esa historia es de la Parte 6). Y la gran virtud del modelo es que es **composable**: componentes que contienen componentes que contienen componentes — esa lógica linda de componer que asociás con un buen backend, funcionando en el front. Se puede tener un front prolijo *y* pesado; no hay contradicción.

---

## 7. 🔴 Props y estado: por dónde viaja la información

Si la UI es una función del estado, la pregunta del millón es: **¿dónde vive el estado y cómo circula por el árbol?** Dos mecanismos, complementarios.

### 7.1 Props: de padre a hijo, siempre para abajo

Las **props** (de *properties*) son la información que un componente padre le pasa a sus hijos. Imaginate un objeto lleno de cosas, del que cada hijo toma la parte que necesita:

```
                    │ props
                    ▼
              [ Componente ]
          ┌─────────┼──────────┐
          │ props   │ props    │ props
          ▼         ▼          ▼
    [ Comp. ]   [ Comp. ]   [ Comp. ]
        │ props                 │ props
        ▼                       ▼
    [ Comp. ]               [ Comp. ]
```

La flecha va **siempre de arriba hacia abajo** — de padre a hijo, nunca al revés. Ejemplo concreto: un navbar (padre) con sus ítems (hijos componentes): a cada ítem le pasa por props qué título mostrar, qué texto, a qué `href` redirigir cuando lo cliqueen. El hijo botón sabe qué hacer *gracias a lo que el padre le pasó*.

**Aclaración importante, porque la intuición de la carrera tira para otro lado: esto NO es herencia.** El padre no le está transmitiendo comportamiento al hijo, ni el hijo "extiende" al padre. Le está pasando **valores** — un dato, o incluso una función — como quien pasa argumentos. Composición, no herencia. (Existe un patrón donde un componente recibe *otros componentes* por props y decide qué hacer con ellos — pero eso es composición elevada al cubo, no herencia tampoco.)

Y dos propiedades de contrato: las props son **read-only** — el hijo las recibe y **no puede modificarlas**. Si algo tiene que poder cambiar, eso no es una prop: es estado.

> 🕳️ **Madriguera — Higher-Order Components**
> El patrón de "componentes que reciben componentes y los envuelven" tiene nombre (HOC) y toda una literatura: componentes-fábrica que le agregan capacidades a otros. Fue muy dominante en el React de la era de clases; los hooks lo desplazaron bastante.
> *Volvé al camino — esto se profundiza aparte, otro día.*

### 7.2 State: el valor que persiste (y quién lo toca)

El **estado** (*state*) es el dato **propio de un componente**, que puede cambiar en el tiempo. Pensalo por analogía con Java: como un atributo de una clase, con su **setter** — un método específico para modificarlo. Contrato espejo del de las props: el estado **sí se modifica**, pero **solo mediante su setter, y solo donde fue declarado**.

| | **Props** | **State** |
|---|---|---|
| ¿Quién lo define? | El padre (lo pasa) | El propio componente |
| ¿Se puede modificar? | **No** — read-only | **Sí** — vía su setter |
| ¿Para qué sirve? | Configurar al hijo desde afuera | Datos propios que cambian con el tiempo |

En el React moderno (funciones), el estado se declara con `useState`:

```jsx
function ReactHooksExample() {
  // useState devuelve un ARRAY de dos cosas:
  //   [ el valor actual , la función para cambiarlo (el "setter") ]
  // y le pasás el valor inicial. Acá: el texto arranca en 'Click button'.
  const [buttonText, setButtonText] = useState('Click button');

  return (
    // JSX: parece HTML, pero las llaves inyectan JavaScript.
    // onClick: cuando el usuario cliquea, se ejecuta el setter con el valor nuevo.
    <button onClick={() => setButtonText('Thank you')}>
      {buttonText}   {/* ← acá se renderiza el valor ACTUAL del estado */}
    </button>
  );

  // Resultado esperado: ves un botón que dice "Click button";
  // lo tocás, y pasa a decir "Thank you". Sin recargar nada, sin tocar el DOM:
  // cambió el estado → React re-renderizó. UI = f(estado), en vivo.
}
```

Un detalle finísimo que explica por qué esto funciona: **los componentes son funciones que se re-ejecutan constantemente** — cada cambio de props o de estado vuelve a correr la función entera para recalcular qué renderizar. ¿Y entonces cómo no se pierde el valor del estado en cada re-ejecución? Porque `useState` es **persistente**: React guarda ese valor *por afuera* de la función, y en cada re-ejecución te devuelve el actual. La función es efímera; su estado, no.

### 7.3 Controlled components: el padre es el dueño

Ahora, la jugada que combina todo — el patrón que la industria llama *controlled components*. Situación: un contador. Un componente muestra el valor, otro componente es el botón que suma. ¿Dónde vive el estado?

**En el padre de ambos.** El padre es el dueño del valor **y** del método que lo cambia; a sus hijos les reparte, por props, lo que cada uno necesita:

```jsx
// EL PADRE: dueño del estado Y de la función que lo modifica.
function Contador() {
  const [cuenta, setCuenta] = useState(0);   // el valor vive ACÁ

  return (
    <div>
      {/* Al hijo que MUESTRA, le paso el valor (una prop-dato). */}
      <Visor valor={cuenta} />

      {/* Al hijo que ACCIONA, le paso una FUNCIÓN (una prop-función):
          en JavaScript las funciones se pasan como cualquier valor. */}
      <Boton alSumar={() => setCuenta(cuenta + 1)} />
    </div>
  );
}

// HIJO 1: solo renderiza lo que le dieron. No sabe de dónde sale.
function Visor({ valor }) {
  return <h1>{valor}</h1>;
}

// HIJO 2: solo ejecuta lo que le dieron cuando lo cliquean.
// No puede tocar el estado directamente — no es suyo.
function Boton({ alSumar }) {
  return <button onClick={alSumar}>+1</button>;
}

// Resultado esperado: un número (arranca en 0) y un botón "+1".
// Cada click: el botón ejecuta la prop-función → el setter corre EN el padre
// → cambia el estado → el padre se re-renderiza → el Visor muestra el nuevo valor.
```

¿Por qué el baile de pasar la función? Porque **el estado solo puede modificarse donde se declaró**. El botón no tiene acceso al estado del padre; lo único que puede hacer es ejecutar la función que el padre le prestó. Y ¿por qué el estado va en el padre y no en el botón? Porque el padre necesita **repartirlo entre varios hijos** — si el valor viviera adentro del botón, el Visor no podría verlo. Regla práctica: el estado vive en el ancestro común más cercano de todos los que lo necesitan.

> **Para el parcial, si te preguntan** — *¿Qué diferencia hay entre props y state?*
> Las props son valores que un padre le pasa a sus hijos, fluyen solo hacia abajo y son read-only: el hijo no puede modificarlas (y no son herencia — se pasan valores y funciones, no comportamiento). El state es el dato propio de un componente, declarado con `useState`, que persiste entre re-renders y solo se modifica mediante su setter, en el componente que lo declaró. Para que un hijo dispare un cambio, el padre le pasa la función modificadora como prop (controlled components).

---

## 8. 🔴 Del prop drilling al patrón Flux

### 8.1 El antipatrón

El modelo "todo baja por props" tiene un modo de falla famoso. Imaginate: el estado vive arriba de todo, y lo necesita un componente **cinco niveles más abajo**. ¿Qué hacés? Se lo pasás al hijo, que se lo pasa a su hijo, que se lo pasa a su hijo… y **los del medio no hacen nada con esa prop** más que cascadearla. Eso es el antipatrón **prop drilling** — "taladrar" props a través de capas que no las usan:

```
   SIN Context (prop drilling)              CON Context
        [ App ]● estado                        [ App ]● estado
           │ prop ↓                        ┌ ─ ─ ─ ─ ─ ─ ─ ─ ┐
        [ Page ]  ← no la usa, la pasa       Context.Provider
           │ prop ↓                        │  [ Page ]        │
        [ Panel ] ← no la usa, la pasa        [ Panel ]         (nadie del medio
           │ prop ↓                        │  [ Boton ]◄──────┼─ se entera)
        [ Boton ] ← POR FIN la usa          ─ ─ ─ ─ ─ ─ ─ ─ ┘   consume directo
```

### 8.2 React Context

La solución nativa: **React Context**. *Wrappeás* (envolvés) el subárbol con un proveedor de contexto, y cualquier descendiente — hijo, nieto, tataranieto — puede **consumir ese estado directamente**, sin que las capas intermedias lo transporten. El caso de manual es el **theme**: el modo claro/oscuro es un dato global que vive arriba de todo, y el botoncito que lo cambia está enterrado en lo profundo del árbol — pasarlo por diez niveles de props sería una locura; con Context, el botón lo consume y listo.

Con una letra chica: cuando el valor del contexto cambia, se re-evalúan el componente y sus hijos — Context **no evita** que los componentes intermedios del árbol se re-rendericen. Para ese ajuste fino existen otros mecanismos (Redux, entre otros, lo previene).

### 8.3 El patrón Flux

Y acá se abre la historia grande. React tardó en tener Context; en el medio, la comunidad parió **80.000 libraries de manejo de estado** — Redux la más famosa, y sigue vigente (en el TP probablemente usen alguna de esta familia). La clave: **todas se basan en el mismo patrón**, y ese patrón hay que conocerlo. Se llama **Flux**, y en criollo es esto: un **objeto maestro** — el **store** — donde vive un montón de estado, y cada componente lo consulta directamente, sin cadenas de props:

```
              ┌──────────────────────────────────┐
              ▼                                  │
   [ ACTION ] ──► [ STORE ] ──► [ VIEW ] ──► (interacción del usuario)
   "pasó algo"    el objeto      consulta        dispara otra action…
   que cambia     maestro con    el store y
   el estado      TODO el        se renderiza
                  estado         con eso
```

El loop completo: la **view** consulta al store ("dame las cosas") y se pinta en función de eso. El usuario interactúa → se dispara una **action** (un evento que describe qué pasó) → la action **cambia el store** → cambió el estado → la view **se re-renderiza** para reflejarlo → y vuelta a empezar. ¿Te suena la música? Es `UI = f(estado)` elevado a arquitectura: **lo que manda es lo que hay en el store.** Flux es superador de las props para estado global, sí — pero mirá sus implementaciones y vas a encontrar constraints y ceremonia propios: no es gratis tampoco.

> **Para el parcial, si te preguntan** — *¿Qué es el prop drilling y cómo se soluciona? ¿Qué es el patrón Flux?*
> Prop drilling es el antipatrón de pasar una prop a través de componentes intermedios que no la usan, solo para que llegue a un descendiente profundo. Se soluciona con React Context (un proveedor envuelve el subárbol y cualquier descendiente consume el estado directamente) o con libraries de manejo de estado global. Estas se basan en el patrón Flux: un store central con el estado, views que lo consultan para renderizarse, y actions que lo modifican — cada cambio del store re-renderiza las views: la UI como función del estado, a escala de aplicación.

---

## 9. 🟡 Hooks: la caja de herramientas

Los **hooks** son funciones especiales de React — las reconocés por el prefijo `use` — que le "enganchan" capacidades a los componentes de función. Nacieron para simplificar un mundo anterior: cuando los componentes eran clases, tenían **lifecycle methods** — métodos del ciclo de vida: "ejecutá esto antes de montarte en el DOM", "esto después de montarte", "esto al desmontarte". Los hooks reemplazaron esa ceremonia. El mapa:

```
   Manejar estado ┐                        ┌ Traer datos (fetch)
   Toggle de valor├── [useState]   [useEffect]──┤ Reaccionar a cambios
   Guardar input  ┘                        └ Tareas de limpieza

   Compartir estado┐                       ┌ Estado complejo
   Theme           ├── [useContext] [useReducer]──┤ Lógica de estado
   Config global   ┘                       └ Manejo de forms

                    [useCallback] · [useMemo]
                     memoizar funciones · valores
                     (optimización de performance)
```

- **`useState`** — ya lo conocés de la sección 7: estado local persistente.
- **`useEffect`** — ejecuta código en momentos del ciclo de vida. El uso típico: **se ejecuta cuando el componente se monta en el DOM**, y ahí es donde hacés el fetch inicial — traés del servidor la info a renderizar y la guardás en un estado. También sirve para el final de la vida: desuscribirse de eventos y limpiar al desmontar (*clean up*). ⚠️ Nota de época: el uso de `useEffect` para traer datos está en discusión activa en la comunidad — hay una corriente fuerte demostrando que casi nunca hace falta. Conocelo, porque está en todos lados; sabé que el viento sopla hacia otras soluciones.
- **`useContext`** — el consumo del Context de la sección 8: compartir estado, theme, config global sin drilling.
- **`useReducer`** — otra forma de manejar estado, pensada para **estado complejo**: mucha lógica, formularios grandes, transiciones elaboradas.
- **`useCallback` y `useMemo`** — la pareja de **memoización** (recordar un resultado para no recalcularlo). ¿Por qué existen? Por lo que viste en 7.2: el componente es una función que se **re-ejecuta entera** ante cada cambio — y con él, se re-crean las funciones que declara adentro y se re-hacen sus cálculos. Si una función o un cálculo son caros, eso duele. `useCallback` memoiza **funciones** y `useMemo` memoiza **valores**: persisten entre re-ejecuciones, como el estado. (Se controlan con un *dependency array* — la lista de valores cuyos cambios invalidan lo memoizado.)

> 🕳️ **Madriguera — Memoización fina**
> Cuándo memoizar y cuándo no, cómo llenar bien el dependency array, qué se rompe si lo llenás mal: es un tema con profundidad propia y opiniones fuertes. Para esta clase alcanza el concepto: son la válvula de escape para el costo de re-ejecutar.
> *Volvé al camino — esto se profundiza aparte, otro día.*

Si esto te resulta abstracto, está bien: son los conceptos más avanzados de la parte, y el objetivo es el de siempre — que cuando la documentación de React te diga "useEffect", ya lo tengas escuchado y sepas en qué cajón va.

---

## 10. 🟡 Routing: recuperar el botón "atrás"

Última cuenta pendiente de la lista de desventajas de la sección 5. Los browsers regalan una facilidad histórica: **navegar hacia atrás y adelante en la historia**. Ese mecanismo es viejo y piensa en documentos: cada página visitada, una entrada en la historia. ¿Qué pasa en una SPA? Que documento hay **uno solo** — las "pantallas" son transiciones hechas con Ajax y estado. Resultado: el botón atrás no funciona *out of the box*; navegás diez pantallas, tocás atrás, y te vas de la aplicación entera.

La solución vino por librerías que usan la **History API** del browser (otra Web API: manipular la historia de navegación programáticamente). La canónica en React: **React Router**, que *simula* la navegación — cada "pantalla" registra su entrada en la historia, con URLs propias, y con su hook de historia (`useHistory`) podés ir y volver sin problemas. Declarativo, como todo en React: rutas que dicen qué componente renderizar para cada path:

```jsx
<Router>
  <div>
    <Header />   {/* el Header queda fijo; lo que rota es la ruta activa */}

    {/* "si el path es exactamente /, renderizá Home": */}
    <Route exact path="/"       component={Home} />
    <Route       path="/about"  component={About} />
    <Route       path="/topics" component={Topics} />
  </div>
</Router>
```

(Aviso de terreno: React Router lleva **muchas** versiones encima y la API cambió varias veces — el concepto es estable, la sintaxis exacta se chequea contra la versión que uses.)

---

## Checkpoint — Parte 5

*(Sin respuestas a propósito: recuperación activa. Las respuestas van al complemento.)*

1. ¿Cuál es el defecto estructural del modelo "pedir documentos" y qué le pasa al usuario en cada interacción?
2. ¿Qué es Ajax y por qué su nombre envejeció mal? Ordená la evolución de sus formas.
3. ¿Qué dos "puertas" abre Ajax? ¿Qué rol pasa a jugar el JSON?
4. Explicá el truco del post-it: ¿qué es la UI optimista y qué la hace posible?
5. ¿Qué prometían los frameworks JS, en términos de UX y de negocio? ¿Cómo terminó la guerra?
6. ¿Qué es JSX y por qué el browser jamás lo ve?
7. Describí el ciclo del virtual DOM ante un cambio de estado. ¿Por qué conviene la copia?
8. "La UI es una función del estado": explicalo, y contrastá imperativo vs. declarativo con un ejemplo.
9. Ventajas y desventajas de una SPA — las cuatro y las cuatro.
10. ¿Qué es el SEO, cómo indexan los robots, y qué ve exactamente un robot cuando visita una SPA?
11. ¿Por qué pasar props NO es herencia?
12. Props vs. state: quién los define, quién puede cambiarlos, para qué sirve cada uno.
13. En el contador: ¿por qué el estado vive en el padre, y por qué al botón se le pasa una función?
14. Los componentes se re-ejecutan constantemente. ¿Por qué el estado no se pierde?
15. ¿Qué es el prop drilling, qué lo soluciona, y qué limitación tiene Context igual?
16. Dibujá (mentalmente alcanza) el loop del patrón Flux: store, view, action. ¿Qué frase de React realiza a escala de arquitectura?
17. ¿Para qué sirve useEffect, y qué nota de época hay que tener presente?
18. ¿Qué problema resuelven useCallback/useMemo, y de qué comportamiento de los componentes nace ese problema?
19. ¿Por qué el botón "atrás" se rompe en una SPA y cómo lo arregla React Router?

---

## Lo que viene — Parte 6

La SPA dejó dos cabos sueltos gordos: un robot de Google mirando un div vacío, y una primera carga pesada. La Parte 6 cierra la clase con el estado del arte: las **estrategias de rendering** — CSR, SSR, SSG, la combinación incremental de Next.js y los server components de Remix, con sus líneas de tiempo comparadas — más tres cajas de herramientas que tu TP va a necesitar sí o sí: **tiempo real** (long polling, WebSockets, SSE — tus notificaciones), **UX y CSS tooling** (mobile first, Tailwind, shadcn), **module bundlers** (el "bundle" que venimos nombrando desde la Parte 4, por fin explicado) y el clásico que hace llorar entregas: **CORS**. Cierra el anexo de RPC/RMI.

**FIN DE LA PARTE 5**
