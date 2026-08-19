# Apunte Maestro — Clase 01 · Parte 3: REST I — principios y Richardson 0 → 2

**Unidad:** clase01 · **Parte 3 de 5**

Cubre el núcleo de la unidad: qué es REST, sus seis principios, HTTP como transporte real, y los tres primeros escalones del Richardson Maturity Model — recursos, verbos, seguridad e idempotencia, y códigos de respuesta. HATEOAS, stateless en profundidad y caching quedan para la Parte 4.

**Leyenda de marcas:** 🔴 central / evaluable · 🟡 secundario · 🟢 mencionado al pasar.

---

## 1. Qué es REST 🔴

**REST — Representational State Transfer**, formulado por Roy Fielding.

Lo primero y lo más importante: **REST es un estilo de arquitectura**, no un protocolo ni un estándar. Es un conjunto de buenas prácticas, restricciones y consejos para diseñar APIs. Nadie te va a rechazar un request por "no ser REST" — no hay un validador. Es un criterio de diseño, y por eso se puede cumplir en distintos grados. Justamente por eso existe una escala para medirlo, que veremos en la sección 4.

Mirá el contraste con lo que viene de la Parte 2. SOAP modelaba `GetPrice` con un parámetro `Item`: **una operación**. REST modela `/prices/apples`: **un recurso**. Ese giro —de verbos a sustantivos— es la idea central, y todo lo demás se desprende de ahí.

---

## 2. Los seis principios 🔴

### 2.1. Cliente-Servidor

La API es **un contrato entre dos partes que evolucionan de forma independiente**. El servidor no sabe quién lo consume; el cliente no sabe cómo está implementado el servidor. Mientras el contrato se respete, cada lado puede cambiar por dentro sin coordinar con el otro.

Compará con RMI de la Parte 1, donde cambiar la firma de un método rompía a todos los clientes: acá la independencia es un objetivo explícito de diseño.

### 2.2. Stateless

**Toda request debe contener toda la información necesaria para ser resuelta.** El servidor no mantiene estado conversacional entre requests: no "se acuerda" de vos. Si hay estado, vive en el cliente y se reenvía en cada llamada.

Esto tiene consecuencias grandes en escalabilidad y en autenticación, y por eso se desarrolla completo en la Parte 4.

### 2.3. Cache aware

Las respuestas **deben poder marcarse como cacheables**. Un mismo pedido que no cambió no debería recalcularse ni retransmitirse. El mecanismo concreto —headers, validación condicional— también va en la Parte 4.

### 2.4. Interfaz uniforme

Los recursos se identifican con **URIs** que llevan **IDs únicos**, y se nombran con **sustantivos en plural**. Un mismo recurso puede tener **múltiples representaciones**: el mismo usuario servido como HTML, JSON o XML.

Ese es el sentido de la R de REST: lo que viaja no es el recurso, es **una representación** de él.

### 2.5. Sistema de capas

Entre cliente y servidor pueden existir **N intermediarios** —caches, proxies, balanceadores de carga, capas de autenticación— y todos deben ser **transparentes para el cliente**. El cliente le habla a una URL y no sabe ni le importa cuántas piezas hay detrás.

Este principio parece burocrático hasta que se viola. En la sección 6.5 vamos a ver un caso concreto en el que un código de respuesta mal elegido filtra la arquitectura interna.

### 2.6. Código bajo demanda (opcional)

El servidor puede enviar **código ejecutable** al cliente, que el cliente corre. Es el único principio opcional, y **prácticamente no se usa**: está desaconsejado por riesgos de seguridad y de performance.

---

## 3. HTTP como transporte 🔴

En teoría REST es agnóstico del transporte. **En la práctica corre casi siempre sobre HTTP**, y la razón es que HTTP ya trae de fábrica casi todo lo que REST pide: verbos, códigos de estado, headers de caché, negociación de contenido.

La anatomía de un request HTTP:

```
    ┌─────────────────────── método
    │      ┌──────────────── recurso (path)
    │      │          ┌───── query parameters
    ▼      ▼          ▼
   GET /users?country=AR&page=2      HTTP/1.1
   Host: api.example.com          ◄── domain
   Accept: application/json       ◄── headers: información extra para el servidor
   Authorization: Bearer abc123

   { ... }                        ◄── body (opcional)
```

Todo request HTTP tiene: un **dominio**, un **recurso**, opcionalmente **query parameters**, un **método**, y **headers**. El **body es opcional**: los `GET` normalmente no lo llevan, los `POST` normalmente sí.

Es la misma información que un mensaje SOAP metía todo adentro del XML. La diferencia es que acá **cada cosa vive en el lugar que HTTP ya tenía previsto para ella**, y por lo tanto toda la infraestructura de internet —proxies, caches, CDNs— la entiende sin que nadie le explique nada.

---

## 4. Richardson Maturity Model 🔴

Una escala de cuatro escalones para evaluar **cuán RESTful** es una API. Cada nivel agrega algo sobre el anterior.

| Nivel | Qué agrega |
|---|---|
| **0** | Swamp of POX — nada |
| **1** | Recursos |
| **2** | Verbos HTTP y códigos de estado |
| **3** | HATEOAS |

### 4.1. Nivel 0 — *Swamp of POX* 🟡

"POX" es *Plain Old XML*: el pantano del XML de toda la vida.

Un **único endpoint**, un **único verbo** (típicamente `POST`), y todo el significado metido en el payload XML o JSON. Técnicamente habla HTTP, pero **no lo aprovecha en absoluto**: HTTP funciona acá como un caño por el que pasan bytes.

**Esto no es REST.** SOAP, por su diseño, vive en este nivel.

### 4.2. Nivel 1 — Recursos 🔴

Aparecen los **recursos con URI propia**, nombrados con **sustantivos en plural**. La API empieza a parecerse a un sistema de archivos.

```
/users                              # la colección de usuarios
/users/{userID}                     # un usuario puntual, identificado por su ID

/users/{userID}/hobbies             # los hobbies DE ese usuario (subrecurso)
/users/{userID}/hobbies/{hobbieID}  # un hobby puntual de ese usuario
```

Se lee como una ruta de carpetas: cada segmento acota al anterior. La jerarquía de la URL **es** la relación entre los datos.

**Uso incorrecto:**

```
/usersService                                          ❌

{
    "user_id": "cbbf467b-c24c-4360-9889-b22d7107edc7"
}
```

Acá el problema no es cosmético. `/usersService` no nombra un recurso, nombra **un servicio** — un lugar al que le mandás pedidos. El identificador del usuario, que debería estar en la URL identificando *qué* recurso querés, quedó escondido en el body. Resultado: **la URL ya no identifica nada**, y todo lo que HTTP construye sobre la URL —cachear, rutear, versionar, autorizar por path— deja de funcionar.

Comparación directa:

- ✅ `GET /users/123`
- ❌ `POST /getUser?id=123`

### 4.3. Nivel 2 — Verbos y códigos de estado 🔴

Se aprovechan los **verbos** HTTP para expresar la acción y los **códigos de estado** para expresar el resultado. La URL dice *sobre qué*, el verbo dice *qué le hago*, el código dice *cómo salió*.

#### Los verbos

| Verbo | Propósito |
|---|---|
| `GET` | Traer un recurso |
| `POST` | Crear un recurso |
| `PUT` | Modificar (reemplazar) un recurso |
| `PATCH` | Modificar parcialmente un recurso |
| `DELETE` | Eliminar un recurso |
| `OPTIONS` | Consultar qué operaciones admite un recurso |

**Uso correcto** — el sustantivo nunca cambia, cambia el verbo:

```http
GET     /users                 # traer la colección
GET     /users/b22d7107edc7    # traer uno
POST    /users                 # crear uno nuevo (el ID lo asigna el servidor)
PUT     /users/b22d7107edc7    # reemplazar ese usuario completo
PATCH   /users/b22d7107edc7    # modificar algunos campos de ese usuario
DELETE  /users                 # eliminar la colección
DELETE  /users/b22d7107edc7    # eliminar ese usuario
OPTIONS /users                 # qué se puede hacer sobre la colección
OPTIONS /users/b22d7107edc7    # qué se puede hacer sobre ese usuario
```

**Uso incorrecto** — el verbo metido dentro de la URL:

```http
GET     /getUsers                    ❌
GET     /getUser/b22d7107edc7        ❌
POST    /createUser                  ❌
PUT     /modifyUser/b22d7107edc7     ❌
PATCH   /patchUser/b22d7107edc7      ❌
DELETE  /deleteUsers                 ❌
DELETE  /deleteUser/b22d7107edc7     ❌
```

Fijate la redundancia: `DELETE /deleteUser` dice "eliminar" dos veces. El verbo HTTP ya lo decía. Y peor: nada obliga a que coincidan — nada impide un `GET /deleteUser/123`, que sería un desastre. Al meter la acción en la URL, se pierde la garantía de que el verbo signifique algo.

---

## 5. Seguridad e idempotencia 🔴

Dos propiedades de los verbos que **casi seguro entran en el parcial**, porque son conceptuales, cortas y fáciles de preguntar.

**Seguro:** el método **no tiene efectos de lado** — es de solo lectura. No modifica el estado del servidor.

**Idempotente:** el **estado final del servidor es el mismo** si la request se ejecuta una vez o N veces.

| Método | Seguro | Idempotente |
|---|:---:|:---:|
| `GET` | Sí | Sí |
| `OPTIONS` | Sí | Sí |
| `PUT` | No | Sí |
| `DELETE` | No | Sí |
| `PATCH` | No | No\* |
| `POST` | No | No |

Los cuatro razonamientos que hay que poder reproducir:

**¿Por qué `GET` es seguro e idempotente?** Porque no modifica el recurso. Hacerlo una o mil veces da el mismo resultado y deja el servidor exactamente igual.

**¿Por qué `PUT` no es seguro pero sí idempotente?** No es seguro porque **pisa** el recurso: modifica el estado. Es idempotente porque **siempre lo pisa con el mismo contenido** — mandar el mismo `PUT` diez veces deja el recurso idéntico a mandarlo una.

**¿Por qué `PATCH` "depende"?** Porque depende de cómo modeles la modificación. Un `PATCH` con una operación **absoluta** (`set x = 5`) es idempotente: aplicalo N veces y `x` sigue valiendo 5. Un `PATCH` con una operación **relativa** (`x += 1`) no lo es: cada ejecución cambia el resultado. Por eso el asterisco en la tabla.

**Y la pregunta capciosa:** un `DELETE` que la primera vez devuelve `204` y la segunda `404`, ¿sigue siendo idempotente? **Sí.** La idempotencia se define sobre **el estado final del recurso en el servidor**, no sobre el código de respuesta. Después de la primera llamada el recurso ya no existe; las siguientes no cambian ese estado. Que la respuesta sea distinta es información sobre lo que pasó, no un cambio de estado.

> **Por qué importa en serio:** si un cliente manda un request y la conexión se corta antes de recibir respuesta, no sabe si el servidor lo procesó. **Con un método idempotente puede reintentar sin miedo**; con uno que no lo es, reintentar puede crear dos usuarios o cobrar dos veces. Es el fundamento de toda política de reintentos — precisamente una de las preguntas que quedaron abiertas con sockets en la Parte 1.

---

## 6. Códigos de respuesta 🔴

El código de estado es **la respuesta semántica**: le dice al cliente qué pasó, sin que tenga que parsear el body para averiguarlo.

### 6.1. `1xx` — Informativo 🟢

- **`101 Switching Protocols`** — cambio de protocolo, por ejemplo al hacer *upgrade* a WebSockets.

### 6.2. `2xx` — Éxito 🔴

Suelen ser bastante semánticos: cada uno dice algo distinto.

- **`200 OK`** — respuesta general de éxito para `GET`, `POST` o `PUT`.
- **`201 Created`** — un `POST` **creó** un recurso. Más preciso que `200`.
- **`202 Accepted`** — la request se recibió y **se va a procesar de forma asincrónica**. No terminó todavía.
- **`204 No Content`** — éxito **sin body**, solo headers. Habitual en `PUT` y `DELETE`.

### 6.3. `3xx` — Redirección 🟡

El cliente debe completar una acción adicional.

- **`301 Moved Permanently`** — el recurso se mudó definitivamente; la nueva dirección va en el header `Location`.
- **`302 Found`** — movido temporalmente; el cliente HTTP debería redirigir.
- **`304 Not Modified`** — se usa con `If-None-Match`: el cliente puede reutilizar su copia cacheada. Es la pieza clave del caching de la Parte 4.
- **`307 Temporary Redirect`** — redirección temporal **preservando el método** original.

### 6.4. `4xx` — Error del cliente 🔴

- **`400 Bad Request`** — request malformada. El cliente **no debería reintentarla igual**: va a volver a fallar.
- **`401 Unauthorized`** — **no estás autenticado**: no mandaste credenciales, o son inválidas. (El nombre engaña: dice "unauthorized" pero significa "unauthenticated".)
- **`403 Forbidden`** — **estás autenticado pero no tenés permiso**. Sé quién sos; no podés.
- **`404 Not Found`** — el endpoint existe pero el recurso no. A veces se usa **deliberadamente en lugar de `403`**, para no filtrar información: responder `403` confirma que el recurso existe, y eso ya le dice algo a un atacante.
- **`405 Method Not Allowed`** — ese verbo no está ruteado para ese recurso.
- **`406 Not Acceptable`** — falló la negociación de contenido (el cliente pidió un formato que el servidor no sabe producir). Poco común: lo habitual es defaultear a algo.
- **`409 Conflict`** — conflicto con el estado actual del recurso. El cliente debería poder resolverlo y reintentar.
- **`412 Precondition Failed`** — falló una precondición, típicamente de ETags. Vuelve en la Parte 4.

### 6.5. `5xx` — Error del servidor 🔴

- **`500 Internal Server Error`** — algo explotó del lado del servidor.
- **`502 Bad Gateway`** — el servidor, actuando como proxy o cliente de otro servicio, recibió una respuesta inválida.
- **`503 Service Unavailable`** — el servicio no está disponible.
- **`504 Gateway Timeout`** — se venció el tiempo de espera actuando como proxy.

> ⚠️ **El caso del `502` — acá se cierra un hilo abierto en la sección 2.5.**
>
> Si nuestro servicio devuelve un `502` al cliente, le está informando que **internamente actuamos como proxy de otro sistema**. Eso:
>
> 1. **Viola el principio de sistema de capas:** los intermediarios debían ser transparentes, y acabamos de exponer que existen.
> 2. **Abre superficie de ataque:** un atacante puede deducir la arquitectura interna a partir de los códigos que devolvemos y usar eso para orientar sus intentos.
>
> Lo correcto es traducir la falla interna a un código propio y coherente con lo que nuestra API promete.

### 6.6. Nivel 3 — HATEOAS 🟢

El cuarto escalón del modelo. Es el tema con el que abre la Parte 4.

---

## Para el parcial, si te preguntan

> **¿Qué es REST?**
>
> Un estilo de arquitectura formulado por Roy Fielding: un conjunto de buenas prácticas y restricciones para diseñar APIs. No es un protocolo ni un estándar, por lo que se cumple en grados —de ahí que exista el Richardson Maturity Model para evaluarlo—. Sus principios son cliente-servidor, stateless, cache aware, interfaz uniforme, sistema de capas y código bajo demanda (opcional). En la práctica corre sobre HTTP.

> **¿Qué son los niveles del Richardson Maturity Model?**
>
> Nivel 0 (*Swamp of POX*): un único endpoint y un único verbo, sin aprovechar HTTP. Nivel 1: recursos con URI propia, nombrados con sustantivos en plural. Nivel 2: uso de los verbos HTTP y de los códigos de estado para expresar semántica. Nivel 3: HATEOAS, respuestas que incluyen los links a los recursos alcanzables. Una API RESTful típica implementa el nivel 2 y rara vez llega al 3.

> **Diferencia entre seguro e idempotente.**
>
> Seguro significa que el método no tiene efectos de lado: es de solo lectura y no modifica el estado del servidor. Idempotente significa que el estado final del servidor es el mismo se ejecute la request una o N veces. Son independientes: `PUT` no es seguro —modifica— pero sí es idempotente, porque siempre pisa el recurso con el mismo contenido.

> **¿Por qué está mal `POST /createUser`?**
>
> Porque mete la acción en la URL, que debe identificar un recurso y no una operación. El verbo HTTP ya expresa la acción, así que hay redundancia, y además nada garantiza que verbo y URL coincidan. Lo correcto es `POST /users`: el sustantivo en plural identifica el recurso y el verbo dice qué se hace con él.

---

## Checkpoint

Respondelas sin volver al texto. Las respuestas van al complemento.

1. ¿Por qué se dice que REST es un estilo de arquitectura y no un protocolo, y qué consecuencia tiene eso?
2. Enumerá los seis principios de REST e indicá cuál es opcional y por qué casi no se usa.
3. ¿Por qué en la práctica REST corre sobre HTTP si en teoría es agnóstico del transporte?
4. ¿Qué está mal en `/usersService` con el `user_id` en el body? Nombrá dos consecuencias concretas.
5. Escribí las rutas y verbos correctos para: listar usuarios, crear uno, reemplazar uno y borrar uno.
6. `PUT` no es seguro pero sí idempotente. Explicá ambas mitades.
7. ¿Bajo qué condición un `PATCH` es idempotente y bajo cuál no?
8. Un `DELETE` devuelve `204` la primera vez y `404` la segunda. ¿Sigue siendo idempotente? Justificá.
9. ¿Qué relación hay entre la idempotencia y una política de reintentos?
10. Diferencia entre `401` y `403`. ¿Por qué a veces se responde `404` en lugar de `403`? ¿Y por qué devolver un `502` puede ser un problema de diseño y de seguridad?

---

## Qué viene en la Parte 4

Los tres principios que quedaron enunciados y sin desarrollar: **HATEOAS** (el nivel 3 del modelo), **stateless en profundidad** —el problema del cliente que se autentica contra un servidor de un cluster y cae en otro, y por qué la respuesta es JWT y no *sticky sessions*—, **escalabilidad** vertical y horizontal, y todo el **caching**: `Cache-Control`, validación condicional, ETags, `Vary`, y el locking optimista con `If-Match`.

---

**FIN DE LA PARTE 3**
