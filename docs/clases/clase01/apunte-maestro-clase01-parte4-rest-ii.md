# Apunte Maestro — Clase 01 · Parte 4: REST II — HATEOAS, stateless, escalabilidad y caching

**Unidad:** clase01 · **Parte 4 de 5**

Cierra REST con los tres principios que en la Parte 3 quedaron enunciados sin desarrollar. HATEOAS completa el Richardson Maturity Model; stateless y cache aware se abren completos, porque son los que explican cómo una API REST escala. Y termina con el locking optimista, que es caching aplicado a un problema de concurrencia.

**Leyenda de marcas:** 🔴 central / evaluable · 🟡 secundario · 🟢 mencionado al pasar.

---

## 1. Nivel 3 — HATEOAS 🟡

**HATEOAS — Hypermedia As The Engine Of Application State.** El cuarto y último escalón del Richardson Maturity Model.

La idea se entiende mejor con una analogía que con la sigla. Pensá cómo usás un sitio web: entrás a una página, y no necesitás que nadie te documente las URLs — **la página misma trae los links** a donde podés ir. Vas descubriendo el sitio navegándolo.

HATEOAS propone lo mismo para una API. Dos propiedades:

1. **Navegabilidad:** cada respuesta incluye los links a los recursos alcanzables desde ahí.
2. **Descubribilidad:** un cliente puede recorrer una API que no conoce, igual que navega una página web.

```json
{
    "content": {
        "price": 499.00,
        "name": "iPad",
        "links": [ {
            "rel": "self",
            "href": "http://localhost:8080/product/1"
        } ]
    },
    "links": [ {
        "rel": "product.search",
        "href": "http://localhost:8080/product/search"
    } ]
}
```

Cómo se lee: **`rel`** declara la *relación* del link con el recurso. `"self"` es la dirección de este mismo recurso — el producto sabe dónde vive. Y el bloque `links` de afuera ofrece `product.search`: desde este producto podés ir a buscar otros, **y el cliente lo descubre sin haber leído documentación alguna**. Eso es la descubribilidad.

Y el caso más simple, la paginación:

```json
{
  "hyperlink" :
  [
    "self" : "http://example.org/foo/bar",
    "next" : "http://example.org/foo/bar/2"
  ]
}
```

`self` es dónde estás; `next` es la página siguiente, **ya armada por el servidor**. El cliente no tiene que saber construir esa URL — y ese es justamente el caso en que HATEOAS sí se usa hoy.

### 1.1. Por qué casi nadie lo implementa

Existen **múltiples estándares** para expresar estos links: HAL, JSON API, JSON-LD, Collection+JSON, CPHL, Siren, Uber, Yahapi.

Ahí está buena parte de la explicación de su baja adopción: **la falta de un estándar único**. Es el fenómeno clásico de la proliferación de estándares — hay N formas de hacer lo mismo, alguien propone una que las unifique a todas, y ahora hay N+1. Sin un ganador claro, ningún cliente puede asumir un formato, y el beneficio de la descubribilidad se evapora.

> **En la práctica:** una API RESTful **rara vez llega al nivel 3**, pero es aconsejable y habitual que implemente el **nivel 2**. El uso concreto más común de HATEOAS hoy es **la paginación** — links `next` y `prev`, sobre todo con cursores, donde construir la URL siguiente no sería obvio para el cliente.

---

## 2. Stateless en profundidad 🔴

En la Parte 3 quedó enunciado: toda request debe traer toda la información necesaria para ser resuelta. Ahora el problema concreto que lo vuelve indispensable.

### 2.1. El problema

Tenés un **cluster** de N servidores detrás de un balanceador de carga. Un cliente se autentica contra el **Server I**, que guarda esa sesión en su memoria. Todo bien. Pero la request siguiente, por decisión del balanceador, cae en el **Server II**.

```
   CLIENTE                    SERVER I                  SERVER II
      │                          │                          │
      │──────── Auth ───────────►│                          │
      │                          │ [guarda la sesión         │
      │                          │  en SU memoria] ✓        │
      │                          │                          │
      │──────── Do Thing ───────►│ ✓ (te reconoce)          │
      │                          │                          │
      │──────── Do Thing ───────────────────────────────────►│
      │                          │                          │
      │                          │                    ¿¿ Y ESTE QUIÉN ES ??
      │                          │                    No tiene la sesión.
```

**¿Cómo sabe el Server II que el cliente está autenticado?** Esa es toda la pregunta.

### 2.2. Las respuestas que NO son stateless (y qué cuestan)

| Opción | Qué hace | Qué cuesta |
|---|---|---|
| **Sticky sessions** | Atar el cliente a un servidor: siempre le pega al mismo | **Rompe la elasticidad horizontal.** Y cuando un nodo se cae, rebalancear tiene costo alto: todos sus clientes pierden la sesión |
| **Session replication** | Replicar el estado de sesión entre todos los servidores | Costoso y complejo, y **escala peor cuantos más nodos tenés** (cada nodo nuevo hay que sincronizarlo con todos) |
| **Session en DB central** | Todos los servidores consultan una base común | Introduce un **SPOF** — *single point of failure*, un punto único de falla: si esa base se cae, se cae todo |

Las tres comparten el mismo vicio de fondo: **intentan que el servidor recuerde**. Y recordar, en un sistema distribuido, es caro.

### 2.3. La solución stateless

Darlo vuelta: **que el cliente traiga la prueba en cada request**. El mecanismo habitual es **JWT — JSON Web Token**: un token firmado por el servidor con un secreto, que contiene datos no sensibles del usuario.

```
   CLIENTE                    SERVER I                  SERVER II
      │                          │                          │
      │── Do Thing (+ token) ───►│ ✓ verifica la firma      │
      │                          │   y responde             │
      │                          │                          │
      │── Do Thing (+ token) ───────────────────────────────►│ ✓ verifica la firma
      │                          │                          │   y responde
      │                          │                          │
                          Ningún servidor guardó nada.
                    Cualquiera puede atender cualquier request.
```

El servidor **verifica el token en cada request** —comprueba que la firma sea válida— y no mantiene ningún estado conversacional. Como todos los servidores conocen el secreto, **cualquiera puede atender a cualquier cliente**. Agregar o quitar nodos deja de ser un problema.

> **Distinción que conviene tener clara: sesión vs cookie.** No son lo mismo ni están al mismo nivel. La **cookie** es un header HTTP, un mecanismo de **transporte**: un lugar donde guardar y mandar datos. La **sesión** es un patrón de **estado**: la idea de que el servidor recuerde quién sos entre requests. Podés transportar un token por cookie sin tener sesión, y ahí seguís siendo stateless.

---

## 3. Escalabilidad 🔴

Stateless no es una obsesión purista: es lo que habilita una forma concreta de crecer.

```
   ESCALADO VERTICAL                    ESCALADO HORIZONTAL
   (agrandar la instancia)              (agregar más instancias)

        ┌─────────────┐                  ┌───┐ ┌───┐ ┌───┐ ┌───┐
        │             │   ▲              │ ▪ │ │ ▪ │ │ ▪ │ │ ▪ │
        │      ▪      │   │              └───┘ └───┘ └───┘ └───┘
        │             │   │ más CPU
        └─────────────┘   │ más RAM      ┌───┐ ┌───┐ ┌───┐ ┌───┐
          ┌─────────┐     │              │ ▪ │ │ ▪ │ │ ▪ │ │ ▪ │
          │    ▪    │     │              └───┘ └───┘ └───┘ └───┘
          └─────────┘     │
           ┌───────┐      │              ┌───┐ ┌───┐ ┌───┐ ┌───┐
           │   ▪   │      │              │ ▪ │ │ ▪ │ │ ▪ │ │ ▪ │
           └───────┘      │              └───┘ └───┘ └───┘ └───┘

   Una máquina cada vez más grande       Muchas máquinas iguales
```

**Vertical** — agregar recursos a la misma máquina: más CPU, más RAM, hasta llegar a un mainframe.

- Tiene **límites teóricos**: la cantidad de hilos, la RAM máxima, y **rendimientos decrecientes** a partir de cierta escala (duplicar el hardware no duplica la capacidad).
- **No siempre es mala idea**, y esto es importante: los motores SQL, los mainframes bancarios y los sistemas donde distribuir sale carísimo escalan vertical con buenas razones. Es el criterio de la Parte 0 otra vez — depende del contexto, no hay una opción superior en abstracto.

**Horizontal** — agregar más máquinas.

- Lo **habilita el diseño stateless**: si ningún nodo guarda estado, sumar nodos es trivial. Con sticky sessions, no.
- También se apoya en la **inmutabilidad** y en esquemas tipo **Kubernetes (k8s)**, que orquestan asignando *millicores* a pods en lugar de máquinas virtuales enteras — granularidad mucho más fina.

> 🕳️ **Madriguera — Millicores y pods**
> Un *pod* es la unidad mínima que Kubernetes despliega, y un *millicore* es la milésima parte de un núcleo de CPU. Permiten pedir "medio núcleo" en vez de una máquina entera, y así empaquetar muchos servicios chicos en el mismo hardware.
> *Volvé al camino — Kubernetes tiene su propio espacio más adelante en la materia.*

---

## 4. Caching 🔴

El tercer principio pendiente. El caching de HTTP vive **en los headers**, de request y de response, y toda la infraestructura de internet ya sabe leerlos: por eso REST sobre HTTP lo obtiene casi gratis.

### 4.1. `Cache-Control`

La directiva principal. Le dice a las caches qué pueden hacer con esta respuesta.

| Directiva | Qué significa |
|---|---|
| `private` | Cacheable **solo en el cliente** (browser, app). Para contenido personalizado |
| `public` | Cacheable **en intermediarios** (CDN, proxies) y **reutilizable entre usuarios** |
| `max-age=N` | La respuesta es fresca por N segundos desde que se generó (`s-maxage` para caches públicas) |
| `no-cache` | Se **puede** cachear, pero hay que **revalidar** contra el origen antes de reutilizar |
| `no-store` | **No se guarda.** Ni siquiera se escribe en disco |
| `stale-while-revalidate` | Se puede servir una respuesta **vencida** mientras se revalida en segundo plano |

> ⚠️ **Cuidado con `public`.** Marcar como pública una respuesta que depende del header `Authorization` significa que un intermediario la va a **servir al usuario siguiente**. Es una de las formas más directas de filtrar datos privados.
>
> Y ojo con `no-cache`: **no** significa "no cachear" —eso es `no-store`—. Significa "cacheá, pero preguntá antes de usarlo".

Así funciona `max-age` en el tiempo:

```
  CLIENTE 1                    CACHE                      SERVIDOR
     │  GET /doc                 │  vacía → reenvía          │
     │──────────────────────────►│──────────────────────────►│
     │                           │  200 OK                   │
     │  200 OK                   │◄─ Cache-Control: max-age=100
     │◄──────────────────────────│  [guarda: /doc, age=0]    │
     │  Age: 0                   │                           │

  CLIENTE 2   (10 segundos después)
     │  GET /doc                 │  FRESCA (10 < 100)        │
     │──────────────────────────►│  devuelve lo guardado     │
     │  200 OK · Age: 10         │  ── no molesta al servidor ──
     │◄──────────────────────────│                           │

  CLIENTE 3   (100 segundos después)
     │  GET /doc                 │  VENCIDA (100 ≥ 100)      │
     │──────────────────────────►│  revalida contra origen   │
     │                           │──────────────────────────►│
     │  200 OK · Age: 0          │◄─ ¡sigue fresca!          │
     │◄──────────────────────────│  resetea age y devuelve   │
```

Fijate el Cliente 2: **el servidor nunca se enteró de que existió**. Eso es lo que compra el caching.

### 4.2. Validación condicional

Cuando la respuesta venció, en vez de retransmitirla entera se puede **preguntar si cambió**.

| Header | Qué dice |
|---|---|
| `If-Modified-Since` | "Devolveme el recurso **solo si cambió** desde esta fecha" |
| `ETag` | Identificador único (un hash) de **esta versión** del recurso. Lo devuelve el servidor |
| `If-None-Match` | "Si el ETag **coincide**, devolveme `304` y nos ahorramos el body" |
| `If-Match` | Para `PUT`/`PATCH`: "aplicá la modificación **solo si** el ETag coincide". Si no, `412` |

```
   ┌──────┐   GET /index.html                          ┌──────┐
   │  PC  │  ────────────────────────────────────────► │ SRV  │
   │      │                                            │      │
   │      │   200 OK                                   │      │
   │      │   ETag: "ba7816bf8f01fb4"    ◄──────────────      │
   └──────┘   [+ el documento entero]                  └──────┘

              (más tarde, el cliente quiere revalidar)

   ┌──────┐   GET /index.html                          ┌──────┐
   │  PC  │   If-None-Match: "ba7816bf8f01fb4"         │ SRV  │
   │      │  ────────────────────────────────────────► │      │
   │      │                                            │      │
   │      │   304 Not Modified            ◄─────────────      │
   └──────┘   [SIN BODY — usá tu copia]                └──────┘
```

> **Qué ahorra realmente un ETag, y qué no.** El servidor **siempre tiene que calcular el hash** del recurso: no nos ahorramos ni CPU ni la latencia de ese cómputo. Lo que se ahorra es **el envío del body por la red**. Para respuestas grandes el ahorro es sustancial; para respuestas chiquitas, casi nulo.
>
> Un ETag es esencialmente una función de hash —lleva un dominio grande a uno chico— y por lo tanto **admite colisiones**.

### 4.3. `Vary`

Le indica a las caches qué dimensiones, **más allá del método y la URL**, distinguen una respuesta de otra.

- `Vary: Accept` — negociación de contenido (JSON vs XML).
- `Vary: Accept-Encoding` — compresión (gzip vs brotli).
- `Vary: Authorization` — **clave**: impide servir contenido privado de un usuario a otro.
- `Vary: *` — la respuesta es incacheable.

```
  CLIENTE 1 (pide gzip)         CACHE                     SERVIDOR
     │  GET /doc                  │  vacía → reenvía         │
     │  Accept-Encoding: gzip     │─────────────────────────►│
     │───────────────────────────►│  200 OK                  │
     │◄───────────────────────────│◄─ Content-Encoding: gzip │
     │                            │   Vary: Accept-Encoding  │
     │              [guarda con clave: /doc + gzip]          │

  CLIENTE 2 (pide brotli)         │                          │
     │  GET /doc                  │  NO matchea la clave     │
     │  Accept-Encoding: br       │  → reenvía               │
     │───────────────────────────►│─────────────────────────►│
     │◄───────────────────────────│◄─ Content-Encoding: br   │
     │              [guarda también: /doc + br]              │

  CLIENTE 3 (pide brotli)         │                          │
     │  GET /doc                  │  ✓ MATCHEA               │
     │  Accept-Encoding: br       │  devuelve lo guardado    │
     │───────────────────────────►│  ── sin tocar el servidor ──
     │◄───────────────────────────│                          │
```

> **Para que algo sea cacheable de verdad, el valor tiene que repetirse entre requests.** Si la respuesta *varía* por usuario y no lo declaramos con `Vary`, la cache va a **servir contenido incorrecto** — el de otro.

---

## 5. Locking optimista con ETags 🔴

El mismo ETag que sirve para ahorrar ancho de banda resuelve un problema completamente distinto: **dos clientes que modifican el mismo recurso a la vez**.

El flujo:

```
  1. GET /users/12345
     ◄── ETag: "686897..."           el cliente se lleva la versión y su huella

  2. [el cliente modifica localmente]

  3. PUT /users/12345
     If-Match: "686897..."           "aplicá esto SOLO si sigue siendo esa versión"

  4. El servidor compara:
     ┌── coincide  ──► aplica el PUT ✓
     └── no coincide ─► 412 Precondition Failed
                        (alguien modificó el recurso en el medio)
```

Cuando llega un `412`, el cliente tiene dos caminos:

- **Sobrescribir a la fuerza**, ignorando el cambio ajeno (por ejemplo con `If-None-Match: *`).
- **Hacer un `GET` nuevo**, resolver el conflicto contra la versión actual, y reintentar el `PUT`.

Este patrón se llama **optimistic concurrency control** — control de concurrencia optimista. El nombre describe la apuesta: **asumimos que los conflictos son raros**, así que no bloqueamos nada por adelantado; simplemente los **detectamos** cuando ocurren y los resolvemos. Lo contrario sería el pesimista: bloquear el recurso mientras alguien lo edita, con todo el costo que eso implica.

---

## Para el parcial, si te preguntan

> **¿Por qué REST exige que el servidor sea stateless?**
>
> Porque permite escalar horizontalmente. Si el servidor guardara la sesión en su memoria, un cliente autenticado contra un nodo no sería reconocido al caer en otro, y habría que recurrir a sticky sessions (que rompen la elasticidad), replicación de sesión (costosa y que escala peor con más nodos) o una base central (que introduce un punto único de falla). Con un diseño stateless el cliente manda en cada request toda la información necesaria —típicamente un token firmado, como JWT— y cualquier nodo puede atender cualquier request.

> **Diferencia entre escalado vertical y horizontal.**
>
> Vertical es agregar recursos a la misma máquina (más CPU, más RAM); tiene límites teóricos y rendimientos decrecientes, aunque sigue siendo la opción correcta en casos como motores SQL o mainframes. Horizontal es agregar más máquinas, y lo habilita el diseño stateless junto con la inmutabilidad y orquestadores como Kubernetes.

> **¿Qué ahorra un ETag?**
>
> Ahorra el envío del body por la red, no cómputo: el servidor siempre tiene que calcular el hash del recurso para compararlo. Cuando el cliente manda `If-None-Match` y el ETag coincide, el servidor responde `304 Not Modified` sin body y el cliente reutiliza su copia cacheada. El ahorro es sustancial en respuestas grandes.

> **¿Qué es el locking optimista y cuándo conviene?**
>
> Es un control de concurrencia que asume que los conflictos son raros: en vez de bloquear el recurso, se detecta el conflicto al escribir. El cliente obtiene el ETag con un `GET` y lo reenvía en el `PUT` mediante `If-Match`; si el recurso cambió en el medio el ETag no coincide y el servidor responde `412 Precondition Failed`, ante lo cual el cliente puede sobrescribir forzadamente o releer, resolver y reintentar. Conviene cuando la contención real es baja, porque evita el costo de bloquear.

---

## Checkpoint

Respondelas sin volver al texto. Las respuestas van al complemento.

1. ¿Qué dos propiedades aporta HATEOAS, y por qué tiene baja adopción pese a ser el nivel 3 del modelo?
2. Describí el problema del cliente que se autentica contra un nodo de un cluster y cae en otro.
3. Nombrá las tres soluciones no-stateless y el costo específico de cada una.
4. ¿Qué diferencia hay entre una sesión y una cookie?
5. ¿Por qué el escalado vertical no siempre es la peor opción? Dá un caso.
6. ¿Qué relación hay entre el diseño stateless y el escalado horizontal?
7. Diferencia entre `no-cache` y `no-store`.
8. ¿Qué riesgo concreto tiene marcar como `public` una respuesta que depende de `Authorization`?
9. ¿Qué ahorra y qué NO ahorra un ETag? ¿Para qué sirve `Vary`?
10. Explicá el flujo completo del locking optimista, incluyendo qué código devuelve el servidor ante un conflicto y qué opciones tiene el cliente.

---

## Qué viene en la Parte 5

Con REST completo, la última parte muestra dónde se rompe. Primero, cómo **las APIs reales se alejan de REST puro** apenas hay que filtrar, paginar, ordenar o versionar — y qué problemas introduce eso. Después, las dos alternativas que tomaron caminos distintos: **GraphQL**, que rompe deliberadamente la semántica de REST y deja que el cliente pida la forma de la respuesta, y **gRPC**, que pega la vuelta completa y vuelve a RPC —el de la Parte 1— reversionado sobre HTTP/2 y Protocol Buffers.

---

**FIN DE LA PARTE 4**
