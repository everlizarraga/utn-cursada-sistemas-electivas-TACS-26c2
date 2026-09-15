# 📘 Apunte maestro — clase04 · NoSQL
## Parte 5 de 5 — Los cuatro tipos y cómo decidir

**Unidad:** `clase04` · clase 4 del cronograma (01/09/2026) · **Tema:** noSQL
**Partes de la unidad:** 1 · Por qué existen las NoSQL y cómo se escala — 2 · Sharding — 3 · Escribir en más de un nodo — 4 · Las garantías — 5 · Los cuatro tipos y cómo decidir *(esta)*

**Leyenda y convenciones:** las de la Parte 1 (🔴🟡🟢 · 🕳️ · ⚠️ · 📌 · 📋). Cada concepto se explica una vez; lo que ya se explicó se cita.

**Qué cubre esta parte.** El catálogo: las cuatro familias de bases NoSQL (clave-valor, wide column, documental, grafos), cada una con su implementación de referencia, su patrón de acceso, sus trampas y su lugar en CAP. Después, las reglas para elegir, y una distinción que atraviesa cualquier sistema real: la base operativa (OLTP) y la base analítica (OLAP) no son la misma.

**Qué asume de las partes anteriores.** Sharding por hash y anillo (Parte 2). Replicación, quorum W/R y consistencia eventual (Parte 3). CAP, ACID, BASE y la regla de "nombrar la implementación y justificar el costo" (Parte 4).

---

## 1. Cuatro familias, cuatro patrones de acceso 🔴

| # | Familia | Estructura en una línea | Ejemplos |
|---|---|---|---|
| 01 | **Clave-valor** (*key-value*) | Un map simple: key → valor. | Redis · Memcached |
| 02 | **Wide column** (también *column family*) | Key + columnas, sin esquema fijo. | Cassandra · DynamoDB |
| 03 | **Documental** | Colecciones de JSON jerárquicos. | MongoDB · Elasticsearch |
| 04 | **Grafos** | Nodos y relaciones, ambos con propiedades. | Neo4j · Titan |

**Cada una optimiza un patrón de acceso distinto.** Esa es la idea que ordena todo lo que sigue: no son cuatro formas de guardar lo mismo, son cuatro respuestas a "¿cómo voy a acceder a estos datos?". Y de ahí sale un concepto importante para sistemas reales: **no hay una que mate a todas.** Cada una se usa para su caso de uso, y en un mismo sistema **se combinan**. A eso se le llama **persistencia políglota**: un sistema que habla con varias bases, cada una para lo que mejor hace.

---

## 2. Clave-valor: Redis 🔴

### 2.1 Qué es

Lo más simple que hay: una base que tiene **clave → valor**, un map. Entrás con una key y te da un **valor opaco** (la base no interpreta qué hay adentro: es un string, un blob, lo que sea).

```
   session:ab12    →  "user=42"                ← sesión de un usuario logueado
   user:42:name    →  "Marcos"                 ← un atributo suelto
   cart:42         →  [item_5, item_8]         ← el carrito del usuario 42
   counter:views   →  15783                    ← un contador
```

Fijate cómo se arman las keys: con un prefijo que dice qué es (`session:`, `user:`, `cart:`) y un identificador. No hay tablas ni relaciones: hay keys.

| Aspecto | Cómo es |
|---|---|
| **Performance** | Pensadas para **leer y escribir rápido**: muy alto throughput, latencias por debajo del milisegundo. |
| **Storage** | La mayoría son **in-memory** o están optimizadas para velocidad. Algunas persisten a disco o se pueden configurar para que lo hagan, pero la prioridad es siempre la performance. |
| **Escala** | Mucha. |
| **Queries** | **Poca flexibilidad**: no permite queries de alta complejidad. Entrás por la key y punto. |
| **Usos típicos** | **Sesiones, caché, carritos, contadores.** |

**Redis** es la implementación de referencia, y está *polenteada*, como Postgres en la Parte 4: tiene un montón de features que se van del concepto básico de clave-valor. Se puede usar como clave-valor plano, pero también trae: opciones para ir guardando a disco (para no tener que levantar todo de vuelta si se cae la memoria), tipos de datos geográficos, bitmaps, muchos tipos de sets, series de tiempo, backups, semáforos distribuidos (sección 2.3) y tres millones de cosas más.

### 2.2 ¿Redis corre adentro de la aplicación o aparte?

Podrías embeberlo, pero **no se suele hacer: Redis corre en su propio nodo.** Y eso dispara una pregunta que da para largo: si Redis es rapidísimo pero está en otra máquina, tengo un **salto de red** en el medio (un pedido HTTP, con su tiempo). ¿No pierdo lo que gané? Hay varios trucos, en distintas capas, para tenerlo aislado pero cerca:

1. **Near cache en la library cliente.** Cuando te bajás Redis, configurás en tu server una biblioteca para pegarle. Esa biblioteca a veces hace *near cache*: se convierte en **una caché de Redis del lado del cliente**, al lado de tu aplicación. Te persiste parte de lo que trabajás con Redis, con invalidación y algunas cosas custom, para que tengas acceso súper rápido sin salir a la red cada vez.
2. **Mismo nodo, salto ínfimo.** A veces tu *working set* (los datos que Redis realmente tiene en uso) son 100 MB: es poca memoria. Entonces, en un nodo de 16 cores y 31 GB, podés configurar que Redis y tu aplicación se hablen por HTTP pero **estén en la misma computadora**. Hay salto de red, pero es ínfimo.
3. **La comparación justa.** Postgres también lo tenés, en general, en otro nodo, y además tiene la latencia de **ir a buscar a disco**. Redis tiene la latencia de red y después busca en memoria, que es rapidísimo. La red la pagás igual; lo que ahorrás es el disco.

El truco 2 tiene un límite: si tu aplicación corre en **múltiples nodos** y todos usan el mismo Redis, no podés tenerlo en cada nodo (tendrías N instancias de Redis, cada una con datos distintos). Y hay un caso de uso que justamente exige **una sola instancia compartida**:

### 2.3 Semáforos distribuidos

Un semáforo (el de sistemas operativos: un mecanismo para que solo un proceso a la vez entre a una sección crítica) **sirve dentro de un mismo nodo**. Cuando tu aplicación corre en cinco nodos, un semáforo normal no te sirve: cada nodo tiene el suyo y no se ven entre sí. Lo que se hace es **crear un lock como un dato en Redis**:

```
   lock:procesar-pedidos   →  "nodo-3"      ← el nodo 3 tiene el lock

   nodo 1: intenta escribir lock:procesar-pedidos → ya está escrita → espera
   nodo 2: intenta escribir lock:procesar-pedidos → ya está escrita → espera
   nodo 3: termina, borra la key
   nodo 1: intenta de nuevo → escribe → ahora tiene el lock
```

Los demás nodos intentan escribir esa clave; si ya está escrita, se quedan esperando hasta ganar el lock. Como todos tienen que ver **la misma clave**, necesitás una instancia de Redis compartida, no una por nodo. Con la app en muchos nodos, esto es lo que hace que "un Redis por nodo" sea un quilombo: se puede, pero es un quilombo.

> 🕳️ **Madriguera — Bases vectoriales y embeddings**
> Usar Redis como "base de conocimiento" de un agente de IA (texto partido en chunks, y buscar "proyectos de Buenos Aires") no es clave-valor: es una **base vectorial**. Un modelo de *embeddings* convierte cada chunk en un vector, la base los carga en un espacio vectorial, y buscar es medir similitud (coseno y otras cosas de álgebra). Emparentado con el *full-text search* difuso. Redis tiene un módulo para eso, como para todo.
> *Volvé al camino — esto se profundiza aparte, otro día.*

**En CAP:** Redis busca **disponibilidad**.

> 📌 **Para el parcial, si te preguntan: ¿qué es una base clave-valor y cuándo la usás?**
> Es un map key → valor opaco, típicamente en memoria, optimizado para leer y escribir rápido con muy alto throughput y latencia sub-milisegundo, a costa de no permitir queries complejas: se accede solo por la key. Se usa para sesiones, caché, carritos, contadores y locks distribuidos. Implementación: Redis (o Memcached). En CAP prioriza disponibilidad.

---

## 3. Wide column: Cassandra y Dynamo 🔴

### 3.1 Qué es: clave-valor donde el valor tiene estructura

Es un par clave-valor, pero **el valor tiene estructura interna**: columnas. Y es **schemaless en los campos no indexados**: podés agregar columnas sin declararlas de antemano. El ejemplo de referencia son sensores que registran temperatura y humedad, con un flag de alerta:

| SensorID **(PK, partition key)** | Timestamp **(SK, sort key)** | Temperature | Humidity | Alert |
|---|---|---|---|---|
| 1 | 4653958801 | 20 | 5 | — |
| 2 | 1650928601 | 20 | 5 | — |
| 2 | 1650928602 | 24 | 5 | — |
| 3 | 5650328211 | 20 | 6 | — |
| 4 | 7650528471 | — | 8 | — |
| 4 | 7650528481 | — | 9 | true |

Mirá las filas del sensor 4: no tienen temperatura, y no pasa nada; no hay "columna nula", la columna simplemente no está en esa fila. Eso es el schemaless.

Las dos columnas marcadas, PK y SK, son la clave de todo:

- **Partition key (PK).** Acá, el ID del sensor. Es la key que **se hashea para decidir en qué nodo vive la fila**: el sharding de la Parte 2, ahora con nombre de columna. Tercer sentido de "partición" en esta clase: no es el particionamiento por negocio (Parte 1) ni la partición de red (Parte 3); es la porción del anillo donde cae una fila. Puede ser **compuesta** (varias columnas juntas forman la PK).
- **Sort key (SK).** Acá, el timestamp. Define **el orden** en que se guardan las filas dentro de una partición. Puede haber **varias**: por ejemplo año, mes y día (redundante con timestamp, pero sirve para verlo). Y cuando hay varias, **el orden importa**: no podés ordenar por la tercera si no ordenaste primero por la primera y la segunda, porque los datos ya vienen ordenados de entrada en ese orden de prioridad.

### 3.2 Cómo se consulta, y por qué eso lo define todo

Las queries, en Cassandra por ejemplo, **se parecen mucho a un SELECT de SQL** sobre una tabla. Pero son **muy restringidas**:

- **Generalmente es obligatorio filtrar por partition key.** Entrás por una PK concreta, y recién después, sobre la SK, podés jugar (rangos, orden).
- **Sobre la PK no podés usar cualquier operador.** No podés pedir "dame el rango de PKs entre 1 y 3": tenés que acceder a una PK.
- **No hay joins.** Hay un montón de operaciones que no se pueden hacer; tienen que ser queries simples. Es más polenta que clave-valor, pero sigue siendo muy poco flexible.
- Se puede hacer un *full scan* (recorrer toda la tabla sin PK), y la base te lo permite. La historia real: un full scan sobre DynamoDB, de las primeras veces que alguien lo usaba, **tiró abajo toda la aplicación**. Se pudo. Se cayó todo.

De ahí sale **la gran restricción**, la regla número uno de wide column:

> **Siempre que diseñes con una wide column, tenés que pensar primero el patrón de acceso.** Si no lo pensás bien, estás frito.

Concretamente: si querés acceder a la misma información **de dos maneras distintas**, tenés que tener **dos tablas distintas**, con los mismos datos guardados de forma distinta, una por cada patrón de acceso. Es el principio de "duplicar información para cuidar el cómputo" de la Parte 3, sección 2.5, llevado al extremo. A cambio, te asegura una velocidad de acceso rapidísima a la información que necesitás, por más cantidad de datos que tengas.

> 🕳️ **Madriguera — Diagrama de Chebotko**
> Es la metodología de modelado para Cassandra: partís de las queries que la aplicación va a hacer y de ahí derivás las tablas, una por patrón de acceso, en un diagrama que muestra qué tabla sirve a qué query. Es el "patrón de acceso primero" convertido en método.
> *Volvé al camino — esto se profundiza aparte, otro día.*

### 3.3 Hay una forma que anda y es mal diseño: el índice global de Dynamo

Estás en DynamoDB. Tenés tu tabla con su PK y su SK, y accedés por ahí. Un día necesitás entrar por **otro campo**. Dynamo te ofrece una salida: crear un **GSI**, *Global Secondary Index*, un índice global. Lo que hace por atrás:

```
   Tabla original                   GSI (lo arma Dynamo, por atrás)
   PK = SensorID, SK = Timestamp    PK = Humidity (la columna que pediste)
   ┌────────────────────┐           ┌────────────────────┐
   │ todas tus filas    │ ────────► │ todas tus filas,   │  eventualmente
   │ ordenadas por      │  copia    │ reindexadas por    │  consistente con
   │ sensor y tiempo    │           │ humedad            │  la original
   └────────────────────┘           └────────────────────┘
```

Agarra **todas tus filas**, las busca por la columna que le dijiste, y **arma otra tabla en paralelo**, que es **eventualmente consistente** con la tuya (Parte 3, 3.4): tu tabla y el índice pueden estar desfasados un rato. Funciona. Y es un antipatrón, por dos razones:

1. **Pagás el doble**, porque es literalmente otra tabla, con su almacenamiento y sus escrituras.
2. **Es la señal de que elegiste mal.** Si tenés una wide column con una PK y necesitás **tres PKs más**, y la solución es "hago tres GSI", lo que pasó es que **elegiste la base equivocada**: tu patrón de acceso no es fijo, y wide column está hecha para patrones fijos.

### 3.4 Time series, costo, disco y por qué es rápida

- **Time series** son una familia de las wide column, un subcaso: guardar **eventos que ocurrieron**, como datos de sensores o logs. El ejemplo de arriba es exactamente eso.
- **Son muy baratas y muy rápidas**, y schemaless: si mañana quiero una columna nueva, la agrego y listo. Pero **se complejizan rápido** en cuanto la consulta se sale del patrón.
- **Viven en disco.** Algunas ofrecen una versión en memoria, mucho más cara: la velocidad de Redis con las capacidades de una wide column.
- **¿Por qué es más rápida que una SQL?** No necesariamente lo es. Suele ser muy rápida porque **el patrón de acceso es fijo**: entro siempre por SensorID y Timestamp, y la base está hiper-optimizada para que entre por ahí. SQL tiene que tener en cuenta muchas más cosas, porque tiene que darte **flexibilidad** en la consulta. Y ojo: una relacional con un buen índice también anda rápido; solo que tiene su costo, no es tan fácil.

**En CAP:** Cassandra por defecto es AP (Parte 4), pero es **configurable**: con los parámetros W y R (Parte 3) podés correrla hacia consistencia o hacia disponibilidad. Siempre con P, porque está distribuida.

> 📌 **Para el parcial, si te preguntan: ¿qué es una base wide column y cuál es su restricción principal?**
> Es clave-valor donde el valor tiene estructura de columnas, schemaless en los campos no indexados. Cada fila tiene una partition key (que decide en qué nodo vive) y una o más sort keys (que ordenan dentro de la partición). Las queries parecen SQL pero son muy restringidas: se entra por partition key, sin joins. Su restricción principal es que hay que diseñar pensando el patrón de acceso: un acceso distinto exige otra tabla con los mismos datos. Ideal para eventos, logs y series de tiempo. Implementaciones: Cassandra, DynamoDB. En CAP: configurable, con P.

---

## 4. Documental: Mongo 🔴

### 4.1 Qué es

Colecciones de **documentos**, generalmente en JSON, con estructura **jerárquica**: un documento tiene campos, listas y objetos anidados. Un documento de ejemplo (las aclaraciones en `//` no son parte del JSON):

```jsonc
{
  "_id": "usr_42",                      // identificador del documento
  "name": "Marcos",
  "tags": ["admin", "pro"],             // una lista, adentro del documento
  "address": {                          // un objeto anidado
    "city": "Buenos Aires",
    "zip": "C1425"
  }
}
```

Compará con lo que sería en una relacional: una tabla de usuarios, una de tags con clave foránea, una de direcciones. Acá es **un solo documento**, y con un acceso te traés todo.

| Aspecto | Cómo es |
|---|---|
| **Schema** | **Dinámico y flexible.** No estás atado a las columnas de una tabla: agregás o sacás campos sin problema, sin columnas en nulo. Es *schemaless*, o mejor dicho *schema on read*: el esquema no lo impone la base al guardar, lo aplica quien lee. Flexible para cambios. |
| **Mapeo con objetos** | Mucho más fácil: cuando trabajás con objetos, el documento mapea **casi uno a uno**. El impedance mismatch de la Parte 1 casi desaparece. |
| **Queries** | Permite consultar por cualquier campo e indexar cualquier campo. En ese sentido, tiene **las mismas features que una SQL**. |
| **Trade-off** | Más flexibilidad que wide column, **menos escalabilidad extrema**. |
| **Ejemplos** | MongoDB, CouchDB, Elasticsearch, DocumentDB. |

### 4.2 El stack MERN y por qué pasó de moda 🟡

Hubo una época en la que esto fue una moda: el **stack MERN** (o MEAN): **M**ongo, **E**xpress (un framework para hacer servers en Node), **R**eact (o **A**ngular), **N**ode. El chiste era que usabas **JSON en la red, objetos de JavaScript en el server y JSON binario en Mongo**: no eran iguales, pero eran muy parecidos, y entonces cualquiera podía hacer algo full stack, punta a punta, más o menos con lo mismo. Un developer con poca experiencia, o que conoce una sola cosa, podía trabajar en muchos lados del stack.

Pasó de moda, y **no porque algo lo superara**. Dos cosas: surgieron otros lenguajes, y **Mongo es fácil de empezar pero cuando escala no es tan amigable**. Las queries se ponen bastante feas y complejas, y necesitás alguien que sepa consultarla y hacerlo de manera óptima, por la performance. No es trivial como en una relacional, donde agregás un índice y listo. Historia real: alguien usó Mongo para una cosa muy chica y **tiró abajo el nodo donde corría**, porque no sabía; a los dos días la app no funcionaba.

De ahí un principio que se repite en toda la materia: **las tecnologías muy de nicho o muy complejas, si no tenés el equipo que las conozca, no convienen.**

### 4.3 Elastic y la analítica 🟡

Lo que está de moda desde hace años, más que Mongo, es **Elasticsearch**. Mucha gente lo usa para cosas analíticas: permite queries más largas, análisis de logs, traerte estadísticas de un usuario, agregaciones (sumas, promedios). Te acerca a lo que hace una relacional con funciones de agregación, sin ser relacional.

### 4.4 ¿Una documental tiene más throughput que una relacional?

📋 Esta pregunta salió por el **TP**, que tiene como requerimiento soportar mucho throughput. La respuesta: **sí, cuando lo que en una relacional te tomaría muchos joins, en una documental es una sola búsqueda**, porque podés tener todo embebido en el documento y con un acceso te traés todo. Pero **depende de cómo lo modeles**: la ventaja aparece si diseñaste el documento para que consumir esa información sea sencillo. El throughput no viene de "usar Mongo", viene de haber embebido bien.

**En CAP:** Mongo es **configurable**: más AP o más CP según cómo lo configures (Parte 4), siempre con P.

> 📌 **Para el parcial, si te preguntan: ¿qué es una base documental y qué ventajas tiene sobre una relacional?**
> Guarda colecciones de documentos JSON jerárquicos. Ventajas: esquema flexible (schema on read: se agregan o sacan campos sin columnas nulas), mapeo casi uno a uno con los objetos del código, y queries e índices sobre cualquier campo. Lo que en una relacional serían varios joins puede ser un solo acceso si el documento está bien embebido. Trade-off: menos escalabilidad extrema que wide column, y las queries se complican al escalar. Implementación: MongoDB. En CAP: configurable, con P.

---

## 5. Grafos: Neo4j 🔴

### 5.1 Qué es

Bases basadas en **nodos**, que representan entidades, y **aristas** (relaciones), que representan cómo se conectan. **Ambos tienen propiedades.** Son muy útiles para problemas donde **lo importante está en la relación**: redes sociales, recomendaciones, detección de fraude.

```
                            (Brooke Langton)                                       (Clint Eastwood)
                                    │ ACTED_IN                                             │ ACTED_IN
                                    │                                                      │ DIRECTED
                                    ▼                                                      ▼
 (Keanu Reeves) ─ACTED_IN─► [The Replacements] ◄─ACTED_IN─ (Gene Hackman) ─ACTED_IN─► [Unforgiven]
                                    ▲                                                      ▲
                                    │ ACTED_IN                                             │ ACTED_IN
                             (Orlando Jones)                                       (Richard Harris)

 (Robin Williams)   ← nodo Person sin relaciones

   ( ) nodo Person     [ ] nodo Movie     ─►  relación con su tipo
```

Actores y películas son nodos de dos tipos; las aristas dicen si actuaron (`ACTED_IN`) o dirigieron (`DIRECTED`). Robin Williams es un nodo sin relaciones.

### 5.2 Las queries son semánticas: Cypher

Las queries se escriben de forma **semántica**, casi como se dice la pregunta: "dame los amigos de tal que también siguen a X". Los lenguajes son **Cypher** (Neo4j) y **Gremlin**. La pregunta del ejemplo: **actores que actuaron en una película con Gene Hackman, en la que no haya actuado Robin Williams.**

```cypher
MATCH (gene:Person {name:"Gene Hackman"})-[:ACTED_IN]->(movie:Movie),
//    ^ buscá el nodo Person que se llama Gene Hackman
//                                     ^ que tenga una relación ACTED_IN hacia un nodo Movie;
//                                       a esa película la llamamos "movie"
      (other:Person)-[:ACTED_IN]->(movie),
//    ^ buscá otros nodos Person que también tengan ACTED_IN hacia ESA misma "movie"
      (robin:Person {name:"Robin Williams"})
//    ^ y ubicá el nodo Person de Robin Williams (para usarlo en la condición)
WHERE NOT (robin)-[:ACTED_IN]->(movie)
//    ^ condición: que NO exista una relación ACTED_IN de Robin hacia esa película
RETURN DISTINCT other
//    ^ devolvé los "other", sin repetidos

// ¿CÓMO FUNCIONA?
// 1. Se ubican las películas de Gene Hackman: The Replacements y Unforgiven.
// 2. Para cada una, se buscan las personas con ACTED_IN hacia ella.
// 3. Se descartan las películas donde Robin Williams actuó (acá, ninguna:
//    Robin no tiene relaciones, así que las dos películas pasan el filtro).
// 4. Resultado esperado: los co-actores de Gene en ambas películas:
//    Keanu Reeves, Brooke Langton, Orlando Jones (The Replacements),
//    Clint Eastwood, Richard Harris (Unforgiven).
//    Detalle: nada excluye a Gene mismo del patrón "other", así que también
//    aparece; para sacarlo se agrega "AND other <> gene" al WHERE.
```

Compará con SQL: la misma pregunta necesitaría varias tablas (personas, películas, actuaciones), varios joins y una subconsulta con NOT EXISTS. Y cuando las relaciones tienen **varios niveles de separación** (amigos de amigos de amigos), en SQL las queries terminan siendo medio recursivas y **casi ilegibles**; en grafos se escriben muy fácil.

Cypher tiene también cláusulas de agrupación para contar (cuántas películas actuó tal), pero el foco de la base no está en las agregaciones: está en **recorrer relaciones**.

### 5.3 Casos reales

- **Facebook:** te recomienda una página porque **amigos de amigos** la vieron. Relaciones anidadas con varios niveles, fáciles acá, casi imposibles en SQL.
- **Fraude:** cargás un montón de transacciones sabiendo cuáles fueron fraudulentas, y buscás **patrones entre las fraudulentas**: tarjetas, personas, conexiones.
- **Scoring crediticio:** con Neo4j, calcular la probabilidad de que una persona te pague o no, y en base a eso qué interés le cobrás y cuánta plata le prestás.

### 5.4 La limitación: no escalan horizontal

A diferencia de las otras tres, las bases de grafos **no escalan horizontalmente**: **no hay sharding**, porque es difícil distribuir un grafo (cortarlo en pedazos rompe justamente las relaciones que son su valor). Tienen que **vivir dentro de un mismo nodo**.

**En CAP:** ⚠️ *En la materia se enseña que las bases de grafos son CA: como viven en un solo nodo, no hay particiones que tolerar y ofrecen consistencia y disponibilidad. Estrictamente, CAP no aplica a un sistema no distribuido (Parte 4, 1.2); "CA" es la forma de decir "no está distribuida". Para el parcial: grafos = CA.*

> 📌 **Para el parcial, si te preguntan: ¿qué es una base de grafos y cuándo la usás?**
> Modela nodos (entidades) y relaciones, ambos con propiedades, y se consulta con lenguajes semánticos como Cypher. Se usa cuando las relaciones son tan importantes como los datos: redes sociales (amigos de amigos), recomendaciones, detección de fraude, scoring. Su limitación es que no escala horizontal: no hay sharding, vive en un solo nodo. Implementación: Neo4j. En CAP: CA.

---

## 6. Dónde cae cada una en CAP 🔴

Todas las bases **distribuidas horizontalmente tienen que tener P**: si están repartidas en nodos, la red se puede partir, y tienen que seguir andando. Lo que eligen es entre C y A. La única que no está distribuida es la de grafos.

| Familia | Implementación | CAP | Por qué |
|---|---|---|---|
| Clave-valor | Redis | **A** (+P) | Responder rápido, siempre. |
| Wide column | Cassandra | **AP por defecto, configurable** | "Te doy el dato que tengo"; W y R la corren. |
| Documental | Mongo | **Configurable** (más AP o más CP) | Parámetros de escritura y lectura. |
| Grafos | Neo4j | **CA** ⚠️ | Vive en un solo nodo. |
| Relacional distribuida | Postgres, MySQL | **CA** ⚠️ | Consistencia por diseño (Parte 4, 1.4). |

---

## 7. ¿Qué elegir? 🔴

No hay una única solución ni una bala de plata. Pero hay reglas útiles para detectar la más indicada:

| # | Regla | Qué significa |
|---|---|---|
| 01 | **Analizar primero** | Antes de elegir, entender **la carga**, **los patrones de acceso** a la información y **los requisitos de consistencia**. |
| 02 | **ACID por default** | Se arranca con una relacional. Sacrificar ACID tiene que estar **justificado por un caso concreto**: un motivo claro por el cual la relacional no alcanza. |
| 03 | **Tipo de consulta** | Pensar siempre la **complejidad de las queries** que voy a hacer: las relaciones, las agregaciones. |
| 04 | **Analytics aparte** | Las queries pesadas van a una **base secundaria** (BigQuery, réplicas, ETL). Es la sección 8. |

*ETL*: *Extract, Transform, Load*, el proceso que saca datos de una base, los transforma y los carga en otra.

Juntando las reglas con el catálogo, el mapa de decisión que la clase deja armado:

| Si el problema es… | Patrón | Elegís | Ejemplo a nombrar |
|---|---|---|---|
| Transacciones, joins, integridad | Relaciones y consistencia fuerte | **Relacional** (el default) | Postgres |
| Sesiones, caché, contadores, locks | Acceso por key, latencia mínima | **Clave-valor** | Redis |
| Eventos, logs, sensores, series de tiempo | Escritura masiva, acceso por key fija + orden | **Wide column** | Cassandra, DynamoDB |
| Objetos con estructura variable, todo embebido | Documento completo en un acceso | **Documental** | MongoDB |
| Amigos de amigos, fraude, recomendaciones | Recorrer relaciones | **Grafos** | Neo4j |
| Reportes, agregaciones sobre millones de filas | Analítica | **OLAP aparte** | BigQuery, Snowflake, Redshift |

Y siempre con la regla de la Parte 4, sección 6.4: **nombrar la implementación, saber explicarla, justificar el costo.**

---

## 8. OLTP vs OLAP: la base operativa no es la analítica 🔴

### 8.1 Qué es cada una

- **OLTP**, *Online Transaction Processing*: la base **transaccional**. Transacciones en tiempo real, baja latencia, operaciones chicas. Es la **base primaria** de tu aplicación.
- **OLAP**, *Online Analytical Processing*: la base **analítica**. Optimizada para recibir **pocas queries pero pesadas**, con mucho análisis y agregación.

| Dimensión | OLTP | OLAP |
|---|---|---|
| **Propósito** | Transacciones en tiempo real | Análisis y reportes |
| **Volumen por query** | Pocos registros | Millones de registros |
| **Frecuencia** | Muchísimas queries por segundo | Pocas queries, pesadas |
| **Latencia objetivo** | Milisegundos | Segundos a minutos |
| **Fuente** | Base primaria | Data warehouse, réplicas, ETL |

### 8.2 No mezclarlas

Lo importante: **no está bueno mezclarlas.** Si tenés tu base transaccional, no tendrías que estar haciendo la analítica ahí. Se tiene una **base secundaria**: si Postgres es la primaria, la analítica va a BigQuery, Snowflake, Redshift o Databricks. La razón es evitar **sobrecargar la base primaria** y no interrumpir el negocio ni generar una caída. Pensalo así: una query analítica pesada puede tardar 15 minutos; para la de reporting no importa, lanzás un job asíncrono, volvés y está. Pero tener **colgada la base de producción 15 minutos** es un problema: esa base está recibiendo un montón de requests de usuarios al mismo tiempo, mientras que la de reporting la consultás vos y, con suerte, otro analista. Son dominios distintos; para la analítica hasta podés aprovisionar un nodo en el momento, correr la consulta y borrarlo.

Cómo llegan los datos de una a otra, según el negocio:

- **Dumps periódicos.** A la noche corrés un dump (un volcado completo, como el del backup de la Parte 3), copiás la base de producción a la OLAP, y al otro día generás reportes sobre la OLAP sin tocar producción. Puede ser diario, cada hora, cada X tiempo.
- **Change data capture (CDC).** En vez de un dump, **cada cambio** que pasa en producción se manda a la analítica: cada fila que agregás, y también cada delete, viaja como un dato. Así la analítica tiene todos los movimientos que se hicieron en la transaccional.

Caso real: en Despegar, producción corría sobre MariaDB, y a la noche corrían dumps de distintas tablas hacia un data lake; los análisis se hacían sobre esas tablas.

### 8.3 Columnar: qué tiene BigQuery que no tenga Postgres

¿En qué se diferencian arquitecturalmente? La clave es **el almacenamiento columnar**. Imaginate un reporte habitual: de las 30 millones de transacciones del día, dame el percentil del monto y el agrupado por tipo de transacción. En una relacional, eso es escanear 30 millones de filas. En una base columnar tenés **archivos donde solamente hay columnas**: todas las columnas de "monto" juntas, todas las de "tipo" juntas, en vez de todas las filas juntas.

```
   Por filas (OLTP)                      Por columnas (OLAP columnar)
   fila 1: [id, monto, tipo, fecha]      monto: [ 120, 85, 3400, 15, ... ]
   fila 2: [id, monto, tipo, fecha]      tipo:  [ "compra", "compra", "retiro", ... ]
   fila 3: [id, monto, tipo, fecha]      fecha: [ ... ]
   → para el percentil del monto,        → leés SOLO la columna monto,
     leés todas las filas enteras          contigua, comprimida
```

Con eso: en vez de procesar fila por fila de manera ingenua, podés tener números **precomputados** y agregar mucho más rápido; podés **comprimir** cada columna, porque son millones de valores del mismo tipo (todos números, todos strings); y tenés herramientas como buckets temporales y percentiles listas para mezclar. **La sintaxis es la misma que en Postgres** (una query SQL); lo que cambia es cómo está hecha la búsqueda por atrás: la data no está guardada de forma relacional, está hecha para reportes.

**Columnar no es wide column.** Ojo con esto porque los nombres se parecen. Wide column (sección 3) tiene consultas muy limitadas, entra por partition key. Columnar tiene **las mismas consultas que Postgres**; lo que cambia es la implementación de esa búsqueda o agrupación: guardar las columnas juntas en vez de las filas juntas. Uno es un tipo de base NoSQL; el otro es una forma de almacenar para analítica.

### 8.4 Duplicados, storage tiers, data warehouse y data lake 🟡

**¿Cuando se dumpea, se limpia la base de origen?** No: las ventas del día **quedan duplicadas**, unos datos en la analítica y otros en producción. En principio se duplica, y después en la parte analítica se aplican estrategias según la necesidad.

**Storage tiers.** En analítica podés configurar niveles de almacenamiento (en Databricks se llaman *gold, silver, bronze*): qué datos quiero acceder rápido y seguido, y cuáles tener "por ahí" para un reporte mensual. El histórico que solo se consulta si algún día viene una auditoría va al storage más barato y más lento. Lo que uso para el reporte diario va a uno más optimizado, con índices y cosas más copadas, que cuesta más por giga pero es menos información, o se borra a los 7 días.

**Data warehouse vs data lake.** Dos estrategias para armar la parte analítica:
- **Data warehouse:** la solución ordenada. Definís una **estructura** de cómo se guardan los datos, y procesos que, al extraer la información, ya la agrupan y crean tablas nuevas con datos pre-agregados, para consultas más rápidas.
- **Data lake:** más tirado de los pelos. A medida que un analista pide información, habla con los devs y van tirando **tablitas sueltas** en una misma base, que no tienen por qué estar relacionadas. Lo valioso: si tenés microservicios con bases separadas (una arquitectura donde cada servicio tiene su base propia), el data lake te permite **cruzar información de distintos microservicios**, porque todas las tablas dumpeadas están en la misma base. Podés hacer queries que en tu sistema productivo no podrías, con SQL transaccional y reportes full relacional, con mucha más riqueza de datos.

> 🕳️ **Madriguera — RDF y la web semántica**
> RDF es un lenguaje de modelado de metadatos para la web semántica, basado en *triples*: sujeto, predicado, objeto ("producto A" — "es compatible con" — "producto B"). Se usaba para ontologías y bases de conocimiento, y se parece a los grafos: los triples son nodos y aristas. Prácticamente murió hace años.
> *Volvé al camino — esto se profundiza aparte, otro día.*

> 📌 **Para el parcial, si te preguntan: ¿qué diferencia hay entre OLTP y OLAP y por qué no se mezclan?**
> OLTP (Online Transaction Processing) es la base transaccional primaria: muchísimas operaciones chicas por segundo, latencia de milisegundos. OLAP (Online Analytical Processing) es la base analítica: pocas queries pesadas sobre millones de registros, latencia de segundos a minutos, con almacenamiento columnar. No se mezclan para no sobrecargar la base de producción: la analítica va a una base secundaria (BigQuery, Snowflake, Redshift) alimentada por dumps periódicos o change data capture, en un data warehouse o un data lake.

---

## 9. Cierre de la unidad 🔴

El recorrido completo, en una línea por parte: los datos crecieron y el hardware cambió, y por eso se escala horizontal (Parte 1); repartir por cálculo es sharding, y el anillo con vnodes es cómo se hace bien (Parte 2); escribir en varios nodos trae conflictos, y quorum es la forma de resolverlos sin perder consistencia (Parte 3); CAP, ACID y BASE son los nombres de lo que se gana y se pierde (Parte 4); y hay cuatro familias, cada una para un patrón de acceso, más una base analítica aparte (Parte 5).

Lo que la cátedra evalúa de todo esto es una sola habilidad: **frente a un sistema descripto, elegir base por base, nombrando la implementación, sabiendo cómo funciona y justificando el costo.** Con la relacional como default y un motivo concreto para cada vez que te apartás de ella.

---

## ✅ Checkpoint — Parte 5

*(Sin respuestas: van en el complemento de la unidad.)*

1. ¿Qué es la persistencia políglota? Dá un ejemplo de sistema que use tres bases distintas y decí para qué cada una.
2. ¿Por qué Redis corre en su propio nodo si eso agrega un salto de red? ¿Qué trucos existen para que no duela?
3. Explicá cómo se implementa un semáforo distribuido con Redis y por qué no sirve un semáforo del sistema operativo.
4. En la tabla de sensores, ¿qué pasa si consulto "todas las lecturas con humedad mayor a 8" sin partition key? ¿Y qué debería haber hecho?
5. Tenés múltiples sort keys: año, mes, día. ¿Podés ordenar solo por día? ¿Por qué?
6. ¿Por qué el GSI de Dynamo es un antipatrón si funciona? ¿Qué te está diciendo sobre tu elección?
7. ¿Qué significa "schema on read"? ¿Qué gana y qué pierde un equipo que trabaja así?
8. Un compañero del TP dice "usemos Mongo porque tiene más throughput". ¿Qué condición tiene que cumplirse para que eso sea cierto?
9. Reescribí en palabras, línea por línea, la query Cypher de la sección 5.2. ¿Qué cambiaría para excluir a Gene Hackman del resultado?
10. ¿Por qué las bases de grafos no shardean? ¿Qué implica eso en CAP?
11. Explicá la diferencia entre columnar y wide column. ¿Cuál acepta las mismas queries que Postgres?
12. ¿Qué diferencia hay entre un dump nocturno y change data capture? ¿Cuándo elegirías cada uno?
13. Ejercicio de parcial: un e-commerce necesita carrito y sesiones, catálogo de productos con atributos variables por categoría, historial de clics de cada usuario, recomendaciones "quienes compraron esto también compraron", pagos, y reportes mensuales de ventas. Elegí base para cada problema, con implementación y justificación.

---

**FIN DE LA PARTE 5 — Apunte maestro clase04 · NoSQL — FIN DE LA UNIDAD**
