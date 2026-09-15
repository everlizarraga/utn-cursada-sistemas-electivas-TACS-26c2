# 📘 Apunte maestro — clase04 · NoSQL
## Parte 4 de 5 — Las garantías: CAP, ACID, BASE y NewSQL

**Unidad:** `clase04` · clase 4 del cronograma (01/09/2026) · **Tema:** noSQL
**Partes de la unidad:** 1 · Por qué existen las NoSQL y cómo se escala — 2 · Sharding — 3 · Escribir en más de un nodo — 4 · Las garantías *(esta)* — 5 · Los cuatro tipos y cómo decidir

**Leyenda y convenciones:** las de la Parte 1 (🔴🟡🟢 · 🕳️ · ⚠️ · 📌 · 📋). Cada concepto se explica una vez; lo que ya se explicó se cita.

**Qué cubre esta parte.** Los nombres de las tensiones que se vinieron eligiendo sin nombrar. El teorema CAP (consistencia, disponibilidad, tolerancia a particiones: solo dos de tres) le pone nombre a "consistencia contra disponibilidad". ACID y BASE son los dos modelos de garantías, y le ponen nombre a "garantías fuertes contra escala". Después, qué garantías te da el motor y cuáles quedan en tu código, y NewSQL: la familia que intenta tener ACID con escala horizontal, y por qué casi nunca es la respuesta.

**Qué asume de las partes anteriores.** Consistencia como "quien lee recibe la última escritura" (Parte 3, 1.2), partición de red (Parte 3, 2.1), quorum con V1/V2 (Parte 3, 2.3), consistencia eventual (Parte 3, 3.4), el comportamiento de Cassandra "te doy el dato que tengo" (Parte 3, 3.4), y que las relacionales escalan horizontal aunque no sea lo típico (Parte 1, 5.3).

---

## 1. Teorema CAP 🔴

### 1.1 El enunciado

```
                        C  Consistencia
                       ╱ ╲
                 CA   ╱   ╲   CP
                     ╱     ╲
   Disponibilidad  A ─────── P  Tolerancia a particiones
                        AP

   Un sistema distribuido se para sobre UN lado del triángulo: CA, CP o AP.
   Nunca en los tres vértices a la vez.
```

Se lo llama también **teorema de Brewer**. Dice que en un sistema distribuido podés asegurar dos de las tres propiedades, **nunca las tres**. Las tres letras son cosas que ya aparecieron en las Partes 1 y 3; acá van con su definición formal.

### 1.2 Las tres propiedades

**C — Consistencia** (*Consistency*). Cada lectura recibe **el estado más actualizado, o un error.** No hay un estado intermedio que puedas llegar a leer. Y vale insistir en lo que ya vimos con el quorum: **no es que todos los servidores tengan el último dato**, es que **vos, como cliente, leas el último dato.** En el ejemplo V1/V2 de la Parte 3, el nodo 3 sigue con V1 y el sistema es consistente igual, porque quien lee con R = 2 se cruza con V2. Con W + R > N no hace falta que todos tengan el último dato, y es consistente.

**A — Disponibilidad** (*Availability*). Cada request, sea de lectura o de escritura, **recibe una respuesta.** No importa si es el estado más actualizado o no. En criollo: **la base te contesta.** Con el mismo ejemplo: si en vez de leer de dos nodos leo de uno solo y me toca el nodo 3, me traigo V1, un dato viejo. Perdí la consistencia, pero tuve disponibilidad: pedí algo y me lo dieron.

**P — Tolerancia a particiones** (*Partition tolerance*). El sistema **sigue funcionando aunque haya problemas de red**: particiones (los nodos incomunicados entre sí, como se definió en la Parte 3), mensajes perdidos, delays. Si se corta la comunicación entre mis servidores, el sistema sigue andando, al menos para una parte.

Una aclaración que evita confusiones: **CAP es para sistemas distribuidos.** Si tu sistema no está distribuido, si es una sola máquina, CAP no aplica: no hay red entre nodos que se pueda partir, y no hay dos réplicas que puedan diferir.

### 1.3 Dos ejemplos para fijarlo

**WhatsApp prioriza disponibilidad.** No necesariamente queremos consistencia: si justo no leímos el último mensaje, mandamos otro y se nos ordena distinto, no es tan grave. Lo que sí queremos es que la app **responda siempre**.

**Un sistema bancario prioriza consistencia.** Con transacciones de dinero **no podemos no mostrar consistencia**: sí o sí tenemos que ver siempre la plata real que hay en la cuenta. Prefiere fallar antes que mostrar un saldo viejo.

### 1.4 Dónde cae cada sistema

- **Relacionales distribuidas: CA.** ⚠️ *En la materia se enseña que los relacionales distribuidos son CA: consistencia y disponibilidad, sin tolerar particiones. En la teoría más estricta, en un sistema distribuido la partición de red no se elige, ocurre, así que la decisión real es siempre C o A cuando la red se parte. Para el parcial: relacionales distribuidas = CA.*
- **Cassandra: bien AP.** Es exactamente el comportamiento que quedó pendiente de nombrar en la Parte 3: "te doy el dato que tengo, no sé si es el último". Responde siempre y tolera particiones; la consistencia es la que se sacrifica.
- **Mongo: configurable.** Podés hacerlo más AP o más CP, según cómo lo configures (como los parámetros W y R del quorum, Parte 3).
- Cada tipo de base tiene sus características, y se ubica en el triángulo al presentar cada tipo, en la Parte 5.

> 📌 **Para el parcial, si te preguntan: ¿qué dice el teorema CAP y qué significa cada letra?**
> En un sistema distribuido solo se pueden garantizar dos de tres propiedades: consistencia (cada lectura recibe el estado más actualizado o un error), disponibilidad (cada request recibe una respuesta, sea o no la más actualizada) y tolerancia a particiones (el sistema sigue funcionando con fallas de red). Ejemplos: un banco prioriza consistencia; WhatsApp prioriza disponibilidad; Cassandra es AP, Mongo es configurable, las relacionales distribuidas son CA. CAP no aplica a un sistema que no está distribuido.

---

## 2. ACID 🔴

### 2.1 Las cuatro garantías

ACID es **el modelo clásico de las bases relacionales**: prioriza la consistencia estricta. Son cuatro garantías sobre las **transacciones** (un conjunto de operaciones que se ejecutan como una unidad), y el ejemplo para las cuatro es el mismo: **mover plata de la cuenta A a la cuenta B**, que son dos operaciones (restar en A, sumar en B).

| Letra | Garantía | Qué significa | En el ejemplo bancario |
|---|---|---|---|
| **A** | **Atomicidad** (*Atomicity*) | Una transacción con N operaciones **se ejecuta entera o no se ejecuta.** | O se resta en A y se suma en B, o no pasa nada. |
| **C** | **Consistencia** (*Consistency*) | La base **pasa de un estado consistente a otro.** Si falla en el medio, la transacción se **rollbackea** (se deshace); no puede quedar por la mitad. | Si saqué la plata de A y por algún motivo no pude sumarla en B, tengo que volver todo atrás. No puedo quedar en un estado donde la plata salió de A y nunca llegó a B. |
| **I** | **Aislamiento** (*Isolation*) | Las transacciones que corren en paralelo **no se afectan entre sí**: operan como si fueran secuenciales, primero una y después la otra. | Dos movimientos simultáneos sobre la misma cuenta dan el mismo resultado que si hubieran corrido uno después del otro. |
| **D** | **Durabilidad** (*Durability*) | Una vez que la transacción se **comitea** (se confirma), el dato **no se pierde**: persiste ante fallas. | Si el banco confirmó la transferencia y se corta la luz, la transferencia sigue hecha. |

Sobre el aislamiento, un detalle: en SQL **es configurable**. El motor te deja elegir el nivel: si permitís o no leer datos de una transacción que todavía no se completó (una *lectura sucia*: ves algo que quizás después se deshaga), o si solo leés lo que ya está comiteado. Lo habitual es esto último: **leer solamente lo que está confirmado** (el nivel *read committed*).

ACID es lo que las bases relacionales garantizan desde que existen (Parte 1, años 70): **confiabilidad.** Pero tiene un precio: dificulta la escalabilidad horizontal. ⚠️ *En la materia se enseña que ACID escala vertical y con dificultad horizontal. En la práctica existen relacionales con ACID horizontales, como Galera Cluster (Parte 1, 5.3), pero son la excepción y cuestan más. Para el parcial: ACID → vertical, difícil horizontal.*

### 2.2 Dos "C" distintas 🟡

Fijate que "consistencia" apareció con dos sentidos y conviene no mezclarlos. La **C de CAP** habla de lecturas: que quien lee reciba el último dato. La **C de ACID** habla del estado de la base: que cada transacción la deje en un estado válido, sin quedarse por la mitad. Cuando en esta clase se dice "consistencia" a secas en contexto de réplicas y nodos, es la de CAP; en contexto de transacciones, la de ACID.

> 📌 **Para el parcial, si te preguntan: ¿qué es ACID?**
> Son las cuatro garantías transaccionales del modelo relacional: atomicidad (la transacción se ejecuta entera o no se ejecuta), consistencia (la base pasa de un estado consistente a otro; si falla, rollback), aislamiento (transacciones en paralelo no se afectan, operan como si fueran secuenciales) y durabilidad (lo comiteado no se pierde). Ejemplo: una transferencia bancaria resta en A y suma en B, o no hace nada. Prioriza consistencia estricta a costa de la escalabilidad horizontal.

---

## 3. BASE 🔴

### 3.1 El juego de palabras y las tres letras

Como contrapartida de ACID (ácido) aparece **BASE** (básico). Es un juego de palabras, un poco forzado: la sigla se armó para que sonara así, y cada concepto queda medio raro. Pero muestra lo que se asocia al **NoSQL distribuido**: priorizar, generalmente y no en todos los casos, **la disponibilidad por sobre una consistencia estricta.**

| Letra | Garantía | Qué significa |
|---|---|---|
| **BA** | **Disponibilidad básica** (*Basic Availability*) | La base está disponible **la mayor parte del tiempo**. No es un 100 % asegurado, pero intenta responder siempre, aunque a veces con **datos parciales o viejos** (*stale*). |
| **S** | **Estado blando** (*Soft State*) | El estado de la base **puede cambiar con el tiempo incluso sin que nadie escriba**, por la propagación de las réplicas: un dato que hoy no está en un nodo, mañana sí, sin que llegara ninguna escritura nueva. |
| **E** | **Consistencia eventual** (*Eventual consistency*) | Si no hay más escrituras, **eventualmente todos los nodos convergen** al mismo estado. En el medio, podés leer cosas que no son las últimas. Es la misma consistencia eventual de la Parte 3, 3.4. |

Lo que BASE implica: **podemos ver datos que no son los últimos**, quedan desactualizados por un tiempo. A cambio, **el sistema escala y responde mejor**, y a su vez va a converger a la consistencia.

### 3.2 Cuándo está bien ver datos viejos

BASE aplica bien cuando ver un dato desactualizado **no afecta al negocio**. Casos concretos:

- **Estados históricos para trazabilidad.** El estado en que estuvo un producto (disponible, después comprado): se persiste para saber por dónde pasó, no hace falta que sea al instante.
- **Un like en Instagram.** Verlo desactualizado no te cambia la vida.
- **La mayoría de la información de redes sociales.** No es crucial verla en el mismo momento en que pasa.
- **Feeds.** Un post que aparece unos segundos más tarde no rompe nada.
- **Analítica.** No necesariamente tiene que ser en tiempo real.
- **El estado "online" de un usuario.** A veces ves que está online y se desconectó hace unos segundos; no es crucial ver lo último de lo último.

> 📌 **Para el parcial, si te preguntan: ¿qué es BASE y cuándo aplica?**
> Es el modelo de garantías asociado al NoSQL distribuido, que prioriza disponibilidad sobre consistencia estricta: disponibilidad básica (responde casi siempre, aunque con datos viejos), estado blando (el estado cambia con el tiempo por la propagación de réplicas) y consistencia eventual (sin nuevas escrituras, todos los nodos convergen). Aplica cuando ver un dato desactualizado no afecta al negocio: likes, feeds, estado online, analítica, trazabilidad.

---

## 4. ACID vs BASE 🔴

| Dimensión | ACID | BASE |
|---|---|---|
| **Consistencia** | Fuerte, inmediata | Eventual |
| **Disponibilidad** | Puede sacrificarse | Prioridad absoluta |
| **Transacciones** | Multi-operación, con rollback | Por operación, típicamente |
| **Escalabilidad** | Vertical; horizontal difícil ⚠️ *(ver 2.1)* | Horizontal nativa |
| **Uso típico** | Banca, inventario, reservas | Web, social, big data, analytics |

Leído en una frase: ACID tiene consistencia fuerte, y para distribuirse tiene que **sacrificar disponibilidad**, porque no puede sacrificar consistencia. A BASE le importa más la **alta disponibilidad**, y la consistencia puede ser eventual. **No hay uno mejor:** resuelven problemas distintos, y cuál corresponde lo decide el negocio.

---

## 5. Qué te da el motor y qué queda en tu código 🔴

Hay una diferencia práctica que no se ve en las tablas y pesa mucho al elegir. Transacciones, auditoría, joins, y un montón de garantías más: **los motores relacionales te las dan en la capa de base de datos**, adentro del motor. Con las NoSQL, por lo general **terminás haciéndolas en la capa de aplicación**: en el código, en el lenguaje que estés usando.

| Necesidad | En una relacional | En una NoSQL |
|---|---|---|
| **Join** | Lo hace la base. | Lo hacés vos. (Mongo hoy permite joins, pero no es una operación recomendada: no es rápida.) |
| **Auditoría** (registro de quién cambió qué y cuándo) | Te la da el motor. | Depende del motor: puede dártela o no. |
| **Transacciones multi-operación** | Nativas, con rollback. | ¿Cómo hacés una transacción entre varios documentos? Es un problema tuyo. |

ACID, visto así, es un **set de features muy maduro**: tiene un montón de cosas que las NoSQL te dan por otro lado, con menos madurez. Las bases van evolucionando y las NoSQL van sumando features, pero la conclusión práctica es directa: **si tu sistema hace muchos joins, te conviene una relacional.**

### 5.1 Postgres tiene de todo 🟡

Un caso que aparece siempre en esta discusión: Postgres. Podés meterle un JSON adentro (el tipo **JSONB**, JSON binario indexable), con lo que una relacional termina guardando documentos sin esquema fijo. Es medio hackear la base, pero funciona. Y Postgres tiene, además, caché, autenticación, tareas programadas y mucho más: es una base *polenteada*, a nivel features de lo más amplio que hay. De ahí una regla práctica muy común en la industria: **si tenés un proyecto y no sabés qué va a ser, arrancá con Postgres.** Tiene buen nivel y muchas herramientas para casi cualquier cosa.

---

## 6. NewSQL: ACID con escala horizontal, a qué precio 🔴

### 6.1 La idea

NewSQL es el intento de **mezclar lo mejor de SQL y NoSQL**: mantener las transacciones ACID, pero con la escalabilidad horizontal como la manejan las NoSQL. Es la familia que apareció nombrada en la línea de tiempo de la Parte 1 y que recién ahora se puede explicar, porque necesita todo lo anterior.

Lo que promete, en tres piezas:

| Pieza | Qué es |
|---|---|
| **Relacional clásico** | Tablas, joins, SQL estándar. |
| **Escalable por diseño** | Sharding automático (Parte 2), múltiples nodos, alta disponibilidad. |
| **ACID distribuido** | Transacciones con consistencia fuerte **entre shards**: una transacción que toca datos en distintos servidores y aun así es atómica. |

Ejemplos: **Google Spanner**, **CockroachDB**, **VoltDB**, **YugabyteDB**.

### 6.2 ¿Y CAP? Cómo hacen para "tener todo"

La pregunta obligada: si CAP es un teorema, ¿cómo pueden ofrecer las tres cosas? La respuesta es que **no las ofrecen: reducen la frecuencia del problema.** Estas bases tienen una infraestructura **muy sofisticada** (redes estables, relojes atómicos, sincronización por GPS) para que una partición de red sea **extremadamente rara**. Con eso, en la práctica **parece** que tuvieran C, A y P al mismo tiempo. En realidad, lo que hacen es **sacrificar la disponibilidad en escenarios extremos** que, con esa infraestructura, no deberían ocurrir. Y además tienen métodos configurables para elegir el nivel de consistencia o de disponibilidad: se van **moviendo dentro del triángulo CAP** según la necesidad. La sensación de que manejan los tres pilares es eso, una sensación.

⚠️ *En la materia se presenta NewSQL como "ACID distribuido, queries SQL, consistencia fuerte, sin sacrificar escala". Para el parcial, respondé con esa definición, y agregá que CAP sigue valiendo: lo que hacen es volver las particiones muy raras con infraestructura cara y sacrificar disponibilidad en los casos extremos.*

### 6.3 Por qué no se usan tanto

- **Cuestan muchísimo.** Toda esa infraestructura se paga. Comparadas con las otras bases, cuestan mucho más, y en general no amerita la diferencia de gasto por lo que ofrecen.
- **Con lo que ya existe se logra lo mismo.** Con las relacionales actuales y las NoSQL que se vienen viendo, se resuelven los mismos problemas sin gastar tanto.
- **Madurez y comunidad.** No es que sean poco confiables (las hace gente que sabe de verdad); es que **las usó poca gente**. Cuando tenés un problema con CockroachDB, andá a encontrar a alguien al que le haya pasado lo mismo; un problema de Postgres lo cruzaron miles. Con Spanner, los primeros resultados de una búsqueda son alguien preguntando si alguien lo usó alguna vez.

En una frase: **ACID con escalabilidad horizontal es un unicornio.** Se puede, pero ¿a qué costo, con qué implementación, y cómo?

> 📋 **Consigna de cátedra:** NewSQL es parte de la materia. Hay que investigarlo y entenderlo como lo que es; lo que se pide es sentido común al aplicarlo.

### 6.4 La pregunta de parcial que ya se tomó 🔴

Hay un ejercicio típico de esta materia: **te describen un sistema con distintos problemas, y tenés que decir qué base de datos usás para resolver cada uno.** En una cursada anterior alguien contestó: *"pongo todo en una NewSQL"*. Está mal, por tres motivos que valen para cualquier respuesta de ese ejercicio:

1. **No justificó el costo.** NewSQL tiene un costo gigante, y elegirla sin justificarlo es no haber entendido el problema.
2. **No nombró la implementación.** No podés decir "uso NewSQL", ni "uso una NoSQL". Tenés que decir **cuál**: Spanner, CockroachDB, la que sea.
3. **No pudo explicar cómo funciona.** Si nombrás una implementación, tenés que hablar de cómo es. Si no tenés experiencia en NewSQL y estás hablando, por ejemplo, de un banco regional, es imposible que uses NewSQL: es **matar un mosquito con un cañón sin saber cómo funciona el cañón.**

> 📌 **Para el parcial, si te preguntan: "dado este sistema, ¿qué base usás para cada problema?"**
> Para cada problema del enunciado: (1) identificá el patrón (mucha escritura, lecturas por clave, relaciones, analítica…), (2) nombrá **el tipo y la implementación concreta** (nunca "una NoSQL" a secas: el motor específico), (3) justificá con las garantías que ese problema necesita (CAP, ACID o BASE) y (4) justificá el **costo**. La relacional es el default: apartarse de ella se argumenta. "Todo en NewSQL" pierde en los cuatro puntos.

---

## 7. Lo que queda abierto 🔴

Con CAP, ACID y BASE ya tenés el vocabulario para decir qué garantiza un sistema y a qué renuncia. Lo que falta es el catálogo: **cuáles son las bases NoSQL**, qué patrón de acceso optimiza cada una, dónde cae cada una en CAP, y las reglas para elegir en un caso concreto. Con eso se puede responder el ejercicio de 6.4 sin caer en el cañón. Es la Parte 5.

---

## ✅ Checkpoint — Parte 4

*(Sin respuestas: van en el complemento de la unidad.)*

1. ¿Por qué CAP no aplica a un sistema que corre en una sola máquina?
2. Con N = 3, W = 2, R = 1: ¿el sistema es consistente en el sentido de CAP? ¿Es disponible? Justificá con V1/V2.
3. "Consistencia significa que todos los nodos tienen el mismo dato." ¿Por qué esa definición está mal para CAP?
4. Un sistema de reservas de vuelos: ¿prioriza C o A? ¿Y el contador de "vistas" de un video? Justificá.
5. ¿Qué diferencia hay entre la C de CAP y la C de ACID? Dá un ejemplo de cada una.
6. Explicá atomicidad y consistencia de ACID con la transferencia bancaria. ¿Qué pasaría sin cada una?
7. ¿Qué es el nivel de aislamiento *read committed* y qué evita?
8. ¿Qué significa "estado blando" en BASE? ¿Cómo puede cambiar una base sin que nadie escriba?
9. Nombrá tres casos donde la consistencia eventual es aceptable y uno donde no, y decí qué los distingue.
10. ¿Cómo hacen las NewSQL para que "parezca" que cumplen C, A y P? ¿Qué sacrifican en realidad?
11. Te preguntan qué base usarías para un banco regional y respondés "NewSQL". Enumerá las tres cosas que te van a objetar.

---

## Qué viene en la Parte 5

Los cuatro tipos de bases NoSQL, con su ejemplo y su lugar en CAP: clave-valor (Redis), wide column (Cassandra, Dynamo), documental (Mongo) y grafos (Neo4j). Persistencia políglota, las reglas para elegir, y OLTP vs OLAP: por qué la base operativa y la base analítica no son la misma.

---

**FIN DE LA PARTE 4 — Apunte maestro clase04 · NoSQL**
