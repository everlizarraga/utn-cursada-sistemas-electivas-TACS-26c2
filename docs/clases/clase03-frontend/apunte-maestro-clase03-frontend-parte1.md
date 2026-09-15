# TACS — Clase 03 · Arquitectura Web y Frontend
## Parte 1 — Del protocolo al browser

> **Sobre esta parte.** Cubre el terreno sobre el que se construye toda la clase: qué es un cliente HTTP y qué tiene de especial el browser, qué pasa exactamente entre que escribís una URL y ves la página, la pila de protocolos donde vive la web, por qué respetar el estándar HTTP no es un capricho, qué significa "frontend" en la industria y por qué le importa al negocio. **No cubre** los lenguajes de la web (HTML, CSS, JavaScript y las Web APIs — Parte 2) ni nada de lo que pasa adentro del browser al ejecutar código (Parte 3 en adelante).
>
> **De dónde venís.** Se asume la clase 01: HTTP como protocolo, verbos, status codes, JSON y APIs REST. Todo lo demás se explica acá.
>
> **Convención 👁️ (vale para todas las partes de esta clase).** A veces una parte necesita mostrar código o pantallas de algo que se enseña recién en una parte posterior, porque es la evidencia del concepto que se está explicando. Esos bloques van marcados con `👁️ → Parte N`: son para **mirar**, no para entender línea por línea todavía. Si te generan más preguntas que respuestas, es normal — la parte indicada las responde.

---

## 1. 🟢 El mapa de la clase

Esta clase es distinta a la de Docker. Aquella era "lo necesario para el TP", con las manos en la terminal. Esta es **deliberadamente teórica y panorámica**: el objetivo no es que salgas sabiendo programar en React, sino que al terminar puedas abrir la documentación de cualquier framework de frontend y **saber qué estás mirando** — que las palabras te suenen, que sepas qué buscar y por qué existe cada pieza. Un curso práctico de React sería una materia aparte; acá se construye el mapa.

Que sea panorámica no significa superficial: el nivel de profundidad esperado es real. La historia que la clase recorre, y que estas partes siguen en orden, es más o menos así: la web de los 90 era simple y primitiva — pedías un documento, te llegaba, lo mirabas. Después vino la web 2.0 y la interactividad. Después las SPAs y los frameworks, que se pelearon a muerte por el mercado (las "frontend wars"). Y de esa guerra salió el panorama actual, que es un espacio de **estrategias** más que de ganadores. Cada eslabón de esa cadena existe porque el anterior se quedó corto en algo — y ese "algo" es lo que vas a ir viendo aparecer, dolor por dolor, a lo largo de las partes.

El recorrido completo de la clase 03:

| Parte | Tema | El dolor que resuelve |
|---|---|---|
| **1 (esta)** | Del protocolo al browser | ¿Sobre qué está parado todo? |
| **2** | HTML, CSS, JavaScript y las Web APIs | ¿Con qué se construye una página? |
| **3** | Adentro del browser: engine, runtime y event loop | ¿Cómo ejecuta JavaScript una máquina que no lo entiende? |
| **4** | Del texto al pixel: render, estándares y performance | ¿Por qué mi página tarda, y cómo lo mido? |
| **5** | Ajax, React y la SPA | ¿Cómo dejo de recargar el documento entero por cada cambio? |
| **6** | Estrategias de rendering, tiempo real, CSS tooling, bundlers y CORS | ¿Cómo sirvo, actualizo y empaqueto todo esto en serio? |
| **Anexo** | RPC, RMI y gRPC | Cierre pendiente del bloque de comunicación entre procesos |

---

## 2. 🟡 Clientes, servidor, y qué tiene de especial el browser

El punto de partida es el diagrama que ya viste mil veces en la carrera:

```
 [PC de escritorio] ─┐
                     │
 [Laptop] ───────────┼───( Internet )─────── [Servidor]
                     │
 [Teléfono] ─────────┘
      Clientes          request HTTP ──►
                        ◄── response HTTP
```

Los clientes le hablan al servidor por HTTP a través de Internet: mandan una **request**, reciben una **response**. Hasta acá, nada nuevo respecto de la clase 01. La pregunta interesante es otra: **¿qué cosas pueden ser ese cliente?**

Un **cliente HTTP** es cualquier programa capaz de armar una request HTTP, mandarla y procesar la response. La familia es más grande de lo que parece:

- **Los browsers**: Chrome, Firefox, Safari, Edge. *(Ojo con el vocabulario: browser/navegador, no "buscador" — el buscador es Google, que es un sitio al que llegás usando un navegador.)*
- **Herramientas de desarrollo**: **Postman** (el más conocido, con interfaz visual para armar requests) e **Insomnia** (misma idea, más nuevo y liviano, con menos ceremonia).
- **Herramientas de línea de comandos**: **curl** (transfiere datos por URL desde la terminal) y **wget** (similar, orientado a descargas). Detalle fino: la terminal en sí **no** es un cliente HTTP — es donde corren clientes como curl.
- **Los lenguajes de programación**: Ruby, Python, Java, Node.js, .NET… ninguno *es* un cliente HTTP, pero todos **traen uno de referencia** en su librería estándar. Cualquier programa que escribas puede hacer una request y, en ese acto, es un cliente HTTP.

Ahora, de toda esa familia, el browser es el bicho raro. Postman tiene interfaz, sí, pero el browser es el único que hace algo cualitativamente distinto con la response: **la renderiza**. curl te muestra el HTML como texto crudo; el browser lo convierte en una página que ves y tocás. Y alrededor de esa capacidad carga con un ecosistema entero de estándares y responsabilidades que ningún otro cliente tiene — lo vas a ver en detalle cuando lleguemos a las Web APIs (Parte 2) y al render (Parte 4).

Guardá esta idea, porque es la columna vertebral de la clase:

> **El browser = un cliente HTTP + render + estándares.** Todo lo que sigue en las seis partes es, en el fondo, desarmar ese "y mucho más".

> **Para el parcial, si te preguntan** — *¿Qué diferencia hay entre un browser y un cliente HTTP como curl?*
> Ambos pueden mandar requests HTTP y recibir responses, pero el browser además **renderiza** el contenido recibido (convierte HTML/CSS/JS en una página visual e interactiva) e implementa un conjunto de estándares y APIs (DOM, Fetch, CORS, etc.) que un cliente "pelado" no tiene. curl te devuelve el documento crudo; el browser te devuelve la experiencia.

---

## 3. 🔴 ¿Qué pasa cuando escribís una URL?

Es LA pregunta clásica — de la materia, y de las entrevistas. No para recitarla de memoria, sino para poder **intuir cada paso**: más adelante (Parte 4) vas a medir la performance de una página con un diagrama que tiene exactamente estas etapas, y si no sabés qué es cada una, el diagrama es ruido.

Primero, la anatomía de lo que escribiste:

```
   http://example.com/product/electric/phone
   └─┬─┘  └────┬────┘ └───────┬──────┘ └─┬─┘
  Scheme    Domain          Path      Resource

  Scheme   → el protocolo a usar (http, https)
  Domain   → el nombre del servidor al que le hablás
  Path     → la ruta dentro de ese servidor
  Resource → el recurso concreto que pedís
```

Y ahora el viaje completo, en seis pasos:

```
  (vos)
    │  ① Escribís la URL y das Enter
    ▼
 ┌─────────────┐
 │   BROWSER   │── ② ¿Tengo la IP de example.com en caché? ──► [DNS Cache]
 │             │        │ si no está…
 │             │── ②bis Consulta recursiva ──► [DNS Resolver] ──► [DNS Server]
 │             │                                    ◄── acá está: 93.184.216.34
 │             │
 │             │══ ③ Abre una conexión TCP contra esa IP ══════╗
 │             │── ④ Manda la request HTTP ────────────────────►║ ┌────────────┐
 │             │◄─────────────────────── ⑤ Vuelve la response ══╣ │ WEB SERVER │
 │  ⑥ RENDER  │                                                ╚═└────────────┘
 └─────────────┘
```

Paso por paso:

1. **Escribís la URL.** El browser la descompone en sus partes: ya sabe el protocolo, a quién hablarle y qué pedirle.
2. **Resolución de nombre.** `example.com` es un nombre para humanos; la red necesita una **IP** (la dirección numérica de la máquina). El browser mira primero su **caché de DNS** — si ya resolvió ese dominio hace poco, se ahorra el viaje. Si no lo tiene, dispara una **consulta DNS recursiva**: le pregunta a un resolver, que va preguntando en la jerarquía de servidores DNS hasta volver con la IP. Pensalo como tu agenda de contactos: vos conocés a la gente por nombre, pero para llamarla necesitás el número.
3. **Conexión TCP.** Con la IP en la mano, el browser abre una conexión TCP contra el servidor. En el medio pasan cosas de redes — la ruta que siguen los paquetes salta de router en router (*hops*) — pero lo esencial es el **handshake**: el apretón de manos que establece la conexión (siguiente sección).
4. **Request HTTP.** Recién con la conexión abierta viaja lo que vos realmente querías decir: la request (por ejemplo, `GET /product/electric/phone`).
5. **Response HTTP.** El servidor hace lo que tenga que hacer puertas adentro — para este viaje no importa cómo — y devuelve la response: status code, headers y el contenido.
6. **Render.** El browser toma ese contenido y lo **dibuja**. Este paso es el que lo separa del resto de los clientes HTTP, y es tan grande que tiene su propia parte más adelante (Parte 4).

> **Para el parcial, si te preguntan** — *¿Qué pasa cuando escribís una URL en el browser?*
> (1) El browser parsea la URL; (2) resuelve el dominio a una IP vía DNS, mirando primero su caché y si no con una consulta recursiva; (3) abre una conexión TCP contra el servidor (handshake); (4) envía la request HTTP; (5) recibe la response HTTP; (6) renderiza el contenido. Los pasos 2 a 5 son protocolos parados uno arriba del otro; el 6 es exclusivo del browser.

---

## 4. 🔴 El viaje visto por protocolo

El mismo viaje, ahora con la lupa en cada protocolo. Son cuatro actos, siempre en este orden:

### 4.1 DNS — de nombre a IP

```
 [DNS Client] ──── Recursive Query ────► [Local DNS Server]
              ◄─── Query Response ─────
```

Una pregunta ("¿qué IP tiene `example.com`?") y una respuesta. "Recursiva" significa que si el servidor DNS local no sabe la respuesta, **él** se encarga de ir a preguntarle a la jerarquía de DNS y volver con el dato — el cliente hace una sola pregunta y espera.

### 4.2 TCP — el apretón de manos

Antes de que viaje un solo byte de HTTP, cliente y servidor tienen que acordar que van a hablar. Eso es el **three-way handshake**:

```
 Cliente                              Servidor
    │ ───────────  SYN  ───────────►     │    "quiero abrir una conexión"
    │ ◄─────────  SYN/ACK  ──────────    │    "dale — ¿vos me recibís a mí?"
    │ ───────────  ACK  ───────────►     │    "sí. conexión abierta."
    │ ◄══════ canal establecido ══════►  │
```

Tres mensajes: `SYN` (synchronize: pido abrir), `SYN/ACK` (acepto y pido confirmación de vuelta), `ACK` (acknowledge: confirmado). Como una llamada telefónica: "¿Hola?" — "Hola, sí, te escucho, ¿me escuchás?" — "Sí, hablemos". Recién después de eso hay canal.

Este handshake va a volver a aparecer con nombre y apellido en la Parte 4, cuando midamos cuánto **cuesta** en tiempo — spoiler: cada ida y vuelta con el servidor se paga, y hay un límite famoso de cuánto entra en el primer viaje.

> 🕳️ **Madriguera — Redes por dentro**
> Hops, ruteo, control de congestión, retransmisiones: todo lo que pasa debajo de este handshake es el territorio de Comunicación de Datos y Redes de Datos en la carrera. Para esta materia alcanza con saber que TCP te da un canal **confiable y ordenado**, y que establecerlo cuesta ida-y-vueltas.
> *Volvé al camino — esto se profundiza aparte, otro día.*

### 4.3 HTTP — por fin, la conversación

Con el canal abierto, viaja lo que ya conocés de la clase 01:

```
 GET /doc HTTP/1.1  ─────────────────────►

 ◄─────────────────────  HTTP/1.1 200 OK
                         Last-Modified: date2
                         Etag: "xyz2"
```

Una request con verbo y recurso; una response con status code y headers. Los headers `Last-Modified` y `Etag` que se ven ahí identifican la versión del recurso — son la base del cacheo HTTP.

> 🕳️ **Madriguera — Caché HTTP**
> Con `Etag` y `Last-Modified` el browser puede preguntar "¿cambió esto desde la última vez?" y el servidor responder `304 Not Modified` sin reenviar nada — ahorro enorme de tráfico. El cacheo reaparece como etapa medible del viaje en la Parte 4, pero su mecánica fina (validadores, `Cache-Control`, expiración) da para una profundización propia.
> *Volvé al camino — esto se profundiza aparte, otro día.*

### 4.4 HTML — lo que llegó

¿Y qué es lo que vuelve en esa response cuando pediste una página? Un documento de texto con esta pinta:

```html
<!-- 👁️ → Parte 2: esto es evidencia de qué viaja en la response,
     no algo que necesites entender línea por línea todavía.
     El lenguaje HTML se explica completo en la Parte 2. -->
<!DOCTYPE html>
<html>
    <head>
        <title>Example</title>
        <link rel="stylesheet" href="style.css">
    </head>
    <body>
        <h1>
            <a href="/">Header</a>
        </h1>
        <nav>
            <a href="one/">One</a>
            <a href="two/">Two</a>
            <a href="three/">Three</a>
        </nav>
    </body>
</html>
```

Texto plano estructurado con etiquetas. El paso ⑥ del viaje — el render — consiste en convertir **esto** en una página visible. Cómo está armado este lenguaje es el arranque de la Parte 2; cómo el browser lo convierte en pixels es la Parte 4.

---

## 5. 🔴 La pila: dónde vive la web

Los cuatro actos de recién no flotan en el aire — son capas paradas una arriba de la otra. Este diagrama es el mapa de situación de toda la clase:

```
 ┌───────────┬───────────┬─────────────┐   ▲
 │           │           │  Web APIs   │   │
 │   HTML    │    CSS    ├─────────────┤   │
 │           │           │ JavaScript  │   │   LA WEB
 ├───────────┴───────────┴─────────────┤   │
 │                HTTP                 │   │
 ├──────┬──────────────────────┬───────┤   ▼
 │ DNS  │                      │  TLS  │
 ├──────┤         TCP          ├───────┘
 │ UDP  │                      │
 ├──────┴──────────────────────┴───────┐
 │                 IP                  │
 └─────────────────────────────────────┘
```

Leyéndolo de abajo hacia arriba:

- **IP** es la base: hace que un paquete llegue de una máquina a otra a través de la red.
- Sobre IP viven **TCP** (el canal confiable del handshake) y **UDP** (su hermano sin garantías: manda paquetes sin establecer conexión ni asegurar que lleguen — más rápido, menos seguro). **DNS** corre sobre esta capa, y **TLS** se monta sobre TCP para cifrar la conversación — es la S de HTTPS.
- Sobre todo eso, **HTTP**: el idioma de requests y responses de la clase 01.
- Y arriba de HTTP, las cuatro piezas con las que se construye lo que ves: **HTML** (estructura), **CSS** (apariencia), **JavaScript** (comportamiento) y ese cuarto cuadradito, **Web APIs**, que es el que casi todo el mundo omite cuando recita "la web es HTML, CSS y JS" — y que resulta ser la pieza que conecta a las otras tres con el browser. Las cuatro se desarrollan completas en la Parte 2.

**"La web"** es de HTTP para arriba. Todo lo de HTTP para abajo es la *foundation*: la infraestructura de redes sobre la que la web está construida (y que en la carrera se estudia en las materias de redes). **Frontend web = dominar la mitad de arriba del diagrama.**

> 🕳️ **Madriguera — TLS / HTTPS**
> TLS cifra el canal TCP: nadie en el medio puede leer ni modificar lo que viaja. Suma pasos propios al handshake (negociación de claves y certificados), o sea que también suma latencia al viaje de la sección 3. La criptografía detrás es un mundo — y una electiva entera de la carrera.
> *Volvé al camino — esto se profundiza aparte, otro día.*

---

## 6. 🟡 Respetar el estándar (o dos historias de terror)

HTTP no es solo un formato: es un **contrato**. Los status codes, los verbos y sus semánticas existen para que cualquier cliente pueda razonar sobre cualquier servidor sin conocerlo. ¿Y si un servidor decide no respetar el contrato? Técnicamente puede — el protocolo no explota. Pero mirá lo que pasa.

### Historia 1: el 200 mentiroso

Imaginate un servidor que responde **siempre** `200 OK`, y mete el resultado real adentro del body:

```
HTTP/1.1 200 OK              ← el sobre dice "todo bien"

{ "status code": 400,        ← la carta dice "salió todo mal"
  "detail": "Bad Request" }
```

Para el protocolo, eso fue un éxito. Para cualquier cliente, monitoreo, caché o herramienta estándar, **también** — porque todos leen el sobre, no la carta. El único que sabe que algo falló es quien parsea ese body con lógica a medida.

No es un ejemplo teórico: a Uber Eats le pasó en India, y el resultado fue **un fin de semana entero de comida gratis**. Su API en ese momento devolvía únicamente 200s; éxito, estado o error había que deducirlo parseando el mensaje del body. Cuando algo empezó a fallar en el flujo de cobro, ningún sistema lo detectó como error — porque, para HTTP, no lo era. Con status codes bien usados, detectar la falla hubiera sido trivial: cualquier dashboard muestra un pico de 4xx/5xx al instante.

La moraleja conecta directo con tu TP: los status codes no son decoración de la API — son **la interfaz de observabilidad más barata que existe**.

### Historia 2: "por favor no refresques la página"

La segunda historia es una pantalla real de un sistema de reservas:

> *"Please do not refresh this page as this may result in a duplicate booking."*

Traducción: "nuestro POST no es idempotente, y en vez de arreglarlo, te pasamos el problema a vos, usuario". La idempotencia — que repetir la misma operación produzca el mismo resultado que hacerla una vez (clase 01) — es exactamente la propiedad que falta acá. Un refresh reenvía la request; si el servidor no está preparado para reconocer la operación repetida, te cobra dos veces el pasaje.

Cuando una página te suplica que no toques nada, no estás viendo un mensaje de cortesía: estás viendo **una decisión de diseño rota que se volvió cartel**.

---

## 7. 🔴 ¿Qué cosa es frontend?

Ahora sí, la definición que da nombre a la clase. En la industria, **frontend** es todo sistema **de cara al usuario** (*user-facing*): la superficie con la que una persona interactúa con tu organización, tu programa, tu negocio.

Y eso es mucho más ancho que "una página web":

- **Aplicaciones web** servidas por HTTP — el caso de esta clase.
- **Aplicaciones de escritorio** — con tecnologías como Electron (apps de escritorio construidas con tecnología web: VS Code, Discord) o WPF (el framework de UI de escritorio de Microsoft/.NET).
- **Aplicaciones mobile** — nativas de iOS/Android, o con React Native (escribir la app con React y que corra como app nativa).
- **CLIs** — interfaces de línea de comandos, hoy de moda de nuevo por las herramientas de IA. Un agente al que le decís "comprame tal cosa en Mercado Libre" y va y lo hace **es un frontend**: es la cara por la que el usuario opera.
- **Terminales físicas**, clientes pesados, clientes livianos — todo lo que sea la punta visible del sistema.

O sea: frontend no implica pantalla linda con botones. Implica **usuario del otro lado**. El resto de la clase se concentra en el frontend **web** — la variante que se maneja con la mitad de arriba de la pila de la sección 5.

### Web vs. nativo: dos filosofías de fallo

Vale la pena un contraste, porque explica mucho de lo que viene después. El desarrollo nativo (Java en Android, por ejemplo, o Flutter) tiende a ser **rígido y todo-o-nada**: tipado más duro, y si algo falla, la app revienta y el error te aparece en la consola de desarrollo. La web es lo contrario: **está hecha para fallar con gracia** (*gracefully*). Podés tener media página rota y la otra media se muestra igual; un error de JS muchas veces ni te enterás. Se le critica — "los estándares son flojos", "se rompe todo en silencio" — y algo de razón hay, pero es en parte injusto: es una **decisión de diseño**. La web prioriza que *algo* se renderice y sea usable antes que la corrección formal absoluta. Porque en la web, el atributo rey es la **usabilidad**: si no tenés usabilidad, no tenés nada.

> **Para el parcial, si te preguntan** — *¿Qué es frontend?*
> Frontend es todo sistema de cara al usuario (user-facing): la superficie por la que una persona interactúa con el sistema. Puede ser una aplicación web, una app de escritorio o mobile, una CLI, una terminal física o incluso un agente. No se limita a interfaces gráficas web, aunque el frontend web (HTML/CSS/JS sobre HTTP) sea el caso más común.

---

## 8. 🟡 ¿Por qué ocuparse del frontend? (spoiler: plata)

Cerrando la parte, la pregunta de negocio. ¿Por qué dedicarle una clase — y gente, y plata — al frontend?

Porque **la métrica que le importa a tu organización pasa por ahí**. Ventas, conversión, usuarios registrados, retención: elijas la que elijas, el usuario la ejecuta a través del frontend. Sin front no convertís, no vendés, no sumás usuarios. Podés tener el backend más elegante del planeta, formalmente verificado, escrito en Haskell con programación funcional pura — si el usuario no puede comprar, **no vendés**, y todo lo demás es decoración cara.

Es la misma vara de siempre en esta materia: la tecnología es un medio; el fin es el valor que le llega al cliente y al negocio. El frontend es, literalmente, el lugar donde ese valor se cobra. Y esta idea vuelve con números concretos en la Parte 4: cada segundo que tu página tarda en cargar se traduce, medible y documentadamente, en usuarios que se van.

> **Para el parcial, si te preguntan** — *¿Por qué es importante el frontend para una organización?*
> Porque las métricas de negocio (conversión, ventas, adquisición y retención de usuarios) se concretan a través del frontend: es la interfaz donde el usuario ejecuta las acciones que generan valor. Una falla o mala experiencia ahí impacta directo en el negocio, sin importar la calidad del backend detrás.

---

## Checkpoint — Parte 1

*(Sin respuestas a propósito: son para recuperación activa. Las respuestas van al complemento de la unidad.)*

1. ¿Qué diferencia a un browser de un cliente HTTP "pelado" como curl?
2. Nombrá cuatro tipos de cliente HTTP que no sean browsers.
3. Enumerá, en orden, los seis pasos entre escribir una URL y ver la página.
4. ¿Por qué el browser consulta primero su caché de DNS? ¿Qué pasa si no encuentra el dominio ahí?
5. ¿Qué es el three-way handshake de TCP y en qué momento del viaje ocurre?
6. En la pila de la web: ¿dónde "empieza" la web y qué queda por debajo? ¿Qué rol cumplen TLS y UDP?
7. ¿Por qué responder `200 OK` con el error adentro del body es un problema? ¿Qué ilustra el caso de Uber Eats en India?
8. Una página te pide "no refresques o podrías duplicar tu reserva". ¿Qué propiedad le falta a ese sistema y por qué el cartel es un parche?
9. ¿Qué es frontend según la industria? Dá tres ejemplos que no sean páginas web.
10. ¿Por qué se dice que la web "está hecha para fallar gracefully", y qué atributo prioriza esa decisión?

---

## Lo que viene — Parte 2

La response llegó y trae texto: HTML. En la Parte 2 se abren los cuatro cuadraditos de arriba de la pila: **HTML** (la estructura y por qué sus etiquetas *hablan*), **CSS** (la apariencia y su regla de especificidad — incluida la forma más peligrosa de usarla), **JavaScript** (el comportamiento) y las **Web APIs** — el cuadradito que quedó pendiente en la sección 5, y sin el cual no se entiende nada de lo que el browser hace después.

**FIN DE LA PARTE 1**
