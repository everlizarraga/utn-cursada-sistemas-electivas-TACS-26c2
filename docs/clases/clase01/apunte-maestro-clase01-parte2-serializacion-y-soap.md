# Apunte Maestro — Clase 01 · Parte 2: Serialización de datos y SOAP

**Unidad:** clase01 · **Parte 2 de 5**

Cubre cómo se representan los datos que viajan por el cable: el problema del *mismatch* entre lenguajes y máquinas, los formatos de texto plano (XML, JSON, YAML), los formatos binarios (con Protocol Buffers en detalle) y SOAP, el primer gran estándar de comunicación entre servicios construido encima de todo esto.

**Leyenda de marcas:** 🔴 central / evaluable · 🟡 secundario · 🟢 mencionado al pasar.

---

## 1. El problema: nadie representa los datos igual 🟡

En la Parte 1 quedó abierta una pregunta. RMI ataba cliente y servidor al mismo lenguaje. Si queremos soltar esa atadura —que un servidor Java atienda a un cliente JavaScript— hay que resolver algo primero: **acordar cómo se representan los datos en el cable**.

Y no es un acuerdo trivial, porque los lenguajes no coinciden ni siquiera en lo más básico.

### 1.1. El mismatch entre lenguajes

| Concepto | Java | JavaScript |
|---|---|---|
| Booleano | `boolean` / `Boolean` | `Boolean` |
| Enteros | `byte`, `short`, `int`, `long` | `Number` (o `BigInt`) |
| Decimales | `float`, `double` | `Number` |
| Carácter | `char` | *(no existe)* |
| Cadena | `String` | `String` |
| Ausencia | `null` | `null` **y** `undefined` |
| Otros | `Object`, Collections | `Object`, arrays, `Symbol`, `Function` |

**El mapeo no es uno a uno, y ahí está el problema.** Mirá los choques concretos:

- Java distingue cuatro tamaños de entero (`byte`, `short`, `int`, `long`); JavaScript los colapsa todos en un único `Number` que además **no distingue entero de flotante**. ¿A qué tipo Java se convierte un `Number` que vale `35`?
- JavaScript **no tiene `char`**. Un `char` de Java, ¿viaja como string de longitud 1?
- JavaScript tiene **dos formas de ausencia**, `null` y `undefined`. Java tiene una. ¿Cuál es cuál?
- Java tiene *boxing* (`int` primitivo vs `Integer` objeto) y toda una jerarquía de Collections; JavaScript tiene arrays y mapas.

Cualquier estándar que quiera comunicar estos dos lenguajes tiene que tomar posición sobre cada uno de esos choques. Por eso hace falta un formato de serialización explícito, definido aparte de los lenguajes.

### 1.2. Y el problema no termina en el lenguaje

Debajo del lenguaje hay más desacuerdos:

- **Procesador:** palabras de 8, 16, 32 o 64 bits.
- **Arquitectura de CPU:** i386, ARM, Apple M, POWER-PC, microcontroladores; familias CISC y RISC.
- **Endianness:** *big endian* vs *little endian* — el orden en que se guardan los bytes de un número en memoria. Una placa de red y un sistema operativo pueden interpretar los mismos bytes de forma distinta.

> 🕳️ **Madriguera — Endianness**
> El número `0x1234` se guarda como `12 34` en *big endian* y como `34 12` en *little endian*. Si dos máquinas con criterios opuestos intercambian bytes crudos sin acordar el orden, leen números distintos.
> *Volvé al camino — para esta unidad alcanza con saber que el desacuerdo existe y que por eso hace falta un formato.*

> **El uso de máquinas virtuales** (la JVM de Java, el V8 de JavaScript) ayuda a homogeneizar el entorno entre distintos sistemas operativos, pero **no elimina la necesidad de un formato de serialización** bien definido cuando cruzamos la frontera del proceso. La VM te unifica el adentro; el cable sigue siendo tierra de nadie.

---

## 2. Plain text encoding 🔴

La familia de formatos textuales es la más difundida: son *human-readable* —los podés abrir y leer— y simples.

### 2.1. XML — Extensible Markup Language

Un lenguaje de marcado basado en **tags que abren y cierran**.

```xml
<!-- XML soporta comentarios: este es uno -->
<user>
    <name>Mario</name>              <!-- ⚠️ el cierre es </name>, con la barra ADELANTE.
                                         Escribir <name/> como cierre es un error frecuente:
                                         <name/> es un elemento vacío, no un tag de cierre. -->
    <lastname>Santos</lastname>
    <hobbies>
        <hobbie>Art</hobbie>        <!-- Un array se representa REPITIENDO el mismo tag.  -->
        <hobbie>Sport</hobbie>      <!-- Que esto "sea un array" es convención de lectura, -->
        <hobbie>TACS</hobbie>       <!-- no sintaxis: XML no tiene noción de array.        -->
    </hobbies>
    <age>35</age>                   <!-- ¿35 es un número o el string "35"? XML no lo dice. -->
</user>
```

Características:

- Soporta strings y números, pero **sin XSD es imposible distinguir tipos** de forma confiable: todo es texto entre tags.
- Los **arrays son tags repetidos**; la interpretación de "esto es una lista" no está en el estándar.
- **Soporta comentarios.**

**XSD — XML Schema Definition.** Es un estándar, escrito en XML, que permite **validar** un documento XML y sus tipos de datos. Resuelve la ambigüedad anterior: el schema declara que `quantity` es un entero positivo y que `price` es decimal.

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema">

<!-- Declara que existe un elemento llamado "shiporder"... -->
<xs:element name="shiporder">
  <xs:complexType>                                      <!-- ...que no es un valor simple sino
                                                             una estructura con hijos.        -->
    <xs:sequence>                                       <!-- Los hijos van EN ESTE ORDEN.     -->
      <xs:element name="orderperson" type="xs:string"/> <!-- Tipado explícito: string.        -->
      <xs:element name="item" maxOccurs="unbounded">    <!-- maxOccurs="unbounded" = acá SÍ
                                                             se declara que es una lista.     -->
        <xs:complexType>
          <xs:sequence>
            <xs:element name="title"    type="xs:string"/>
            <xs:element name="note"     type="xs:string" minOccurs="0"/>  <!-- opcional -->
            <xs:element name="quantity" type="xs:positiveInteger"/>       <!-- entero > 0 -->
            <xs:element name="price"    type="xs:decimal"/>               <!-- decimal   -->
          </xs:sequence>
        </xs:complexType>
      </xs:element>
    </xs:sequence>
    <xs:attribute name="orderid" type="xs:string" use="required"/>        <!-- obligatorio -->
  </xs:complexType>
</xs:element>

</xs:schema>
```

**Ventajas:** legible por máquina y por humano · esquema de autovalidación (XSD) · soporta comentarios.
**Desventajas:** los tags repetidos lo hacen pesado (mitigable con compresión) · difícil de leer sin herramientas · ambigüedad de tipos sin XSD.
**Usos típicos:** RSS, XHTML, SOAP.

### 2.2. JSON — JavaScript Object Notation

```json
{
  "user": {
    "name": "Mario",
    "lastname": "Santos",
    "hobbies": ["art", "sports", "TACS"],
    "age": 24,
    "graduated": true
  }
}
```

Mirá lo que resolvió respecto de XML, en las mismas cinco líneas:

- **Estructura jerárquica** clara, con llaves y anidamiento explícito.
- **Tipos diferenciados:** `"Mario"` es string por las comillas, `24` es número por su ausencia, `true` es booleano, y existe `null`. Como JSON es transporte y no lenguaje, **no se ocupa de limitar la precisión** de los números.
- **Arrays nativos:** `[...]` es sintaxis, no convención.

**JSON Schema** cumple para JSON el mismo rol que XSD para XML:

```json
{
  "$id": "https://example.com/person.schema.json",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Person",
  "type": "object",
  "properties": {
    "firstName": {
      "type": "string",
      "description": "The person's first name."
    },
    "lastName": {
      "type": "string",
      "description": "The person's last name."
    },
    "age": {
      "description": "Age in years which must be equal to or greater than zero.",
      "type": "integer",
      "minimum": 0
    }
  }
}
```

*(El código va sin comentarios porque JSON no los admite — es su desventaja, listada acá abajo. Las anotaciones van afuera:)*

| Línea | Qué hace |
|---|---|
| `$id` | Identificador único de este schema |
| `$schema` | Qué versión del estándar JSON Schema se está usando |
| `"type": "object"` | El documento validado debe ser un objeto |
| `description` | Documentación del campo: no valida nada, explica |
| `"type": "integer"` | Entero, no cualquier número: más estricto que el `number` de JSON |
| `"minimum": 0` | Restricción de **valor**, no solo de tipo: la edad no puede ser negativa |

**Ventajas:** muy legible · tipado · arrays claros.
**Desventajas:** **no soporta comentarios** · `number` no distingue entero de punto flotante.
**Usos típicos:** JavaScript, APIs HTTP.

### 2.3. YAML — YAML Ain't Markup Language

```yaml
# YAML sí soporta comentarios, a diferencia de JSON.

- otherHobbies: &id001    # '&id001' define un ANCLA: le pone nombre a este nodo
    - FOSS                #  para poder reutilizarlo más abajo sin repetirlo.
    - Linux

- user:
    name: Mario           # Sin llaves ni comillas: la INDENTACIÓN define la jerarquía.
    lastname: Santos
    hobbies:              # Una lista se escribe con guiones, un ítem por línea.
        - arts
        - sports
        - TACS
    otherHobbies: *id001  # '*id001' es la REFERENCIA: acá se inserta el nodo anclado arriba.
    age: !!str 35         # '!!str' FUERZA el tipo: 35 se interpreta como el string "35",
                          #  no como número. Es el control de tipos custom de YAML.
```

Características:

- **Superset de JSON:** todo JSON válido es YAML válido.
- **Indentación obligatoria**, lo que lo vuelve más legible para humanos.
- Soporta **referencias** (anclas) para reutilizar nodos sin repetirlos.
- Permite **tipos de datos custom** y forzar la interpretación de un tipo.

> ⚠️ **Cuidado con el tipado implícito.** YAML interpreta demasiado por su cuenta: el literal `no` sin comillas se lee como el booleano `false`. Es la fuente clásica de bugs silenciosos en archivos de configuración.

**Ventajas:** legibilidad · referencias · tipos custom.
**Desventajas:** **no hay un esquema de validación oficial** (existen varios, ninguno estandarizado).
**Usos típicos:** Swagger / OpenAPI, archivos de configuración de infraestructura.

### 2.4. Comparativa y regla práctica

| | **XML** | **JSON** | **YAML** |
|---|---|---|---|
| Legibilidad | Máquina y humano | Humano | Humano (la mejor) |
| Jerarquía | Tags anidados | Llaves anidadas | Indentación |
| Arrays | Tags repetidos (convención) | Nativos | Guiones |
| Tipos | Ambiguos sin XSD | Diferenciados | Diferenciados + custom |
| Validación | XSD (oficial) | JSON Schema (oficial) | Sin estándar oficial |
| Comentarios | Sí | **No** | Sí |
| Peso | El más pesado | Liviano | Liviano |
| Usos típicos | RSS, XHTML, SOAP | JavaScript, APIs | OpenAPI, configs |

> **La regla práctica: XML para HTML, JSON para APIs, YAML para configuraciones.** No es una ley, es lo que verás en el 90% de los casos.

Sobre el peso en el cable: **XML tiende a ser más pesado que JSON** por la repetición de tags. La **compresión** (gzip, brotli) achica mucho esa diferencia, pero no es gratis: comprimir y descomprimir cuesta CPU. Es un intercambio explícito — **más cómputo a cambio de menos red**.

---

## 3. Binary encoding 🔴

Los formatos binarios atacan las tres limitaciones que quedan: compatibilidad entre lenguajes, performance de serialización, y tamaño del mensaje. El precio es que dejan de ser legibles por humanos.

Los representativos: **Protocol Buffers** (Google), **Cap'n Proto**, **Thrift** (Facebook), **FlatBuffers**, **Avro**.

### 3.1. Protocol Buffers en detalle

**Características**

- **Language-neutral:** soportado en múltiples lenguajes; cada uno suele traer su librería o herramienta.
- **Platform-neutral.**
- **Alta performance** en serialización y deserialización.
- **Self-describing messages** (opcional): permite serializar y deserializar sin necesitar siempre el archivo `.proto`.
- **Proto3 JSON mapping:** puede emitir JSON, para interoperar con quien no habla binario.
- **Evolucionable:** el esquema puede cambiar sin romper compatibilidad — con una condición, que viene abajo.

**Desventajas**

- **No es human-readable.** No podés abrir el mensaje y leerlo.
- Disponible en menos lenguajes que JSON, y requiere librerías de terceros.

### 3.2. Cómo se ve un mensaje

```proto
message Person {
  required string name  = 1;   // ← ese "= 1" NO es un valor por defecto: es el NÚMERO DE CAMPO
  required int32  id    = 2;   //   (field tag). Es la clave de todo Protobuf, ver abajo.
  optional string email = 3;   // 'optional' = puede no venir. 'required' = tiene que venir.

  enum PhoneType {             // Un enum: conjunto cerrado de valores posibles.
    MOBILE = 0;
    HOME   = 1;
    WORK   = 2;
  }

  message PhoneNumber {        // Un mensaje anidado dentro de otro: estructura compuesta.
    required string number = 1;
    optional PhoneType type = 2 [default = HOME];   // valor por defecto si no viene
  }

  repeated PhoneNumber phone = 4;   // 'repeated' = es una lista de PhoneNumber (0 o más).
}
```

> 🔴 **¿Qué es el número al lado de cada campo?** Es **la clave de la retrocompatibilidad**. En el mensaje binario **no viajan los nombres de los campos, viajan esos números** — por eso pesa tan poco. Mientras se respete la numeración, el esquema puede evolucionar: agregar campos nuevos con números nuevos, renombrar campos existentes (el nombre no viaja) o deprecar campos, **sin romper a los clientes viejos**. Reutilizar un número para otra cosa sí los rompe.

### 3.3. El flujo de trabajo

```
  ┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
  │ 1. Escribís el   │        │ 2. El compilador │        │ 3. Compilás ese  │        │ 4. Usás esas     │
  │    .proto que    │───────►│    protoc genera │───────►│    código junto  │───────►│    clases para   │
  │    define la     │        │    código        │        │    con tu        │        │    serializar y  │
  │    estructura    │        │                  │        │    proyecto      │        │    deserializar  │
  └──────────────────┘        └──────────────────┘        └──────────────────┘        └──────────────────┘
         salida                      salida                      salida
    ┌──────────────┐          ┌──────────────────┐        ┌──────────────────┐
    │ archivo      │          │ .java, .py, .cc  │        │ clases           │
    │ .proto       │          │ u otros fuentes  │        │ compiladas       │
    └──────────────┘          └──────────────────┘        └──────────────────┘
```

En una línea: **definís el contrato una vez en `.proto`, y Protobuf te genera (transpila) el código de cada lenguaje que lo necesite.** Ese código generado es el que serializa y deserializa en tu aplicación.

Fijate el contraste con RMI de la Parte 1: allá el contrato era una interfaz **de un lenguaje** y por eso ataba; acá el contrato es un archivo **neutral** del que se derivan las implementaciones de cada lenguaje.

---

## 4. SOAP 🟡

**SOAP — Simple Object Access Protocol** fue una de las primeras estandarizaciones fuertes de comunicación entre servicios web. Es XML aplicado a la comunicación entre servicios.

### 4.1. Características

- **Extensible:** admite extensiones, por ejemplo de seguridad (WS-Security).
- **Neutral respecto del transporte:** puede correr sobre HTTP, SMTP o AMQP. No está casado con HTTP — cosa que, como veremos en la Parte 3, sí distingue a REST en la práctica.
- **Independiente del paradigma:** funciona sobre cualquiera.

### 4.2. La estructura: el envelope

Todo mensaje SOAP es un **sobre** (`Envelope`) con **encabezado** (`Header`) y **cuerpo** (`Body`):

```xml
<?xml version="1.0"?>
<soap:Envelope                                            <!-- El sobre: contiene todo.     -->
xmlns:soap="http://www.w3.org/2001/12/soap-envelope"      <!-- 'xmlns' declara un namespace:
                                                               evita que los tags de SOAP
                                                               choquen con los tuyos.        -->
soap:encodingStyle="http://www.w3.org/2001/12/soap-encoding">

<soap:Header>       <!-- Metadatos: autenticación, transacciones, ruteo. Opcional. -->
...
</soap:Header>

<soap:Body>         <!-- La carga útil: qué se pide o qué se responde. -->
...
  <soap:Fault>      <!-- Bloque estandarizado de ERROR. SOAP modela las fallas
  ...                    dentro del mismo mensaje, no fuera de él.          -->
  </soap:Fault>
</soap:Body>

</soap:Envelope>
```

**Request** — pedir el precio de las manzanas:

```xml
<?xml version="1.0"?>
<soap:Envelope
xmlns:soap="http://www.w3.org/2003/05/soap-envelope/"
soap:encodingStyle="http://www.w3.org/2003/05/soap-encoding">

<soap:Body>
  <m:GetPrice xmlns:m="https://www.w3schools.com/prices">  <!-- La OPERACIÓN que se invoca.
                                                                Los parámetros van adentro. -->
    <m:Item>Apples</m:Item>                                <!-- El parámetro.               -->
  </m:GetPrice>
</soap:Body>

</soap:Envelope>
```

**Response:**

```xml
<?xml version="1.0"?>
<soap:Envelope
xmlns:soap="http://www.w3.org/2003/05/soap-envelope/"
soap:encodingStyle="http://www.w3.org/2003/05/soap-encoding">

<soap:Body>
  <m:GetPriceResponse xmlns:m="https://www.w3schools.com/prices">  <!-- Convención: la respuesta
                                                                        se llama igual + Response -->
    <m:Price>1.90</m:Price>
  </m:GetPriceResponse>
</soap:Body>

</soap:Envelope>
```

Notá qué se está modelando: `GetPrice` es **una operación con parámetros**, no un recurso. SOAP piensa en verbos, en procedimientos. Guardate eso — es exactamente lo que REST invierte.

### 4.3. WSDL

**WSDL — Web Services Description Language** es un formato XML para **describir servicios de red**. Aporta:

- **Discovery:** un cliente puede descubrir qué operaciones ofrece el servicio.
- **Documentación** del contrato.
- **Autogeneración de clientes** a partir de la definición.

```xml
<?xml version="1.0"?>
<definitions name="StockQuote" targetNamespace="http://example.com/stockquote.wsdl"
xmlns:tns="http://example.com/stockquote.wsdl" xmlns:xsd1="http://example.com/stockquote.xsd"
xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/" xmlns="http://schemas.xmlsoap.org/wsdl/">

    <types>   <!-- 1) TIPOS: qué estructuras de datos existen. Se declaran con XSD. -->
        <schema targetNamespace="http://example.com/stockquote.xsd"
              xmlns="http://www.w3.org/2000/10/XMLSchema">
            <element name="TradePriceRequest">
                <complexType> <all> <element name="tickerSymbol" type="string"/> </all> </complexType>
            </element>
            <element name="TradePrice">
                <complexType> <all> <element name="price" type="float"/> </all> </complexType>
            </element>
        </schema>
    </types>

    <!-- 2) MENSAJES: qué viaja de ida y qué de vuelta, usando los tipos de arriba. -->
    <message name="GetLastTradePriceInput">  <part name="body" element="xsd1:TradePriceRequest"/> </message>
    <message name="GetLastTradePriceOutput"> <part name="body" element="xsd1:TradePrice"/> </message>

    <!-- 3) PORT TYPE: qué operaciones existen (el equivalente a una interfaz). -->
    <portType name="StockQuotePortType">
        <operation name="GetLastTradePrice">
            <input  message="tns:GetLastTradePriceInput"/>
            <output message="tns:GetLastTradePriceOutput"/>
        </operation>
    </portType>

    <!-- 4) BINDING: CÓMO se transmiten esas operaciones (protocolo concreto: SOAP sobre HTTP). -->
    <binding name="StockQuoteSoapBinding" type="tns:StockQuotePortType">
        <soap:binding style="document" transport="http://schemas.xmlsoap.org/soap/http"/>
        <operation name="GetLastTradePrice">
            <soap:operation soapAction="http://example.com/GetLastTradePrice"/>
            <input>  <soap:body use="literal"/> </input>
            <output> <soap:body use="literal"/> </output>
        </operation>
    </binding>

    <!-- 5) SERVICE: DÓNDE vive concretamente el servicio (la dirección). -->
    <service name="StockQuoteService">
        <documentation>My first service</documentation>
        <port name="StockQuotePort" binding="tns:StockQuoteBinding">
            <soap:address location="http://example.com/stockquote"/>
        </port>
    </service>
</definitions>
```

La estructura se lee de adentro hacia afuera: **qué tipos existen → qué mensajes los usan → qué operaciones los intercambian → sobre qué protocolo → en qué dirección.**

> **El gran valor de WSDL** era que las herramientas podían **generar el código del cliente automáticamente** a partir de la definición del servicio: apuntabas tu IDE al WSDL y te aparecían las clases listas para usar.
>
> **Su gran costo:** la verbosidad —mirá el tamaño de ese archivo para describir *una* operación que devuelve *un* número— y la obligación de mantener la definición actualizada en el servidor.

---

## Para el parcial, si te preguntan

> **¿Por qué hace falta un formato de serialización?**
>
> Porque los extremos de la comunicación no coinciden en cómo representan los datos: los lenguajes tienen conjuntos de tipos distintos y no mapeables uno a uno (Java distingue cuatro tamaños de entero, JavaScript los unifica en `Number`), y por debajo difieren el tamaño de palabra del procesador, la arquitectura de CPU y el endianness. Un formato de serialización es el acuerdo explícito sobre cómo se representan los datos en el cable, independiente de ambos extremos.

> **¿Cuándo conviene un formato binario en vez de JSON?**
>
> Cuando pesan la performance de serialización, el tamaño del mensaje y la generación de código multi-lenguaje más que la legibilidad. Los formatos binarios como Protocol Buffers serializan más rápido, ocupan menos —no viajan los nombres de los campos, solo sus números— y generan clientes tipados para cada lenguaje. Se paga con la pérdida de legibilidad humana y con menos soporte que JSON.

> **¿Qué permite que un esquema de Protocol Buffers evolucione sin romper clientes?**
>
> La numeración de campos. En el mensaje binario viajan los números, no los nombres, así que mientras se respete la numeración existente se pueden agregar campos nuevos, renombrar los actuales o deprecarlos sin afectar a los clientes viejos.

---

## Checkpoint

Respondelas sin volver al texto. Las respuestas van al complemento.

1. Nombrá tres choques concretos entre los tipos de Java y los de JavaScript, y explicá por qué obligan a definir un formato de serialización.
2. ¿Por qué usar una máquina virtual no elimina la necesidad de serializar?
3. ¿Cómo representa XML un array, y por qué eso es un problema?
4. ¿Qué resuelven XSD y JSON Schema, y qué le falta a YAML en ese aspecto?
5. Nombrá una ventaja de JSON sobre XML y una de XML sobre JSON.
6. ¿Qué costo tiene comprimir para achicar la diferencia de peso entre XML y JSON?
7. ¿Qué representa el número al lado de cada campo en un `.proto` y por qué es central?
8. Describí las cuatro etapas del flujo de trabajo de Protocol Buffers.
9. ¿Qué partes tiene un mensaje SOAP y qué va en cada una?
10. ¿Qué aportaba WSDL y cuál era su costo?

---

## Qué viene en la Parte 3

Arranca **REST**, el núcleo de la unidad. SOAP modelaba operaciones —`GetPrice`, `GetLastTradePrice`— sobre un transporte cualquiera. REST invierte las dos cosas: modela **recursos** en vez de operaciones, y se apoya en **HTTP** como transporte real en vez de ignorarlo. Vamos a ver sus seis principios y a subir el **Richardson Maturity Model**: recursos, verbos, idempotencia y códigos de respuesta.

---

**FIN DE LA PARTE 2**
