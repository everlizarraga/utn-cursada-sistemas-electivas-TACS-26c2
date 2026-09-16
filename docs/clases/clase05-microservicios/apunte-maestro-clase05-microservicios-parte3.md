# 📘 Apunte maestro — Clase 05 — Microservicios
## Parte 3 de 6 — La arquitectura refactorizada

> **Unidad:** `clase05` · TACS 2C 2026 · Clase del 08/09/2026 · Apunte maestro, Parte 3 de 6 · Leyenda e índice completo en la Parte 1.

**Qué cubre esta parte.** El mapa completo del e-commerce después de N iteraciones, y todo lo que se aprende mirándolo: qué gana cada equipo con su ciclo de vida propio, qué le cuesta a la organización, por qué un servicio es cross a todos, por qué cada uno tiene su base, por qué se integra solo por API, qué pasa cuando aparece un segundo front. Después, tres temas que salen del mapa y que son preguntas de entrevista: versionar una API ("APIs are forever"), autenticar servicios entre sí (mTLS), y las llamadas circulares y el N+1.

**Qué NO cubre.** Qué pasa cuando una caja del mapa falla y cómo se contiene el daño: eso es la Parte 4.

**De dónde venís.** Parte 2: la migración iteración por iteración, el contrato como única forma de hablar entre servicios, y la regla de que solo la app dueña conoce su base.

---

## 1. El mapa 🔴

Este diagrama es el objeto de estudio de esta parte y de la siguiente. Leelo completo antes de seguir: **cajas** rectangulares son aplicaciones; **cilindros** (marcados `(DB)`) son bases de datos; cada **flecha** es una llamada, y va **del que llama al que responde** (HTTP entre aplicaciones; acceso directo cuando apunta a un cilindro). Todo lo del dibujo está adentro de la frontera del sistema, salvo dos cosas. Afuera quedan el **usuario**, a la izquierda, y los **servicios externos de catálogo** (los proveedores de terceros), a la derecha.

```
                                             ┌──────────┐         ┌──────────┐
                                             │ Pol.Com. │(DB)     │ Catálogo │(DB)
                                             └────▲─────┘         └────▲─────┘
                                                  │                    │
                                           ┌──────┴──────┐        ┌───┴──────┐     ┌────────────────┐
   o      ┌────────┐     ┌────────────┐    │             │        │          │     │   Servicios    │
  /|\ ───►│ Front  │────►│ Destacados │───►│  Políticas  │───────►│  Datos   │────►│  externos de   │
  / \     └─┬──┬─┬─┘     └──────────┬─┘    │ Comerciales │        └──────────┘     │    catálogo    │
usuario     │  │ │      ┌──────────┐│   ┌─►│             │                         └────────────────┘
            │  │ └─────►│ Facetado ││   │  └──────┬──────┘
            │  │        └───┬────┬─┘│   │         │
            │  │            │    └──┼───┘         │
            ▼  │            │       │             │
  ┌───────────┐│            │       │             │
  │ Header y  ││            │       │             │
  │  footer   ││            │       │             │
  └─────┬─────┘│            │       │             │
        ▼      ▼            ▼       ▼             ▼
  ┌────────────────────────────────────────────────────┐
  │                  Sesión y usuario                  │
  └───────────────┬─────────────────────────┬──────────┘
                  ▼                         ▼
            ┌──────────┐              ┌──────────┐       ┌──────────┐
            │ Usuarios │(DB)          │   P13N   │──────►│   P13N   │(DB)
            └──────────┘              └──────────┘       └──────────┘
```

*(El único cruce sin conexión está marcado con ┼: la bajada de Destacados a Sesión y usuario cruza la ida de Facetado a Políticas Comerciales. La lista de abajo es la que manda.)*

**Las 8 aplicaciones** (son las responsabilidades de la Parte 2, §1, cada una convertida en caja):

| Aplicación | Responsabilidad |
|---|---|
| **Front** | Sirve la página; punto de entrada del usuario. |
| **Destacados** | Productos resaltados. |
| **Facetado** | Filtros y categorías. |
| **Header y footer** | Login/sesión, carrito, contenido estático, newsletter. |
| **Políticas Comerciales** | Markup, promociones, restricciones. |
| **Datos** | El acceso al catálogo: lee su base propia y los servicios externos de catálogo. |
| **Sesión y usuario** | Quién es el usuario, qué sesión tiene. |
| **P13N** | Personalización (P13N es una abreviatura tipo "i18n": P + 13 letras + N). |

**Las 4 bases:** Pol.Com. (de Políticas Comerciales), Catálogo (de Datos), Usuarios (de Sesión y usuario) y P13N (de P13N).

**Las 18 llamadas** (quién llama a quién):

| Desde | Hacia |
|---|---|
| Usuario | Front |
| Front | Destacados · Facetado · Header y footer · Sesión y usuario |
| Header y footer | Sesión y usuario |
| Destacados | Políticas Comerciales · Sesión y usuario |
| Facetado | Políticas Comerciales · Sesión y usuario |
| Políticas Comerciales | Pol.Com. (DB) · Datos · Sesión y usuario |
| Datos | Catálogo (DB) · Servicios externos de catálogo |
| Sesión y usuario | Usuarios (DB) · P13N |
| P13N | P13N (DB) |

Fijate en un patrón: **ninguna aplicación llama a una base que no sea la suya.** Políticas Comerciales necesita productos, y no va a la base Catálogo: va a Datos. Esa es la regla del contrato de la Parte 2, ahora aplicada en todo el sistema (se retoma en 3.5).

---

## 2. Lo crítico: cada caja tiene su ciclo de vida 🔴

Antes de mirar las cajas una por una, lo más importante del mapa no se ve en el dibujo: **cada caja va a tener su ciclo de vida.** Hay un equipo que maneja el Front y otro que maneja Políticas Comerciales, y cada uno decide cuándo buildea, cuándo testea, cuándo deploya.

Eso es la ventaja más grande de microservicios, y es exactamente la respuesta a los problemas de la Parte 1. Compará:

| Antes (monolito) | Ahora (por caja) |
|---|---|
| Un deploy unificado, sacrificado, con un técnico de cada parte a las 4 AM | **Cada parte se puede deployar todos los días** |
| Un binario gigante | Un binario **más liviano** |
| 30 personas tocando el mismo artefacto | El equipo tiene **ownership** (propiedad y responsabilidad) sobre su pieza |
| Fronteras difusas entre módulos | **Un contrato que hay que mantener siempre** |

Se gana **muchísimo en agilidad**. Pero con dos condiciones y un costo que no se ven en el mapa:

- **Condición 1: scope suficiente.** Cada caja tiene que tener suficiente responsabilidad como para justificar un equipo y un ciclo de vida (lo contrario, el "servicio anémico", está en la Parte 5).
- **Condición 2: gente suficiente.** Ocho ciclos de vida necesitan ocho equipos, o al menos ocho responsables claros.
- **El costo: necesito un equipo de infraestructura.** En el monolito, si se caía la aplicación, **todos nos ocupábamos de levantarla**. Con ocho cajas, ¿quién vela por la comunicación entre servicios? ¿Quién mira si se cayó una? ¿Quién resuelve la **observabilidad** (poder ver qué está pasando adentro del sistema a partir de sus logs, métricas y trazas; tiene su clase propia, la 10)? Tiene que haber una solución **cross**, que no pertenece a ningún equipo de features. Eso es gente, y es costo.

Anotalo desde ya: **microservicios no es una bala de plata.** Cada ventaja de esta parte tiene su desventaja en la Parte 5.

> **Para el parcial, si te preguntan: ¿cuál es la ventaja principal de una arquitectura de microservicios?**
> Que cada servicio tiene su propio ciclo de vida: el equipo dueño lo puede buildear, testear y deployar de forma independiente, todos los días si hace falta, con un binario más liviano y ownership claro, siempre que mantenga el contrato que expone. Se pasa de un deploy unificado y sacrificado a deploys chicos y frecuentes. A cambio, se necesita scope y gente suficientes por servicio, y un equipo de infraestructura que resuelva de forma transversal la comunicación, el monitoreo y las caídas.

---

## 3. Observaciones sobre el mapa 🔴

Ahora sí, caja por caja. Cada observación enseña una regla general de microservicios usando el ejemplo.

### 3.1 Sesión y usuario es cross a todo

Mirá cuántas flechas llegan a **Sesión y usuario**: desde Front, Header y footer, Destacados, Facetado y Políticas Comerciales. Es la caja más llamada del sistema.

La lectura correcta **no** es "los servicios se comunican con el front a través de usuario". Es al revés: **cada servicio llama a Sesión y usuario para sacar cosas que complementan su respuesta.** Políticas Comerciales necesita saber *de qué usuario* se trata: quizás tiene que buscar un ID, quizás un flag (¿es VIP?, ¿es nuevo?), quizás un segmento. Y en ese momento ya depende del servicio de usuarios. Y así con todas.

La consecuencia es obvia y hay que decirla igual: **si se cae Sesión y usuario, se cae todo.** Ese es el punto de mirar el mapa: encontrar las cajas de las que dependen todas las demás, porque son las que más te duelen. Cómo se contiene ese daño es el tema de la Parte 4.

### 3.2 P13N: la personalización

En el mapa, P13N recibe una sola flecha (desde Sesión y usuario) y tiene su base. Pero lo que **no** se ve es que seguramente hay atrás un montón de procesos (de tipo analytics, cron jobs, dumps) que **ingestan** datos de todos lados para construir la personalización, y que a la vez P13N va a ser invocada desde un montón de lugares: para recomendaciones, para settings, para el perfil del usuario. Es una caja con muchas más implicancias de las que muestra el dibujo.

**¿Por qué se separa de Usuarios?** No necesariamente hay que separarla; acá está hecho así. La razón por la que tiene sentido: personalización puede hacer un montón de cosas que **no son responsabilidad del usuario**.

Caso real de un sitio muy conocido de viajes: un visitante entra **sin loguearse** y, para comprar, mete una tarjeta de crédito. Las tarjetas tienen categoría: Black, Gold, Platinum. El servicio de personalización miraba eso y, en base a la categoría de la tarjeta, **recomendaba viajes** más o menos caros. Es una decisión de negocio que se toma sobre **la sesión** (una IP, una tarjeta), no sobre un usuario que ni siquiera existe todavía. Eso no le corresponde al servicio de usuarios; le corresponde a uno que tiene otras responsabilidades. Contraejemplo: el carrito de compras es muy de sesión/usuario; la categoría de la tarjeta, no tanto.

La regla general: **la separación de cajas sigue a la separación de responsabilidades**, no al parecido de los datos. Y cuando la responsabilidad no está clara, aparece uno de los problemas de la Parte 5.

### 3.3 Bases de datos: muchas y distintas

Cuatro cilindros, cuatro bases. **Tengo muchas DB diferentes, y eso me permite diferentes esquemas y tecnologías.** Las puedo mezclar todas, si quiero (hasta cierto punto; teóricamente puedo). Con los modelos de la clase 04:

- Políticas Comerciales puede ser **documental**.
- Catálogo puede ser **documental**.
- P13N puede tener un **grafo** adentro, o algo tipo Dynamo / **wide column**.
- Usuarios puede ser **relacional**.

No tengo que tener una tecnología única de persistencia. Y como cada base está detrás de una API, **a los demás no les importa**: Políticas Comerciales no sabe ni necesita saber qué motor usa Datos. Esto resuelve el problema 4.10 de la Parte 1.

### 3.4 Cantidad de aplicaciones: ocho

**Tengo 8 aplicaciones, todas requieren monitoreo, y cada una tiene su ciclo de vida propio.** Lo del ciclo de vida ya está en la sección 2. Lo del monitoreo es un requisito nuevo: soy un equipo, quiero levantar una aplicación, y **me tienen que funcionar un montón de cosas *out of the box* desde el día uno** (monitoreo, logs, alertas, la autenticación de la sección 5). Antes eso no pasaba: había una aplicación y se la miraba a mano. Con ocho, hacen falta herramientas y estándares (Parte 5, §6.4 y Parte 6, §3).

### 3.5 Integraciones por WS: solo la app dueña conoce su DB

**WS** (*web services*): servicios expuestos por la red, vía HTTP; en la práctica, APIs REST.

- **Cada app expone servicios.**
- **No hay acceso vía capa de datos.**
- **Solo la app dueña conoce a su DB.**

Idealmente no hay integraciones que no sean por API (REST o la que se elija). Es la regla del contrato de la Parte 2, elevada a norma del sistema entero. Toda excepción es temporal y con fecha de vencimiento (Parte 2, §6.3), y la Parte 5 la lista entre las malas prácticas.

### 3.6 Aprovechar diferentes apps: el front mobile, y el BFF

Ventaja: **es más fácil utilizar concerns de diferentes apps desde otras.** Tengo el Front web; si mañana tengo que hacer una **app mobile**, la pego a mis mismas APIs: la app mobile llama a Destacados y a Facetado directamente, y **no repito código**.

```
   ┌────────────┐
   │ App mobile │─ ─ ─ ─ ─ ┬ ─ ►  Destacados
   └────────────┘          └ ─ ►  Facetado
   ┌────────────┐
   │   Front    │────────────►  Destacados, Facetado, Header y footer, Sesión y usuario
   └────────────┘
```

⚠️ Esto **tiene su parte buena y su parte mala**, aunque suele presentarse solo como buena. Lo que termina pasando en la realidad cuando tenés más de un front es que **hacés un BFF** (*Backend For Frontend*): una caja nueva por cada tipo de cliente (web, mobile) que arma la respuesta que ese cliente necesita. Es una especie de API Gateway (Parte 2, §5), pero **con lógica de negocio**: agrega, recorta y adapta lo que devuelven los servicios de atrás. Así que "reutilizar es cierto, hasta ahí": las APIs se reutilizan, pero la capa que las combina para cada front hay que escribirla igual.

---

## 4. Contratos: "APIs are forever" 🔴

Pregunta natural mirando el mapa: si necesito **cambiar la interfaz** entre dos cajas, ¿quién se encarga? ¿Los dos equipos? ¿Alguien dedicado?

Es el problema clásico del **versionado de APIs**, y hay una frase que lo resume: **"APIs are forever."** Yo hago una API, y la API que expongo es la API que **me comprometo a dar**. Mis clientes construyeron sobre ella.

### 4.1 "Agregar un campo no rompe nada": a veces

Hay quien dice: "podés agregar un campo a una respuesta y no pasa nada, porque solo estás agregando". **A veces.** Depende de cómo parsea el cliente:

- Si el cliente usa **Jackson** (la library de Java que serializa y deserializa JSON) en modo estricto, un campo desconocido en la respuesta **es un error**.
- Si el cliente hace **parseo binario** y lee "el siguiente campo está en el byte N", agregar un campo corre todos los offsets y rompe la lectura.

Entonces, agregar un campo puede romper la API del otro. **No es tan lineal.**

### 4.2 Cómo se versiona

Cuando tengo que versionar una API, tengo que hacer algo donde **mantenga lo viejo y dé la ruta nueva**. La estrategia común: la versión en la URL, `/v1/...` y `/v2/...`. Las dos rutas funcionan al mismo tiempo, y **yo tengo que ir migrando a mis clientes** a la nueva.

La responsabilidad es **compartida**, pero el peso cae del lado del servicio:

1. Yo, servicio, **ofrezco las dos rutas** y las dos funcionan.
2. Miro en la herramienta de monitoreo (New Relic, Datadog, Grafana; Parte 6, §3.3) que **vos empieces a consumir la nueva** y que te funcione.
3. Cuando nadie más consume `/v1`, la apago.

Balance: lo **bueno** es que a mí no me importan los internos del otro servicio, ni a él los míos. Lo **malo** es que si cambio el contrato, **tengo que estar muy consciente de mis clientes**: quiénes son, qué consumen, cuándo migran. Esto es lo que hace que cambiar una API sea engorroso (Parte 5).

### 4.3 El problema de los IDs

Un caso que parece un detalle y rompe mucho. **Un servicio referencia IDs de otro servicio** y los guarda en su propia base: Destacados guarda "el producto con ID 4521 es destacado". Ahora Datos, dueño de los productos, decide migrar de una base relacional con **IDs autoincrementales** (1, 2, 3, …) a MongoDB, que **no tiene autoincremental**: usa **UUIDs** (identificadores únicos generados, del estilo `550e8400-e29b-…`). ¿Qué pasó con el 4521 que guardó Destacados? Se rompió.

Dos salidas:

- **Usar un ID de dominio**: un identificador que sea del negocio (el SKU del producto, por ejemplo) y no de la base, para no "comerte" en tu servicio una abstracción de la base de datos de otro.
- **O declarar en el contrato** que los IDs son autoincrementales, y entonces el dueño ya no es libre de cambiarlos.

Son **decisiones muy blandas**, que no parecen técnicas cuando las tomás, y después las cosas se rompen de un montón de formas. El contrato es más que la lista de endpoints: incluye lo que tus clientes asumen de tus datos.

> **Para el parcial, si te preguntan: ¿qué significa "APIs are forever" y cómo se cambia una API que otros consumen?**
> Que la API que expongo es un compromiso con mis clientes, que construyeron sobre ella; incluso agregar un campo puede romperlos, según cómo parseen. Para cambiarla, mantengo la versión vieja y expongo la nueva (por ejemplo `/v1` y `/v2`), migro a los clientes gradualmente verificando con monitoreo que consumen la nueva, y apago la vieja cuando nadie la usa. La responsabilidad es compartida, pero el servicio tiene que conocer a sus clientes.

> **Para el parcial, si te preguntan: un servicio guarda IDs de otro servicio. ¿Qué riesgo hay y cómo se evita?**
> Que el dueño de esos IDs cambie de tecnología de base (de autoincremental a UUID, por ejemplo) y los IDs guardados dejen de tener sentido. Se evita usando IDs de dominio (del negocio, no de la base) o dejando explícito en el contrato el tipo de ID, de modo que el dueño no lo cambie sin versionar.

---

## 5. Autenticación entre servicios 🟡

Otra pregunta que sale del mapa: ¿tiene que haber **autenticación entre servicios**? ¿Datos tiene que verificar que quien lo llama es Políticas Comerciales y no cualquiera?

La respuesta honesta: **sí, tendría que haber**, aunque hay demasiados lugares donde no se hace. Y puede ser de dos niveles: **a nivel infraestructura** o **a nivel negocio**.

### 5.1 mTLS: a nivel infraestructura

Cuando consumís un servicio HTTP y la URL tiene la **S** de `https`, eso es HTTP con el protocolo **TLS** (*Transport Layer Security*). ¿Qué es TLS, brevemente? Una forma de usar **certificados** para saber que **el otro es quien dice ser**. Para poder hacerlo, confiamos en una **autoridad certificante** (un tercero) que firma los certificados; cada uno tiene el suyo, y valida los de los demás en base a esa firma.

**mTLS** (*mutual TLS*) es lo mismo pero **en los dos sentidos**: no solo el cliente verifica al servidor, sino que **el servidor verifica al cliente**. Analogía: cada servicio tiene un **DNI** (su certificado). Al consumir a otro, se presenta con su DNI; el otro chequea contra su lista de DNIs válidos: si está, pasa a consumir; si no, lo rechaza directamente.

```
   Políticas Comerciales                          Datos
   ┌───────────────────┐                   ┌───────────────────┐
   │  certificado PC   │── "soy PC" ──────►│ ¿PC está en mi    │
   │                   │                   │  lista de válidos?│
   │                   │◄── "soy Datos" ───│  certificado D    │
   │ ¿Datos es válido? │                   │                   │
   └───────────────────┘                   └───────────────────┘
          ambos validan al otro contra la autoridad certificante
```

Es un estándar de la industria, muy usado sobre todo en la comunicación **entre proveedores** (servicio de una empresa con servicio de otra), aunque adentro de las empresas a veces no se use.

### 5.2 Quién lo hace: no vos

El punto que conecta con la sección 2: **los equipos de features no van a estar haciendo esto.** Si trabajás en Facetado, no vas a preguntarte cómo está hecho el mTLS. Tiene que haber un **equipo de infraestructura** y un **equipo de seguridad** que lo hagan por atrás: que **renueven el certificado automáticamente**, que se cercioren de que cuando tu request sale, lleva lo que tiene que llevar. Vos estás **completamente abstraído**.

Es decir: **aumentan los costos, aumenta la cantidad de gente, aumenta la complejidad.** De nuevo, la misma cuenta que en la sección 2.

> 🕳️ **Madriguera — Service Mesh**
> La forma moderna de abstraer mTLS, reintentos y timeouts de las aplicaciones es un *service mesh*: una capa de red que se mete entre los servicios y resuelve eso sin tocar el código de cada uno. Tiene su clase propia, la 13.
> *Volvé al camino — esto se profundiza aparte, otro día.*

### 5.3 A nivel negocio: tokens y permisos

Además de (o en vez de) la infra, puedo tener autenticaciones **más de negocio**: con **tokens** (un valor que el llamador presenta y el servicio valida), un flujo de **OAuth** entre más de un servicio (un protocolo de autorización delegada), **permisos granulares** (este servicio puede leer productos pero no modificarlos). Se puede hacer un montón.

La realidad de la mayoría de los casos es "pegarle a la URL y listo". Pero estas cosas **empiezan a pasar en empresas que escalan**. Y ahí comparás con el monolito: ¿podía tener permisos? Sí, pero **estábamos todos en el mismo proceso**. Cuando en el medio hay **una capa de red**, todo se empieza a complicar.

> **Para el parcial, si te preguntan: ¿qué es mTLS y para qué se usa entre microservicios?**
> Es TLS mutuo: cada servicio se presenta con su certificado, firmado por una autoridad certificante de confianza, y el que recibe la llamada valida que el llamador es quien dice ser antes de responder (el cliente también valida al servidor). Se usa para autenticar la comunicación servicio a servicio a nivel infraestructura, y lo resuelven equipos de infra y seguridad de forma transparente para los equipos de features.

---

## 6. Llamadas circulares y N+1 🟡

En la Parte 1 (§4.6) viste que en un monolito el módulo A que llama a B y B que llama a A es un bucle que estás **obligado a resolver**. En microservicios te lo dejan pasar: el servicio B llama a A, y A después llama a B, y nada te frena. **Es un dolor tremendo.**

Peor todavía, en microservicios y en REST en general aparece el **problema N+1**: te pido una lista de N usuarios (una llamada), y para cada usuario quiero hacer algo, que implica llamar a otro servicio (N llamadas), que después **vuelve a invocar a usuarios**, y usuarios me llama a mí… Puede armarse una **cadena de llamadas desastrosa**, y se ha visto.

```
   Yo ──► Usuarios: "dame la lista"          (1 llamada)
   Yo ──► Otro: "procesá el usuario 1"       ┐
   Yo ──► Otro: "procesá el usuario 2"       │ N llamadas
   ...                                       │
   Yo ──► Otro: "procesá el usuario N"       ┘
                 └──► Usuarios ──► Yo ──► ...   ← y si hay un bucle, algo raro hay
```

Hay que tener cuidado con **cómo se diseñan los flujos**. Una llamada que dispara otra es normal; **si hay un bucle, algo pasa, algo raro hay** en el reparto de responsabilidades (3.2). Y todo ese overhead de red es una de las desventajas de la Parte 5.

---

## Checkpoint — Parte 3

*(Sin respuestas: van al complemento.)*

1. Mirando el mapa, ¿qué regla se cumple con las cuatro bases de datos y qué habilita?
2. ¿Qué gana un equipo con el "ciclo de vida propio" y qué le exige a cambio (dos condiciones y un costo)?
3. ¿Por qué Sesión y usuario es la caja más peligrosa del mapa? ¿Qué lectura incorrecta hay que evitar sobre ella?
4. Con el caso de la tarjeta de crédito, explicá por qué la personalización no es responsabilidad del servicio de usuarios.
5. ¿Qué significa "solo la app dueña conoce su DB" y qué pasa si dos apps comparten una base?
6. ¿Por qué "reutilizar servicios desde un front mobile" es una verdad a medias? ¿Qué es un BFF?
7. "Agregar un campo a una respuesta no rompe nada." Da dos casos donde sí rompe.
8. Describí el procedimiento para pasar una API de v1 a v2 sin romper a los clientes.
9. Un servicio guarda IDs autoincrementales de otro. ¿Qué puede pasar y qué es un ID de dominio?
10. ¿Qué diferencia hay entre TLS y mTLS? ¿Quién lo implementa en una organización con microservicios?
11. ¿Qué es el problema N+1 y por qué con microservicios puede terminar en un bucle entre servicios?

---

## Qué viene en la Parte 4

Qué pasa cuando una caja del mapa falla o se pone lenta, y cómo se evita que arrastre a las demás: timeouts, rate limiting, circuit breaker, reintentos y colas. El caso completo de backpressure paso a paso, desde los proveedores externos hasta el usuario que aprieta F5. Y la otra forma de hablar entre servicios: colas de mensajes y asincronismo, con RabbitMQ y Kafka, la dead letter queue, y cuándo el asincronismo suma y cuándo complica.

---

**FIN DE LA PARTE 3 — Apunte maestro clase05 — Microservicios**
