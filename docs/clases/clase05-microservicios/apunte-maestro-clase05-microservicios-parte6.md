# 📘 Apunte maestro — Clase 05 — Microservicios
## Parte 6 de 6 — Requisitos de la organización, y cierre

> **Unidad:** `clase05` · TACS 2C 2026 · Clase del 08/09/2026 · Apunte maestro, Parte 6 de 6 · Leyenda e índice completo en la Parte 1.

**Qué cubre esta parte.** Lo que una organización tiene que tener para que microservicios funcione: provisionamiento rápido y cloud, escalabilidad horizontal transparente, monitoreo con herramientas específicas (APM) y trazabilidad de requests, ciclos de vida independientes, testing con ambientes bajos consistentes, y disponibilidad. Cada requisito es la respuesta a un problema de las Partes 3 a 5. Cierra con las dos preguntas de siempre: ¿monolito o microservicios hoy?, y ¿qué es "saber microservicios"?

**Qué NO cubre.** Cómo se implementa cada requisito: infraestructura, observabilidad y service mesh tienen sus clases (6, 10 y 13).

**De dónde venís.** Parte 5 terminó con "no todas las organizaciones están preparadas; hay que evaluarlo y tener la espalda suficiente". Esta parte es la lista de con qué se evalúa.

---

## 1. No todas las organizaciones están preparadas 🔴

**No todas las organizaciones están preparadas para dar el salto.** Lo que sigue son recomendaciones: seis capacidades que hay que tener antes, no después, de partir el monolito. La primera es la más básica.

### 1.1 Provisionamiento rápido, y por qué cloud

**Provisionamiento** es conseguir infraestructura (máquinas, nodos, instancias) y ponerla a disposición de una aplicación. Necesito que sea **rápido**: **para escalar, si es necesario, donde aprieta el zapato**. Si Facetado está saturado, quiero darle máquinas a Facetado hoy, no en tres semanas cuando compras apruebe el servidor.

**Esta idea se lleva muy bien con las infraestructuras denominadas "cloud"**, porque aprovechan el feature llamado **escalabilidad elástica**: la infraestructura crece y se achica según la demanda. Es, de hecho, lo que viene diciendo toda la materia: si puedo usar Docker (clase 02), puedo usar Kubernetes, y puedo **escalar horizontalmente de manera automática o manual**, tengo **una ventaja gigante** sobre alguien con un monolito. De vuelta, esto suma complejidad: si escalo horizontalmente tengo que decidir **con qué límites, con qué nodos, qué pasa si me quedo atascado**. Son decisiones que hay que tomar.

> 🕳️ **Madriguera — Kubernetes**
> El orquestador de containers que, entre otras cosas, automatiza el provisionamiento y el escalado. Ya sonó en la clase 02 y vuelve con service mesh en la clase 13. Por ahora, alcanza con saber para qué existe.
> *Volvé al camino — esto se profundiza aparte, otro día.*

---

## 2. Escalabilidad 🔴

Sobre el mapa de la Parte 3, imaginate que **Facetado** es la caja que aprieta: todos los usuarios filtran, y Facetado no da abasto. Escalar en microservicios es escalar **esa** caja:

```
                                        ┌──────────┐
                              ┌────────►│ Facetado │──┐
                              │         └──────────┘  │
   Front ──► (balanceador) ───┼────────►┌──────────┐  ├──► Políticas Comerciales, Sesión y usuario
                              │         │ Facetado │──┤
                              │         └──────────┘  │
                              └────────►┌──────────┐  │
                                        │ Facetado │──┘
                                        └──────────┘
   el Front no se entera: sigue llamando a "Facetado"; el resto del mapa no cambia
```

Acá juega sobre todo la **escalabilidad horizontal** (agregar réplicas, clase 04) que ya viste, y lo que se necesita es:

- **Agregar servers al cluster.**
- **Incorporar los servers a PROD** (producción).
- **Balancear la carga de forma fácil y dinámica.**

Lo importante: si yo agrego nodos, **mis clientes, salvo casos súper específicos, no se van a enterar**; va a ser **transparente**. Compará con el monolito (Parte 1, §4.7): allá escalar era escalar todo. **Escalar los servicios es algo fundamental**, y solo funciona si el requisito 1 (provisionamiento) está resuelto.

---

## 3. Monitoreo 🔴

**No es lo mismo mirar una aplicación que mirar ocho.** Es un pro y una contra a la vez: puedo ver cada pieza por separado, pero **tengo que mirarlas todas**.

- **Ahora tengo más puntos de falla y más aplicaciones para seguir.**
- **Algo que quizás antes hacía de forma rudimentaria** (mirar logs, tirar comandos en el server, JMX, que son las extensiones de Java para inspeccionar un proceso en ejecución) **ya no me alcanza**. Lo que antes era ver **un** log ahora es ver **N** logs.
- **Necesito herramientas específicas para poder monitorear**: con **alarmas reactivas** (avisan cuando algo ya falló) **y proactivas** (avisan cuando algo va a fallar: la latencia sube, el disco se llena), **pero no manuales**. Dependiendo de la importancia del negocio, puede llegar a **7×24**.

### 3.1 Trazabilidad: ¿qué request es cuál?

El problema nuevo: **una request entra por el Front y pasa por ocho servicios.** En cada uno deja un log. ¿Qué request es cuál? ¿Qué request salió de qué request? Necesito mecanismos para **reconstruir el camino** de una request a través de todos los servicios. Se puede resolver con un **ID compartido** que viaje en cada llamada, con **headers**, o con un **servicio de telemetría** que lo haga por vos. No hay una única respuesta, **pero algo tengo que hacer**. Antes, una request era una response y listo. La **observabilidad** (poder ver qué pasa adentro del sistema a partir de sus logs, métricas y trazas) **levanta muchísimo la vara** de todo esto; tiene su clase, la 10, con tracing distribuido.

### 3.2 ¿Y si me cae Log4Shell? 🙈

El ejemplo de lo que cuesta tener ocho (o treinta mil) artefactos en vez de uno. **Log4Shell** fue una vulnerabilidad en una library de logging de Java muy común (log4j), que estaba en un montón de servicios, y que permitía **ejecución remota de código**: le mandabas una request especial y ejecutabas código en el servidor. Fue una **zero-day** (una vulnerabilidad que se descubre cuando ya está siendo explotada, sin parche disponible), y encima fue descubierta en Minecraft. Hubo que salir a **parchear todos los servicios**.

Si tenés un monolito, **parcheás el monolito y sos feliz**. Si tenés **30.000 microservicios**, como alguna empresa grande de la región, tenés que parchear **30.000 artefactos**. Imaginate: los managers de cada vertical con un Excel ("esto lo parché, esto no"), el equipo de infra corriendo scripts para ver qué es vulnerable, un deadline, y un montón de equipos perdiendo un montón de horas en resolverlo y en verificar que quedó resuelto. Con el monolito no te pasaba.

Conclusión: **las capacidades que necesito son mucho más altas, y me pueden golpear de una manera mucho más extendida.**

### 3.3 APM: New Relic, Datadog, Grafana

Las herramientas específicas se llaman **APM** (*Application Performance Monitoring*). Las de referencia: **New Relic** y **Datadog** (las más potentes, pagas) y **Grafana** (una versión open source de algunas de las cosas que tienen las otras dos).

**Qué son.** Imaginate un portal web donde entrás y tenés **el listado de todos tus servicios**, quizás tus bases de datos, **alertas**, **latencia**, insights de **cuánto tarda cada endpoint**, **cuántas requests por minuto**, cuál es el **percentil 90 de latencia** (el tiempo por debajo del cual responde el 90 % de las requests; mucho más útil que el promedio, porque muestra la cola lenta), cómo están tus **pods o nodos**, cuánta **memoria** consume un servicio, si hay **throttling** (si al servicio le están frenando requests por límite de recursos). Podés sacar métricas, contadores, un montón de cosas. Tres vistas típicas, para que te imagines la herramienta:

- **Vista de bases de datos**: una tabla con cada instancia, sus lecturas y escrituras por segundo, conexiones abiertas y **replication lag** (cuántos segundos atrás está una réplica respecto del master), con las instancias en problema en rojo y una lista de eventos y alertas recientes.
- **Vista por endpoint de una app**: cada endpoint con su tiempo de respuesta, y para el seleccionado, un desglose de ese tiempo (cuánto fue **encolado**, cuánto fue **aplicación**, cuánto fue **red**), el throughput y la transferencia de datos en el tiempo.
- **Mapa de servicios**: el servicio en el centro, a la izquierda quiénes lo llaman y a la derecha de qué depende (base, cache, otros servicios), cada uno con su latencia y sus llamadas por minuto. Es el mapa de la Parte 3, pero **vivo** y con números.

**Cuál es la premisa básica.** Cuando levantás tu aplicación, en algún lado tenés que **llamar a una SDK**, o en Java meter un **agente**: un binario o library que se mete en tu aplicación y le da esa "magia" (reporta todo lo anterior). ¿Cómo la metés? Con una **API key**: pagás el servicio, te dan la key, y se va armando el ecosistema de monitoreo. Conclusión operativa: **todo servicio tiene que salir con esa library**. Te la da la empresa, no la tenés que hacer vos; vos la tuneás y la configurás a tus necesidades. (Es uno de los "un montón de cosas *out of the box* desde el día uno" de la Parte 3, §3.4.)

**Contras.** Dos, y son grandes:

- **Son muy caros.**
- **Tienen vendor lock-in**: te acoplan a su library, así que si querés cambiar de proveedor no es tan fácil. Con el tiempo surgieron estándares como **OpenTelemetry** (clase 10) para que la aplicación reporte en un formato neutro y el proveedor sea intercambiable; pero en principio hay bastante acoplamiento con el vendor.

Aun así, **es fundamental**: no se imagina una empresa grande sin algo de esto.

> **Para el parcial, si te preguntan: ¿qué es un APM y por qué es un requisito para microservicios?**
> Un APM (Application Performance Monitoring; New Relic, Datadog, Grafana) es una herramienta que, mediante un agente o SDK que se incluye en cada servicio con una API key, centraliza en un portal el estado de todos los servicios: latencia por endpoint, requests por minuto, percentiles, recursos, alertas, mapa de dependencias. Es requisito porque con N aplicaciones ya no alcanza con mirar logs a mano: hay más puntos de falla, hay que reconstruir el camino de cada request a través de los servicios, y hacen falta alarmas automáticas. Contras: son caros y generan vendor lock-in.

---

## 4. Ciclo de vida 🔴

Antes era **un** ciclo de vida, el del monolito. Ahora tengo **N ciclos de vida**, y **para poder tener más equipos, necesito que sean independientes en su ciclo de vida**. Dos planos:

- **Procesos**: que cada equipo **pueda completar todo su ciclo de vida sin intervención externa**, sin blockers de afuera. Nadie tiene que pedir permiso a otro equipo para deployar.
- **Técnico**: un **esquema de deploy rápido**. **Más aplicaciones implica más deploys**, y si cada uno cuesta, la ganancia se pierde.

Lo que eso provoca es que **tengas deploys mucho más habituales**: desde **uno por día hasta N por día**, sin downtime ni inconvenientes. Y ahora sí, las preguntas que en la Parte 1 (§6.5) quedaban incómodas para el monolito, respondidas desde acá:

| Pregunta | Monolito (Partes 1 y 2) | Microservicios |
|---|---|---|
| **¿Cada cuánto?** | Cada dos o tres meses, con las features acumuladas | **De uno a N por día**, sin downtime. |
| **¿A qué hora?** | A las 4 AM, cuando hay menos tráfico | **Probablemente a cualquier hora**: estoy cambiando una pieza chiquita de un engranaje de un sistema más complejo. Varía según la empresa. |
| **¿De qué forma? ¿Disruptivo o no?** | Un lote grande, con un técnico por parte | Depende de la feature, pero **probablemente sea menos disruptivo**. |
| **¿Puedo hacer un deploy fácil?** | No: build, CI, 4.000 tests, todo | Sí: binario liviano, tests de mi dominio. |
| **¿Qué pasa si algo sale mal? ¿Rollback?** | Rollback de todo, hotfix caro | **El rollback es muy barato**: es hacer rollback de **un** deploy. |

Es decir: **poder realizar deploys no disruptivos de forma fácil y segura.** Ese es el requisito, y es lo que hace real el "cada caja tiene su ciclo de vida" de la Parte 3 (§2).

---

## 5. Testing 🔴

**Necesito poder probar de forma rápida y barata.** Lo bueno: los tests son mucho **más chiquitos, más localizados**: estoy testeando **mi dominio**, no 4.000 tests de todo el mundo (Parte 1, §4.3).

### 5.1 La desventaja: los ambientes bajos

Hay algo que es cierto y es una desventaja de microservicios: **mantener ambientes actualizados para múltiples aplicaciones puede ser un dolor de cabeza.** Los **ambientes bajos** (beta, sandbox, staging; el nombre varía según la empresa, y en 5.2 "sandbox" se usa con un sentido preciso; son las copias del sistema donde se prueba antes de producción) hay que **mantenerlos consistentes**.

El caso: yo desarrollo una feature en mi microservicio, **la subo a beta, y rompo beta**, porque mi servicio tiene un bug. Genial: rompí beta **en mi servicio**, no pasa nada. Pero después viene otro equipo que quiere hacer una prueba, y su prueba **me llama a mí**, y yo estoy roto. Con ocho servicios y ocho equipos subiendo a beta cuando quieren, **beta puede estar rota por un montón de lados**, o tener cosas no contempladas o no hechas. Entonces **"testeo en beta y subo a prod" no vale**: beta no es confiable por definición. Mantener un ambiente bajo que funcione **no es fácil**.

### 5.2 Sandbox, y el costo de la infra duplicada

¿Cómo se resuelve? Hay empresas que tienen lo que llaman **sandbox**: **una copia de producción donde los cambios se meten sí o sí de a uno**. Tengo algo idéntico a prod, y puedo meter **un** cambio para hacer una prueba. No es como beta, donde cualquiera puede deployar en cualquier momento y romper todo.

| | Beta | Sandbox |
|---|---|---|
| Quién deploya | Cualquiera, cuando quiere | De a uno, controlado |
| Estado | Puede estar roto por varios lados | Idéntico a prod más un cambio |
| Sirve para | Probar tu servicio en contexto | Validar un cambio antes de prod |

El costo: no solo hay que **meter orden a nivel organización** (quién sube a sandbox y cuándo), sino que hay que tener **bastante infra**. No triplicar, pero sí **la misma disposición, la misma topología de red, la misma infra, probablemente menos escalada**. Y eso suma complejidad.

### 5.3 Beta simple y dinámica: por ejemplo, con un header 🟢

Un requisito más ambicioso: un ambiente de beta **simple y dinámico**, por ejemplo **con un header**. La idea, en una línea: la request lleva un header que indica que debe ir a la versión beta del servicio, y el ruteo la manda ahí, sobre la misma infraestructura de producción, sin levantar un ambiente entero aparte.

> **Para el parcial, si te preguntan: ¿por qué el testing es a la vez más fácil y más difícil con microservicios?**
> Más fácil porque cada equipo testea su propio dominio con tests chicos y localizados. Más difícil porque hay que mantener ambientes bajos consistentes para muchas aplicaciones: en beta cualquiera puede deployar y romper, así que "probar en beta" no garantiza nada; se necesita un sandbox (copia de producción con cambios de a uno), que exige orden organizacional y replicar la topología de infraestructura, con su costo.

---

## 6. Disponibilidad 🔴

**"Una cadena es tan fuerte como su eslabón más débil."** En el mapa, si Sesión y usuario se cae, se cae todo (Parte 3, §3.1). Entonces:

- **¿Qué pasa si una de las aplicaciones falla? ¿Cómo manejo un eventual fallo?** Con los mecanismos de control de backpressure de la Parte 4.
- **Necesito estar preparado para una eventual caída de un server o una app.** Cualquier parte de mi aplicación se puede degradar.
- **Todas mis aplicaciones críticas tienen que ser HA** (*High Availability*, alta disponibilidad: que sigan funcionando aunque falle alguna de sus piezas).
- **Esto también aplica para las bases de datos.**

### 6.1 Por qué HA es muy difícil

Seguramente lo escuchaste en otras materias, y la verdad es que **la high availability es muy difícil**, tanto como la consistencia en NoSQL (clase 04). Seguí la escalera:

1. "Tengo **más de una instancia** de mi aplicación, es HA." **No**: se cae la base de datos y se cayó la aplicación.
2. "Bueno, pero tengo **la base de datos replicada**, y no se cae." **No**: se cae **la región de AWS**, o la *availability zone* (una región es un conjunto de datacenters en una zona geográfica; una availability zone es uno de los datacenters aislados dentro de ella), y se cayó tu aplicación entera. Es más: **se cayó tu empresa entera**, probablemente. Seguro viste alguna noticia de "se cayó us-east-1" (la región más usada de AWS) y con ella medio internet.
3. ¿Cómo lográs HA de verdad? **Tenés que tener disponibilidad por región**: tu sistema replicado en más de una región del mundo. **Es carísimo.** Tenés que estar dispuesto a eso.

Quizás con el monolito era mucho más fácil, o el HA aplicaba de otra manera: había una cosa sola que mantener arriba, no ocho más sus bases.

> **Para el parcial, si te preguntan: ¿qué es high availability y por qué es difícil en microservicios?**
> Que las aplicaciones críticas (y sus bases) sigan funcionando aunque falle alguna pieza. Es difícil porque cada nivel tiene un punto de falla superior: varias instancias no alcanzan si se cae la base; la base replicada no alcanza si se cae la región o availability zone del proveedor cloud; la HA real exige disponibilidad por región, que es carísima. En microservicios el sistema es una cadena de servicios, y es tan fuerte como su eslabón más débil, por lo que hay que preparar cada uno para degradarse de forma controlada.

---

## 7. Cierre: las dos preguntas 🟡

### 7.1 ¿Monolito o microservicios? ¿Qué conviene hoy?

Lo que probablemente te dijeron en Diseño de Sistemas, "arrancá por un monolito y cuando crezcas partilo, no arranques por microservicios", **está bien**. Y hoy, con la IA, **las cosas apuntan a estar más cerca que antes**. ¿Por qué? Porque **un LLM puede leer código que está junto más fácil**. Si tengo un *design system*, lo apunto una vez sola. Si tengo un monorepo de frontend y tengo que hacer un módulo nuevo, tengo el módulo de al lado para copiarme el código y las buenas prácticas, sin hacer un repo nuevo ni construir todo el contexto de nuevo.

Desde hace años se veía un poco más **servicios** que **microservicios** (el caso Amazon de la Parte 5, §6.3), y hoy vamos todavía más por ahí. Se puede googlear como **SOA** (*Service Oriented Architecture*): medio lo mismo que vimos, pero **con servicios más grandes**. No hay respuesta correcta; hay criterio, y esta unidad es el criterio.

### 7.2 ¿Qué es "saber microservicios"?

Muchas vacantes piden "saber microservicios". ¿Cuánto hay que saber para decir que sabés? La respuesta de la cátedra: **esta clase te da una buena idea.** Si sabés lo que está acá, lo entendés y **sabés dar ejemplos**, sabés más que el 80 % de los desarrolladores que dicen saber microservicios. Estás capacitado para decir que sí.

Sobre las vacantes que piden Kafka y Kubernetes: **hay más humo del que parece.** Muchas veces recursos humanos les pregunta a varios equipos qué tecnologías usan y las mete todas en una publicación, y una persona que sepa todo eso es muy difícil de encontrar (y pedírselo a un junior, más). Kafka es una pieza complejísima de entender; Kubernetes, peor. Y es casi un antipatrón que a alguien que entra a trabajar en, digamos, facturación le pidan Kubernetes: **la empresa debería tenerlo abstraído** (Parte 3, §5.2). Si los sabés, buenísimo, adelante; pero no es lo que define saber microservicios.

---

## Checkpoint — Parte 6

*(Sin respuestas: van al complemento.)*

1. ¿Qué es el provisionamiento rápido y por qué microservicios "se lleva bien con cloud"?
2. Facetado está saturado. Describí cómo se escala en microservicios y qué ven los clientes de Facetado.
3. ¿Por qué mirar logs a mano deja de alcanzar? ¿Qué diferencia hay entre alarmas reactivas y proactivas?
4. Una request atraviesa ocho servicios. ¿Cómo reconstruís su camino? Nombrá tres mecanismos.
5. ¿Qué enseña Log4Shell sobre el costo operativo de microservicios?
6. ¿Cómo se integra un APM a un servicio y qué te da a cambio? ¿Cuáles son sus dos contras?
7. Respondé las preguntas del deploy (frecuencia, hora, forma, facilidad, rollback) para microservicios y comparalas con el monolito.
8. ¿Por qué "testeo en beta y subo a prod" no es válido? ¿Qué es un sandbox y qué cuesta tenerlo?
9. Reproducí la escalera de por qué HA es difícil: instancias, base replicada, región.
10. ¿Por qué hoy las arquitecturas tienden a servicios más grandes (SOA)? ¿Qué tiene que ver la IA?
11. ¿Qué significa "saber microservicios" según esta unidad? ¿Qué tiene de humo una vacante que pide Kafka y Kubernetes a un junior?

---

## Fin de la unidad

Las seis partes, en una frase cada una:

1. El monolito es la decisión correcta al principio, y sus problemas (gente, deploys, una falla que arrastra todo) son los que empujan a partirlo.
2. Se parte pedazo a pedazo, sin parar la máquina, y lo que queda entre las piezas es un contrato.
3. Cada caja gana su ciclo de vida; a cambio, contratos que son para siempre, autenticación entre servicios y un equipo de infra.
4. Cuando una caja falla, sin timeouts, circuit breakers y colas, la falla llega hasta el usuario y el F5.
5. Todo lo anterior tiene contracara: overhead, complejidad, management, y a veces conviene volver.
6. Para que funcione, la organización necesita provisionamiento, escalabilidad, monitoreo, ciclos de vida, ambientes y HA. Y criterio.

---

**FIN DE LA PARTE 6 — Apunte maestro clase05 — Microservicios · FIN DE LA UNIDAD**
