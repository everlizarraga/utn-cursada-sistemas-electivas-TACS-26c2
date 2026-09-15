# 📘 Apunte maestro — clase04 · NoSQL
## Parte 3 de 5 — Escribir en más de un nodo

**Unidad:** `clase04` · clase 4 del cronograma (01/09/2026) · **Tema:** noSQL
**Partes de la unidad:** 1 · Por qué existen las NoSQL y cómo se escala — 2 · Sharding — 3 · Escribir en más de un nodo *(esta)* — 4 · Las garantías — 5 · Los cuatro tipos y cómo decidir

**Leyenda y convenciones:** las de la Parte 1 (🔴🟡🟢 · 🕳️ · ⚠️ · 📌 · 📋). Cada concepto se explica una vez; lo que ya se explicó se cita.

**Qué cubre esta parte.** Los dos problemas que quedaron abiertos: el activo/pasivo escribía en un solo nodo (Parte 1) y el sharding dejaba cada dato en un único servidor (Parte 2). Acá se escribe en varios nodos a la vez (activo/activo), aparece el conflicto que eso genera y las tres formas de resolverlo (consenso, gana la última escritura, quorum), se combinan sharding y replicación en un mismo sistema, y se aclara por qué replicar no es hacer backup.

**Qué asume de las partes anteriores.** Replicación como copias del mismo dato en varios nodos, activo/pasivo, replicación síncrona vs asíncrona y elección de líder (Parte 1, secciones 6 y 7). Anillo de hash consistente con virtual nodes (Parte 2, secciones 4 y 5).

---

## 1. Replicación activo / activo 🔴

### 1.1 Cómo funciona

```
                    ( Cliente )
                   ╱           ╲
        lectura y escritura    lectura y escritura
                 ▼               ▼
           ┌──────────┐ sync  ┌──────────┐
           │    A     │◄─ ─ ─►│    B     │
           │  Activo  │       │  Activo  │
           └──────────┘       └──────────┘
```

A diferencia del activo/pasivo, acá **los dos nodos son activos**: el cliente puede **escribir y leer en cualquiera de los dos**, al mismo tiempo, y los nodos se sincronizan entre sí. Con eso:

- **Escala**, porque las escrituras ya no entran por un solo lugar.
- **Es altamente disponible**, porque si cae uno, el otro sigue atendiendo lecturas y escrituras, sin failover.

### 1.2 El problema: conflictos

Es bastante evidente en el diagrama. Si dos clientes escriben a la vez **sobre la misma clave**, cada uno en un nodo distinto:

```
   A:  user_23 = "Sebastián"
   B:  user_23 = "Seba"
```

¿Cuál es el nombre del usuario 23? Los dos nodos tienen una respuesta distinta y las dos son "válidas". Ya no podemos quedarnos de brazos cruzados: **hay que hacer algo.** Y fijate el corrimiento: hasta la Parte 2 el problema era la disponibilidad (que el sistema responda aunque se caigan máquinas). Acá el problema pasa a ser **la consistencia de los datos**.

Vale definir ese término ahora, porque va a estar en toda la parte: **consistencia** es que quien lee **reciba la última escritura**. Si escribí "Seba" y después leo "Sebastián", el sistema fue inconsistente conmigo. Fijate que no dice nada de cuántos nodos tienen qué: es una garantía hacia el que lee.

### 1.3 ¿Hay un nodo que diga la verdad?

Pregunta natural: en activo/activo, ¿me interesa tener un nodo que sea la "fuente de la verdad", o me fijo en momentos de baja carga que no haya inconsistencias? La respuesta es que **depende de la estrategia que elijas para resolver el conflicto**: en algunas hay un nodo que dice la verdad, en otras no hay ninguno. Las tres estrategias son la sección 2.

Un contraste que ayuda a ubicarse: una base relacional te da la **consistencia por diseño**, porque cada escritura se confirma completa o no se confirma. Pero por eso mismo el esquema "escribo en cualquier lado y leo de cualquier lado" **no escala tan bien** en una relacional: cada escritura tendría que confirmarse en todos los nodos antes de darse por hecha, para que no se cuele nada inconsistente. Las estrategias que siguen son, justamente, distintas maneras de relajar o de pagar ese costo.

> 📌 **Para el parcial, si te preguntan: ¿qué gana y qué problema trae la replicación activo/activo?**
> Gana escalabilidad y alta disponibilidad: ambos nodos sirven lectura y escritura al mismo tiempo, y no hay un único punto de escritura. El problema son los conflictos: dos escrituras concurrentes sobre la misma clave en nodos distintos dejan valores distintos, y hace falta una estrategia para decidir cuál gana. El problema deja de ser la disponibilidad y pasa a ser la consistencia.

---

## 2. Resolución de conflictos: cuál gana 🔴

Cuando dos nodos aceptan escrituras, hace falta decidir cuál gana. Hay tres formas, y van de la más estricta a la más laxa.

### 2.1 Consenso (Raft, Paxos)

Los nodos **se ponen de acuerdo antes de escribir**. Los algoritmos de consenso más usados son **Raft** y **Paxos**. La mecánica, a grandes rasgos: hay un **líder** que coordina, y una escritura se **compromete por mayoría**: solo se da por hecha cuando la mayoría de los nodos la aceptó.

Lo que te da:
- **Consistencia fuerte** y un **orden de operaciones**: todos ven las escrituras en el mismo orden.

Lo que te cuesta:
- **Más latencia**, porque cada escritura espera el acuerdo.
- **Pérdida de disponibilidad cuando hay una partición de red.** Una *partición de red* es cuando los nodos quedan **incomunicados entre sí** aunque cada uno siga vivo: la red se cortó en dos (o más) islas. Si el algoritmo necesita mayoría y una isla no la tiene, esa isla **no puede escribir**: prefiere no responder antes que responder mal. Al ir por consenso, estás **sacrificando disponibilidad**.

Ojo con la palabra: "partición de red" no tiene nada que ver con el particionamiento por negocio de la Parte 1. Aquello era una decisión de diseño; esto es una falla.

> 🕳️ **Madriguera — Raft y Paxos por dentro**
> Son protocolos con elección de líder, términos (mandatos numerados), logs replicados y votaciones por mayoría. Para esta materia no hace falta saber cómo funcionan; el sitio de Raft tiene una visualización interactiva que lo hace entendible en diez minutos, si querés chusmear.
> *Volvé al camino — esto se profundiza aparte, otro día.*

### 2.2 Timestamp: gana la última escritura (*last write wins*)

Una forma mucho más sencilla: **cada escritura lleva un timestamp** (la marca de tiempo del momento en que se hizo), y **la más reciente pisa a la anterior**. Cuando hay más de un valor, gana el de fecha más nueva.

Lo que te da:
- **Es simple.**
- **Latencia baja** y **disponibilidad alta**: nadie espera a nadie para escribir.

Lo que te cuesta:
- **Perdés consistencia.** Podés perder **updates válidos**: si dos updates son concurrentes, uno pisa al otro y lo que decía el otro desaparece, aunque fuera legítimo. En el ejemplo, "Seba" con un timestamp un milisegundo más nuevo pisa a "Sebastián", y nadie se entera de que "Sebastián" existió.
- **Requiere relojes sincronizados** entre los nodos: si cada máquina tiene la hora distinta, "la última" no es la última.

Eso no significa que no sirva para ningún caso. Hay casos en los que **no me importa tanto la consistencia** y puedo sacrificar un poquito. Depende del negocio y del caso de uso.

Un detalle de implementación que suele confundir 🟡: con timestamp puro, en una arquitectura activo/activo, **escribís en un solo nodo** (el que te tocó, A o B) y le ponés la marca de tiempo. El desempate se hace al leer: **leés de todos y te quedás con el más nuevo.** Por eso, comparado con lo que sigue, el timestamp te obliga a leer más nodos para estar seguro. Le conviene a un sistema con muchas escrituras y pocas lecturas. Cómo se implementa exactamente depende del motor.

### 2.3 Quorum: W + R > N

La tercera estrategia es la que más cuesta ver y la más usada. Se definen tres números:

| Letra | Qué es |
|---|---|
| **N** | Cuántas **réplicas** tengo de cada dato: la cantidad de nodos. |
| **W** | En cuántos nodos tengo que **escribir** (recibir confirmación) para dar una escritura por hecha. |
| **R** | De cuántos nodos tengo que **leer** para dar una lectura por válida. |

Y la ecuación:

> **Si W + R > N, puedo asegurar consistencia.**

#### El caso: N = 3, W = 2, R = 2

Tengo tres réplicas. Configuro: cada escritura tiene que quedar confirmada en al menos dos nodos, y cada lectura tiene que consultar al menos dos nodos. 2 + 2 = 4 > 3, así que la ecuación dice que voy a tener consistencia. Veamos por qué, paso a paso.

```
  PASO 1 — Escritura exitosa: escribo el dato "V2"
  Se confirma en un quorum de 2 nodos (1 y 2). El nodo 3 se queda con el dato viejo "V1".

      ●(1)      ●(2)      ○(3)
      V2        V2        V1
      └─ quorum de escritura, W = 2 ─┘

  PASO 2 — Lectura posterior: consulto 2 nodos
  Me tocan, por ejemplo, el nodo 2 y el nodo 3.

      ○(1)      ●(2)      ●(3)
                V2        V1
                └─ quorum de lectura, R = 2 ─┘

  PASO 3 — Intersección: el nodo 2 se solapa
  La lectura ve "V2" y "V1". Comparando timestamps, el cliente sabe que "V2" es
  el más nuevo. Consistencia asegurada.

      ○(1)      ◉(2)      ●(3)
                 ▲
           está en los dos quorums
```

Es el **principio del palomar** aplicado a réplicas: si escribí en 2 de 3 y leo de 2 de 3, **no hay forma de elegir dos nodos para leer sin que al menos uno sea de los que tienen la escritura**. Hay intersección garantizada entre el conjunto de escritura y el de lectura. Por más que una de mis lecturas traiga un dato viejo, la otra trae el último, y con timestamps sé cuál es cuál.

Funciona igual con otras combinaciones. Si escribo en **1** y leo de los **3** (1 + 3 = 4 > 3): por más que lea dos datos viejos, entre los tres está el último. Si escribo en los **3** y leo de **1** (3 + 1 = 4 > 3): cualquier nodo que lea tiene el último. Lo que no funciona es, por ejemplo, W = 1 y R = 1: puedo escribir en el nodo 1 y leer del nodo 3, y nunca cruzarme con la escritura.

#### Cuatro cosas que hay que tener claras

**W y R son cantidades, no cuáles.** Decís *en cuántos* escribís y *de cuántos* leés, no *en cuáles*. Qué nodos te tocan es aleatorio, no rotan en orden. El ejemplo del paso 2 lee del 2 y del 3, pero podría haber leído del 1 y el 2, o del 1 y el 3; y la escritura podría haber caído en el 1 y el 3. La garantía vale para **cualquier** combinación, y por eso es una garantía.

**Consistente no es "todos tienen el último dato".** En el paso 3 el sistema es consistente y el nodo 3 sigue con "V1". Con W + R > N **no hace falta que todos los nodos tengan el último dato**; hace falta que el que lee lo encuentre. Es exactamente la definición de la sección 1.2.

**Se configura según cómo se usa la base.** Escribir en tres y leer de uno funciona, pero ¿para qué escribir tres veces si escribiendo dos y leyendo dos alcanza? Escribir cuesta. Entonces se elige según qué es más caro para vos: si la base **se lee mucho y se escribe poco**, conviene **R bajo** (leer de menos nodos) y **W alto**; si **se escribe mucho y se lee poco**, al revés. Son parámetros configurables del motor: los tenés en Mongo y en Cassandra (dos bases NoSQL que se ven en la Parte 5).

**A veces ni siquiera buscás consistencia.** Dependiendo del caso de uso puede no interesarte, y la base te deja configurar **W = 1, R = 1**. Es válido; solo que renunciás a la garantía.

Un caso particular que aparece mucho: configurar **mayoría** para las dos, es decir **W = R = ⌊N/2⌋ + 1** (la mitad redondeada hacia abajo, más uno). Con N = 3, eso es 2 y 2, el ejemplo de arriba. Cumple la desigualdad y es el balance típico entre performance y consistencia. Pero la regla general es la desigualdad, no la mayoría.

> 📋 **Info de cursada.** Estos parámetros se ven en profundidad en *Datos NoSQL*, electiva de quinto, donde se trabaja Mongo a fondo.

> 📌 **Para el parcial, si te preguntan: ¿por qué W + R > N garantiza consistencia?**
> Porque si escribo en W nodos y leo de R nodos, y W + R supera el total N, los dos conjuntos se intersecan obligatoriamente (principio del palomar): al menos uno de los nodos que leo tiene la última escritura. Comparando timestamps entre lo que traigo, me quedo con el más nuevo. Ejemplo: N = 3, W = 2, R = 2. No exige que todos los nodos tengan el último dato, solo que el que lee lo encuentre. W y R son cantidades, no nodos fijos.

### 2.4 Las tres, una al lado de la otra

| | Consenso (Raft, Paxos) | Timestamp (last write wins) | Quorum (W + R > N) |
|---|---|---|---|
| **Cómo decide** | Acuerdo por mayoría antes de escribir, con un líder. | La escritura con timestamp más nuevo pisa. | Escribo en W, leo de R; la intersección trae el último. |
| **Consistencia** | Fuerte, con orden de operaciones. | Se pierde: updates válidos pueden pisarse. | Asegurada si W + R > N. |
| **Latencia** | Alta. | Baja. | Intermedia, configurable. |
| **Disponibilidad** | Se pierde ante partición de red. | Alta. | Configurable. |
| **Requisito** | Mayoría alcanzable. | Relojes sincronizados. | Elegir W y R según carga. |
| **Cuándo** | No puedo permitir inconsistencia. | Puedo sacrificar consistencia; muchas escrituras. | Balance entre performance y consistencia. |

### 2.5 Un principio que aparece detrás: duplicar información 🟡

El quorum con W > 1 guarda el mismo dato más de una vez. ¿No es desperdiciar memoria? Sí, y es a propósito. Hay un principio, no siempre exacto pero útil:

> **Las bases relacionales cuidan el almacenamiento. Las no relacionales cuidan el cómputo.**

Normalizar una relacional (no repetir información, como enseñan las formas normales) hace que no guardes nada dos veces; pero cuando tenés que hacer un **join**, es **computacionalmente pesado**. Las no relacionales van al revés: **no les importa guardar el dato dos veces, les importa accederlo rápido.** El costo aparece después: al duplicar información tenés que **mantenerla en los dos lados**. Cada vez que actualizás, no podés actualizar una copia y la otra no: tenés que estar atento a que siempre que cambia una, cambie la otra.

### 2.6 Estas estrategias son para activo/activo, y se combinan

Las tres son resoluciones de conflictos **para cuando hay más de un nodo donde escribir**. Con un solo punto de entrada (activo/pasivo) técnicamente no hacen falta.

Y ninguna de las estrategias vistas hasta acá es "superadora" de otra: **se combinan**. Podés tener activo/pasivo y a la vez sharding. Cassandra combina replicación y sharding; Mongo combina quorum y sharding; un esquema relacional puede combinar activo/activo con replicación. No es una u otra: es qué mezcla le sirve a cada sistema.

---

## 3. Sharding + replicación: cómo se arma un sistema real 🔴

En sistemas reales se suelen usar los dos combinados: el sharding de la Parte 2 para repartir, y la replicación para no depender de un solo nodo por dato.

### 3.1 El coordinador y el replication factor

```
                        key "K"
                          │
                          ▼
                  (Server 6)   (Server 1) ◄── dueño de K
               ╱                      ╲  replica
        (Server 5)                    (Server 2)
               ╲                      ╱  replica
                  (Server 4)   (Server 3)

   Replication factor = 3
   Los servers 1, 2 y 3 guardan los datos de las keys
   del arco entre el server 6 y el server 1.
   Los servers 2, 3 y 4 guardan los datos de las keys
   del arco entre el server 1 y el server 2.
```

Cuando llega una operación hay un nodo que actúa como **coordinador**: recibe la operación y decide dos cosas. Primero, **en qué shard cae la key**, con el hash sobre el anillo, como en la Parte 2. Segundo, **a qué réplicas tiene que ir**, porque hay un **replication factor**: cuántas veces se guarda cada dato. Con replication factor 3, el dato de la key K se guarda en su dueño (server 1) y además en las **dos réplicas siguientes en sentido horario** (servers 2 y 3). Cada server, entonces, guarda sus propios datos y los de sus vecinos anteriores.

A partir de ahí aparecen tres decisiones clave.

### 3.2 ¿Sincrónica o asincrónica?

Las dos funcionan, con consistencia y latencia muy distintas. Es la misma decisión de la Parte 1 (sección 7.3), ahora con varias réplicas:

- **Síncrona:** tengo que esperar que **todas las réplicas confirmen** que escribieron. Es más consistente, pero **más lenta** y **con menos disponibilidad** (si una réplica no responde, la escritura se traba).
- **Asíncrona:** te respondo rápido y **después** replico la información. Ganás latencia, pero durante ese rato hay réplicas atrasadas.

La decisión es una sola: **sacrifico latencia o sacrifico consistencia.**

### 3.3 ¿Cuántas réplicas?

El replication factor define **durabilidad** (que el dato no se pierda: cuantas más copias, más difícil perderlo) y **costo** (cada copia ocupa disco y hay que escribirla). Más réplicas, más seguro y más caro.

### 3.4 ¿De dónde leer?

Hay distintas estrategias según el modelo del motor:

| Estrategia | Cómo | Qué obtengo |
|---|---|---|
| **Con líder** (*leader-based*) | Leo siempre del **primario**, el nodo que recibió la escritura. | Consistencia: el primario siempre tiene el último dato. |
| **De réplicas** | Leo de cualquier réplica. | Es **más rápido**, pero tengo **consistencia eventual**. |
| **Sin líder** (*leaderless*) | Leo de **varias** réplicas. | Consistencia si combino bien lecturas y escrituras: **quorum, W + R > N**. |

**Consistencia eventual** merece su línea: es cuando el sistema garantiza que las réplicas *van a* converger al último valor, pero no *cuándo*. Aparece al leer de réplicas con escritura asíncrona: puede ser que el dato ya se haya actualizado, pero **todavía no en el nodo del que estoy leyendo**. Leo algo viejo, y un rato después, en ese mismo nodo, leo lo nuevo.

**Sin líder** es el esquema de Cassandra: no hay un primario; leés de varias réplicas y, si querés consistencia, configurás R y W para que sumen más que N. ¿Se lee de todas o de la mayoría? Depende de cómo lo configures: tenés N servidores, y R y W los definís vos. Según eso vas a leer de X shards y te vas a asegurar el último dato, o no.

Hay motores, Cassandra entre ellos, cuya filosofía por defecto es la de leer de cualquier réplica: **"te doy el dato que tengo; no sé si es el último."** Priorizan responder antes que responder con lo último. Retené este comportamiento, porque tiene nombre y se lo damos en la Parte 4.

> 📌 **Para el parcial, si te preguntan: ¿de dónde se lee en un sistema con réplicas y qué se gana en cada caso?**
> Con líder: se lee del primario y se garantiza el último dato. De réplicas: es más rápido pero da consistencia eventual, porque con escritura asíncrona la réplica puede no tener todavía la última escritura. Sin líder (como Cassandra): se lee de varias réplicas y la consistencia se asegura configurando quorum, W + R > N.

### 3.5 Qué te da la combinación

Con hash consistente y virtual nodes, si un servidor se cae la carga se reparte entre varios y no colapsa ninguno (Parte 2). Con replicación encima, además, **si se cae un servidor no se pierde la data**, porque está escrita también en otros nodos. Eso es lo que resuelve el problema abierto al final de la Parte 2: con sharding solo, si se te cae la conexión y perdés el acceso al server C, la data que estaba en C **la perdés** (si lo apagás vos, a propósito, podrías repartir sus datos antes; si se cae, no). Con la información replicada, entrás por otro nodo y ahí está.

---

## 4. Replicación no es backup 🔴

Pregunta obligada: si tengo la data replicada en tres nodos, ¿ya tengo backup? **No.** Las políticas de backup son cosa separada; no podés tener esto y decir que tenés un backup.

```
   REPLICACIÓN                                BACKUP
   "si me cayó un nodo,                       dos piezas:
    la info está en otro"                     1. backup: tomar la información y
                                                 dumpearla (volcarla a un archivo
   asegura no perder data                        o snapshot, guardado aparte)
   MIENTRAS un servidor está caído            2. restore / recovery: volver a
                                                 cargar ese snapshot en una base
   no hay snapshot, no hay restore
```

Un backup tiene dos partes: el **backup** como tal, agarrar la información y *dumpearla* (volcarla entera a un archivo, un snapshot de la base, guardado en otro lado), y el **restore** o *recovery*, el mecanismo para volver a levantar la base desde ese snapshot. La replicación no tiene nada de eso: no hay una foto de la base almacenada aparte ni un mecanismo para restaurarla. Lo que te asegura es otra cosa: **no perder data cuando un servidor está caído**, porque la información está escrita también en otros nodos. Si borrás un dato por error, la replicación lo borra prolijamente en todas las réplicas. El backup es el que te lo devuelve.

> 📌 **Para el parcial, si te preguntan: ¿la replicación reemplaza al backup?**
> No. La replicación asegura no perder data mientras un nodo está caído, porque el dato está escrito en otros nodos. Un backup es un snapshot de la base guardado aparte más un mecanismo de restore para volver a cargarlo; la replicación no tiene ni snapshot ni restore. Son políticas separadas.

---

## 5. Lo que queda abierto 🔴

En esta parte aparecieron tres tensiones que se repiten en cada decisión: consistencia contra latencia (síncrono o asíncrono), consistencia contra disponibilidad (consenso pierde disponibilidad en una partición de red; Cassandra prefiere responder aunque no sea lo último), y garantías fuertes contra escala (la relacional consistente por diseño que no escala tan bien en activo/activo). Se estuvo eligiendo de qué lado pararse en cada una, sin ponerle nombre. La Parte 4 se lo pone, y ordena qué garantiza cada familia de bases y a qué renuncia.

---

## ✅ Checkpoint — Parte 3

*(Sin respuestas: van en el complemento de la unidad.)*

1. ¿Qué diferencia práctica hay entre activo/pasivo y activo/activo para un cliente que escribe? ¿Y para uno que lee?
2. Definí consistencia como se usa en esta parte. ¿Por qué "todos los nodos tienen el mismo dato" no es la definición?
3. ¿Por qué el consenso pierde disponibilidad ante una partición de red? ¿Qué preferiría hacer un nodo aislado: responder o callarse?
4. ¿Qué es una partición de red y en qué se diferencia del particionamiento de la Parte 1?
5. Con *last write wins*, dos clientes actualizan el mismo carrito con un milisegundo de diferencia. ¿Qué pasa con el update del primero? ¿Es aceptable? ¿En qué negocio sí y en cuál no?
6. N = 5. Proponé dos configuraciones de W y R que garanticen consistencia y una que no. Justificá con el palomar.
7. Un sistema se lee muchísimo y se escribe poco. ¿Cómo configurarías W y R? ¿Por qué?
8. "Con quorum, todos los nodos terminan con el último dato." ¿Verdadero o falso? Explicá con el ejemplo de V1 y V2.
9. ¿Qué hace el coordinador cuando llega una escritura a un sistema con sharding y replication factor 3?
10. ¿Qué es la consistencia eventual y en qué estrategia de lectura aparece?
11. Borraste por error una tabla completa en un sistema con replication factor 3. ¿La replicación te salva? ¿Qué te salvaría?

---

## Qué viene en la Parte 4

Las garantías: el teorema CAP (consistencia, disponibilidad, tolerancia a particiones: solo dos de tres), ACID vs BASE como los dos modelos de garantías, dónde conviene cada uno, qué garantías da el motor y cuáles quedan en la aplicación, y NewSQL como el intento de tener ACID con escala horizontal.

---

**FIN DE LA PARTE 3 — Apunte maestro clase04 · NoSQL**
