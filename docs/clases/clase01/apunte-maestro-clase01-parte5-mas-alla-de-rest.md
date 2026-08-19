# Apunte Maestro — Clase 01 · Parte 5: Más allá de REST — query params, GraphQL y gRPC

**Unidad:** clase01 · **Parte 5 de 5**

Cierra la unidad mostrando dónde REST toca su techo: qué pasa apenas hay que filtrar, paginar, ordenar o versionar, y las dos alternativas que salieron por caminos opuestos — GraphQL, que rompe la semántica de REST a propósito, y gRPC, que vuelve al RPC de la Parte 1 reversionado.

**Leyenda de marcas:** 🔴 central / evaluable · 🟡 secundario · 🟢 mencionado al pasar.

---

## 1. Alejándose de REST puro 🔴

En la práctica **las APIs reales rara vez son 100% RESTful**, y el techo se toca rápido. Basta una pregunta de diseño perfectamente normal:

> **¿Cómo armarías una API RESTful para listar los usuarios de un sistema filtrando por fecha de nacimiento, país y sexo?**

Pensala un segundo antes de seguir. Con lo visto en la Parte 3, un filtro debería ser… ¿un recurso? ¿un subrecurso?

### 1.1. La respuesta purista, y por qué no sobrevive

Si insistís en que todo sea un recurso identificado por su path, tenés que crear **una ruta por cada combinación posible de filtros**:

```
/users

/users/birth_date/{birth_date}
/users/country/{country}
/users/gender/{gender}

/users/birth_date/{birth_date}/country/{country}
/users/birth_date/{birth_date}/gender/{gender}

/users/country/{country}/gender/{gender}
/users/country/{country}/birth_date/{birth_date}

/users/gender/{gender}/country/{country}
/users/gender/{gender}/birth_date/{birth_date}

/users/birth_date/{birth_date}/country/{country}/gender/{gender}
/users/birth_date/{birth_date}/gender/{gender}/country/{country}
/users/country/{country}/gender/{gender}/birth_date/{birth_date}
/users/country/{country}/birth_date/{birth_date}/gender/{gender}
/users/gender/{gender}/country/{country}/birth_date/{birth_date}
/users/gender/{gender}/birth_date/{birth_date}/country/{country}
```

**Dieciséis rutas.** Y contra todas ellas:

```
/users?birth_date=1995&country=argentina&gender=female
```

**Una.** Ese contraste es todo el argumento. Y notá que con tres filtros ya son dieciséis; con un cuarto la cosa explota, porque hay que contemplar cada subconjunto de filtros y además cada orden posible dentro de cada subconjunto.

**La conclusión es incómoda pero honesta: los query params no son REST** —la URL deja de identificar un recurso y pasa a describir una consulta— **y aun así son lo que se usa, siempre.** Es un caso de manual del criterio de la Parte 0: la pureza no es un valor en sí, el valor está en la solución que sirve.

### 1.2. Los cuatro usos de los query params

**Filtrado y combinación:**

```
GET /users?country=AR&gender=F
```

**Selección de campos** — pedir solo lo que se necesita:

```
GET /users
{
  "users": [
    {
      "name": "Cosme",
      "lastname": "Fulanito",
      "birth_date": "1955-10-10",
      "gender": "male",
      ...                              // vienen TODOS los campos
    }
  ]
}

GET /users?fields=birth_date,gender    // ⚠️ 'fields' declara QUÉ CAMPOS querés,
                                       //    no filtra QUÉ usuarios. Son cosas distintas:
                                       //    para filtrar, el param va suelto (?gender=male).
{
  "users": [
    {
      "birth_date": "1955-10-10",      // solo los campos pedidos
      "gender": "male"
    }
  ]
}
```

**Paginación:**

```
GET /users?limit=10&offset=33          // desde el registro 33, traeme 10
GET /users?page=10&pageSize=5          // la página 10, de 5 en 5
GET /users?page=2&size=50              // o con cursores, en vez de números de página
```

**Ordenamiento:**

```
GET /users?orderBy=birth_date
GET /users?orderBy=birth_date&order=dec
GET /users?sort=createdAt:desc
GET /users?$sort[birth_date]=-1
```

Fijate que **cada API inventa su propia sintaxis**. No hay estándar. Eso nos lleva directo a los problemas.

### 1.3. Los problemas que introducimos

**Cache aware.** Cada combinación de query params genera **una URL distinta**, y por lo tanto **una entrada de cache distinta**. El cache se fragmenta: en vez de tener un `/users` muy caliente que se sirve a todos, tenés cientos de variantes frías. Es el principio de la Parte 4 degradándose por el uso.

**Documentación obligatoria.** Los parámetros **no se descubren solos**. Un cliente no tiene forma de saber que existe `?orderBy` ni qué valores acepta, salvo que alguien lo documente. Comparalo con HATEOAS de la Parte 4: aquello prometía descubribilidad, esto la destruye. Por eso las APIs modernas vienen con OpenAPI o similar — no es burocracia, es la única manera de que el cliente sepa qué puede pedir.

### 1.4. Versionado 🔴

**REST no estandariza el versionado.** No hay una forma oficial, y las tres que se usan tienen cada una su costo:

| Estrategia | Ejemplo | Ventaja | Costo |
|---|---|---|---|
| **En la URL** | `/v1/users`<br>`facebook.com/v2.1/dialog/oauth` | Simple, visible, fácil de rutear | **Rompe la idea de que la URL representa un recurso**: `/v1/users` y `/v2/users` son el mismo recurso en dos URLs |
| **Header custom** | `X-API-Version: 2`<br>`Version: 1` | Limpio: la URL sigue siendo el recurso | Menos visible; se olvida fácil al debuggear |
| **Vía `Content-Type`** | `application/vnd.myapi.v2+json`<br>`application/json+v1` | Formalmente correcto: es otra representación del mismo recurso | Complejo de operar y de explicar |

> ⚠️ **Para el parcial:** las tres son válidas. No hay una "correcta" — hay tres con trade-offs distintos, y eso es exactamente lo que se espera que puedas discutir.

---

## 2. GraphQL 🔴

**GraphQL** es un **lenguaje de queries y mutaciones** sobre datos, más un runtime que las ejecuta. Está *orientado a grafos*: en vez de que el servidor defina la forma de cada respuesta, **el cliente describe qué forma quiere y el servidor la construye**.

### 2.1. Los dos problemas que ataca

En REST las respuestas tienen **forma fija**: el endpoint devuelve lo que devuelve. De ahí salen dos males simétricos:

```
   OVER-FETCHING                        UNDER-FETCHING
   (te dan de más)                      (te dan de menos)

   Necesitás:  nombre                   Necesitás:  usuario + sus tasks
   Te llegan:  nombre, apellido,        Te llega:   usuario
               email, teléfono,
               dirección, fecha…        → tenés que hacer MÁS REQUESTS
                                          para completar la vista
   → ancho de banda desperdiciado
```

GraphQL deja que el cliente **declare exactamente qué campos necesita**, y devuelve exactamente eso. Los dos problemas desaparecen por construcción.

### 2.2. El problema N+1

El caso concreto de under-fetching, y el que más duele. Supongamos que las tasks son un subrecurso del usuario, y queremos las tasks de **todos** los usuarios de un board:

```
   PASO 1 — traer los usuarios del board
                                                   ┌──────────┐
   [Cliente] ◄─── users where board = x ──────────►│   User   │   requests = 1
              [ u:1, u:2, u:3, u:4 ]               │ Database │   → /boards/1/users
                                                   └──────────┘

   PASO 2 — traer las tasks de CADA usuario
              tasks where user = 1  ◄──────────►
                    [ t:1, t:2 ]                   ┌──────────┐
              tasks where user = 2  ◄──────────►   │   Task   │   requests = 4
   [Cliente]        [ t:3, t:4 ]                   │ Database │   → /users/1/tasks
              tasks where user = 3  ◄──────────►   └──────────┘     /users/2/tasks
                    [ t:5 ]                                         /users/3/tasks
              tasks where user = 4  ◄──────────►                    /users/4/tasks
                    [ ]

   ┌────────────────────────────────────────────┐
   │  2 tipos de entidad  →  5 requests totales │
   │  Con N usuarios      →  1 + N requests     │
   └────────────────────────────────────────────┘
```

De ahí el nombre: **1 + N requests**. Con 500 usuarios en el board, 501 llamadas.

En GraphQL se resuelve con **una sola query** que anida usuarios y sus tasks.

> **Ojo con la letra chica, porque es lo que distingue entender de repetir:** el problema **no desaparece, se mueve de capa**. Alguien tiene que implementar del lado del servidor el *resolver* que trae esas tasks, y ese resolver puede ser tan ineficiente como las N llamadas originales. Lo que GraphQL aporta es que **la interfaz para pedirlo ya viene dada**: el cliente pide una vez, y la optimización queda del lado del servidor, donde se puede resolver bien y una sola vez para todos.

### 2.3. Cómo se ve

**El servidor publica un schema** que define los tipos y sus relaciones:

```graphql
type Book {
    id: ID!              # el '!' significa NO NULO: este campo siempre viene
    title: String!
    published: Date
    price: String
    author: Author!      # relación: un Book apunta a un Author (obligatorio)
}

type Author {
    id: ID!
    firstName: String!
    lastName: String     # sin '!': puede venir nulo
    books: [Book!]       # relación inversa: una lista de Books
}                        # ...y acá está el GRAFO: Book → Author → Books → …
```

**El cliente manda la query en el body de un `POST`**, describiendo la forma que quiere:

```graphql
{
    book(id: "1"){       # quiero el book con id 1
        title            # de él, solo el título
        author {         # y de su autor...
            firstName    # solo el nombre
        }                # nada más: ni price, ni published, ni lastName
    }
}
```

**El servidor responde con esa forma exacta:**

```json
{
    "data": {
        "title": "Black hole blues",
        "author": {
            "name": "Mario Santos"
        }
    }
}
```

Compará la query con la respuesta: **son la misma estructura**. Eso es lo que quiere decir "el cliente describe la forma de la respuesta".

### 2.4. `GET` en vez de `POST`

También se puede hacer `GET` con la query codificada en la URL:

```
http://myapi/graphql?query={book(id: "1"){title,author{firstName}}}

http://myapi/graphql?query=%7Bbook%28id%3A%20%221%22%29%7Btitle%2Cauthor%7BfirstName%7D%7D%7D
```

La segunda línea es la misma query, URL-encoded — así viaja de verdad. La desventaja salta a la vista: **URLs muy largas**, que además **no se pueden comprimir** porque viajan en la línea de request, no en el body. Guardate esto: en la sección 2.6 hay una técnica que lo resuelve.

### 2.5. Qué trae el lenguaje

Del lado de las **queries**:

- **Aliases:** renombrar campos en la respuesta.
- **Fragments:** reutilizar pedazos de query en varias consultas.
- **Variables** y **valores por defecto**.
- **Directivas:** `@include(if: Boolean)` y `@skip(if: Boolean)` — incluir o saltear campos condicionalmente.
- **OperationName:** nombrar la operación.
- **Arguments** y **Fields:** los ladrillos básicos.

Hay **tres tipos de operación**:

| Operación | Qué hace |
|---|---|
| **Query** | Lectura pura (*read-only fetch*) |
| **Mutation** | Escritura seguida de una lectura |
| **Subscription** | Permite al cliente **escuchar eventos en tiempo real** desde el servidor |

```graphql
query GetAllPets {          # una QUERY: solo lee
  pets {
    name
    petType
  }
}
```

```graphql
mutation AddNewPet($name: String!, $petType: PetType!) {   # una MUTATION: escribe.
  addPet(name: $name, petType: $petType) {                 # $name y $petType son VARIABLES,
    id                                                     # tipadas y obligatorias (!).
    name                                                   # Y lo que va entre llaves es lo que
    petType                                                # querés que te devuelva DESPUÉS de
  }                                                        # crear: escritura + lectura en una.
}
```

Del lado del **ecosistema**:

- **Introspección:** el cliente puede **preguntarle al servidor qué tipos y campos soporta** — conceptualmente parecido a `OPTIONS` en HTTP. Es lo que hace posible el autocompletado.
- **Validación:** como el schema es tipado, se puede determinar **antes de ejecutar** si una query es válida.
- **Query resolver:** la función responsable de poblar los datos de un campo del schema. Es donde se implementa el trabajo real.
- **Herramientas:** **GraphiQL** (un IDE web para explorar el schema y probar queries, con introspección y autocompletado), **GraphQL Playground**, **Apollo Server**.

### 2.6. Caching en GraphQL

Acá aparece el costo grande. Como los requests son **`POST` con body**, el caching HTTP tradicional —todo lo de la Parte 4— **deja de aplicar directamente**:

- El **HTTP caching sigue valiendo** para los endpoints que lo permitan.
- **El caching de `POST` no es posible** en la mayoría de las caches HTTP.
- Hay una tensión de fondo entre **customization** (que cada cliente pida una forma distinta) y **optimization** (reutilizar respuestas): si cada query es única, no hay nada que reutilizar.
- **ETag y `Last-Modified`** siguen siendo aplicables a nivel aplicación.
- Existen capas de **app cache, server cache y shared cache**.

**Automatic Persisted Queries (APQ)** es la técnica que resuelve a la vez el problema de las URLs largas y el del caching:

```
   CLIENTE                        ENGINE                     SERVIDOR
      │                             │                            │
      │  hash: '4fa973c'            │                            │
      │──── request optimista ─────►│  🤷 no conozco ese hash    │
      │                             │                            │
      │◄─── Persisted query not found                            │
      │                             │                            │
      │  hash: '4fa973c'            │                            │
      │  + query GetPosts {         │                            │
      │      posts { id, title } }  │                            │
      │────────────────────────────►│  👍 la registro con        │
      │                             │     ese hash               │
      │                             │───── query GetPosts ──────►│
      │                             │                            │ [ejecuta]
      │◄──────────── response ───────────────────────────────────│
      │                             │                            │
      │   ── de acá en más, alcanza con mandar el hash ──         │
```

1. El cliente **calcula un hash de la query**.
2. Manda **solo el hash**.
3. Si el servidor **conoce** esa query (la tiene registrada), la ejecuta. Si no, responde pidiendo el texto completo; el cliente lo manda, el servidor la **registra con ese hash**, y de ahí en adelante alcanza con el hash.

Resultado: requests **chiquitas**, **cacheables por HTTP** porque ahora entran en un `GET`, y el cliente no reenvía la query entera cada vez.

### 2.7. El balance

| Pros | Contras |
|---|---|
| Evita over-fetching y under-fetching | **Optimización de las queries** (el resolver puede ser un desastre) |
| **Contrato por naturaleza:** el schema es el contrato | **Solo JSON** |
| **Introspección incorporada** | **Caching** (todo lo de arriba) |
| Validación previa a la ejecución | **Madurez del ecosistema** |
| | **Potencial de *bikeshedding***: tiempo gastado en cuestiones tangenciales como errores de red, negociación de contenido y caché |

> 🕳️ **Madriguera — Bikeshedding**
> La ley de la trivialidad: los equipos dedican desproporcionadamente más discusión a lo fácil de opinar (el color del cobertizo de bicicletas) que a lo difícil e importante (el diseño de la central nuclear).
> *Volvé al camino.*

> **GraphQL rompe deliberadamente la semántica de REST.** No es un sucesor: tiene trade-offs distintos, y **puede convivir con APIs REST** dentro de una misma arquitectura.

---

## 3. gRPC 🔴

Con gRPC **pegamos la vuelta completa y volvemos a RPC** — el de la Parte 1 —, pero reversionado: un framework moderno de Google construido sobre **HTTP/2** y con **Protocol Buffers** —el de la Parte 2— como formato por defecto.

### 3.1. Qué trae

- **Framework RPC** completo, no solo un formato.
- **Integración transparente con Protocol Buffers** como lenguaje de definición de interfaz y como formato del payload.
- **Payload agnostic** en teoría (Protobuf es el default).
- **Client-side load balancing:** el balanceo lo decide el **cliente**, eligiendo a qué servidor pegarle. Al revés de lo habitual.
- **Tracing** integrado.
- **Health checking.**
- **Authentication.**
- **Cascading call-cancellation:** cancelar una llamada **cancela en cadena las llamadas hijas** que disparó.
- **Flow control** a nivel de aplicación.
- **Full-duplex streaming:** ambos extremos mandan y reciben simultáneamente.
- **Generación de código** adaptada a cada lenguaje, igual que Protobuf.

```proto
// The greeting service definition.
service Greeter {                                          // 'service' define el CONTRATO
  // Sends a greeting                                      //  (lo que en RMI era una interfaz).
  rpc SayHello (HelloRequest) returns (HelloReply) {}      // 'rpc' declara un método remoto:
}                                                          //  qué recibe y qué devuelve.
```

Mirá ese archivo y volvé mentalmente a la Parte 1. Es la misma idea que `RMIInterface` — declarar métodos invocables a distancia — pero escrita en un lenguaje **neutral** del que se genera código para cualquier stack, en vez de una interfaz Java que ata a ambos extremos a Java.

### 3.2. Por qué HTTP/2

gRPC corre sobre HTTP/2, y de ahí saca buena parte de sus capacidades:

```
   HTTP/1.1 (secuencial)                  HTTP/2 (multiplexado)

   1  abre conexión                       1  abre conexión
   2  GET /index.html ──►                 2  GET /index.html ──►
   3  ◄── respuesta                       3  ◄── respuesta
   4  GET /styles.css ──►                 4  GET /styles.css  ──┐
   5  ◄── respuesta                          GET /scripts.js ──┴──►  (en paralelo,
   6  GET /scripts.js ──►                 5  ◄── ambas respuestas     misma conexión)
   7  ◄── respuesta                       6  RENDERIZA
   8  RENDERIZA                           7  conexión abierta
   9  cierra conexión
                                             ↑ menos latencia total
```

- **Multiplexing:** múltiples requests **en paralelo sobre la misma conexión TCP**, sin esperar que termine la anterior.
- **Streaming bidireccional** y full-duplex.
- **Encabezados comprimidos** (HPACK).

### 3.3. Qué se paga

- **Documentación** históricamente más pobre que la de REST.
- **Manejo de errores custom**, distinto del modelo HTTP clásico al que la mayoría de los clientes está acostumbrada.
- **Menor soporte en browsers:** requiere gRPC-Web y un proxy en el medio.

### 3.4. ¿gRPC es más performante que REST? 🔴

**Depende.** Y esta respuesta no es una evasiva: es la respuesta correcta, y probablemente la pregunta más representativa de toda la materia.

Lo que gRPC te da de fábrica —full-duplex, autenticación, multiplexing, serialización binaria— **HTTP + JSON no te lo da**. Si tu caso de uso los usa, gRPC gana con claridad.

Pero **para un CRUD simple que no explota ninguna de esas features**, REST puede ser **más simple, más cacheable y más operable**. Y hay un factor que suele subestimarse: la enorme cantidad de APIs REST con JSON que existen hizo que se desarrollaran **librerías muy eficientes** y que la comunidad sea **mucho más amplia** — menos bugs, más casos probados en producción.

> **La frase que cierra la unidad:** una tecnología **más de nicho, mejor en cierto caso de uso, o más nueva, no implica un sucesor ni una mejora.** Es exactamente el criterio de la Parte 0 —los incentivos y el valor, no la novedad— aplicado al último tema del recorrido.

---

## 4. El recorrido completo 🟢

Vale la pena ver la unidad entera de un vistazo, porque cada tema fue una respuesta al problema que dejó abierto el anterior:

```
  Sockets              bajo nivel; te dan control y te cobran TODO
     │                 ¿mejores abstracciones?
     ▼
  RPC / RMI            simple, pero acopla lenguaje y miente sobre lo remoto
     │                 ¿cómo hablamos entre lenguajes distintos?
     ▼
  Serialización        XML · JSON · YAML · Protobuf → el acuerdo sobre los datos
     │                 ¿y un estándar de comunicación encima?
     ▼
  SOAP                 estandariza, pero modela OPERACIONES y es verborrágico
     │                 ¿y si modelamos RECURSOS y usamos HTTP en serio?
     ▼
  REST                 recursos + verbos + códigos + caché + stateless
     │                 ¿y cuando hay que filtrar, paginar, versionar?
     ▼
  Query params         funciona, no es REST puro, fragmenta la caché
     │
     ├──► GraphQL      el cliente pide la forma. Rompe REST a propósito.
     │                 Gana flexibilidad, pierde caché HTTP.
     │
     └──► gRPC         vuelve a RPC sobre HTTP/2 + Protobuf.
                       Gana performance y features, pierde simplicidad y browsers.
```

Ninguna de las flechas es una sustitución. **Todas conviven hoy en producción**, y elegir entre ellas es un problema de contexto, no de ranking.

---

## Para el parcial, si te preguntan

> **¿Por qué las APIs reales se alejan de REST puro?**
>
> Porque en cuanto hay que filtrar, seleccionar campos, paginar u ordenar, mantener la URL como identificador de recurso obligaría a crear una ruta por cada combinación de filtros —con tres filtros ya son dieciséis rutas—. Se resuelve con query params, que no son REST pero son lo que se usa. El costo es que fragmentan la caché, porque cada combinación genera una URL distinta, y que obligan a documentar la API, porque los parámetros no se descubren solos.

> **¿Qué problemas de REST resuelve GraphQL?**
>
> El over-fetching y el under-fetching, porque el cliente declara qué campos necesita en lugar de recibir una respuesta de forma fija, y el problema N+1, porque una sola query puede anidar recursos relacionados en vez de hacer 1+N requests. Es importante notar que el N+1 no desaparece: se mueve al resolver del servidor, pero la interfaz para pedirlo ya viene dada. A cambio se pierde el caching HTTP tradicional, porque las queries viajan como `POST` con body.

> **¿Qué son las Automatic Persisted Queries y qué resuelven?**
>
> Una técnica en la que el cliente calcula un hash de la query y manda solo ese hash. Si el servidor la tiene registrada la ejecuta; si no, pide el texto completo, la registra con ese hash y a partir de ahí alcanza con el hash. Resuelve dos problemas a la vez: el tamaño excesivo de las URLs cuando las queries van por `GET`, y la imposibilidad de cachear, ya que las requests pasan a ser chicas y cacheables por HTTP.

> **¿gRPC es más performante que REST?**
>
> Depende del caso de uso. gRPC ofrece de fábrica serialización binaria, multiplexing sobre HTTP/2, streaming full-duplex y autenticación, cosas que HTTP con JSON no da; si la aplicación las aprovecha, gana. Para un CRUD simple que no las usa, REST suele ser más simple, más cacheable y más operable, y además cuenta con librerías muy maduras y una comunidad mucho más amplia. Que una tecnología sea más nueva o más de nicho no la convierte en sucesora ni en mejora.

---

## Checkpoint

Respondelas sin volver al texto. Las respuestas van al complemento.

1. ¿Cuántas rutas hacen falta para filtrar por tres criterios manteniendo REST puro, y por qué? ¿Por qué los query params no son REST y se usan igual?
2. ¿Qué dos problemas introducen los query params?
3. Nombrá las tres estrategias de versionado con su ventaja y su costo.
4. Explicá over-fetching y under-fetching con un ejemplo de cada uno.
5. Describí el problema N+1 y en qué sentido GraphQL lo resuelve y en qué sentido no.
6. ¿Qué significa que en GraphQL "el cliente describe la forma de la respuesta"? ¿Cuáles son sus tres tipos de operación?
7. ¿Por qué el caching HTTP tradicional no aplica directamente a GraphQL?
8. Explicá el flujo completo de las Automatic Persisted Queries.
9. ¿Qué le aporta HTTP/2 a gRPC? ¿En qué se parece y en qué se diferencia del RMI que vimos al principio?
10. ¿Bajo qué condiciones elegirías REST por sobre gRPC?

---

## Cierre de la unidad

Con esto termina el contenido de APIs. Quedan como cierre las lecturas que la cátedra recomienda para profundizar por tu cuenta:

- **El camino hacia la gloria de REST** (Martin Fowler, sobre el Richardson Maturity Model) — https://martinfowler.com/articles/richardsonMaturityModel.html
- **Métodos seguros e idempotentes** (MDN) — https://developer.mozilla.org/en-US/docs/Glossary/Idempotent
- **HTTP Cache** — http://odino.org/rest-better-http-cache/
- **Protocol Buffers** (Google) — https://developers.google.com/protocol-buffers/
- **Charla de Mercado Libre** — https://youtu.be/0zZ9Poyywq4
- **GraphQL** — https://graphql.org/
- **GraphQL caching** (Apollo) — https://www.apollographql.com/blog/graphql-caching-the-elephant-in-the-room-11a3df0c23ad/
- **gRPC** — https://grpc.io/
- **GraphQL + Thrift en Airbnb** — https://medium.com/airbnb-engineering/reconciling-graphql-and-thrift-at-airbnb-a97e8d290712

---

**FIN DE LA PARTE 5 — FIN DE LA UNIDAD clase01**
