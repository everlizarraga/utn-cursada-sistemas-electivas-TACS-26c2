# 📘 Apunte maestro — clase04 · NoSQL
## Parte 2 de 5 — Sharding

**Unidad:** `clase04` · clase 4 del cronograma (01/09/2026) · **Tema:** noSQL
**Partes de la unidad:** 1 · Por qué existen las NoSQL y cómo se escala — 2 · Sharding *(esta)* — 3 · Escribir en más de un nodo — 4 · Las garantías — 5 · Los cuatro tipos y cómo decidir

**Leyenda y convenciones:** las de la Parte 1 (🔴🟡🟢 · 🕳️ · ⚠️ · 📌 · 📋). Cada concepto se explica una vez; lo que ya se explicó se cita.

**Qué cubre esta parte.** La segunda forma de *descomponer* los datos entre máquinas: repartirlos **por cálculo**, sin criterio de negocio. Primero qué es una función de hash, y después los tres approaches de sharding, cada uno resolviendo el problema que dejó el anterior: módulo N, hash consistente y hash consistente con virtual nodes.

**Qué asume de la Parte 1.** Escalar horizontal es sumar máquinas, y hay dos formas de repartirles el trabajo: clonar los datos o descomponerlos (sección 5.2). El particionamiento por negocio (sección 8) es la primera forma de descomponer, y su límite es que necesita un criterio natural del dominio. Esta parte ataca ese límite.

---

## 1. Sharding: repartir por cálculo, no por negocio 🔴

En la Parte 1, para dividir los datos de Argentina y Brasil hacía falta que el negocio tuviera esa frontera. Acá la idea es otra:

> **Sharding:** distribuir los datos en N servidores **sin depender del negocio**. La asignación **se calcula, no se modela.**

En vez de mirar el dominio y decidir "esto va acá", se toma una función que, dado un dato, devuelve a qué servidor va. Y esa función reparte los datos automáticamente entre los servidores. Lo que se gana:

- **Escalar horizontal sin pensar cómo dividir el dominio.** No hace falta encontrar una frontera de negocio.
- **Un balanceo de carga más eficiente.** Ya no se habla de cliente grande y cliente chico: se busca directamente una buena distribución de los datos entre las máquinas.

Este mecanismo es el que usan muchas bases NoSQL distribuidas. Hay tres approaches de implementación, y se presentan en orden porque cada uno arregla lo que el anterior dejó mal:

| # | Approach | En una línea |
|---|---|---|
| 1 | **Hash distribuido** | Módulo simple. Rápido, pero frágil ante cambios. |
| 2 | **Hash consistente** | Un anillo de hashes. Minimiza el rebalanceo. |
| 3 | **Hash + virtual nodes** | Distribución pareja al caer o sumar nodos. |

Antes de los tres, la herramienta que usan todos.

---

## 2. Qué es una función de hash 🔴

### 2.1 El caso: un DNI, tres servidores

Tengo tres servidores y quiero guardar personas identificadas por DNI. Necesito una regla que, mirando el DNI, me diga en cuál de los tres servidores va. Una regla cualquiera: *tomo el último dígito y calculo el resto de dividirlo por 3*. Con el DNI 31674167: el último dígito es 7, y 7 dividido 3 tiene resto 1. Ese DNI va al servidor 1. Y siempre que aparezca el DNI 31674167, la regla va a dar 1. Eso es una función de hash aplicada a este problema.

### 2.2 La definición práctica

Una **función de hash** tiene una definición matemática formal; la que importa acá es la práctica: **va de un dominio grande a una imagen chica.** La entrada es algo con muchísimos valores posibles; la salida, algo con pocos.

- **Par o impar** es una función de hash: el dominio son todos los enteros (0, 1, 2, 3, 4, 5, 6…), infinitos; la imagen es {par, impar}, dos valores.
- **De la IP de un cliente a un servidor**: el dominio son todas las IPs posibles; la imagen, los servidores que tengo. Una función malísima pero válida: sumar todos los dígitos de la IP, dividir por algo, tomar el resto, y ese resto elige el servidor.

Dos propiedades que hacen que sirva para repartir datos:

1. **Cae dentro de un conjunto fijo.** Si tengo diez servidores configurados, la función tiene que devolver siempre un número entre 0 y 9: cada salida es un servidor que existe. (El conjunto se puede cambiar, pero en cada momento es fijo.)
2. **Es determinista:** el mismo X va **siempre** al mismo lugar. Si el DNI 31674167 hoy cae en el servidor 1, mañana tiene que caer en el servidor 1, o no lo voy a encontrar. Esto es lo que garantiza que después puedas *buscar* el dato: aplicás la misma función y sabés dónde está.

Notá lo que la función **no** es: no es uno a uno. Muchísimos DNIs distintos caen en el mismo servidor, y eso es a propósito: la gracia es justamente ir de muchos a pocos.

### 2.3 Sharding no es sinónimo de hash

En implementaciones productivas vas a terminar usando una función de hash, pero **podrías usar el criterio que quieras**. Es muy común, por ejemplo, shardear **por día**: la "función" es el día del mes en que se creó el registro, y tenés 31 particiones lógicas. Por mes, tenés 12. Por primera letra del apellido, tenés las letras del abecedario. (*Particiones lógicas*: divisiones del conjunto de datos; no necesariamente hay una máquina física por cada una.)

El problema aparece cuando el criterio reparte mal. Si shardeás por primera letra del apellido, **nadie cae en la W** y montones caen en la G. Entonces necesitás una función que haga que los nodos tengan **carga más o menos equitativa**, que sea justa, como una especie de *load balancing* (repartir el trabajo parejo entre máquinas) aplicado a los datos. Y que además sea determinista: que X vaya siempre al mismo lado. Una función de hash bien elegida cumple las dos.

> 📌 **Para el parcial, si te preguntan: ¿qué diferencia hay entre particionamiento y sharding?**
> El particionamiento divide los datos por un criterio de negocio (región, cliente): la división se modela mirando el dominio. El sharding los distribuye por cálculo, típicamente con una función de hash sobre una key, sin depender del negocio: la asignación se calcula. El sharding permite escalar horizontal sin pensar cómo dividir el dominio y busca una distribución de carga pareja.

---

## 3. Approach 1 — Hash distribuido (módulo N) 🔴

### 3.1 La idea

Cada elemento tiene una **key** que se puede hashear (el DNI, un ID, una IP). Se calcula **hash módulo N**, donde N es la cantidad de servidores, y el resto es el servidor. Con la regla de la sección 2.1, tres servidores numerados 0, 1 y 2, y la key "último dígito del DNI":

```
   31674167          key = último dígito = 7
                     7 mod 3 = 1
                     → servidor 1
```

Numerar los servidores desde 0 tiene una gracia: **el resto es directamente el número de servidor.** Resto 0 → servidor 0, resto 1 → servidor 1, resto 2 → servidor 2.

### 3.2 Cómo se reparten quince DNIs

| Servidor 0 (último dígito mod 3 = 0) | Servidor 1 (mod 3 = 1) | Servidor 2 (mod 3 = 2) |
|---|---|---|
| 7846113**3** | 1403648**7** | 7213356**2** |
| 3717903**0** | 7659373**4** | 3020959**8** |
| 6459782**3** | 9297107**7** | 2723677**2** |
| 8845036**9** | 9418283**1** | 6369873**8** |
| 3292179**6** | 5222920**4** | 2331803**5** |

Los que terminan en 0, 3, 6 o 9 (múltiplos de 3) van al servidor 0; los que tienen resto 1 (terminan en 1, 4, 7) al servidor 1; los que tienen resto 2 (terminan en 2, 5, 8) al servidor 2. Es **simple, rápido y fácil de implementar**. Y en este caso quedó una distribución medianamente pareja, cinco por servidor, con buena performance.

### 3.3 Por qué en la práctica no escala

Hasta acá parece ideal. Hay dos razones por las que no lo es, una menor y una de fondo.

**La menor: la distribución podría salir mal.** Podrían venir justo todos los DNIs terminados en 0, 3, 6 y 9, y caer todos en el servidor 0. Es difícil que pase: si la key es lo bastante aleatoria, es poco probable. Y ojo con la key elegida: con el último dígito solo hay diez valores posibles, así que la granularidad es baja; con un hash sobre el DNI entero sería más fina. Pero este no es el problema real.

**La de fondo: N está adentro de la función.** La cantidad de servidores forma parte del cálculo. Cambia N, y cambia dónde cae **todo**. Y N cambia por dos motivos igual de cotidianos: se cae un servidor, o querés agregar uno.

**Se cae el servidor 1.** Quedan dos, y la función pasa a ser mod 2: último dígito par → servidor 0, impar → servidor 2. Con los mismos quince DNIs:

| Antes: servidor 0 | Antes: servidor 1 *(caído)* | Antes: servidor 2 |
|---|---|---|
| 78461133 → **se va al 2** | 14036487 → al 2 | 72133562 → **se va al 0** |
| 37179030 → queda en 0 | 76593734 → al 0 | 30209598 → **se va al 0** |
| 64597823 → **se va al 2** | 92971077 → al 2 | 27236772 → **se va al 0** |
| 88450369 → **se va al 2** | 94182831 → al 2 | 63698738 → **se va al 0** |
| 32921796 → queda en 0 | 52229204 → al 0 | 23318035 → queda en 2 |

Los cinco del servidor caído tenían que moverse sí o sí; eso es inevitable. El problema es el resto: del servidor 0 se van tres de cinco, del servidor 2 se van cuatro de cinco. **Doce de los quince DNIs cambian de servidor**, y en general con esta operación migran alrededor de dos tercios de los datos. Datos que estaban perfectamente bien ubicados se mueven de una máquina sana a otra máquina sana, solo porque cambió la fórmula. A eso se le llama **tormenta de re-hashing**, y tiene un costo altísimo de performance: mover datos entre máquinas mientras el sistema sigue atendiendo.

**Se agrega un servidor.** Paso de tres servidores a cuatro (0, 1, 2 y el nuevo 3) y la función pasa a ser mod 4. El problema es simétrico:

| Antes: servidor 0 | Antes: servidor 1 | Antes: servidor 2 | Nuevo: servidor 3 |
|---|---|---|---|
| 78461133 → **al 3** | 14036487 → **al 3** | 72133562 → queda | *recibe:* |
| 37179030 → queda | 76593734 → **al 0** | 30209598 → **al 0** | 78461133 |
| 64597823 → **al 3** | 92971077 → **al 3** | 27236772 → queda | 64597823 |
| 88450369 → **al 1** | 94182831 → queda | 63698738 → **al 0** | 14036487 |
| 32921796 → **al 2** | 52229204 → **al 0** | 23318035 → **al 1** | 92971077 |

Once de quince se mueven, y solo cuatro terminan en el servidor nuevo: los otros siete se reacomodan entre servidores que ya existían. Mismo problema, distinta dirección.

**Veredicto:** el hash distribuido es rápido pero frágil ante cambios. Como agregar y perder servidores es parte de operar un sistema, **no escala operativamente**.

> 📌 **Para el parcial, si te preguntan: ¿por qué el sharding por módulo N no escala?**
> Porque la cantidad de servidores N está dentro de la función de asignación (hash mod N). Cuando cae o se agrega un servidor cambia N, cambia la función, y prácticamente todos los datos se reasignan, incluso los que estaban en servidores sanos: una tormenta de re-hashing con un costo enorme de performance. Es simple y rápido, pero frágil ante cualquier cambio en la cantidad de nodos.

---

## 4. Approach 2 — Hash consistente 🔴

### 4.1 La idea: hashear también los servidores, sobre un anillo

Cambia un poco la idea. Ahora **no solo hasheo las keys: también hasheo los servidores** (por ejemplo, su IP). Y en vez de tomar el resto módulo N, pienso el espacio de salida del hash como un **anillo**, un círculo imaginario. Cada hash, sea de un dato o de un servidor, cae en un punto del anillo.

La regla de asignación: **cada dato va al servidor inmediatamente posterior en el anillo, en sentido horario.** Arrancás desde el punto donde cayó el dato, girás en sentido horario, y el primer servidor que encontrás es el suyo.

```
                        C ●
                     ╱       ╲
              Kate ○           ○ John
              ╱                    ╲
             │                      │
      Jane ○ │                      │ ○ Steve
             │                      │
              ╲                    ╱
              A ●                 ● B
                 ╲             ╱
                     ○ Bill

   ● servidor   ○ dato   → cada dato va al primer ● girando en sentido horario ↻
```

Orden en el anillo, en sentido horario desde arriba: C, John, Steve, B, Bill, A, Jane, Kate, y de vuelta a C. Aplicando la regla:

| Dato | Girando en sentido horario, el primer servidor es… | Servidor |
|---|---|---|
| John | Steve (dato), después **B** | B |
| Steve | **B** | B |
| Bill | **A** | A |
| Jane | Kate (dato), después **C** | C |
| Kate | **C** | C |

El sentido de giro es una **convención**: podría ser antihorario y funcionaría igual. Lo importante es que **el criterio sea siempre el mismo**, porque de eso depende poder encontrar el dato después.

### 4.2 Qué pasa al agregar o sacar un servidor

Acá está la gracia. Como los servidores también tienen su lugar en el anillo, **agregar o sacar uno solo mueve una parte de los datos**: los que quedan entre el servidor afectado y su vecino anterior. Todo lo demás queda igual.

**Agrego un servidor D entre John y Steve.** El único dato cuyo "primer servidor en sentido horario" cambia es John, que ahora encuentra a D antes que a B. Steve sigue en B, Bill en A, Jane y Kate en C. **Se movió un dato.** Compará con el mod N, donde agregar un servidor movía once de quince.

**Se cae C.** Jane y Kate, que estaban en C, siguen girando y el próximo servidor que encuentran es B. Bill sigue en A; John y Steve siguen en B.

- **Solo los datos de C tienen que moverse.** El resto queda intacto en sus servidores.
- **Pero todos los huérfanos caen en el mismo servidor.** B queda con John, Steve, Jane y Kate (cuatro datos) y A con Bill (uno). Se acaba de heredar todo el peso de C, y la carga queda desbalanceada.

Es mejor que mod N, porque no hay que redistribuir todo para todos lados: se **re-hashean pocos elementos**, y ahí es donde el hash consistente le gana al módulo. Pero el servidor consecutivo hereda todo, y el desbalanceo sigue siendo un problema. Todavía no es lo ideal.

### 4.3 ¿Los servidores quedan repartidos parejos en el anillo? 🟡

Lo ideal sería que los servidores queden **equidistantes** en el anillo, así cada uno cubre un arco parecido. Y podés empezar así: con una key que se distribuye de manera uniforme, como los DNIs, podés ubicar un servidor que cubra los últimos dígitos 0 a 3, otro 4 a 6, otro 7 a 9. Pero hay dos límites:

- **Equidistante no garantiza parejo.** Nada te asegura que caigan la misma cantidad de datos entre el 7 y el 9 que entre el 0 y el 3. Los arcos pueden ser iguales y la carga no.
- **Al agregar un servidor, se rompe.** Empezás equidistante; agregá uno, y decime de qué manera sigue siendo equidistante. Nunca puede serlo. Si tengo A, B y C y agrego D cerca de C, alivio a C, pero B sigue con la misma carga. Agregar un nodo no redistribuye la carga entre todos.

Lo que el hash consistente te da es exactamente esto: **cuando sacás o agregás un nodo, re-hasheás pocos datos.** Si además querés que la carga quede **equilibrada**, este approach no te alcanza, y necesitás el tercero.

> 📌 **Para el parcial, si te preguntan: ¿qué resuelve el hash consistente y qué problema le queda?**
> Hashea tanto las keys como los servidores sobre un anillo, y cada dato va al servidor inmediatamente posterior en sentido horario. Al agregar o sacar un servidor solo se mueven los datos de ese sector: minimiza el rebalanceo, a diferencia del módulo N. Lo que le queda: cuando cae un servidor, todos sus datos van al mismo servidor vecino, que hereda todo el peso, así que la carga queda desbalanceada.

---

## 5. Approach 3 — Hash consistente + virtual nodes 🔴

### 5.1 La idea: cada servidor físico aporta muchos puntos al anillo

Suena complicado y no lo es tanto una vez que hace clic. Lo que hacés es **dividir cada servidor en muchos virtual nodes** (vnodes). El servidor físico C no aparece una vez en el anillo: aparece como C0, C1, C2… hasta C9. Lo mismo B y lo mismo A. Y esos puntos se distribuyen de forma aleatoria alrededor del anillo, intercalados.

¿Cómo se consigue que un solo servidor tenga muchos puntos? Hasheando su identidad con un sufijo distinto por vnode:

```
   hash(IP_de_C + "-0")  →  un punto del anillo   → C0
   hash(IP_de_C + "-1")  →  otro punto           → C1
   hash(IP_de_C + "-2")  →  otro                 → C2
   ...
```

Como una función de hash reparte sus salidas por todo el espacio, los diez puntos de C caen desparramados por el anillo, mezclados con los de A y los de B. Todos siguen apuntando al mismo servidor físico C.

La regla de asignación no cambia: cada dato va al **vnode** inmediatamente posterior en sentido horario, y el dato vive en el servidor físico dueño de ese vnode.

### 5.2 El anillo con treinta vnodes

Dibujado como círculo se vuelve ilegible, así que va **desenrollado**: la secuencia de puntos en sentido horario, arrancando arriba y volviendo al principio. Los cinco datos van entre corchetes.

```
  B3  C0  B2  [John]  C4  A3  A2  A1  C6  [Steve]  B5  B6  B7  C2  A8  A4  [Bill]
  C5  C3  B9  A0  A6  B1  [Jane]  C7  A9  B8  B0  C8  C1  A5  C9  [Kate]  B4  A7
  → y vuelve a B3
```

Aplicando la regla (el primer vnode a la derecha de cada dato):

| Dato | Primer vnode en sentido horario | Servidor físico |
|---|---|---|
| John | C4 | C |
| Steve | B5 | B |
| Bill | C5 | C |
| Jane | C7 | C |
| Kate | B4 | B |

### 5.3 Qué pasa cuando cae un servidor

**Se cae B.** Se caen todos sus vnodes a la vez: B0 a B9 desaparecen del anillo. Los datos que apuntaban a un vnode de B siguen girando hasta el próximo vnode **vivo**:

- Steve apuntaba a B5. Sigue: B6 caído, B7 caído, **C2** vivo. Steve pasa a C.
- Kate apuntaba a B4. Sigue: **A7** vivo. Kate pasa a A.

Los datos huérfanos de B **no fueron a parar todos al mismo servidor**: uno fue a C y otro a A. Con más datos, el efecto es el mismo a escala: como los vnodes de B estaban intercalados por todo el anillo, a cada uno le sigue, en sentido horario, un vnode de A o de C, y la carga de B se reparte entre los dos servidores que quedan.

**Se cae C** (con B sano de vuelta). John apuntaba a C4 → sigue a **A3**. Bill apuntaba a C5 → C3 caído → **B9**. Jane apuntaba a C7 → **A9**. Otra vez la carga se reparte: dos a A, uno a B.

Eso es lo que el approach anterior no lograba, y es lo que resumen sus dos propiedades:

- **Distribución pareja.** La carga se reparte entre todos los servidores reales, porque cada uno está presente en todo el anillo.
- **Failover suave.** Si cae un servidor, sus vnodes se reparten entre muchos otros, no entre uno solo.

### 5.4 Detalles que conviene tener claros 🟡

**Los vnodes son un mapa lógico, no máquinas.** Sirven solo para la distribución. Si tenés que ir a ver logs, ves los logs de A: no hay logs de A1 y A2 por separado, porque A1 y A2 **son** A. Tampoco corren en paralelo entre sí: todo lo que cae en A1 y en A2 cae en el mismo servidor físico. Es una forma de repartir el anillo, nada más.

**¿Qué devuelve la función de hash?** Para pensarlo, sirve imaginar que devuelve **grados**: en qué ángulo del círculo cae cada cosa. Eso es lo didáctico. En una implementación real devuelve un entero dentro de un rango fijo y enorme (en Cassandra, por ejemplo, *tokens* de 64 bits), y ese rango es el que se piensa como anillo. Como los vnodes son virtuales, no importa dónde caen exactamente: los vas repartiendo como quieras.

**Particiones calientes.** Puede haber sectores del anillo que se usan mucho más que otros, porque las keys reales no siempre se distribuyen tan parejo como los DNIs. Con vnodes, ese reparto se puede ir ajustando sobre la marcha, moviendo o agregando puntos donde hace falta. Para eso están.

**Quién lo usa.** Cassandra (una base NoSQL distribuida; Parte 5) usa exactamente este esquema: virtual nodes sobre hash consistente. No es algo que vos calcules: lo hace por atrás, con parámetros configurables, como una feature del motor. Y no lo usa solo: lo combina con **replicación** (copias de los datos en varios nodos, como vimos en la Parte 1, sección 6), de modo que un nodo puede guardar además datos que "pertenecen" a otro. Cómo se combinan sharding y replicación es tema de la Parte 3.

> 📌 **Para el parcial, si te preguntan: ¿qué agregan los virtual nodes al hash consistente?**
> Cada servidor físico aporta muchos puntos al anillo (hash de su IP con un sufijo distinto por vnode: hash(IP + "-1"), hash(IP + "-2")…), intercalados con los de los demás servidores. Así, cuando cae un servidor, sus datos se reparten entre muchos otros servidores en lugar de caer todos en el vecino: distribución pareja y failover suave. Es el esquema que usa Cassandra.

---

## 6. Los tres approaches, uno al lado del otro 🔴

| | Hash distribuido (mod N) | Hash consistente | Hash consistente + vnodes |
|---|---|---|---|
| **Qué se hashea** | Las keys. | Las keys **y** los servidores. | Las keys y **muchos puntos por servidor**. |
| **Cómo se asigna** | Resto de dividir por N. | Primer servidor en sentido horario en el anillo. | Primer vnode en sentido horario; el dato va a su servidor físico. |
| **Cae o se agrega un servidor** | Cambia N, se reasigna casi todo (tormenta de re-hashing). | Solo se mueven los datos del sector afectado. | Solo se mueven los datos del sector afectado. |
| **Cómo queda la carga** | Pareja mientras N no cambie. | Desbalanceada: el vecino hereda todo. | Pareja: se reparte entre todos. |
| **Veredicto** | Rápido pero frágil. | Minimiza el rebalanceo. | Distribución pareja y failover suave. |

---

## 7. Lo que queda abierto 🔴

Con sharding ya podés repartir datos entre muchas máquinas sin criterio de negocio, sobrevivir a que caigan o se agreguen servidores, y mantener la carga pareja. Pero fijate qué pasa con cada dato individual: **existe en un único servidor.** Si se cae B, los datos de B se reasignan a otros servidores… pero los datos en sí estaban en B, en su disco. Reasignar dice a dónde deberían ir; no los hace aparecer ahí. Mientras B esté caído, lo que B tenía no está.

Y del otro lado, quedó pendiente de la Parte 1 el límite del activo/pasivo: un solo nodo escribiendo. Las dos cosas se atacan juntas en la Parte 3: escribir en más de un nodo a la vez, y combinar el sharding con copias de los datos.

---

## ✅ Checkpoint — Parte 2

*(Sin respuestas: van en el complemento de la unidad.)*

1. ¿Qué significa que en el sharding "la asignación se calcula, no se modela"? ¿Qué gana un sistema con eso respecto del particionamiento?
2. Explicá con tus palabras qué es una función de hash y por qué "par o impar" es una.
3. ¿Por qué una función de hash para sharding tiene que ser determinista? ¿Qué pasaría si no lo fuera?
4. Shardear por primera letra del apellido es un criterio válido. ¿Qué problema tiene y qué propiedad le falta?
5. Con la regla "último dígito mod 3" y servidores 0, 1 y 2, ¿a dónde va el DNI 40128455? ¿Y si se cae el servidor 1 y la regla pasa a mod 2?
6. ¿Qué es una tormenta de re-hashing y por qué el módulo N la provoca tanto al perder como al agregar un servidor?
7. En el hash consistente, ¿por qué se hashean también los servidores? ¿Qué cambiaría si solo se hashearan las keys?
8. En el anillo de la sección 4, se agrega un servidor E entre Bill y A. ¿Qué datos cambian de servidor?
9. ¿Por qué "equidistantes en el anillo" no garantiza carga pareja?
10. Si el servidor A tiene diez vnodes y se cae, ¿por qué sus datos no van todos al mismo servidor? ¿Qué pasaría con un solo vnode por servidor?
11. Un compañero dice "con virtual nodes tengo A1 y A2 atendiendo en paralelo, así que duplico la capacidad de A". ¿Qué le corregís?

---

## Qué viene en la Parte 3

Replicación activo/activo: escribir en dos nodos a la vez y el conflicto que eso genera; las tres formas de resolverlo (consenso, gana la última escritura, quorum); cómo se combinan sharding y replicación; y por qué replicar no es hacer backup.

---

**FIN DE LA PARTE 2 — Apunte maestro clase04 · NoSQL**
