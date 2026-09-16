# 📘 Apunte maestro — Clase 05 — Microservicios
## Parte 5 de 6 — No todo es bueno

> **Unidad:** `clase05` · TACS 2C 2026 · Clase del 08/09/2026 · Apunte maestro, Parte 5 de 6 · Leyenda e índice completo en la Parte 1.

**Qué cubre esta parte.** La lista honesta de lo que microservicios empeora: responsabilidades que no se sabe en qué caja van, overhead de red, tareas redundantes, un sistema más complejo aunque cada aplicación sea más simple, la coordinación entre equipos y ambientes, y la complejidad de management (el two-pizza team y cómo la IA está cambiando el tamaño de los equipos). Los servicios anémicos y los picoservicios, el caso de Amazon que volvió a un monolito, la estandarización, y la lista de malas prácticas: repo compartido, SDK en vez de API, ambiente compartido, mensajería sin versionar, servicios olvidados.

**Qué NO cubre.** Lo que una organización necesita tener para que esto funcione (provisionamiento, escalabilidad, monitoreo, ciclo de vida, testing, disponibilidad) ni el cierre: Parte 6.

**De dónde venís.** Parte 3: las ventajas de la arquitectura refactorizada. Cada ventaja de allá tiene acá su contracara, y en el parcial se espera que puedas dar las dos con ejemplo.

---

## 1. Responsabilidades que no son tan claras 🔴

Con el mapa de la Parte 3 adelante, tres preguntas sin respuesta obvia:

- **Los destacados llevan políticas comerciales** (markup, promociones). ¿Cómo y dónde se cargan? ¿Los carga Destacados? ¿Los pide a Políticas Comerciales cada vez?
- **Para personalizar la presentación**, ¿qué parte le corresponde al Front y qué parte a P13N? ¿O tengo que meter una caja nueva, un BFF (Parte 3, §3.6)?
- **Las políticas comerciales influyen en los ítems a traer del proveedor externo** (una restricción puede decir "este producto no se vende"). ¿Quién toma esa decisión: Políticas Comerciales o el catálogo (Datos)? ¿Lo meto en el catálogo, o cada vez que pido productos tengo que ir a Políticas Comerciales, asociar con el catálogo y hacer una especie de **join de servicios**?

**Algunas responsabilidades no son tan claras como para determinar si van en una caja u otra.** En el monolito la pregunta ni existía: estaba todo junto. Ahora hay **partes del dominio que pueden no estar muy claras**, y resolverlas no es un problema técnico: **está en la decisión y en la capacidad de la organización** verlo. Es la contracara de la separación de responsabilidades que en la Parte 3 (§3.2) parecía tan limpia con el caso de la tarjeta de crédito.

---

## 2. Overhead por saltos y distribución 🔴

Antes tenía **una llamada** (del cliente a la app) y a lo sumo un acceso a datos. Adentro llamaba funciones, era feliz, y las excepciones y los errores eran bien conocidos.

Ahora tengo **múltiples llamadas internas** para resolver un pedido, y cada una es una llamada de red: **mayor tiempo**, manejo de todo lo que es red (status codes, timeouts, lo de la Parte 4), y mucho **overhead de red**, porque **la carga interna se multiplica varias veces**. Consecuencias:

- **Más tiempo de respuesta al cliente.**
- **Más overhead en la red.**

**Es mitigable** (caches, colas, BFF), **pero lo tengo**: muy probablemente voy a tener **más latencia**. Es exactamente la "baja latencia con pinzas" del monolito de la Parte 1 (§3), vista desde el otro lado.

---

## 3. Redundancia de tareas 🔴

Para resolver algunos pedidos **necesito invocar varias veces al mismo servicio desde diferentes lugares**. En el mapa: **Destacados y Facetado llaman ambos a Políticas Comerciales**, y las dos llamadas nacen de la misma request del Front.

```
                ┌──► Destacados ──► Políticas Comerciales ──► Datos ──► ...
   usuario ──► Front
                └──► Facetado   ──► Políticas Comerciales ──► Datos ──► ...
                                        ↑ el mismo trabajo, dos veces, por una sola página
```

Algunas aplicaciones son invocadas varias veces, desde varios lugares o desde la misma aplicación, y eso **agrega overhead por tareas que se repiten**. Lo que en el monolito era una llamada recursiva ahora es llamar a dos lugares y hacer un join de negocio en el medio. Y puede quedarte **un grafo de servicios horrible para un flujo**. Siempre hay un **camino crítico** (la cadena de llamadas más larga, la que determina cuánto tarda el pedido); si ese camino crítico tiene un garabato gigante, no está bueno.

---

## 4. Aplicaciones más simples, sistema más complejo 🔴

**Si bien las aplicaciones son más simples, el sistema en general es más complejo.** Cualquiera que haya trabajado con más de un servicio lo vivió:

- **Un pedido se desarrolla con la colaboración entre varias aplicaciones.**
- **Si una aplicación falla, el sistema entero puede funcionar de manera incorrecta** (Parte 4).
- **Definir la interacción entre aplicaciones (la API) puede ser engorroso.**
- **Si quiero cambiar la API de una aplicación, necesito coordinar con los clientes** (Parte 3, §4).
- **Si necesito más datos de una aplicación, tengo que coordinar con el equipo responsable.**

El último punto es el que más se sufre, y vale contarlo como pasa. Necesitás un campo más en la respuesta de otro servicio, para algo tuyo. Le pedís al equipo: "sí, dale, te lo sumamos, **son 15 días**" (o un mes; no preguntes por qué). Y vos dependés de ese campo. Quizás no tenés ganas de hacerles un pull request, porque está mal; **a veces ni siquiera ves su repo, por seguridad**. Querés un campo más, o una vista distinta de la misma respuesta, y te complicás **porque la API ya está definida**. En el monolito, ese campo lo agregabas vos.

**Coordinación entre equipos.** El clásico: un **Gantt** (el diagrama de barras que ordena tareas en el tiempo) con tres equipos involucrados en una feature: el equipo 1 hasta acá, el 2 hasta acá, el 3 hasta acá, con las stories en orden según quién es el último cliente. Muy lindo, hasta que uno se atrasa. Y hay que coordinar también los **ambientes bajos** (Parte 6, §5): hacer una beta entre varios equipos no es fácil.

---

## 5. Complejidad a nivel management 🔴

**Si bien me permite transformar un gran equipo en equipos más pequeños, tengo que diseñar una nueva estructura.** Y esa estructura tiene sus propios problemas:

- **A veces es difícil balancear las tareas de todo el stack de forma pareja**: algún equipo puede quedar sobrecargado y otro sin trabajo suficiente.
- **Algunas aplicaciones tienen mucha carga de trabajo en un momento dado y poca en otro** → necesito ser **flexible con los equipos**, mover gente. **No todas las organizaciones están preparadas para este nivel de agilidad.**

### 5.1 El two-pizza team, y lo que la IA le está haciendo 🟡

La estructura clásica se llama **two-pizza team**: el equipo que podés alimentar con dos pizzas. Es variable (hay gente que come más), pero en general **entre 3 y 7 personas** como máximo, y con eso manejabas una serie de microservicios.

Hoy esto está en discusión por la IA. Es una opinión de industria, pero es lo que está pasando: no está claro que tenga sentido tener equipos tan grandes. A veces podés tener **dos o tres personas muy buenas, con conocimiento del negocio**, iterando mucho más rápido, y reasignar a las otras tres. Hay empresas grandes de la región donde ya no hay equipos en el sentido tradicional: son tres, los tres contribuyen en todo, y la IA es una herramienta de uso obligatorio.

Antes, en un equipo tenías gente de back y gente de front, porque había que mantener las dos cosas; sigue existiendo y es válido. Pero hoy, al menos en empresas de producto, **todos tienen que tocar todo**: ser full stack es casi un requisito. Ya no vale el "yo soy frontend, no tengo idea de esa query que falla": levantás una sesión con un agente, le pegás el error y tenés acceso a todo el stack. La IA **baja la barrera inicial** y **sube el ownership**. La contracara: es raro que alguien se quede sin backlog, y que eso salga bien es un desafío; reasignar gente sin el tacto necesario para que las cosas sigan saliendo con calidad es irresponsable.

> 🕳️ **Madriguera — la IA y la forma de los equipos**
> Tres opiniones que salieron al pasar y no se van a evaluar: (1) para que el estilo visual sea consistente cuando cada uno hace una partecita, se apunta la IA al *design system* y a las libraries existentes; (2) medir rendimiento por commits o por tokens consumidos no mide impacto de negocio; (3) una empresa grande te enseña estructura (postmortems, RFCs, observabilidad, "una story no termina cuando deployás"), y eso vale aunque tenga sus cosas raras. Todo esto cambia en escala de meses, así que tomalo como foto, no como regla.
> *Volvé al camino — esto se profundiza aparte, otro día.*

---

## 6. Otros: anémicos, transacciones distribuidas, estandarización 🔴

### 6.1 No pasarse del otro lado: servicios anémicos

**Tengo que tener cuidado al diseñar mi arquitectura y no pasarme del otro lado**: aplicaciones muy livianas, con muy poca responsabilidad. Es lo que se conoce como **servicio anémico** (por analogía con el modelo anémico de dominio: objetos sin comportamiento): un microservicio que hace **pasamanos**, o que almacena una cosita y no hace nada más.

Caso real, de una app de delivery muy conocida, hace unos años: trabajaban con **picoservicios**. Pico, como 10⁻¹², súper-súper-microservicios: **un endpoint, un servicio**. Por algo lo habrán pensado, pero el resultado es que levantabas un servicio que hacía una cosa y tenías **cuarenta Mongos tiradas por ahí**. **Mucho overhead**: un montón de cosas que levantar y un montón de infra que mantener (y en esa época el cloud era más barato que ahora). Es el extremo opuesto del monolito, y duele parecido.

### 6.2 Transacciones distribuidas: el gran problema

**Manejar transacciones distribuidas es un GRAN problema; a veces es tan caro que es mejor no atacarlo.** Lo viste en la Parte 1 (§5): las herramientas existen (idempotencia, colas, compensación, Saga), pero ninguna es amigable, y **en microservicios el problema empeora**. Hay que **no llegar al infierno**: si una operación necesita atomicidad entre dos servicios, la pregunta correcta es si esos dos servicios tienen que ser dos.

### 6.3 El caso Amazon: a veces conviene volver

Lo que termina pasando, y ya pasó hace unos años, es que quizás **es mejor trabajar en servicios que en microservicios**. Hay un artículo de AWS, de hace unos años, sobre un servicio de video que tenían en microservicios y que **refactorizaron a monolito**: hicieron el camino inverso, y **los costos bajaron alrededor de un 90 %**. Tenían un montón de costos de overhead, un montón de serverless, un montón de cosas prendidas que no tenían sentido; se fueron a un monolito más simple y ganaron un montón.

Conclusión: **no hay respuesta correcta; va a depender del problema que tengo.** Ni "monolito malo" ni "microservicios buenos". Ese es el criterio que se evalúa.

### 6.4 Estandarización

**Depende de qué tan alineados quiero que estén mis equipos, pero necesito más trabajo para que, por ejemplo, sigan convenciones de código, utilicen las mismas herramientas, etc.** Concretamente, hay que estandarizar un montón de cosas: **lenguaje de programación, libraries, cómo paso headers (si los paso), runtimes, versionado**.

Por qué: viene un developer entusiasmado y dice "yo quiero levantar esto en Rust, porque mi aplicación va a consumir dos megas". Y sí, está bueno. Pero después: **¿quién lo mantiene? ¿Cómo te doy soporte?** Tengo una library *commons* interesantísima con cosas de negocio, **y no la tenés en Rust**. Hoy la IA te lo soluciona un poco, pero sigue siendo un problema. Por eso en la mayoría de las empresas terminás con **una selección de herramientas**: podés elegir entre Kotlin y Go, pero no Haskell, ni Prolog porque te copa su servidor HTTP (se ha visto). Los estándares los definen **equipos cross de arquitectos** o similares.

La otra cara, que la misma lista reconoce: **esto puede ser un punto a favor de la experimentación.** Si cada servicio es independiente, probar una tecnología nueva en uno solo es barato; el costo aparece cuando la prueba se queda.

**Esto es importantísimo: no todas las organizaciones están preparadas.** Hay que evaluarlo y **tener la espalda suficiente**. Con qué se evalúa es la Parte 6.

> **Para el parcial, si te preguntan: ¿cuáles son las desventajas de una arquitectura de microservicios?**
> Responsabilidades del dominio que no está claro en qué servicio van; overhead por saltos de red (más latencia, más carga interna); redundancia de tareas (el mismo servicio invocado varias veces por un pedido); un sistema más complejo aunque cada aplicación sea más simple (un pedido colabora entre varias apps, definir y cambiar APIs exige coordinar con clientes y equipos); complejidad de management (nueva estructura de equipos, balance de carga entre ellos); riesgo de servicios anémicos; transacciones distribuidas muy caras; y necesidad de estandarizar lenguajes, libraries y herramientas. No hay respuesta correcta: depende del problema, y a veces conviene volver a servicios más grandes.

---

## 7. Malas prácticas de microservicios 🔴

Seis, en el orden en que se suelen cometer.

### 7.1 Tener un repositorio compartido entre microservicios ⚠️

Está en la lista, y **no es tan mala práctica, porque ahora se usa** desde hace años: el **monorepo**, un repo que tiene más de un servicio y a veces comparte libraries entre ellos. Lo que sí es mala práctica es **hacerlo ad hoc**: "vengo yo y hago un repo con más de una cosa adentro". Eso está mal. Pero hay **herramientas para hacer monorepo** que están buenas (surgió sobre todo en frontend y en el mundo TypeScript). Así que: **debatible, y puede estar bien**. Si tu TP es un monorepo, la pregunta a responder es esa: ¿tiene herramienta y convenciones, o es un repo con carpetas?

### 7.2 Preferir SDKs sobre APIs (compartir clientes, no interfaces)

**SDK** (*Software Development Kit*, ya nombrada en la Parte 1, §4.5): en términos prácticos, un conjunto de herramientas para programar; una library con un poquito más de onda, que un proveedor te da para integrarte con su servicio.

¿Por qué es mala práctica usar una library en vez de una API para integrarte con otro microservicio? Porque **la API es estándar** e independiente de la tecnología; **la SDK depende de la tecnología** del que la escribió, y te **acopla** a su implementación. Dos casos reales, horribles:

- Una SDK de una empresa de delivery que, cuando la levantabas, **se conectaba con la cuenta de AWS de esa empresa** para hacer polling de unos mensajes. Tu aplicación, de repente, salía a la red a un servicio de AWS ajeno. Una locura.
- Un sistema de notificaciones que se incluía como un **jar de Java**: al levantarlo **te exigía un data source** para arrancar y **te creaba una tabla en tu base de datos**. Una library que te levanta una tabla en tu aplicación.

La conclusión: **compartir una interfaz, no compartir la implementación.** Vos tenés que interactuar con una API y **desacoplarte lo más que puedas**. Si necesito ida y vuelta, puedo usar un **webhook** (una URL tuya a la que el otro te llama cuando pasa algo), un socket, un endpoint; muchas cosas. Pero no lo puedo atar a una SDK.

⚠️ **No se reduce a "nunca usar SDK".** Las SDKs tienen casos de uso legítimos y hay SDKs recontraútiles: un mapa interactivo (Google Maps), o algo súper dependiente del cliente que es imposible resolver solo con una API. Y cuidado con las SDKs que quedan viejas: un proveedor de pagos tiene una de Java y una de Go, y alguna se queda atrás, y entonces la que queda vieja es la tuya. **La mala práctica es, específicamente, usar una SDK para integrarte con otro microservicio.**

> **Para el parcial, si te preguntan: ¿por qué es mala práctica integrarse a otro microservicio con una SDK en vez de con su API?**
> Porque una SDK comparte implementación, no interfaz: te acopla a la tecnología y a las decisiones internas del otro servicio (puede exigirte recursos, crear tablas en tu base, salir a la red por su cuenta), y queda vieja con su ciclo de vida, no con el tuyo. La API es estándar e independiente de tecnología. La regla es compartir la interfaz y no la implementación; para ida y vuelta se usan webhooks o endpoints. Las SDKs tienen su lugar (mapas, integraciones muy dependientes del cliente), pero no entre microservicios.

### 7.3 Utilizar un ambiente compartido por varios microservicios 🟢

Un ambiente (infraestructura, configuración, base de datos) compartido entre varios servicios reintroduce el acoplamiento que se quiso evitar: lo que uno rompe, lo pagan los demás. Es la versión de infraestructura de "solo la app dueña conoce su DB" (Parte 3, §3.5), y se conecta con el problema de mantener ambientes bajos de la Parte 6 (§5).

### 7.4 Mensajería sin versionar o no retrocompatible

**"APIs are forever"**, de nuevo (Parte 3, §4), y vale igual para los mensajes de una cola (Parte 4): un mensaje es un contrato entre productor y consumidor, y cambiarle el formato sin versionar rompe a quien lo consume, con el agravante de que los mensajes viejos pueden seguir encolados cuando el consumidor ya cambió.

### 7.5 Irse para el otro lado: servicios anémicos

Lo de 6.1: servicios muy, muy chiquitos.

### 7.6 Tener servicios "olvidados" que están en el camino crítico

Dicho así parece exagerado, pero **un montón de veces se olvidan servicios**: hay servicios que quedan **en el éter**. **Quedan huérfanos** cuando reestructuran un equipo y algún servicio queda en el medio, y nadie sabe qué hacer con él. Y si ese servicio está en el **camino crítico** (sección 3) de un flujo, el día que falla nadie sabe quién lo levanta. Son cosas que pasan con microservicios, y que en el monolito no podían pasar: no había nada que olvidar.

> **Para el parcial, si te preguntan: nombrá malas prácticas de microservicios.**
> Repositorio compartido armado ad hoc (el monorepo con herramientas es válido); preferir SDKs sobre APIs para integrarse entre servicios; ambiente compartido por varios microservicios; mensajería sin versionar o no retrocompatible ("APIs are forever"); irse al otro extremo con servicios anémicos; y tener servicios olvidados o huérfanos en el camino crítico.

---

## Checkpoint — Parte 5

*(Sin respuestas: van al complemento.)*

1. Elegí una de las tres preguntas de responsabilidad sobre el mapa (destacados con políticas, personalización front/P13N, restricciones catálogo/políticas) y argumentá una decisión.
2. ¿Por qué "baja latencia" era una ventaja del monolito con pinzas y ahora es una desventaja de microservicios con pinzas?
3. Explicá con el mapa qué es la redundancia de tareas y qué es el camino crítico de un pedido.
4. "Las aplicaciones son más simples pero el sistema es más complejo." Da tres motivos concretos.
5. ¿Qué es un two-pizza team y por qué está en discusión?
6. ¿Qué es un servicio anémico y qué fueron los picoservicios? ¿Qué overhead generan?
7. ¿Qué enseña el caso de Amazon volviendo a monolito?
8. ¿Por qué hay que estandarizar lenguajes y libraries si cada servicio es independiente? ¿Qué se pierde al hacerlo?
9. "Tener un repositorio compartido es mala práctica." ¿Sí, no, depende? Justificá.
10. ¿Qué significa "compartir la interfaz, no la implementación"? Da uno de los dos casos reales de SDK.
11. ¿Cómo aparece un servicio huérfano y por qué es peligroso si está en el camino crítico?

---

## Qué viene en la Parte 6

Lo que una organización tiene que tener para dar el salto: provisionamiento rápido y cloud, escalabilidad horizontal transparente, monitoreo con herramientas específicas (APM: New Relic, Datadog, Grafana) y trazabilidad de requests, con Log4Shell como ejemplo de lo que cuesta parchear 30.000 artefactos; ciclos de vida independientes con deploys a cualquier hora y rollback barato; testing con ambientes bajos que hay que mantener consistentes (beta vs sandbox); y disponibilidad, donde "high availability" es carísima y el eslabón más débil manda. Cierra con las dos preguntas de siempre: ¿monolito o microservicios hoy?, y ¿qué es "saber microservicios"?

---

**FIN DE LA PARTE 5 — Apunte maestro clase05 — Microservicios**
