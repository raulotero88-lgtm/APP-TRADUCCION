# APP-TRADUCCIÓN — Documento de control del proyecto

> **Documento vivo.** Es la única fuente de verdad del estado del proyecto, las decisiones tomadas y por dónde continuar.
> **Última actualización:** 2026-06-04
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
- 🔄 **Vanesa respondiendo el Bloque 1** del cuestionario.
- ⏳ Pendiente: respuestas de los bloques 2–5 y la decisión de §8.

---

## 8. Decisiones pendientes

1. **¿Qué significa "determinista" aquí?** (bifurca la arquitectura)
   - (a) **Sin LLM** → Opción 1 (determinismo total, no genera prosa nueva).
   - (b) **LLM con garantías deterministas** → Opción 2 (híbrido, cubre también marketing).
   - *Pendiente de las respuestas de Vanesa.*
2. **Alcance de la integración:** ¿herramienta autónoma que devuelve Excel/Word vs. integración automática con Drupal?
3. **Idiomas origen:** ¿hay memoria alemán→español o solo inglés→español?
4. **Política de IA:** ¿se permite enviar texto a IA externa o debe quedarse en Azure interno?

---

## 9. Cuestionario de auditoría (discovery) y su estado

| Bloque | Tema | Estado |
|---|---|---|
| 1 | El dolor real (prioridad de errores, criterio de éxito, coste actual) | 🔄 En curso con Vanesa |
| 2 | Qué texto y cuánto (fichas vs. prosa, volumen, formato, idiomas) | ⏳ Pendiente |
| 3 | Materia prima (acceso/calidad de la TM, glosario existente, alineación) | ⏳ Pendiente |
| 4 | Reglas de oro (no traducir, términos trampa, contexto, estilo) | ⏳ Pendiente |
| 5 | Dónde vive y mantenimiento (Drupal vs. fichero, tiempo real vs. lote, política IA, validación, aprendizaje) | ⏳ Pendiente |

*(El detalle completo de las preguntas está en el historial de la conversación; cuando lleguen respuestas se vuelcan aquí.)*

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

### 2026-06-04 — Sesión 1
- Analizada la documentación aportada por Vanesa (5 imágenes).
- Diagnosticados los 3 errores (Spanglish, mistraducción, inconsistencia).
- Establecido el enfoque: **determinista TM + glosario** sobre RAG semántico.
- Presentadas 3 opciones de arquitectura; recomendada la **Opción 2 (híbrido)**.
- Entregado el **cuestionario de auditoría** (5 bloques) para Vanesa.
- Creado el repositorio y este documento de control.
- **Próximo paso:** recoger respuestas del Bloque 1 (y siguientes) y resolver la decisión de §8.
