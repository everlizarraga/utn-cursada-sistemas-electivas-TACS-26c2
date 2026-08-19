# Apunte Maestro — Clase 01 · Parte 0: La materia

**Unidad:** clase01 · **Parte 0 de 5** (+ partes 1 a 5 sobre APIs)

Esta parte cubre lo que hay que saber sobre TACS como materia: de dónde salió, con qué criterio enseña, cómo se cursa y cómo se aprueba. Es la parte más corta del apunte y la única sin contenido técnico. El contenido de APIs arranca en la Parte 1.

**Leyenda de marcas:** 🔴 central / evaluable · 🟡 secundario · 🟢 mencionado al pasar.

---

## 1. De dónde salió TACS 🟢

TACS no se diseñó desde cero. Salió de un grupo de docentes y ayudantes de **TADP** (Técnicas Avanzadas de Programación, otra electiva de la carrera), en una época en que Diseño de Sistemas era una materia floja: no se aprendía ahí a construir software de verdad, y en cambio en TADP sí se veía diseño orientado a objetos con profundidad.

Ese grupo quiso dar un paso más: pasar del diseño en abstracto a lo técnico — abrir frameworks por dentro, meterse en arquitectura, entender cómo funcionan las herramientas que se usan en un trabajo real.

El primer intento fue **desdoblar TADP en dos cursos**: uno más *funky* (diseño y programación funcional) y otro más *techie* (frameworks, arquitectura). No funcionó, y el motivo es interesante porque explica el diseño actual de la materia: **los alumnos llegaban al curso techie sin la base**. Si alguien no sabe qué es un *strategy* o un *decorator*, no hay forma de explicarle cómo funciona un framework de mapeo objeto-relacional por dentro — el framework *es* esos patrones aplicados. Sin la base, la explicación se vuelve magia.

> 🕳️ **Madriguera — Strategy y Decorator**
> Dos patrones de diseño orientado a objetos. *Strategy* encapsula un algoritmo intercambiable detrás de una interfaz común; *Decorator* envuelve un objeto para agregarle comportamiento sin tocar su clase. Se ven en Diseño y en TADP.
> *Volvé al camino — acá solo aparecen como ejemplo de "base previa necesaria".*

La conclusión fue que el curso techie necesitaba al funky **antes**, no al lado. Así nació TACS: como continuación de TADP, no como su reemplazo.

Después cambió el contexto. Diseño se volvió una materia seria donde sí se enseña a construir software, TADP mutó, y TACS quedó actualizando su programa **cuatrimestre a cuatrimestre** — es una materia que se reescribe seguido, porque lo que enseña se mueve rápido.

---

## 2. El leitmotiv y el criterio de éxito 🟡

El objetivo declarado de la materia, y no cambió nunca desde su origen, es **acercar la industria a la academia**.

Eso se traduce en un criterio de éxito concreto y bastante exigente:

> Alguien que cursó y aprobó TACS tiene que poder ir a una entrevista para un puesto **junior o semi-senior** en una empresa mediana y que le vaya razonablemente bien.

La idea detrás es la cercanía: los conceptos que se ven acá ya no son de juguete. Son los que se usan efectivamente en empresas de escala mediana o grande, al menos en el mercado argentino. Eso es difícil de lograr en una materia, no porque el resto no sepa, sino porque **todo tiene una curva de aprendizaje** y la mayoría de las materias se consumen enseñando la base.

Consecuencia práctica sobre cómo estudiar: **el criterio de corrección de la materia es "¿sabrías defender esto en una empresa?"**, no "¿te acordás de la definición?".

---

## 3. Por qué hacemos software 🔴

Esta es la única idea conceptual de la presentación, y es la que más vuelve: **es el marco con el que la cátedra evalúa decisiones durante todo el cuatrimestre**.

La pregunta es por qué hacemos lo que hacemos. Por qué nos interesa la tecnología, por qué aprendemos un lenguaje nuevo, un framework nuevo, por qué buscamos performance o un sistema robusto.

**La respuesta que la materia rechaza:** porque es lo nuevo. Perseguir el *bleeding edge* — la tecnología recién salida, todavía sin madurar — no es una razón. Es un riesgo, y la metáfora es literal: el filo sangra, te puede cortar.

**La respuesta que la materia sostiene:** es una cuestión de **incentivos**, y en última instancia de **dar valor al cliente y al negocio**. Generar valor, generar ventas, generar dinero. La tecnología es el medio; el fin está siempre más abajo, en la organización que paga por ese software.

> 🕳️ **Madriguera — Bleeding edge**
> Un escalón más allá del *cutting edge*: tecnología tan nueva que todavía no tiene comunidad, documentación madura ni casos de producción que la respalden. Quien la adopta paga el costo de descubrir sus bugs.
> *Volvé al camino.*

De acá sale el criterio con el que se juzga cualquier elección técnica en esta materia: **una tecnología no es mejor que otra en abstracto — es mejor o peor para un contexto, con sus incentivos y su costo**. Una herramienta más nueva o más de nicho no implica que sea sucesora ni mejora de la anterior.

> **Para el parcial, si te preguntan: ¿con qué criterio se elige una tecnología?**
>
> Con el criterio de los incentivos y el valor que genera para el cliente y el negocio, no por novedad. La elección es siempre relativa al contexto: hay que evaluar qué problema real resuelve, qué costo tiene adoptarla y sostenerla, y qué gana la organización con eso. Que una tecnología sea más nueva o más específica no la convierte en sucesora ni en mejora de la que reemplaza.

---

## 4. Cómo se cursa 🟡

**Comunicación.** Todo pasa por un **grupo de mail de Google** de la materia. Se abre una conversación por consulta, la reciben todos los docentes y responde cualquiera. **No hay campus virtual.** Por ahí llegan también las presentaciones y los apuntes, así que el mail de la facultad hay que mirarlo — es el único canal.

Además cada grupo tiene un **ayudante asignado**, y a los ayudantes se les puede escribir directo.

**Los ayudantes no son correctores.** Esto es explícito: son gente con experiencia en la industria a la que se le puede preguntar cosas, no un buzón donde se deposita la entrega. Usarlos solo para que corrijan es desperdiciarlos.

**"El techo lo ponen ustedes".** La materia tiene un piso definido por el programa, pero no tiene techo. Se pueden pedir temas, incluso fuera del programa. El modelo de cátedra lo permite: **el que viene a dar un tema es porque lo domina** — lo trabaja, lo maneja o lo estuvo estudiando. Cuando hay un tema que no dominan, traen a alguien de afuera que sí. De ahí que las clases las den distintas personas y que existan clases bonus.

La frase que resume la postura: no es una electiva para tachar una casilla en la libreta.

**Presencialidad.** No hay política de asistencia ni porcentaje mínimo. La mayoría de las clases son virtuales; hay algunas presenciales puntuales (laboratorio, la demo del trabajo práctico, los parciales) y se avisan con tiempo. **Las entregas del trabajo práctico son virtuales.**

**Tecnología del trabajo práctico.** No es obligatorio Java, más allá de lo que sugiera el material viejo. **La tecnología se acuerda con el ayudante**, con un criterio de sentido común: si el ayudante no conoce el lenguaje que elegís, no va a poder darte una corrección de alto nivel que te sirva. Hubo trabajos prácticos hechos en distintos lenguajes y tecnologías.

---

## 5. Cómo se aprueba 🟡

La nota de cursada sale de dos instancias que pesan igual:

| Instancia | Cuándo | Peso |
|---|---|---|
| **Trabajo práctico** | Entregas a lo largo del cuatrimestre | Cuenta como si fuera un parcial |
| **Parcial** | Última clase de la cursada, presencial | Un parcial |

**Promoción:** hace falta **8 o más en ambas**. Con eso se accede al **coloquio**, que es una charla adicional — no un examen escrito más.

**Si no se llega:** cualquier combinación por debajo de eso (por ejemplo un 6 y un 8) manda a **final**.

Detalle no menor sobre el coloquio: es una instancia **oral**. Eso significa que todo lo que se entrega hay que poder explicarlo. Material que no se puede defender no suma.

---

## 6. Una nota sobre APIs 🟢

El contenido técnico de esta unidad —APIs, y REST en particular— la cátedra lo **da por sabido**. Se entrega como apunte para estudiar por cuenta propia, en parte con función diagnóstica: sirve para que cada uno mida dónde está parado antes de que el programa se apoye encima.

Eso significa que **no se explica de nuevo más adelante**, pero sí se usa: Arquitectura Web, Microservicios y buena parte del trabajo práctico se paran sobre esto. Es la Parte 1 en adelante de este apunte.

---

## Checkpoint

Respondelas sin volver al texto. Las respuestas van al complemento.

1. ¿Por qué el primer intento de desdoblar TADP en dos cursos no funcionó, y qué decisión de diseño salió de ese fracaso?
2. ¿Cuál es el criterio de éxito que la cátedra declara para alguien que aprobó la materia?
3. Según la materia, ¿por qué NO se elige una tecnología? ¿Y cuál es el criterio que sí vale?
4. ¿Qué significa que "el techo lo ponen ustedes" y qué característica del equipo docente lo hace posible?
5. ¿Qué combinación de notas habilita el coloquio, y qué tipo de instancia es el coloquio?
6. ¿Quién define la tecnología del trabajo práctico y con qué criterio?

---

## Qué viene en la Parte 1

Arranca el contenido técnico. La pregunta de fondo de toda la unidad: **cómo hacemos que dos procesos que corren en máquinas distintas se comuniquen**. Empezamos por lo más bajo —sockets— y por el primer intento de abstraerlos, RPC/RMI, para entender qué problemas dejaron abiertos. Todo lo que viene después (serialización, SOAP, REST, GraphQL, gRPC) son respuestas a esos problemas.

---

**FIN DE LA PARTE 0**
