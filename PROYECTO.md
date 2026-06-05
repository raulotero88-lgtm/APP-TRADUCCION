# APP-TRADUCCIÓN — Documento de control del proyecto

> **Documento vivo.** Es la única fuente de verdad del estado del proyecto, las decisiones tomadas y por dónde continuar.
> **Última actualización:** 2026-06-05
> **Repo:** https://github.com/raulotero88-lgtm/APP-TRADUCCION

---

## 1. Objetivo

Construir una aplicación/script en **Python** que traduzca de forma **consistente y determinista** las descripciones técnicas de productos de hardware de un e-commerce (originales en **inglés y alemán**) a **español** (y, a futuro, otros idiomas), apoyándose en la terminología y traducciones ya validadas por un traductor humano.

El cliente interno del proyecto es **Vanesa** (traductora). El caso surge de un encargo de su departamento de TI de usar IA para traducir decenas de miles de referencias.

---

## 2. El problema (caso de Vanesa)

- TI debe traducir **decenas de miles** de descripciones de producto (alta densidad terminológica), con **alta repetición** (muchas referencias son variaciones de otras → los mismos tecnicismos se repiten, a menudo idénticos).
- Entran referencias nuevas **constantemente**; mantenerlo con traductor humano es inviable por coste.
- En español deben **convivir** términos que SÍ se traducen (`front camera → cámara frontal`) y términos que NO se traducen (`USB-C`, `NFC`, `Wi-Fi`…).
- Primeras pruebas **sin RAG** en su entorno: **Azure OpenAI + CMS Drupal + BD propietaria**.

### Riesgo declarado
La mezcla de idiomas (Spanglish) y la inconsistencia terminológica dificultan diferenciar productos muy similares y perjudican la decisión de compra.

---

## 3. Diagnóstico (de las pruebas de IT sin RAG)

Evidencia: dos SKUs casi idénticos (M3 Mobile SM25W, solo cambia color) traducidos de forma distinta. Tres tipos de error:

| Origen inglés | SKU 1 | SKU 2 | Error |
|---|---|---|---|
| portable data collection device | dispositivo de recopilación de datos portátil | Dispositivo portátil de adquisición de datos | **Inconsistencia** |
| display size | tamaño de pantalla | Diagonal de pantalla | **Inconsistencia** |
| auto-focus | autoenfoque | auto-focus | **Spanglish** |
| front camera | cámara frontal | front camera | **Spanglish** |
| vibration | vibración | vibration | **Spanglish** |
| battery | batería | Acumulador | **Inconsistencia** |
| hand strap | correa de mano | lazo para llevar | **Mistraducción** |

**Conclusión clave:** el mismo input produce outputs distintos → es un problema de **falta de determinismo y de anclaje terminológico**, no de "el modelo no sabe traducir".

### Prioridad de los errores (Vanesa, Bloque 1)

Vanesa ordenó los tres errores de mayor a menor gravedad — y el orden de *dolor* resulta ser el **inverso** al de dificultad técnica:

| # | Gravedad | Error | Por qué (Vanesa) |
|---|---|---|---|
| 1 | **Más grave** | Mistraducción (`hand strap` → `lazo para llevar`) | Engaña: el comprador lee el texto sin ver la foto e imagina otro objeto. Error de *significado*. |
| 2 | Intermedio | Inconsistencia (`batería`/`acumulador`/`pilas`) | En traducción técnica la variabilidad léxica es un defecto; misma pieza → mismo término. Inventa diferencias que no existen. |
| 3 | **Menos grave** | Spanglish (término sin traducir) | El comprador profesional conoce el inglés del sector; cuesta *fatiga de lectura*, no malentendido. |

**Lectura de diseño:** los tres síntomas colapsan en una sola palanca → **cobertura terminológica + imposición obligatoria del término en la salida**. No son tres problemas, son tres caras del mismo.

---

## 4. Decisión de enfoque: determinista (TM + glosario), no RAG semántico

> **El objetivo real es CONSISTENCIA, y la consistencia es una propiedad determinista. Los embeddings dan "parecido", no "igual" — por eso un RAG semántico puro puede perpetuar el bug actual.**

Esto es, en esencia, un problema clásico y maduro de **Memoria de Traducción (TM) + Gestión Terminológica (glosario/termbase)**, que las herramientas TAO/CAT (el propio **Trados Studio** de Vanesa) resuelven de forma determinista desde hace décadas.

Sobre la hipótesis original de Vanesa ("buscar por el origen inglés y que venga la traducción validada como dato asociado"): **es correcta en el mecanismo**, y es exactamente lo que hace una TM — el español está guardado como el valor emparejado del inglés. No hacen falta embeddings cross-lingües porque **la alineación ya es explícita** en la TMX.

---

## 5. Opciones de arquitectura evaluadas

| Opción | Descripción | Determinismo | Cubre prosa marketing | Recomendación |
|---|---|---|---|---|
| **1 — TM + Glosario puro (sin LLM)** | Coincidencia exacta TM + sustitución por glosario; lo no cubierto se marca para humano | **Total** | No | Ideal para fichas de producto |
| **2 — Híbrido (núcleo determinista + LLM acotado + guardarraíles)** ⭐ | Lo de la 1 + LLM (Azure OpenAI, temp 0) solo en lo nuevo + pasada determinista de *enforcement* terminológico sobre la salida | Alto (garantías sobre la salida) | Sí | **Recomendada** |
| **3 — RAG con embeddings (hipótesis original)** | Vector store + recuperación semántica + generación LLM | Bajo | Sí | Solo como capa *fuzzy* opcional dentro de la 2, no como columna vertebral |

**Recomendación:** Opción 2. Para las fichas de e-commerce, la capa determinista (TM exacta + glosario) hace el grueso; el LLM queda reservado a la prosa, siempre con validación determinista de terminología en la salida.

---

## 5.1. Visión de la solución final (el "norte")

> **En una frase:** una **app integral** tipo *TMS/CAT pipeline headless* para el catálogo (un "mini-Trados automatizado"), con **núcleo determinista (TM + glosario)** que hace el grueso y un **LLM acotado** solo para lo nuevo, con *enforcement* terminológico sobre su salida.

Claves de la visión:
- **App integral → SÍ**, es la forma (el paraguas que orquesta las capas).
- **RAG → NO** como columna vertebral; a lo sumo un *fuzzy match* opcional como **sugerencia**.
- **LLM interno → SÍ**, pero como la pieza **más pequeña y vigilada**, no como el protagonista.

La idea contraintuitiva: el LLM es el componente **más acotado** del sistema. El protagonista es el núcleo determinista (TM + glosario). No es "un RAG" ni "un LLM con buen prompt": es una tubería por capas donde **cada capa cura un síntoma de §3**.

### Tubería

```
        Texto origen (EN / DE)
                │
   [1] Ingesta + normalización
                │
        ¿Hay match EXACTO en la TM?
          ┌─────┴─────┐
       SÍ │           │ NO
          ▼           ▼
  [2] Reutiliza   [3] LLM acotado (Azure, temp 0)
  la traducción       con los términos del glosario
  validada,           inyectados en el prompt
  calcada             │
          │           ▼
          │   [4] ENFORCEMENT terminológico
          │   (forzar los términos del glosario
          │    sobre la salida del LLM)
          └─────┬─────┘
                ▼
   [5] Salida (Excel para revisar / importable a Drupal)
                ▼
   [6] Comercial detecta errores ──► realimenta
       TM + glosario  (lo mantiene el traductor)
```

### Qué cura cada capa
- **[2] TM exacta** → **inconsistencia** (mismo origen → misma traducción, siempre).
- **[3]+[4] Glosario + enforcement** → **Spanglish** y **mistraducción** (el término correcto ya fichado y se impone).
- **[6] Realimentación** → **aprendizaje** del sistema (comercial = QA distribuido; traductor = dueño del glosario).

### El RAG, fuera de la columna vertebral
Un RAG decide por "parecido"; aquí se necesita "idéntico". Único hueco legítimo: un *fuzzy match* como **sugerencia** (no decisión) cuando no hay match exacto — mejora opcional de la Fase 4, nunca la base.

### Confianza (qué está fijo y qué calibrará el Bloque 2)
- 🟢 **La forma** (app integral, núcleo determinista, LLM acotado + enforcement) — estable.
- 🟡 **Las proporciones** (cuánto cubre la TM vs. cuánto cae al LLM) — las fija el Bloque 2 (repetición + estructura de ficha) y la Fase 1.
- 🟡 **Entrega** (Excel vs. Drupal) → §8.2 · **Alemán** → §8.3 · **"Interno"** (Azure vs. IA externa) → §8.4.

> El Bloque 2 no cambia la *forma*, cambia el *tamaño de las cajas*. La **Fase 1** (TMX → índice de match exacto = cajas [1] y [2]) es la que medirá si la solución tiende a "casi sin LLM" o a "híbrido de verdad".

> ⏳ **Pendiente de validación.** Este "norte" hay que contrastarlo con el **esquema de solución que Raúl planteó en clase** para el caso de Vanesa (vídeo por revisar). Cuando se recupere ese esquema, se compara caja por caja con esta §5.1 y se anota aquí si coincide o en qué difiere.

---

## 6. Activos / materia prima disponible

Aportados por Vanesa (en [`documentos/`](documentos/)):

| Archivo | Qué es | Valor |
|---|---|---|
| `Muestras_de_traduccion_realizada_por_IA_que_nos_remite_IT.jpg` | Pruebas de IT (traducción IA sin RAG) | Muestra los 3 errores |
| `Computer-Aided_Translation-ejemplo.jpg` | Vista bilingüe de Trados Studio | Flujo de trabajo actual |
| `Exportacion_a_TMX_de_segmentos_alineados_como_par_de_idiomas.jpg` | Estructura TMX (`<tuv en-GB>` ↔ `<tuv es-ES>`) | **Fuente principal**: pares alineados validados |
| `Extracto_de_pares_de_segmentos_de_una_memoria_de_traduccion.jpg` | Tabla de TM (ID, EN, ES, estado TC/LI+) | Memoria de traducción |
| `Pares_de_documentos_originales_en_ingles_y_sus_traducciones_a_espanol_.jpg` | Word EN + ES (prosa marketing 500-2000 palabras) | Corpus paralelo para marketing |

**Dos tipos de contenido a tratar:**
- **(A) Fichas de e-commerce** → listas de atributos repetitivas → determinismo casi total.
- **(B) Marketing en prosa** (folletos, revista, blog, mailings, RRSS) → necesita generación con anclaje.

---

## 7. Estado actual

🟡 **Fase de auditoría / discovery.** Sin código todavía (decisión deliberada: primero entender el punto de dolor).

- ✅ Documentación de Vanesa analizada.
- ✅ Diagnóstico y opciones presentadas al cliente (Raúl).
- ✅ Cuestionario de auditoría entregado (ver §9).
- ✅ **Bloque 1 respondido** y volcado (ver §9); jerarquía de errores en §3.
- ✅ **Decisión §8.1 cerrada:** enfoque **Opción 2 (híbrido, dos marchas)**.
- ⏳ Pendiente: respuestas de los bloques 2–5 y las decisiones §8.2–§8.4.
- ⏳ Pendiente: **validar el "norte" (§5.1) contra el esquema que Raúl planteó en clase** — revisar el vídeo, extraer su esquema y compararlo con nuestra arquitectura.

---

## 8. Decisiones pendientes

1. ~~**¿Qué significa "determinista" aquí?**~~ ✅ **RESUELTO (2026-06-05, Bloque 1) → Opción 2 (híbrido, dos marchas).**
   - La definición de Vanesa ("el modelo aplica **obligatoriamente** los términos del glosario validado" + "**siempre habrá un %**" + "la **revisión humana no es viable**") solo es coherente con la Opción 2: el "%" presupone un LLM, y "aplicar obligatoriamente los términos" es la capa de *enforcement* determinista sobre su salida. La Opción 1 choca con "revisión humana no viable" (todo lo nuevo quedaría sin cubrir).
   - **Dos marchas:** fichas de e-commerce ≈ modo determinista (TM exacta + glosario, LLM solo si hace falta); marketing en prosa = LLM acotado + *enforcement* terminológico.
2. **Alcance de la integración:** ¿herramienta autónoma que devuelve Excel/Word vs. integración automática con Drupal?
3. **Idiomas origen:** ¿hay memoria alemán→español o solo inglés→español?
4. **Política de IA:** ¿se permite enviar texto a IA externa o debe quedarse en Azure interno?

---

## 9. Cuestionario de auditoría (discovery) y su estado

| Bloque | Tema | Estado |
|---|---|---|
| 1 | El dolor real (prioridad de errores, criterio de éxito, coste actual) | ✅ Respondido (2026-06-05) — ver abajo |
| 2 | Qué texto y cuánto (fichas vs. prosa, volumen, formato, idiomas) | ⏳ Pendiente |
| 3 | Materia prima (acceso/calidad de la TM, glosario existente, alineación) | ⏳ Pendiente |
| 4 | Reglas de oro (no traducir, términos trampa, contexto, estilo) | ⏳ Pendiente |
| 5 | Dónde vive y mantenimiento (Drupal vs. fichero, tiempo real vs. lote, política IA, validación, aprendizaje) | ⏳ Pendiente |

*(El detalle completo de las preguntas está en el historial de la conversación; cuando lleguen respuestas se vuelcan aquí.)*

### Respuestas — Bloque 1 (2026-06-05)

**P1 · Prioridad de los errores.** Jerarquía completa (tabla y razones en §3): **mistraducción** (más grave) > **inconsistencia** > **Spanglish** (menos grave). La mistraducción engaña sobre qué es el producto; la inconsistencia genera incertidumbre sobre si dos productos son realmente distintos; el Spanglish solo fatiga (el comprador es profesional y conoce el inglés del sector).

**P2 · Criterio de éxito.** "Que sea determinista y no alucine", asumiendo que siempre quedará un %. **La revisión humana masiva no es viable.** Por eso el criterio operativo es: en la generación, el modelo debe **aplicar obligatoriamente** los términos extraídos de un **glosario previamente validado**. (→ fija §8.1 = Opción 2.)

**P3 · Coste / quién corrige hoy.** La traducción con IA está **en fase de pruebas**; el **departamento comercial de España** valida las muestras antes de pasar a producción. El nº de referencias hace **inviable** la revisión manual. Modelo de operación futuro propuesto por Vanesa:
- **Comercial** = QA distribuido: detecta errores en su día a día y los deriva al traductor.
- **Traductor humano** = su tarea principal pasa a ser **mantener los glosarios actualizados**, incorporando terminología nueva **antes** del lanzamiento de productos.
- → El **glosario/termbase es el centro de gravedad** del sistema y el punto donde entra el *aprendizaje* (las correcciones de comercial se convierten en entradas de glosario).

**Confirmación externa (no preguntada):** Vanesa ratifica por su cuenta el rechazo al RAG semántico — *"la precisión se pierde en la conversión a ese significado numérico abstracto que es el embedding"* (alinea con §4).

### 3 preguntas imprescindibles
1. De los 3 errores (Spanglish / mal traducido / inconsistente), ¿cuál duele más?
2. ¿Puede Vanesa exportar ella misma la TM de Trados a TMX y está revisada?
3. ¿Debe entrar en Drupal automáticamente o vale un Excel para revisar?

---

## 10. Roadmap por fases (tentativo, se ajustará con las respuestas)

- **Fase 0 — Discovery** *(actual)*: cuestionario y decisiones de §8.
- **Fase 1 — Ingesta de datos:** parsear TMX/TM → índice de coincidencia exacta (clave normalizada → traducción validada).
- **Fase 2 — Glosario/termbase:** lista bilingüe con reglas "traducir / NO traducir"; extracción terminológica semiautomática de la TM.
- **Fase 3 — Motor determinista:** segmentación → match exacto → sustitución por glosario → reensamblaje; lo no cubierto se marca.
- **Fase 4 — (si Opción 2) capa LLM acotada:** *fuzzy match* + glosario al prompt (temp 0) + *enforcement* determinista de terminología sobre la salida.
- **Fase 5 — Validación y métricas:** medir cobertura, consistencia y % de revisión manual contra el criterio de éxito.
- **Fase 6 — Integración/operación:** entrega (Excel/Word o API/Drupal), aprendizaje (realimentar la TM con correcciones).

---

## 11. Stack técnico previsto (tentativo)

- **Lenguaje:** Python.
- **Ingesta:** parseo de TMX/XML (memoria de traducción), TXT/DOCX (corpus paralelo).
- **Almacén determinista:** índice de coincidencia exacta (dict/SQLite) por clave normalizada.
- **Glosario:** estructura bilingüe con flags (traducir/no-traducir, prioridad, longest-match).
- **(Opción 2)** Azure OpenAI, temperatura 0 + caché.
- **Sin dependencia de vector DB** salvo que se decida la capa *fuzzy* semántica opcional.

---

## 12. Registro de sesiones (changelog)

### 2026-06-05 — Sesión 2
- Clonado el repo en local y revisado el estado del proyecto.
- Recibidas y analizadas las **respuestas del Bloque 1** de Vanesa (+ imagen de la `hand strap` que ilustra por qué `lazo para llevar` es una mistraducción de *significado*).
- Establecida la **jerarquía de errores** (mistraducción > inconsistencia > Spanglish) → §3.
- **Cerrada la decisión §8.1 → Opción 2 (híbrido, dos marchas)**, justificada con la propia definición de "determinista" de Vanesa.
- Confirmado el **glosario como centro de gravedad** y el modelo de operación (comercial = QA distribuido; traductor = mantenimiento del glosario antes de cada lanzamiento).
- Documentada la **visión de la solución final** (nueva §5.1) como brújula: app integral con núcleo determinista + LLM acotado + enforcement; RAG fuera de la columna vertebral.
- Anotada tarea pendiente: **revisar el vídeo de clase** y comparar el esquema de solución que Raúl planteó allí con nuestro §5.1.
- **Próximo paso:** pasar a Vanesa el **Bloque 2** del cuestionario (volumen, formato, fichas vs. prosa, idiomas origen).

### 2026-06-04 — Sesión 1
- Analizada la documentación aportada por Vanesa (5 imágenes).
- Diagnosticados los 3 errores (Spanglish, mistraducción, inconsistencia).
- Establecido el enfoque: **determinista TM + glosario** sobre RAG semántico.
- Presentadas 3 opciones de arquitectura; recomendada la **Opción 2 (híbrido)**.
- Entregado el **cuestionario de auditoría** (5 bloques) para Vanesa.
- Creado el repositorio y este documento de control.
- **Próximo paso:** recoger respuestas del Bloque 1 (y siguientes) y resolver la decisión de §8.
