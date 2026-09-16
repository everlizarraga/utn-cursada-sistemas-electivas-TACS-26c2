# 📘 Apunte maestro — Clase 05 — Microservicios
## Parte 2 de 6 — Del monolito a microservicios: la migración

> **Unidad:** `clase05` · TACS 2C 2026 · Clase del 08/09/2026 · Apunte maestro, Parte 2 de 6 · Leyenda e índice completo en la Parte 1.

**Qué cubre esta parte.** El caso concreto que ordena toda la unidad: un e-commerce monolítico con sus capas, treinta personas y una subida cada dos o tres meses. Cómo se migra sin parar la máquina: elegir el primer pedazo, aislarlo, integrarlo desde el monolito sin borrar lo viejo, hacer el rollout de a poco y recién entonces eliminar. El patrón Strangler Fig como camino alternativo, y qué se hace cuando el refactor es disruptivo y hay que migrar datos.

**Qué NO cubre.** La arquitectura a la que se llega después de N iteraciones y sus observaciones: eso es la Parte 3.

**De dónde venís.** Parte 1: los problemas del monolito (no escala en gente, deploys grandes, una falla arrastra todo) y el hack del balanceador por rutas como alivio de corto plazo. Esta parte arranca donde ese hack deja de alcanzar.

---

## 1. El ejemplo: un sistema de venta online 🔴

Todo lo que sigue en la unidad se cuenta sobre el mismo sistema, así que conocelo bien. Es un e-commerce (símil Garbarino o Amazon) que, como monolito, tiene estas **capas o responsabilidades** adentro:

| Responsabilidad | Qué hace |
|---|---|
| **Acceso al catálogo** | Lee los productos, de una base de datos propia o de un **servicio de terceros** (un proveedor externo que expone su catálogo por API). |
| **Políticas comerciales** | Le agrega a cada producto el **markup** (el margen que se suma al costo para armar el precio de venta), las **promociones** y las **restricciones** (qué no se puede vender, a quién, dónde). |
| **Facetado** | Los **filtros** y **categorías** con los que el usuario recorta el catálogo (marca, precio, talle); incluye el orden por usuario. |
| **Destacados** | Los productos que se muestran resaltados en la home o en una categoría. |
| **Front** | Lo que se sirve al navegador. |
| **Header** | Login / sesión y carrito de compras. |
| **Footer** | Contenido estático y suscripción al newsletter. |
| **Búsqueda de productos** | El buscador. |

Nada de esto es exótico: es lo que tiene cualquier tienda online. Lo que importa es que **todo eso vive en un solo artefacto**, y que cada responsabilidad tiene un perfil distinto: el catálogo es lectura intensiva contra terceros, políticas comerciales es lógica de negocio pura, el front es presentación, y el login y la sesión los necesita todo el mundo. Tenelo presente para las Partes 3 y 5, donde ese perfil distinto es lo que justifica (y complica) partirlo.

---

## 2. Iteración #1: el sistema monolítico 🔴

Así arranca la historia:

```
   ┌───────────────────────────┐
   │                           │         ┌──────┐
   │        APLICACIÓN         │────────►│  DB  │
   │                           │         └──────┘
   └───────────────────────────┘
```

- Todo el código está en **un solo artefacto / deployable** (WAR, JAR, etc.).
- **1 GRAN equipo de 30 personas.**
- **1 GRAN subida cada 2 / 3 meses.**
- **1 GRAN ciclo de bugfixing** el mes siguiente a la subida.

Es decir: el equipo entero trabaja sobre un solo artefacto, con ciclos en los que se sube todo junto y después se pasa un período arreglando lo que se rompió, antes de la siguiente versión. Es exactamente la escena de la Parte 1 (secciones 4 y 6), con números.

Dos aclaraciones honestas. Primera: **esta cadencia hoy es lenta.** Con release trains, un monolito puede deployarse bastante más seguido (Parte 1, §6.5). Segunda: aun así, **es una forma de trabajo completamente válida.** Hay empresas que funcionan así y funcionan bien. El punto es que, a medida que crece, van apareciendo todos los problemas de la Parte 1, y llega el momento de pasar a una solución de microservicios.

---

## 3. Cómo se come un elefante 🔴

**Pedazo a pedazo.**

La frase resume la forma de trabajo de toda la migración, y hay que tomarla en serio porque la alternativa (parar todo y rehacer) no existe:

- **El modelo iterativo e incremental me permite agregar valor mientras la rueda sigue girando.** El negocio no espera: mientras migrás, se siguen vendiendo productos y se siguen pidiendo features.
- **No es posible detener la máquina para hacerla de nuevo.** Por eso tengo que ir evolucionando la arquitectura **componente a componente**.
- **No cambio el ciclo de vida a "todo o nada".** Sigo trabajando de manera iterativa; lo que cambia es que ahora una parte de cada iteración va a la migración.

Lo que sí hay que decidir es **qué hago primero, y cuándo**. Eso es la iteración #2.

---

## 4. Iteración #2: empiezo a pensar en microservicios 🔴

El procedimiento, paso a paso. Cada paso tiene su porqué; si te saltás uno, te lo cobra el siguiente.

### 4.1 Tomar un "concern" (¿alcanza con cualquiera?)

Tomo un **concern** o aspecto del sistema: una responsabilidad de la tabla de la sección 1. Por ejemplo, el **catálogo**.

**¿Alcanza con tomar cualquiera? No.** Tengo que tomar algo que de alguna manera **me apriete, me incomode**:

- una **API difícil** de mantener o de consumir;
- una **performance complicada** (necesita infra distinta al resto, es lento, se lleva los recursos);
- algo que tiene, o pide, una **base de datos propia**.

La razón es de valor: la migración tiene costo, y el primer pedazo tiene que ser uno que, al salir del monolito, **me agregue valor de inmediato**. Migrar primero algo que no molesta es pagar el costo sin cobrar el beneficio. El catálogo del ejemplo cumple: pega a terceros, tiene su base, y su performance condiciona todo lo que se muestra.

### 4.2 Aislar y reescribir

Ese concern **lo aíslo**: identifico cuál es su API, cuáles son sus módulos, cuáles son sus clases. Lo aparto. Y **creo una caja que haga esa parte**: la reescribo como un servicio nuevo (hoy reescribir es mucho más fácil que antes), **la pruebo y la comparo contra lo anterior**. La caja nueva tiene que hacer lo mismo que hacía el módulo viejo, y eso se verifica contra el viejo, no contra una spec.

### 4.3 Integrar desde el monolito, sin eliminar lo viejo

Acá está la clave de que la rueda siga girando: **lo integro desde la aplicación monolítica sin eliminar lo viejo.** Cuando el monolito necesita el catálogo, en vez de llamar al módulo interno, hace **una llamada HTTP a la cajita nueva** y vuelve. El código viejo sigue ahí, intacto, disponible.

```
   ┌────────────────────────────────┐        HTTP        ┌────────────┐
   │          APLICACIÓN            │───────────────────►│  Catálogo  │  ← la caja nueva
   │           ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐│                    └────────────┘
   │             Catálogo viejo     │  ← sigue adentro, sin borrar
   │           └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘│
   └────────────────────────────────┘
```

Fijate en la flecha: es **HTTP** (en este caso web; podría ser otro protocolo). Y fijate que la flecha va **de servicio a servicio**, no del monolito a la base de datos del catálogo. Ese detalle se desarrolla en 4.6.

### 4.4 Balancear desde el monolito: rollout paulatino

¿Tengo que mandar **todo** el tráfico a la caja nueva de una vez? No. Desde el monolito puedo **balancear**: hacer un **rollout** (despliegue gradual a los usuarios) por algún aspecto, o por un porcentaje.

- **Por porcentaje**: el 5 % de las llamadas al catálogo van a la caja nueva; el resto, al módulo viejo. Si anda, subo al 20 %, al 50 %, al 100 %.
- **Por feature**, **por ID** de usuario, **por cliente** ("para tal cliente sí, para tal otro no").

Esto se implementa con **feature flags**: booleanos o condiciones en la aplicación (a nivel infra, a nivel código o a nivel configuración) que deciden cómo balancear una parte de la aplicación. Cambiar el flag no requiere deploy. Hay un artículo de Martin Fowler sobre feature flags que las recorre por nivel; lectura opcional.

⚠️ Esto **no es A/B testing**, aunque se parezca. A/B testing es una herramienta de **negocio**: mostrar dos variantes a dos grupos de usuarios para medir cuál convierte mejor. Acá se usa el mismo mecanismo de balanceo, pero con un objetivo técnico: validar la caja nueva con tráfico real y bajo control.

### 4.5 Dejar de usar lo viejo, y eliminarlo

Una vez que el rollout llega al 100 % y la caja nueva se sostiene, **dejo de usar lo viejo y lo elimino**. El "Catálogo viejo" del diagrama desaparece del monolito. Este paso se olvida seguido, y el resultado es un monolito con módulos muertos adentro: código que nadie llama pero que sigue compilando, testeándose y deployándose.

### 4.6 Qué cambió: el contrato

Mirá de nuevo el diagrama de 4.3 y notá dos cosas que parecen básicas pero son el corazón de todo lo que viene:

1. **La flecha es entre servicios.** La aplicación **no conoce la base de datos del catálogo**. Ni siquiera sabe **si** el catálogo tiene una base de datos.
2. **Lo que el catálogo expone es una API** (REST, o la que se haya diseñado). Esa API es un **contrato**: yo, monolito, no sé cómo está hecho el catálogo por adentro; me manejo solo con el contrato que me expone.

Ese es el cambio de fondo respecto del monolito, donde cualquier módulo podía tocar cualquier tabla. A partir de acá, la única forma de hablar con el catálogo es su API, y el equipo del catálogo puede cambiar todo lo de adentro (la base, la tecnología, el modelo) sin avisarle a nadie, **mientras respete el contrato**. Lo que pasa cuando el contrato tiene que cambiar está en la Parte 3.

> **Para el parcial, si te preguntan: ¿cómo se migra un monolito a microservicios?**
> De manera iterativa e incremental, sin detener la aplicación: se toma un concern que incomode (API difícil, performance complicada, base propia), se aísla y se reescribe como un servicio nuevo que se prueba contra el viejo, se integra desde el monolito por HTTP sin eliminar lo viejo, se balancea el tráfico con un rollout paulatino (por porcentaje, feature, ID o cliente, con feature flags), y cuando la caja nueva se sostiene al 100 %, se elimina el código viejo. Se repite N veces con las distintas partes.

> **Para el parcial, si te preguntan: ¿por qué el monolito no conoce la base de datos del catálogo migrado?**
> Porque el catálogo expone una API, y esa API es el contrato: el monolito se comunica solo a través de ella y no sabe (ni le importa) cómo está implementado el catálogo por adentro, incluida su base de datos. Eso es lo que permite que cada servicio cambie sus internos sin afectar a sus clientes.

---

## 5. El otro camino: Strangler Fig 🟡

Hay una variante del mismo proceso que entra por el otro lado. En vez de que el monolito llame a la caja nueva, pongo **un proxy o un API Gateway adelante** de todo (una pieza que recibe todas las requests y las rutea; el API Gateway además puede sumar lógica encima: autenticación, límites, transformación), y separo parte de la lógica en un servicio nuevo. Al principio, todas las rutas van al monolito. De a poco, **algunas rutas dejan de ir al monolito** y van al servicio nuevo, hasta que el monolito queda vacío. Es el patrón **Strangler Fig** (por la higuera estranguladora, que crece alrededor de un árbol hasta reemplazarlo).

```
   Camino de la sección 4                    Camino Strangler Fig
   (entrás siempre por el monolito)          (entrás por el contrato, el gateway)

   usuario ──► MONOLITO ──HTTP──► Catálogo    usuario ──► GATEWAY ──┬──► MONOLITO
                  │                                                 │
              (llamadas                                             └──► Catálogo
               internas)                                        (cada vez más rutas van acá)
```

La diferencia: en el camino de la sección 4 **entrás siempre desde el monolito**, que decide a quién llamar; en Strangler Fig **armás el contrato primero** (el gateway) y de ahí las rutas van al monolito o al servicio nuevo. En Strangler Fig persiste un detalle: una parte del monolito puede seguir llamando a otra parte internamente, así que al final es un poco lo mismo. **Los dos caminos son igualmente válidos.** Elegís según dónde te sea más fácil cortar: en las llamadas internas del monolito o en la entrada.

---

## 6. Sobre los refactors de arquitectura 🔴

Idealmente, cada iteración es como la de la sección 4: la caja nueva convive con la vieja, el rollout es paulatino, y si algo falla se vuelve atrás con un flag. **Idealmente.**

### 6.1 Refactors disruptivos

**Algunos refactors son disruptivos: no pueden convivir con el modelo anterior.** No tienen retrocompatibilidad, o no pueden hacerse rollout de manera paulatina. El caso típico es un **cambio de arquitectura complicado, por ejemplo algo transaccional**: si la operación tiene que ser atómica (Parte 1, §5), no puedo tener la mitad de las escrituras yendo a un lado y la mitad al otro.

- **Idealmente deberíamos tratar de evitar estos cambios.** Pero no siempre podemos.
- **En caso de poder:** rollout paulatino con balanceo por porcentaje (lo que en la jerga se suele llamar "A/B testing", con la salvedad de 4.4), e ir probando cómo responde. Es decir, el camino de la sección 4.
- **En caso de no poder:** no solo hay que testear mucho (**test funcional, de carga**, etc.); por lo general hay que hacer **migración de datos**.

### 6.2 Migración de datos

El problema, con el ejemplo: el monolito interactúa con el catálogo viejo, que tiene **su** base de datos. El catálogo nuevo tiene **otra** base, propia. Si escribo en el catálogo viejo, no escribo en el nuevo. Si escribo en el nuevo, no escribo en el viejo. Mientras los dos conviven, las dos bases **divergen**.

```
   MONOLITO ──► Catálogo viejo ──► DB vieja      ┐  escribo acá...
                                                 ├─  ...y el otro no se entera
   MONOLITO ──► Catálogo nuevo ──► DB nueva      ┘
```

¿Qué hago para **reconciliar** las dos bases? Dos caminos:

- **Replicar escrituras**: una herramienta de migración de datos que, por atrás, haga que **cada escritura a la tabla vieja vaya también a la tabla nueva**. Estas herramientas las dan los proveedores cloud (AWS tiene) o las arma un DBA (*database administrator*, la persona a cargo de las bases).
- **Dump**: levantar un volcado completo de la base vieja y cargarlo en la nueva, en un momento dado.

En los dos casos, el objetivo es **no perder consistencia durante la migración**: que al terminar, la base nueva tenga todo lo que tenía la vieja más lo que pasó en el medio. Y como dice el punto siguiente, **los dos modelos pueden convivir por un tiempo**; hay que pensarlo, y siempre depende de la feature que estés migrando.

### 6.3 Cosas que no me gustan, DE FORMA TEMPORAL

Durante una migración **puedo hacer cosas que no me gustan, de forma temporal.** El ejemplo canónico: **integrar dos aplicaciones a través de la capa de datos**, es decir, comunicar las bases directamente en vez de pasar por las APIs. Idealmente no pasa, no es la idea (va contra el contrato de 4.6 y vas a ver en la Parte 3 que es una regla dura: solo la app dueña conoce su base). Pero en el medio de una migración puede ser la única forma de sostener la consistencia, y está bien **mientras tenga fecha de vencimiento**.

### 6.4 El refactor tiene costo

**El tiempo que me tomo refactorizando tiene costo.** Es tiempo que no va a features, y el negocio lo ve. Por eso el orden importa (4.1): cada pedazo que se extrae tiene que pagar su propio costo con el valor que devuelve.

> **Para el parcial, si te preguntan: ¿qué hacés cuando un refactor de arquitectura es disruptivo y no puede convivir con el modelo anterior?**
> Primero intento evitarlo. Si no puedo hacer rollout paulatino, pongo mucho énfasis en el testing (funcional y de carga) y hago migración de datos: replico las escrituras de la base vieja a la nueva o cargo un dump, para no perder consistencia mientras los dos modelos conviven. Puedo aceptar temporalmente soluciones que no me gustan, como integrar apps por la capa de datos, siempre con fecha de vencimiento.

---

## 7. Iteración #N 🟢

Idealmente, el proceso de la sección 4 se hace **N veces**, con las distintas partes del monolito: el catálogo, después políticas comerciales, después facetado, y así. Cada iteración aísla un concern, lo convierte en una caja con su API y su base, y borra lo viejo. Lo que queda al final es la **arquitectura refactorizada**: un mapa de ocho aplicaciones y cuatro bases que es el objeto de estudio de la Parte 3.

---

## Checkpoint — Parte 2

*(Sin respuestas: van al complemento.)*

1. ¿Por qué una migración de monolito a microservicios tiene que ser iterativa e incremental? ¿Qué pasaría con la alternativa?
2. Te toca elegir el primer concern a extraer. ¿Con qué criterio lo elegís y por qué "cualquiera" no sirve?
3. ¿Por qué la caja nueva se prueba "contra lo anterior" y no contra una especificación?
4. ¿Qué gana el equipo integrando la caja nueva desde el monolito **sin borrar el módulo viejo**?
5. Explicá el rollout paulatino: qué es una feature flag y por qué se prefiere a mandar el 100 % del tráfico de una vez.
6. ¿En qué se diferencia el rollout técnico de un A/B testing?
7. Después de la iteración #2, el monolito "no sabe si el catálogo tiene base de datos". ¿Qué habilita eso?
8. ¿Qué diferencia hay entre el camino "desde el monolito" y el Strangler Fig? ¿Cuándo elegirías cada uno?
9. Un refactor es disruptivo porque involucra una operación transaccional. ¿Qué medidas tomás y por qué el rollout paulatino no alcanza?
10. Describí las dos formas de reconciliar la base vieja con la nueva y qué buscan garantizar.

---

## Qué viene en la Parte 3

La arquitectura a la que se llega después de N iteraciones, dibujada completa: ocho aplicaciones, cuatro bases, un usuario y un proveedor externo. Qué gana cada equipo con su ciclo de vida propio y qué le cuesta a la organización. Las observaciones sobre el mapa (sesión y usuario cross a todo, personalización, bases distintas, integraciones solo por API, apps que se reutilizan y el BFF), el problema de versionar APIs ("APIs are forever"), la autenticación entre servicios con mTLS, y por qué las llamadas circulares y el N+1 duelen más cuando hay red en el medio.

---

**FIN DE LA PARTE 2 — Apunte maestro clase05 — Microservicios**
