# Apunte Maestro — Clase 01 · Parte 1: Comunicar procesos

**Unidad:** clase01 · **Parte 1 de 5**

Cubre el punto de partida de toda la unidad: cómo se comunican dos procesos, primero en la misma máquina y después a través de la red. Sockets como la abstracción más baja con la que trabajamos, y RPC/RMI como el primer intento de tapar esa complejidad. La serialización de los datos que viajan queda para la Parte 2.

**Leyenda de marcas:** 🔴 central / evaluable · 🟡 secundario · 🟢 mencionado al pasar.

---

## 1. El problema de fondo 🟢

Antes de hablar de APIs conviene bajar un escalón, a una pregunta de Sistemas Operativos: **¿cómo hacemos que dos procesos se comuniquen entre sí?**

Pensalo concreto. Tenés dos programas corriendo. Uno calcula algo, el otro necesita ese resultado. Son dos procesos separados: cada uno con su propio espacio de memoria, sin acceso al del otro. No pueden simplemente leer una variable del vecino.

A eso se lo llama **IPC — Inter-Process Communication** (comunicación entre procesos).

### 1.1. Dentro de una misma máquina

Cuando los dos procesos corren en la misma computadora, el sistema operativo ofrece varios mecanismos:

| Mecanismo | Idea |
|---|---|
| **Memoria compartida** | Una región de memoria a la que ambos procesos pueden acceder |
| **Pipes** | Un canal donde uno escribe y el otro lee, como una tubería |
| **Sockets** | Un punto de conexión bidireccional entre dos extremos |
| **Semáforos** | Mecanismo de sincronización para coordinar el acceso a un recurso |
| **Archivos** | Uno escribe, el otro lee (en Linux, además, *todo es un archivo*) |
| **RPC** | Invocar un procedimiento que vive en el otro proceso |

### 1.2. El escalón que importa

La pregunta interesante es la siguiente: **¿y si los procesos corren en computadoras distintas?**

Ahí se cae la mitad de la lista. La memoria compartida deja de existir —no hay una memoria física común— y los semáforos también, porque sincronizan sobre esa memoria que ya no está. Los archivos y los pipes tampoco cruzan el cable por sí solos.

Quedan los mecanismos que entienden de red. **Sobre esa pregunta se construye todo lo que sigue en esta unidad.**

---

## 2. Sockets 🟡

Un **socket** es la abstracción más baja con la que normalmente trabajamos para comunicar procesos a través de la red. Es un extremo de comunicación: un proceso abre uno, el otro abre otro, y entre ambos fluyen bytes.

### 2.1. Dónde viven en el modelo OSI

El modelo **OSI** (Open Systems Interconnection) divide la comunicación en red en siete capas, cada una apoyándose en la de abajo. Los sockets no son una capa: se ubican **entre la capa 4 (transporte) y la capa 5 (sesión)** — es decir, **por debajo de HTTP**.

```
┌────────────────────────────────────┐
│ Layer 7 — Application              │
├────────────────────────────────────┤
│ Layer 6-5 — Presentation · Session │
│            (SSL, HTTP)             │
├────────────────────────────────────┤
│ ▓▓▓▓▓▓▓▓  S O C K E T S  ▓▓▓▓▓▓▓▓  │  ← acá
├────────────────────────────────────┤
│ Layer 4 — Transport (TCP, UDP)     │
├────────────────────────────────────┤
│ Layer 3 — Network (IP)             │
├────────────────────────────────────┤
│ Layer 2 — Data (Ethernet)          │
├────────────────────────────────────┤
│ Layer 1 — Data (par trenzado)      │
└────────────────────────────────────┘
```

Leer esa ubicación es leer la consecuencia: **cuando programás con sockets estás por debajo de HTTP**. Nada de lo que HTTP te regala —métodos, headers, códigos de estado, caché— existe todavía. Si lo querés, lo escribís vos.

Los sockets usan **TCP o UDP** como protocolo de transporte, y **el desarrollador tiene que elegir cuál**.

> 🕳️ **Madriguera — El modelo OSI completo**
> Las siete capas tienen cada una su propio universo de protocolos y responsabilidades. Para esta unidad alcanza con saber ubicar dónde caen los sockets y qué queda por encima (HTTP) y por debajo (TCP/UDP, IP).
> *Volvé al camino — el modelo completo se ve en Redes.*

### 2.2. El flujo con TCP (stream, orientado a conexión)

TCP establece una conexión antes de mandar datos y garantiza que lleguen en orden.

1. El servidor crea el socket y hace **`bind`** a una dirección: IP + puerto. *(Reservar esa dirección para sí.)*
2. Hace **`listen`** y queda bloqueado en **`accept`**, esperando que alguien se conecte.
3. El cliente hace **`connect`**; el servidor acepta, y a partir de ahí ambos intercambian datos con **`read`/`write`** (o `recv`/`send`).

### 2.3. El flujo con UDP (datagram, sin conexión)

UDP no establece conexión: manda paquetes sueltos y no garantiza ni orden ni llegada.

1. El servidor crea el socket y hace **`bind`**.
2. Queda bloqueado en **`recvfrom`**, esperando mensajes.
3. El cliente hace **`sendto`** y el servidor responde.

```
        SERVER                                  CLIENT
     ┌──────────┐                            ┌──────────┐
     │  Socket  │                            │  Socket  │
     └────┬─────┘                            └────┬─────┘
          ▼                                       │
     ┌──────────┐                                 │
     │   Bind   │                                 │
     └────┬─────┘                                 ▼
          ▼                                  ┌──────────┐
     ┌──────────┐ ◄── Block until reply ──── │  Sendto  │
     │ Recvfrom │                            └────┬─────┘
     └────┬─────┘                                 │
          ▼                                       ▼
     ┌──────────┐ ───────── Data ───────────► ┌──────────┐
     │  Sendto  │                             │ Recvfrom │
     └────┬─────┘                             └────┬─────┘
          ▼                                        ▼
     ┌──────────┐                             ┌──────────┐
     │  Sendto  │ ──────────────────────────► │ Recvfrom │
     └──────────┘                             └──────────┘
```

Fijate en la flecha **"Block until reply"**: el cliente manda y queda **bloqueado** esperando. Ese bloqueo es tuyo para administrar — nadie te da un timeout de regalo.

### 2.4. Qué te dan y qué te cobran

**Ventajas**
- Permiten **mantener la conexión abierta** (no armar y desarmar en cada mensaje).
- Dan **mayor control** sobre la conexión.

**Desventajas**
- Son de **bajo nivel**: la API es propensa a errores, y usarla implica conocer el *cómo* y no solo el *qué*.
- Obligan a **implementar un protocolo por encima** para conseguir cosas que no vienen de fábrica: dónde termina un mensaje y empieza el siguiente (*delimitación* y *framing*), reintentos, confirmaciones.
- Hay que **elegir el tipo de socket** —*Datagram*, *Stream* o *Raw*— y el protocolo de transporte, TCP o UDP.
- **Seguridad y autenticación quedan enteras a cargo del desarrollador.**

> 🕳️ **Madriguera — Framing**
> El problema de que TCP entrega un flujo continuo de bytes sin marcar dónde termina cada mensaje. Se resuelve inventando una convención: prefijar la longitud, usar un delimitador, o un largo fijo. Es exactamente uno de los trabajos que un protocolo de más alto nivel ya te resolvió.
> *Volvé al camino.*

### 2.5. Las preguntas que tenés que poder contestar 🟡

Trabajar directo con sockets significa **tomar un montón de decisiones que nadie tomó por vos**. Frente a cualquier implementación sobre sockets, vale desafiarla con estas preguntas:

- ¿Qué pasa si **la conexión se corta**? ¿Hay política de **reintentos**?
- ¿Se valida la **integridad** de lo que llega? ¿Hay *checksum*? ¿Hay protección contra *buffer overflow*?
- ¿Hay **autenticación**? ¿Los datos viajan **encriptados**?

Ninguna de estas tiene respuesta por defecto. Y el peso acumulado de tener que contestarlas todas, en cada proyecto, es lo que empuja la pregunta siguiente: **¿existen mejores abstracciones?**

> 🕳️ **Madriguera — Buffer overflow**
> Escribir más datos de los que entran en el espacio de memoria reservado, pisando lo que había al lado. Con datos que vienen de la red y no se validan, es una vía clásica de ataque.
> *Volvé al camino.*

---

## 3. RPC y RMI 🟡

La primera abstracción histórica que se construyó sobre sockets fue **RPC — Remote Procedure Call** (llamada a procedimiento remoto), y su versión orientada a objetos, **RMI — Remote Method Invocation** (invocación de método remoto).

La idea es de una simplicidad seductora: **hacer que llamar a algo que está en otra máquina se parezca lo más posible a llamar a algo local**. Que escribas `servidor.saludar("Ever")` y te olvides de que hay un cable en el medio.

Veámoslo funcionando y después discutimos el precio.

### 3.1. El código: la interfaz

RMI arranca por el **contrato**: una interfaz que declara qué se puede invocar remotamente. La comparten cliente y servidor.

```java
// 'package' declara el espacio de nombres del archivo (como una carpeta lógica).
package com.mkyong.rmiinterface;

// 'import' trae clases de otros paquetes para poder usarlas por su nombre corto.
import java.rmi.Remote;           // interfaz marcadora: señala "esto es invocable remotamente"
import java.rmi.RemoteException;  // excepción que representa una falla de comunicación remota

// La interfaz extiende Remote. No agrega métodos: es una MARCA que le dice a RMI
// que los métodos de acá se pueden invocar desde otra máquina.
public interface RMIInterface extends Remote {

    // La firma del método remoto: recibe un String, devuelve un String.
    // 'throws RemoteException' es OBLIGATORIO en todo método remoto: declara que la
    // llamada puede fallar por la red (caída, timeout, servidor inalcanzable).
    // Es la única pista, en toda la firma, de que esto no es una llamada local.
    public String helloTo(String name) throws RemoteException;

}
```

Guardate ese `throws RemoteException`. Es el punto donde la abstracción se filtra, y en un rato es el centro del argumento.

### 3.2. El código: el servidor

El servidor **implementa** esa interfaz y se publica para que lo encuentren.

```java
// La clase implementa el contrato (implements RMIInterface) y extiende UnicastRemoteObject,
// la clase base de Java que le da a un objeto la capacidad de ser invocado remotamente:
// se encarga de escuchar en un puerto y de traducir la llamada entrante en una invocación real.
public class ServerOperation extends UnicastRemoteObject implements RMIInterface {

    // ⚠️ Al extender UnicastRemoteObject, Java exige un constructor que declare
    // 'throws RemoteException' (el constructor de la clase padre la lanza).
    // Sin esto, no compila:
    public ServerOperation() throws RemoteException {
        super();
    }

    // '@Override' avisa al compilador que este método implementa uno del contrato.
    // Si la firma no coincidiera exactamente, falla la compilación en vez de fallar en runtime.
    @Override
    public String helloTo(String name) throws RemoteException {
        // System.err es la salida de error de la consola (System.out es la salida estándar).
        // Este print corre EN EL SERVIDOR: es la evidencia de que la llamada cruzó la red.
        System.err.println(name + " is trying to contact!");

        // El valor de retorno viaja de vuelta al cliente por la red.
        return "Server says hello to " + name;
    }

    // 'main' es el punto de entrada de un programa Java.
    // Declara las excepciones que puede lanzar rebind (de red y de URL mal formada).
    public static void main(String[] args) throws Exception {

        // 'Naming' es el registro de nombres de RMI: una guía telefónica donde los objetos
        // remotos se publican con un nombre. Para que esto funcione tiene que haber un
        // registry de RMI corriendo en esa máquina.
        // rebind asocia el nombre "MyServer" a esta instancia, pisando el anterior si existía.
        Naming.rebind("//localhost/MyServer", new ServerOperation());

        System.err.println("Server ready");
    }
}
```

### 3.3. El código: el cliente

El cliente **busca** el objeto por su nombre y lo invoca como si fuera local.

```java
public class ClientOperation {

    public static void main(String[] args)
            throws MalformedURLException, RemoteException, NotBoundException {

        // 'lookup' es la operación inversa a rebind: busca en el registro el objeto
        // publicado con ese nombre y devuelve una referencia a él.
        // El cast (RMIInterface) es necesario porque lookup devuelve un tipo genérico:
        // le decimos al compilador "confiá, esto cumple el contrato RMIInterface".
        // Lo que vuelve NO es el objeto del servidor: es un intermediario (un 'stub')
        // que reenvía cada llamada por la red. Para el código, es indistinguible.
        RMIInterface look_up = (RMIInterface) Naming.lookup("//localhost/MyServer");

        // JOptionPane abre una ventanita de diálogo de escritorio pidiendo texto al usuario.
        String txt = JOptionPane.showInputDialog("What is your name?");

        // 👈 ACÁ ESTÁ TODO EL PUNTO DE RMI.
        // Esta línea es indistinguible de una llamada local a un objeto en memoria.
        // Pero abre una conexión, serializa el argumento, lo manda por la red, espera,
        // recibe la respuesta y la deserializa. Nada de eso se ve.
        String response = look_up.helloTo(txt);

        System.out.println(response);
    }

}
```

```
// ¿CÓMO FUNCIONA?
//
// 1. Arranca el registry de RMI en la máquina del servidor.
// 2. Se ejecuta ServerOperation: crea la instancia y la publica bajo el nombre "MyServer".
//    → El servidor imprime: "Server ready"
// 3. Se ejecuta ClientOperation: hace lookup de "//localhost/MyServer" y recibe un stub.
// 4. Se abre el diálogo. El usuario escribe, por ejemplo, "Ever".
// 5. El cliente invoca look_up.helloTo("Ever"). El stub serializa "Ever" y lo manda.
// 6. El servidor recibe, ejecuta el método real.
//    → El servidor imprime: "Ever is trying to contact!"
// 7. El servidor devuelve "Server says hello to Ever"; el stub lo deserializa.
//    → El cliente imprime: "Server says hello to Ever"
//
// Resultado esperado:
//   consola del SERVIDOR → Server ready
//                          Ever is trying to contact!
//   consola del CLIENTE  → Server says hello to Ever
```

### 3.4. El flujo, en limpio

- **Cliente:** define (comparte) la interfaz con los métodos que va a llamar.
- **Servidor:** hace `bind` a una dirección, escucha requests e implementa los métodos de esa interfaz.
- **Llamada:** el cliente hace un *lookup* de la dirección del servidor, invoca el método y recibe una respuesta.

### 3.5. Qué ganamos

- **Es mucho más simple para el desarrollador que trabajar con sockets.** Comparalo con la lista de decisiones de la sección 2.5: acá no elegiste TCP ni UDP, no hiciste framing, no parseaste bytes.
- **Tenemos tipos de datos y firmas de métodos.** El contrato es una interfaz tipada: el compilador te avisa si llamás mal.

### 3.6. Qué pagamos

- **Acoplamiento fuerte de tecnología.** Cliente y servidor deben compartir lenguaje, o al menos un stack compatible. Un cliente en JavaScript no puede consumir esto.
- **Acoplamiento a detalles de implementación** del lenguaje: el orden y los tipos de los parámetros, la forma interna de serializar. El contrato no es extensible: cambiar la firma rompe a todos los clientes.
- **Múltiples implementaciones incompatibles entre sí:** JRMP, RMI-IIOP, JINI. Elegir una es atarse a ella.
- **Las llamadas son síncronas** —el cliente queda esperando— pero además atraviesan la red, y **los errores de red y los timeouts no se modelan naturalmente**.
- **El manejo de errores se hace con excepciones**, lo que mezcla el canal del flujo normal con el de los errores, y tiene impacto en performance.
- **No comunica lenguajes distintos** de forma transparente.

### 3.7. La abstracción local inexistente 🔴

Este es el talón de Aquiles de RPC/RMI, y el concepto que hay que llevarse de toda esta parte.

**El problema no es que la abstracción sea imperfecta. Es que es demasiado buena.**

Volvé a mirar esta línea:

```java
String response = look_up.helloTo(txt);
```

Es idéntica a una llamada local. Y no lo es: puede tardar 300 milisegundos o no volver nunca, puede fallar porque se cayó la red, porque el servidor está saturado o porque alguien desenchufó un cable. Una llamada local no tiene ninguno de esos modos de falla.

Cuando el desarrollador se olvida de esa diferencia —y la abstracción está diseñada para que se olvide— **modela mal la comunicación**: no pone timeouts, no piensa en reintentos, no considera qué hacer si la respuesta nunca llega, encadena diez llamadas remotas creyendo que son baratas.

La lección de diseño es más general que RMI: **una abstracción que oculta una diferencia esencial no simplifica, engaña**. Lo remoto es esencialmente distinto de lo local, y un buen diseño lo hace visible en vez de taparlo.

> **Para el parcial, si te preguntan: ¿cuál es el principal problema de RPC/RMI?**
>
> La abstracción local inexistente: hacen que una llamada remota se vea igual que una local, cuando no lo es. Eso lleva a modelar mal la comunicación, porque el desarrollador ignora la latencia y los modos de falla propios de la red. A eso se suma el fuerte acoplamiento —de lenguaje, de código y de contrato— que impide comunicar tecnologías distintas.

> **Para el parcial, si te preguntan: ¿por qué no comunicamos servicios directamente con sockets?**
>
> Porque son de muy bajo nivel: obligan a implementar por encima un protocolo propio (delimitación de mensajes, framing, reintentos) y dejan la seguridad, la autenticación y la validación de integridad enteramente a cargo del desarrollador. Cada proyecto tendría que resolver de cero problemas ya resueltos, y sin un estándar común no hay interoperabilidad entre tecnologías distintas.

---

## Checkpoint

Respondelas sin volver al texto. Las respuestas van al complemento.

1. ¿Qué mecanismos de IPC dejan de servir cuando los procesos pasan a correr en máquinas distintas, y por qué?
2. ¿Dónde se ubican los sockets respecto de HTTP, y qué consecuencia práctica tiene eso para quien programa con ellos?
3. Nombrá tres decisiones que tenés que tomar vos si trabajás directo con sockets y que una abstracción más alta ya te resolvería.
4. ¿Cuál es la diferencia de flujo entre un socket TCP y uno UDP del lado del servidor?
5. En el código de RMI, ¿cuál es la única señal, en la firma del método, de que la llamada no es local?
6. ¿Qué gana RPC/RMI respecto de los sockets, y qué paga a cambio?
7. Explicá qué significa "abstracción local inexistente" y por qué es un problema de diseño y no solo un detalle técnico.
8. ¿Por qué un cliente escrito en JavaScript no puede consumir el servidor RMI del ejemplo?

---

## Qué viene en la Parte 2

RPC/RMI dejó abierta una pregunta que no puede esquivarse: si el cliente y el servidor hablan lenguajes distintos, corren en arquitecturas distintas y guardan los números de forma distinta, **¿cómo acordamos qué significan los bytes que viajan por el cable?** Esa es la **serialización de datos**: XML, JSON, YAML, formatos binarios como Protocol Buffers, y el primer gran estándar que se construyó encima, SOAP.

---

**FIN DE LA PARTE 1**
