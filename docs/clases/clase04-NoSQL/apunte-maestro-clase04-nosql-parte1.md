# 📘 Apunte maestro — clase04 · NoSQL
## Parte 1 de 5 — Por qué existen las NoSQL y cómo se escala

**Unidad:** `clase04` · clase 4 del cronograma (01/09/2026) · **Tema:** noSQL
**Partes de la unidad:** 1 · Por qué existen las NoSQL y cómo se escala *(esta)* — 2 · Sharding — 3 · Escribir en más de un nodo — 4 · Las garantías — 5 · Los cuatro tipos y cómo decidir

**Leyenda:** 🔴 central, evaluable · 🟡 secundario · 🟢 mencionado al pasar · 🕳️ madriguera (tangente que no hace falta seguir hoy) · ⚠️ "en la materia se enseña X" · 📌 "Para el parcial, si te preguntan" · 📋 información operativa de la cursada.

**Cómo están escritas las partes.** Cada concepto se explica una sola vez, en la parte donde aparece por primera vez; las partes siguientes lo citan ("como vimos en la Parte 1") y nunca se usa nada antes de explicarlo. Los nombres de productos (Cassandra, Mongo, Redis…) aparecen como ejemplos con una línea la primera vez; su tratamiento completo está en la Parte 5.

**Qué cubre esta parte.** El problema que hace nacer a las bases no relacionales: de dónde vienen, qué cambió en el hardware y en los datos, qué se le empieza a pedir a un sistema cuando crece, cómo se escala (vertical y horizontal), por qué cualquier máquina se cae y las dos primeras estrategias para convivir con eso: replicación activo/pasivo y particionamiento por negocio.

**De dónde venís.** Esta parte no se apoya en nada de las clases 01 a 03. Lo único que asume es que sabés qué es una base relacional (tablas, filas, columnas, SQL): la clase parte de ahí.

---

## 0. Encuadre — tres advertencias antes de arrancar 🔴

Lo que sigue trata de responder cuatro preguntas: cómo surge el concepto de base no relacional, qué necesidades viene a cubrir, cómo funciona por atrás, y en qué casos ayuda — y si ayuda, qué tipo de base elegir. Antes de la primera pregunta, tres advertencias que atraviesan toda la clase:

1. **NoSQL no es superador de SQL.** Es un concepto más nuevo, pero "más nuevo" no significa "mejor". Es otra herramienta, para otros problemas.
2. **La división útil no es SQL vs NoSQL, es relacionales vs no relacionales.** SQL es un lenguaje de consulta; lo que cambia de fondo es el modelo de datos. En este apunte se usan las dos formas de nombrarlas porque así se las nombra en la industria, pero la distinción que importa es la del modelo.
3. **Los casos de uso de las no relacionales son mucho más específicos que los de las relacionales.** Es muchísimo más común haber trabajado con una relacional, y el default sano sigue siendo relacional. Elegir una no relacional exige **un motivo**: se tiene que presentar el caso de uso que la justifique. No se elige "porque nos gusta".

> 📌 **Para el parcial, si te preguntan: ¿cuándo conviene una base no relacional?**
> Cuando el caso de uso lo justifica: por default se usa una base relacional, y solo cuando el volumen, la velocidad, la variedad de los datos o la necesidad de repartir el sistema en muchas máquinas superan lo que la relacional maneja bien, tiene sentido evaluar una no relacional. La elección se argumenta con el caso de uso concreto, no con preferencia.

---

## 1. Un poco de historia 🟡

```
   1960              1970                1990                        2000
    ●─────────────────●───────────────────●───────────────────────────●──────►
 Discos rígidos    Relacionales       Orientadas a objetos        NoSQL / NewSQL
 Bases             Modelo relacional  Object-relational           Google, Facebook,
 jerárquicas       SQL                impedance mismatch          Amazon
```

**Años 60 — aparecen los discos rígidos.** Con ellos empieza a existir la **persistencia**: guardar datos de forma que sobrevivan a que se apague la máquina. Las primeras bases de esa época son las *jerárquicas* (los datos se organizan como un árbol: cada registro cuelga de un registro padre).

**Años 70 — las bases relacionales que conocemos hoy.** Se publica el primer paper del modelo relacional y se introduce SQL como **lenguaje declarativo**. El cambio es enorme: antes había que pensar *cómo* acceder físicamente a los datos; con SQL pasás a describir *qué* datos querés y el motor resuelve cómo traerlos. Se separa la lógica del negocio del acceso físico a la información, y eso dio una flexibilidad que explica por qué estas bases dominan hasta hoy.

**Años 90 — las bases orientadas a objetos.** Intentan unir el mundo de las bases de datos con el de la programación orientada a objetos, para esquivar el **Object-Relational Impedance Mismatch**: los objetos de tu programa casi nunca tienen la misma forma que una tabla. Un objeto tiene listas adentro, referencias a otros objetos, herencia; una tabla es plana, con filas y columnas. Cada vez que persistís un objeto en una relacional hay una traducción de por medio, y esa fricción es el "impedance mismatch". Las bases orientadas a objetos no se impusieron, pero influyeron mucho en los **ORM** (*Object-Relational Mapping*: las bibliotecas que hacen esa traducción por vos), tanto los de estilo *Data Mapper* como Hibernate, como los de estilo *Active Record* como el que usa Rails.

> 🕳️ **Madriguera — Data Mapper vs Active Record**
> Son dos patrones de ORM: en *Active Record* el propio objeto sabe guardarse y buscarse (`usuario.save()`); en *Data Mapper* hay una capa aparte que mueve datos entre objetos y tablas, y el objeto no sabe nada de la base.
> *Volvé al camino — esto se profundiza aparte, otro día.*

**Años 2000 — NoSQL y NewSQL.** Llega la *web 2.0* (la web donde los usuarios generan el contenido: redes sociales, video, comentarios) y con ella el crecimiento masivo de datos. Empresas como Google, Facebook y Amazon empiezan a necesitar crecer más allá de lo que aguanta una máquina, a manejar datos que no son tan estructurados y a responder en tiempo real. La infraestructura de la época no les daba, y tuvieron que buscar alternativas. De ahí salen motores como Mongo, Dynamo o Cassandra (tres bases no relacionales; las vemos una por una en la Parte 5). En la misma línea de tiempo aparece **NewSQL**, otra familia con la que se cierra la clase (Parte 4).

---

## 2. El costo del almacenamiento cambió el problema 🟡

```
 dólares por terabyte (escala logarítmica: cada marca es ×100)
 10¹⁴ ┤●╲
 10¹² ┤  ╲╲──╲
 10¹⁰ ┤●───╲───╲╲
 10⁸  ┤      ╲───╲──╲
 10⁶  ┤          ╲───╲──╲╲
 10⁴  ┤              ╲   ╲╲──╲──────────────── RAM
 10²  ┤               ╲    ╲╲──╲────────────── Flash / SSD
      ┤                 ╲────╲──────────────── Disco
      └────┬─────┬─────┬─────┬─────┬─────┬─────┬──
         1956  1970  1980  1990  2000  2010  2022
```

El costo de almacenar cayó de forma exponencial en las últimas décadas, y en todas las tecnologías: memoria RAM, memorias flash, discos de estado sólido (SSD, sin partes móviles) y discos rígidos. Guardar un terabyte en disco pasó de costar del orden de **10¹² dólares** a **menos de cien**: unos nueve órdenes de magnitud en setenta años. Ojo con el gráfico: el eje vertical es logarítmico, cada marca es cien veces la anterior, así que la caída es mucho más brutal de lo que parece a simple vista.

Eso cambió el paradigma de las bases de datos. Antes el problema era **optimizar el espacio al máximo** para poder guardar los datos. Hoy el espacio dejó de ser el cuello de botella, y el problema pasó a ser **cómo los manejamos**: cómo accedemos a ellos, qué tan rápido respondemos, cómo escalamos el sistema. El foco se corre del almacenamiento al acceso y la escala.

Una forma de verlo: hay tres cosas que se le piden al almacenamiento de un sistema, y tiran en direcciones distintas, como las puntas de un triángulo.

```
              Garantías sobre la información
             (que lo que leés esté bien y al día)
                          ▲
                         ╱ ╲
                        ╱   ╲
                       ╱     ╲
      Velocidad ◄─────●───────● Espacio que ocupo
      de acceso                 (hoy, el lado barato)
```

Con el espacio barato, el diseño se mueve hacia las otras dos puntas: velocidad de acceso y garantías sobre los datos. La complejidad espacial ya no es el problema principal.

Dos matices. Primero, con la demanda de cómputo actual (inteligencia artificial incluida) los precios de memoria y almacenamiento vienen subiendo, pero esa suba es marginal comparada con el costo histórico; no cambia el cuadro. Segundo, no solo evolucionó la tecnología: también evolucionaron los requerimientos. Cuando se diseñaron las primeras bases nadie pensaba en procesar terabytes; hoy es lo normal.

> 📋 **Lectura recomendada por la cátedra:** *Designing Data-Intensive Applications* (Martin Kleppmann, editorial O'Reilly, la de los animales en la tapa). Es un libro generalista sobre sistemas que manejan datos: tiene capítulos sobre almacenamiento, formatos de intercambio de mensajes, relojes en sistemas distribuidos y más. Su introducción desarrolla justamente este cambio de foco (del espacio a la velocidad y las garantías) y muestra cosas que se pueden hacer con una base y no con otra. Recomendado para cualquier ingeniero, no hace falta para la materia.

---

## 3. Big Data: las tres V y los dos patrones de carga 🔴

### 3.1 Qué es y qué no es

Big Data suena a humo, y en parte lo es: **no hay un límite real** a partir del cual "tantos teras" pasan a ser Big Data. No es un umbral de tamaño. Es un **conjunto de requerimientos no funcionales** que aparecen juntos cuando los datos crecen en alguna de tres dimensiones:

| Dimensión | Qué mide | Ejemplo |
|---|---|---|
| **Volumen** | Cuántos datos hay. Terabytes, petabytes. Cantidades que **un único nodo** a veces no puede soportar. | El histórico completo de una red social. |
| **Velocidad** | A qué ritmo llegan. No es lo mismo terabytes acumulados a lo largo de un año que terabytes generados **por día o por minuto**. Se habla de *streams* de alta frecuencia, *throughput* sostenido, baja *latencia*. | Los eventos que genera una app en tiempo real. |
| **Variedad** | En qué formato vienen: logs, JSON, imágenes, eventos, grafos, *blobs*. **No todo encaja en tablas.** | Una plataforma que guarda texto, fotos y relaciones entre usuarios. |

Algunos términos que van a aparecer toda la clase, definidos acá:

- **Nodo:** una máquina (física o virtual) que forma parte del sistema. "Un único nodo" = un solo servidor.
- **Latencia:** cuánto tarda una operación individual en responder (por ejemplo, 5 ms para una lectura).
- **Throughput:** cuántas operaciones procesa el sistema por unidad de tiempo (por ejemplo, 10.000 escrituras por segundo). Un sistema puede tener latencia baja y throughput bajo, o al revés; son medidas distintas.
- *Stream:* flujo continuo de datos que llegan uno tras otro, sin fin. *Blob:* un archivo binario opaco (una imagen, un PDF) que la base guarda sin entender su contenido.

### 3.2 Dónde se queda corto el modelo relacional

Los modelos relacionales funcionan muy bien cuando **los datos están estructurados**: esquema fijo, filas con las mismas columnas, relaciones claras. Cuando los esquemas no son fijos, los datos son diversos o la escala es muy grande, **nos quedamos cortos**, y muchas veces empezamos a necesitar otros tipos de bases para manejarlos bien.

### 3.3 Los dos patrones de carga

En sistemas con mucho tráfico o muchos datos suelen aparecer uno de dos patrones. Conviene reconocerlos, porque cada uno pide cosas distintas del almacenamiento:

| Patrón | Qué pasa | Ejemplos |
|---|---|---|
| **Poco de muchos** | Se guarda poco por usuario, pero hay millones de usuarios. | Perfiles, preferencias, configuración de sesiones. |
| **Mucho de cada uno** | Hay relativamente pocos usuarios, pero cada uno genera muchísimo. | Comportamiento de usuario, telemetría (mediciones que mandan dispositivos o sensores), logs. Pensá en cada clic que hace cada usuario de Netflix mientras busca una película: información enorme, que se analiza. |

En cualquiera de los dos casos empezamos a preocuparnos por cosas nuevas: **cómo escalar las escrituras, cómo distribuir los datos, cómo mantener consultas que no demoren una eternidad.** Y ahí es donde empieza a tener sentido evaluar alternativas a la base relacional.

> 📌 **Para el parcial, si te preguntan: ¿qué es Big Data?**
> No es un umbral de tamaño: es el conjunto de requerimientos no funcionales que aparecen cuando los datos crecen en volumen (más de lo que soporta un único nodo), velocidad (llegan por minuto, no por año) o variedad (formatos que no encajan en tablas). Las tres V. El modelo relacional rinde bien con datos estructurados; cuando alguna de las tres V se dispara, hay que evaluar otros tipos de base.

---

## 4. Requerimientos no funcionales: ya no alcanza con guardar 🔴

Acá cambia el foco. Ya no hablamos solo de **guardar** datos, sino de **cómo se tiene que comportar** el sistema que los guarda. Un *requerimiento no funcional* es exactamente eso: no describe qué hace el sistema, sino cómo lo hace (qué tan rápido, qué tan seguido falla, cuánto aguanta). Como ingenieros en sistemas siempre buscamos garantizar cinco cosas:

| Requerimiento | Qué significa |
|---|---|
| **Escalabilidad** | Crecer sin tener que reescribir la aplicación. |
| **Disponibilidad** | Responder incluso ante fallas parciales. |
| **Performance** | Baja latencia y/o throughput aceptable **bajo carga**, no solo con un usuario. |
| **Robustez** | Evitar las **fallas silenciosas**: el sistema puede fallar, pero el usuario se tiene que enterar. Una operación que "parece" que salió bien y no se guardó es peor que un error visible. |
| **Alta disponibilidad** (*high availability*) | Minimizar los **puntos únicos de falla** (*single points of failure*, SPOF): componentes que, si se caen, tiran todo el sistema. |

Esto nos obliga a repensar cómo **almacenamos**, cómo **leemos**, cómo **procesamos** y cómo **escalamos**. Y en sistemas grandes, estos requerimientos a veces son **más importantes que el modelo de datos** que vamos a elegir: primero hay que garantizar que el sistema responda y crezca; recién después, con qué forma se guardan los datos.

Lo que sigue en esta parte y en las dos siguientes desarrolla dos de esos cinco puntos, escalabilidad y disponibilidad, porque de ellos salen todas las estrategias de distribución de datos que la clase enseña.

---

## 5. Escalabilidad: vertical y horizontal 🔴

La situación concreta: **se llenó el disco**, o **no doy abasto con el throughput**. El sistema creció y la máquina no alcanza. ¿Qué opciones tengo? Dos.

```
 VERTICAL — mejorar la máquina            HORIZONTAL — sumar máquinas
 ┌────────────────────┐                   ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
 │ ████ más CPU       │                   │  ▪  │ │  ▪  │ │  ▪  │ │  ▪  │
 │ ████ más RAM       │                   └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘
 │ ████ mejor disco   │                      └───────┴───────┴───────┘
 └────────────────────┘                          la carga se reparte
       la misma máquina                       entre máquinas más simples
```

### 5.1 Escalamiento vertical

Agregarle a la misma máquina más CPU, más RAM, un disco mejor. Es **fácil**: no cambia nada de la aplicación ni de la arquitectura. Pero:

- **Es caro:** la curva de precio es exponencial. Duplicar la potencia cuesta mucho más que el doble.
- **Es limitado:** hay un techo físico. No podés escalar indefinidamente metiéndole hardware a una máquina.
- **No resuelve la disponibilidad:** por más grande que sea, sigue siendo una sola máquina que se puede caer.

### 5.2 Escalamiento horizontal

Ir agregando **más máquinas en paralelo**, más simples, y distribuir la carga entre ellas. Es **más complejo de manejar**, pero:

- Es **más barato por unidad de capacidad**: muchas máquinas comunes salen menos que una monstruosa.
- Escala **casi sin techo**: si hace falta más, se suma otra.
- **Habilita la alta disponibilidad**: con varias máquinas, la caída de una no es el fin del sistema.

Dentro del escalamiento horizontal hay más de una forma de repartir el trabajo. En el fondo son dos ideas:

```
 CLONAR el conjunto de datos                DESCOMPONERLO
 ┌──────┐ ┌──────┐ ┌──────┐                 ┌──────┐ ┌──────┐ ┌──────┐
 │ ABCD │ │ ABCD │ │ ABCD │                 │  AB  │ │  C   │ │  D   │
 └──────┘ └──────┘ └──────┘                 └──────┘ └──────┘ └──────┘
 todos tienen todo → más máquinas           cada uno tiene una parte → los
 respondiendo las mismas lecturas           datos y la carga se reparten
```

O **clonás** tu conjunto de datos en varias máquinas para acceder más rápido, o lo **descomponés** en pedazos y cada máquina se ocupa de uno. Las dos ideas, con sus variantes, son el resto de esta parte y la Parte 2.

### 5.3 ⚠️ Horizontal no es "mejor" que vertical, ni las relacionales son "verticales"

No es que el escalamiento horizontal sea mejor que el vertical, ni que uno sea el pasado y el otro el futuro. **Son estrategias que se combinan**, y dentro de cada una hay variantes. Y en particular: **los motores relacionales escalan de las dos maneras.** Es mucho más habitual verlos escalados verticalmente, pero podés tener perfectamente un setup relacional escalado horizontalmente:

- **Galera Cluster** es un modo de correr MySQL/MariaDB en varias máquinas a la vez: es síncrono, mantiene SQL y todas las garantías transaccionales de una relacional, y es horizontal.
- También podés armar un esquema donde una máquina escribe y otras la copian y atienden lecturas (lo vemos en la sección 7 de esta parte), y además escalar verticalmente cada una de esas máquinas.

**Entonces, ¿por qué las NoSQL se asocian a escalar horizontal?** Dos razones, y las dos son históricas:

1. **Cuándo nació cada una.** SQL se diseñó en una época en la que escalar horizontal no era común; no nació pensado para eso, aunque hoy tenga estrategias. Las NoSQL nacen en los 2000, cuando el problema era justamente ese, y vienen **pensadas de base** para distribuirse. Por eso a veces resulta más fácil escalarlas.
2. **Qué hardware había.** Hoy tenés un SSD, y podés levantar un Redis (una base que vive en memoria; Parte 5) que es básicamente un mapa en RAM accedido de manera **aleatoria**, y es rapidísimo. En el pasado tenías un disco rígido o una cinta, y esos dispositivos leen mucho más rápido de manera **secuencial** (datos contiguos, uno tras otro) que aleatoria (saltando de un lugar a otro). Un motor relacional, por atrás, está pensado para guardar datos contiguos y leerlos en secuencia; una NoSQL puede hacer algo más orientado al acceso random y aun así ser más rápida, porque corre sobre un SSD donde eso ya no cuesta.

Eso no significa que un motor relacional no se piense para SSD: también evoluciona, y los motores relacionales de hoy son muy completos. La conclusión es la de siempre: **depende del caso de uso** cuál vas a preferir.

> 📌 **Para el parcial, si te preguntan: ¿qué diferencia hay entre escalar vertical y horizontal?**
> Vertical es mejorar la misma máquina (más CPU, RAM, disco): es fácil, pero caro (curva exponencial), tiene un techo físico y no resuelve la disponibilidad. Horizontal es sumar máquinas más simples y repartir la carga: es más complejo de manejar, pero más barato por unidad, escala casi sin límite y habilita la alta disponibilidad. No son excluyentes: se combinan, y los motores relacionales escalan de las dos formas.

---

## 6. Disponibilidad: cualquier máquina se cae 🔴

El concepto clave de todo lo que sigue: **cualquier máquina puede fallar.** Por más buena infraestructura que tengas, por más cara que sea la instancia, se puede caer. Siempre va a haber caídas, problemas de red, tareas de mantenimiento. Y la **única forma de esquivar eso** es tener **múltiples nodos**: redundancia, datos distribuidos, y alguna estrategia para manejarlos.

De ahí salen las estrategias que se van a ver. Son cuatro nombres, y conviene tener el mapa antes de entrar en cada una:

| Estrategia | Idea en una línea | Dónde se desarrolla |
|---|---|---|
| **Replicación** | Copias del mismo dato en varios nodos. Es la forma de *clonar* de la sección 5.2. | Sección 7 (su forma más simple) y Parte 3 (sus variantes). |
| **Activo / Pasivo** | La forma más simple de replicación: un nodo escribe; el otro replica, atiende lecturas y espera para tomar el relevo. | Sección 7. |
| **Particionamiento** | Dividir los datos según un **criterio de negocio** (por región, por cliente). Es una forma de *descomponer*. | Sección 8. |
| **Sharding** | Repartir los datos entre servidores **por cálculo**, sin depender del negocio: la asignación se calcula, no se modela. La otra forma de *descomponer*. | Parte 2. |

Dos de estas cuatro (activo/pasivo y particionamiento) se cierran en esta parte. Vamos con la más simple.

---

## 7. Replicación activo / pasivo 🔴

### 7.1 Cómo funciona

```
                     ( Cliente )
                    ╱           ╲
   escrituras y lecturas         solo lecturas
                  ▼               ▼
            ┌──────────┐ replica ┌ ─ ─ ─ ─ ─ ┐
            │    A     │───────►     A'
            │  Activo  │         │  Pasivo   │
            └──────────┘         └ ─ ─ ─ ─ ─ ┘
```

Hay **un nodo activo**, A, que es el único que maneja las **escrituras**. Y hay **uno o más nodos pasivos**, A', que **replican** lo que A escribe (reciben una copia de cada cambio) y sirven para **lecturas**. El cliente escribe siempre en A; puede leer de A o de A'.

### 7.2 Propiedades

- **Failover.** Si A se cae, se *switchea* a A': el pasivo pasa a ser el activo. Suele hacerse por DNS: el nombre con el que la aplicación llama a "la base" pasa a apuntar a la dirección de A' en vez de a la de A, y la aplicación ni se entera. *Failover* es exactamente eso: el mecanismo por el cual, ante la caída de un nodo, otro toma su lugar.
- **Lecturas distribuidas.** Las réplicas se usan para las consultas de lectura, así que la carga de lectura se reparte entre A y A'. Lecturas más rápidas.
- **Impacto en escritura: poco.** Solo el nodo activo escribe. Agregar pasivos no te da más capacidad de escritura.
- **Impacto en lectura: grande.** Cada pasivo que sumás es una máquina más respondiendo lecturas.

La ventaja, entonces, es doble: distribuís la carga de lectura y tenés failover. La **limitación**: seguís teniendo **un único punto de escritura**. Si tu problema es que no das abasto con las escrituras, activo/pasivo no lo resuelve.

### 7.3 Los dos temas que abre replicar

Decir "A replica en A'" esconde dos decisiones. Las dos aparecen acá por primera vez y vuelven más adelante.

**¿Replicación síncrona o asíncrona?** Cuando el cliente escribe en A, ¿cómo llega ese cambio a A'?

```
 SÍNCRONA                                ASÍNCRONA
 Cliente ──escribe──► A                  Cliente ──escribe──► A
                      A ──copia──► A'                         A ──"listo"──► Cliente
                      A ◄──"ok"─── A'                         A ──copia──► A'   (después)
 Cliente ◄──"listo"── A
 A espera la confirmación de A'          A responde enseguida y replica
 antes de responder                      por atrás; A' puede ir un paso atrás
```

- **Síncrona:** A escribe, le manda la copia a A', **espera que A' confirme**, y recién ahí le responde al cliente. Cuando el cliente recibe el "listo", el dato ya está en los dos nodos.
- **Asíncrona:** A escribe y le responde al cliente **enseguida**; la copia a A' viaja después, por atrás. Es más rápido para el cliente, pero durante un ratito A' está un paso atrás de A.

**¿Quién manda si se cae el activo?** Si se cae el nodo de escritura, hace falta un nuevo activo, y los nodos tienen que ponerse de acuerdo en cuál. Eso se resuelve con un **algoritmo de elección de líder**: un procedimiento por el cual los nodos que quedan acuerdan quién pasa a ser el nuevo activo. El failover de la sección 7.2 se apoya en esto.

### 7.4 Nombres y naturaleza de la estrategia

En el mundo relacional este mismo esquema se llama **master / slave**: un nodo *master* que escribe y N nodos *slave* que replican y atienden lecturas. Activo/pasivo y master/slave son la misma idea vista desde el failover: un nodo de escritura y N de lectura, y cuando el de escritura se cae, otro toma el relevo.

Importante: activo/pasivo **no es una implementación, es un concepto.** Es el fundamento del que salen los mecanismos concretos de failover de cada motor.

> 📌 **Para el parcial, si te preguntan: ¿qué es la replicación activo/pasivo y qué limitación tiene?**
> Un nodo activo recibe todas las escrituras y uno o más nodos pasivos replican sus datos y sirven lecturas. Da failover (si cae el activo, un pasivo toma su lugar, típicamente por DNS) y reparte la carga de lectura. Su limitación es que hay un único punto de escritura: no escala las escrituras.

---

## 8. Particionamiento por negocio 🔴

### 8.1 Cómo funciona

```
          ( clientes de AR )        ( clientes de BR )
                  │                         │
                  ▼                         ▼
          ┌───────────────┐         ┌───────────────┐
          │   Argentina   │         │    Brasil     │
          │    Activo     │         │    Activo     │
          └───────────────┘         └───────────────┘
            base propia               base propia
```

Particionar es dividir los datos **según un criterio de negocio** y mandar cada partición a un nodo distinto. El criterio puede ser la **región** (los datos de Argentina en una base, los de Brasil en otra), el **cliente**, o la **vertical** de negocio. Lo esencial es que **el límite está atado al negocio**: hay algo en el dominio que separa naturalmente los datos, y esa separación se aprovecha para repartirlos. Cada nodo es activo para su porción: atiende lecturas y escrituras de su partición.

Un término que va a aparecer seguido: **tenant**. En un sistema que le da servicio a muchas empresas-cliente, cada empresa es un *tenant* (inquilino): sus datos viven en el mismo sistema pero aislados de los otros. Particionar "por tenant" es darle a cada cliente-empresa su porción.

### 8.2 Tres ejemplos reales

**Cisco, particionando por cliente.** Cisco (la empresa de los routers) maneja datos de tráfico de redes: volúmenes impresionantes. Escalaron **con SQL**, con bases relacionales, particionando por cliente: si el cliente era muy grande, tenía un nodo para él solo, con su propia base; si era chico, se agrupaban varios clientes chicos en un mismo nodo. Es particionamiento por tenant sobre un motor relacional. Dos cosas para retener: la partición la dicta el negocio (el tamaño del cliente), y esto es escalar horizontal con una relacional, como se adelantó en la sección 5.3.

**SQLite, un tenant por archivo.** El caso extremo del mismo criterio. SQLite es una base de datos que vive entera en **un solo archivo**; su contra es que no soporta escrituras concurrentes (dos procesos escribiendo a la vez). Hay sistemas que aprovechan justamente eso: **un archivo SQLite por tenant**. Cada cliente-empresa tiene su base, chiquita, aislada; el sistema "escala" al tamaño de cada cliente en vez de al total, y levanta instancias mucho más chicas. Particionar por tenant te permite eso: que cada partición sea pequeña y manejable.

**Twitter, particionando por usuario grande y chico.** El ejemplo con el que abre *Designing Data-Intensive Applications*: una red social con posteos y notificaciones a seguidores. Cuando un usuario postea, hay que avisarle a sus seguidores. Para un usuario con dos amigos, notificar a dos personas es baratísimo: le dejás el post en la "casilla" (*mailbox*) de cada uno. Para un usuario con millones de seguidores, notificar a todos es carísimo; conviene invertir la lógica y que los seguidores "escuchen" a esos pocos usuarios enormes cuando abren su feed. El sistema trata distinto a **usuarios grandes** y **usuarios chicos**: es una forma de particionamiento, y el criterio otra vez sale del negocio.

La moraleja de los tres: **hay N formas de particionar un sistema en términos de datos, y cuál usar depende del caso de uso.**

### 8.3 Para qué sirve y dónde se limita

Lo que gana el particionamiento:

- **Distribuye la carga:** escala procesamiento, disco y tiempo de respuesta, porque cada nodo maneja solo su porción.
- **Mejora los tiempos de respuesta.** Con la salvedad de que la mejora en lectura **depende de la query**: una consulta que solo toca su partición vuela; una que necesita varias, no.
- **Aísla los problemas y desacopla negocios:** si se cae la base de Argentina, el negocio en Brasil puede seguir. Eso aumenta la disponibilidad.
- Puede ser **transparente o no** para la aplicación, según cómo se implemente: hay motores que enrutan solos cada consulta a su partición, y hay diseños donde la aplicación tiene que saber a qué base ir.

Lo que lo limita: se complica cuando necesitás **combinar información de dos particiones** (*cross-partition*). Si el negocio necesita todo el tiempo datos de los dos lados, en tiempo real, ese criterio de partición **no es viable**; quizás haya otro criterio que sí funcione. Por eso, dependiendo del negocio, este tipo de particionamiento puede o no ser factible: hay que mirar cómo se consultan los datos antes de decidir por dónde cortarlos.

> 📌 **Para el parcial, si te preguntan: ¿qué es el particionamiento y cuál es su límite?**
> Es dividir los datos según un criterio de negocio (región, cliente/tenant, vertical) y asignar cada partición a un nodo distinto. Distribuye la carga, mejora tiempos de respuesta y aísla fallas (si cae una partición, las otras siguen). Su límite son las consultas que necesitan combinar datos de varias particiones: si el negocio las requiere en tiempo real, ese criterio de partición no sirve.

---

## 9. Lo que queda abierto 🔴

Con lo visto hasta acá, tenés dos herramientas para convivir con máquinas que se caen y con datos que crecen:

| Herramienta | Qué resuelve | Qué no resuelve |
|---|---|---|
| Replicación activo/pasivo | Failover y carga de lectura. | Las escrituras: siguen entrando por un solo nodo. |
| Particionamiento | Reparte datos y carga, aísla fallas. | Necesita que **el negocio tenga un criterio natural** para cortar, y sufre las consultas que cruzan particiones. |

La pregunta que queda: **¿y si quiero repartir los datos entre muchas máquinas pero no hay (o no quiero depender de) un criterio de negocio?** Necesito una forma de decidir a qué servidor va cada dato que sea automática, que no requiera pensar cómo dividir el dominio, y que además reparta la carga de manera pareja. Eso es sharding, y es la Parte 2.

---

## ✅ Checkpoint — Parte 1

*(Sin respuestas: van en el complemento de la unidad.)*

1. ¿Por qué el disclaimer de la clase insiste en decir "relacionales vs no relacionales" en lugar de "SQL vs NoSQL"? ¿Qué es lo que cambia de fondo entre unas y otras?
2. ¿Qué es el *Object-Relational Impedance Mismatch* y qué herramientas nacieron para mitigarlo?
3. ¿Cómo cambió el problema de las bases de datos la caída del costo de almacenamiento? ¿Qué pasó a ser el cuello de botella?
4. Definí las tres V de Big Data y explicá por qué "Big Data" no es un umbral de tamaño.
5. ¿Qué diferencia hay entre el patrón "poco de muchos" y "mucho de cada uno"? Dá un ejemplo de cada uno que no esté en el apunte.
6. ¿Qué diferencia hay entre latencia y throughput? ¿Puede un sistema tener una buena y la otra mala?
7. ¿Qué es un punto único de falla? ¿Cuál es el punto único de falla de un esquema activo/pasivo?
8. Una empresa te dice "nosotros usamos SQL, así que no podemos escalar horizontal". ¿Qué le respondés y con qué ejemplo?
9. En replicación activo/pasivo, ¿qué diferencia práctica hay para el cliente entre replicar de forma síncrona y asíncrona?
10. Tu sistema necesita, en cada pantalla, combinar datos de clientes de Argentina y de Brasil en tiempo real. ¿Podés particionar por región? ¿Qué harías?

---

## Qué viene en la Parte 2

Sharding: qué es una función de hash, y los tres approaches para repartir datos por cálculo, cada uno resolviendo un problema del anterior: hash distribuido (módulo N), hash consistente (el anillo) y hash consistente con virtual nodes.

---

**FIN DE LA PARTE 1 — Apunte maestro clase04 · NoSQL**
