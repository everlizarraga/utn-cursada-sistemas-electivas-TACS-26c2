# 🐳 Clase desde Cero — Docker · Roadmap

**Serie:** 7 módulos + 2 archivos de setup + este roadmap · **Alcance:** lo que la materia cubre de Docker y Docker Compose
**Público:** cero conocimiento previo de Docker. Si nunca escuchaste hablar de esto, perfecto: está escrito para eso.

---

## Cómo usar esta serie

1. **En orden.** Cada módulo se para sobre los anteriores. Saltear un módulo rompe la cadena.
2. **Leé primero, ejecutá después.** Todos los comandos traen su salida esperada escrita al lado: podés leer un módulo entero sin tocar la terminal y entender todo. Ejecutar es para fijar, no para descubrir qué pasa.
3. **Los checkpoints no traen respuestas.** Están al final de cada módulo para que recuperes de memoria. Si una no te sale, la sección correspondiente quedó floja: volvé ahí.
4. Un módulo por sesión es un buen ritmo. Dos por sesión es posible en los más livianos (1 y 2 juntos funcionan bien).

## Setup previo (una sola vez, según tu sistema)

Para ejecutar los ejemplos necesitás Docker instalado. Hay un archivo de setup por sistema — **elegí el tuyo, es excluyente** (nadie hace los dos en la misma máquina):

| Tu sistema | Tu archivo |
|---|---|
| macOS | `clase-cero-docker-setup-mac.md` |
| Windows | `clase-cero-docker-setup-windows.md` |
| Linux | ya tenés medio camino hecho: Docker se instala nativo, sin VM — la guía oficial de docker.com es corta y suficiente |

**Cuándo hacerlo:** cuando quieras, pero **obligatorio antes del módulo 4**, que es donde arrancan los comandos. Los módulos 1 a 3 son conceptuales y se leen sin nada instalado. Los setups no son módulos: son un trámite, no un concepto — por eso no llevan número.

## Convención de terminal

Los comandos de esta serie son **shell Unix** y hay un único juego canónico — sin variantes duplicadas. Corre idéntico, sin cambiar un carácter, en:

- **Mac:** la Terminal de siempre
- **Linux:** cualquier terminal
- **Windows:** la **terminal de Ubuntu (WSL)** — la recomendación oficial del material, explicada y configurada en el setup de Windows

**Git Bash** (Windows) es compatible con dos asteriscos — las dos trampas y sus arreglos están en un único recuadro del setup de Windows y no se repiten. **PowerShell y CMD** quedan fuera del material: Docker en sí funciona ahí, pero la sintaxis de alrededor es otro idioma; quien los use, traduce por su cuenta. Única excepción: administrar WSL (`wsl -l -v`, etc.) es territorio Windows y se hace desde PowerShell — todo en el setup.

## Leyenda de bloques

| Marca | Significado |
|---|---|
| 🔴 | Contenido central — alta probabilidad de evaluación |
| 🟡 | Contenido secundario — completa el cuadro |
| 🟢 | Mencionado al pasar — contexto |
| 🕳️ **Madriguera** | Tangente real pero fuera de alcance. Se lee, se suelta, se sigue. No es tarea pendiente. |
| ⚠️ | Trampa: error que te va a pasar si no lo ves venir |
| 🖥️ **Según tu sistema** | El único punto donde Mac, Windows o Linux difieren. Todo lo que no lleva este bloque es idéntico en los tres. |
| 🎓 **Para el parcial, si te preguntan** | Respuesta modelo lista, en formato examen |
| `# ←` | Dentro de una salida de consola: la línea que importa y por qué |
| `...` | Dentro de un comando o de una salida: **recorte** — "acá seguía más y no viene al caso". Jamás se tipea: no es sintaxis |

*(Los archivos de setup son instrumentales: no llevan 🔴🟡🟢 ni 🎓 — nada de ellos entra en un parcial.)*

## Los 7 módulos

| # | Archivo | Tema | Densidad | Qué te llevás |
|---|---|---|---|---|
| 1 | `clase-cero-docker-modulo1-el-problema.md` | El problema antes de Docker | 🟡 | Por qué los procesos pelados chocan, qué cuesta una VM, y qué queremos que todavía no existe |
| 2 | `clase-cero-docker-modulo2-kernel-y-aislamiento.md` | Aislar sin virtualizar | 🔴 | Kernel compartido, namespaces, cgroups, de LXC a Docker, y por qué tu Mac o tu Windows esconden una VM |
| 3 | `clase-cero-docker-modulo3-imagenes-y-capas.md` | La imagen | 🔴 | File system en capas, el Dockerfile, hashes, y por qué un container "parece" un Ubuntu |
| 4 | `clase-cero-docker-modulo4-el-container.md` | El container | 🔴 | Capa de escritura, ciclo de vida, PID 1, y las manos en la consola: run, ps, exec, logs, stop, rm |
| 5 | `clase-cero-docker-modulo5-cache-y-copy-on-write.md` | El costo de las capas | 🔴 | La caché de build, por qué el orden del Dockerfile importa, copy-on-write, y por qué tus datos se evaporan |
| 6 | `clase-cero-docker-modulo6-volumenes-y-redes.md` | Persistir y comunicar | 🔴 | Los tres tipos de montaje, el binding de puertos, redes propias y resolución por nombre |
| 7 | `clase-cero-docker-modulo7-compose-y-registry.md` | Orquestar y distribuir | 🔴 | El YAML de Compose (services/volumes/networks), escalar servicios, y el registry con sus tags |

**Fases:**

```
SETUP (una vez)      FASE A · Fundamentos       FASE B · El corazón           FASE C · Operar en serio
┌──────────────┐     ┌─────────────┐            ┌─────────────────────┐       ┌─────────────────┐
│ setup-mac    │     │ M1  M2      │ ─────────▶ │ M3   M4   M5        │ ────▶ │ M6   M7         │
│ setup-windows│ ──▶ │ el problema │            │ imagen · container  │       │ datos · redes · │
│ (el tuyo)    │     │ y la idea   │            │ · capas             │       │ compose         │
└──────────────┘     └─────────────┘            └─────────────────────┘       └─────────────────┘
 hace falta desde     conceptual, sin comandos   acá se forma el modelo        acá se opera el TP
 el módulo 4
```

## Mapa de hilos que se cierran

Algunos módulos te dejan una molestia a propósito. Es deliberado: primero sentís el dolor, después llega la herramienta que lo cura. Si al terminar un módulo te queda picando "¿y entonces cómo…?", probablemente sea uno de estos:

| Hilo | Se abre en | Se siente en | Se cierra en |
|---|---|---|---|
| H1 — Aislar cuesta carísimo (VMs) | M1 | M1 | **M2** (namespaces + cgroups: aislamiento casi gratis) |
| H2 — "En mi máquina funciona" | M1 | M1 | **M3** (la imagen ES el paquete completo) |
| H3 — ¿Cómo "parece" un Ubuntu si no trae sistema operativo? | M2 | M2 | **M3** (el file system en capas es el disfraz) |
| H4 — ¿El build por qué tarda / por qué vuela? | M3 | M4 | **M5** (la caché de capas y el orden del Dockerfile) |
| H5 — ¿Dónde quedan mis datos? | M4 | M5 (se pierden en serio) | **M6** (volúmenes) |
| H6 — Dos servers en el puerto 8080 | M1 | M4 (aparece el binding) | **M6** (redes: cada container tiene la suya) |
| H7 — Levantar 4 containers a mano es un infierno | M6 | M6 | **M7** (Compose: un YAML, un comando) |

## Relación con el TP de la materia

- **Camino crítico para la Entrega 1** (app corriendo dentro de Docker): setup + módulos 1 a 5, más la parte de Compose del módulo 7.
- **Entrega 2** (base de datos, migración de memoria a DB): suma de lleno el módulo 6 (volúmenes) y el 7 completo (Compose con app + db + network).
- La cátedra evalúa **explícitamente** en el TP: buenas prácticas de Docker, cómo está arquitecturado el `docker-compose`, y qué tipos de imágenes se usan. Los módulos 5 y 7 son donde eso se juega.

## Qué viene después de la serie

Terminados los 7 módulos: práctica sobre el TP real, y después el **apunte maestro** oficial de la unidad — que vas a leer como repaso, porque el entendimiento ya va a haber ocurrido acá.

---

**FIN DEL ROADMAP — Clase desde Cero · Docker**
