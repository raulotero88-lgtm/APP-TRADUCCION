# ⚔️ Mismo proyecto, dos modelos: Opus 4.8 (arquitecto) vs Fable 5 (auditor)

**El caso:** un e-commerce de hardware necesita traducir decenas de miles de fichas de producto (EN/DE → ES). Las pruebas de IT con Azure OpenAI "a pelo" producían 3 tipos de error: **mistraducciones** (`hand strap` → `lazo para llevar`), **inconsistencias** (mismo término → `batería`/`acumulador` según el día) y **Spanglish** (términos sin traducir). La revisión humana masiva es inviable. La traductora (cliente interno) tiene una memoria de traducción de Trados (TMX) con miles de pares validados.

**El experimento (no planificado):** las 2 primeras sesiones del proyecto las llevó **Opus 4.8**. La tercera, con el proyecto ya documentado, la hizo **Fable 5** con un único prompt: *"dime si la solución propuesta es coherente y si es la mejor posible; razona paso a paso"*.

---

## Qué hizo cada modelo

**🏗️ Opus 4.8 — el arquitecto (sesiones 1–2, días 1–2)**
- Diagnosticó la causa raíz: no es que "el modelo no sepa traducir", es **falta de determinismo y anclaje terminológico** (mismo input → outputs distintos).
- Descartó el RAG semántico como columna vertebral (embeddings dan "parecido", el caso exige "idéntico") y lo reformuló como un problema clásico de **TM + glosario** (lo que hace Trados desde hace décadas).
- Diseñó la arquitectura: núcleo determinista (match exacto en TM + glosario) + LLM acotado a temp 0 solo para lo nuevo + *enforcement* terminológico sobre la salida.
- Montó el proceso: cuestionario de discovery en 5 bloques, documento de control, decisiones cerradas con las respuestas literales de la clienta.
- Validó su arquitectura contra el esquema que el profesor dibujó en clase — y detectó qué añadía sobre él (la TM exacta y la verificación de salida).

**🔍 Fable 5 — el auditor (sesión 3, día 6)**
- Leyó el documento de control **y las imágenes de evidencia** (muestras de errores, TMX, extracto de TM, esquema del profesor) antes de opinar.
- Veredicto: **el plan es coherente — cero errores factuales o de lógica interna encontrados** en el trabajo de Opus.
- Pero encontró **8 omisiones**, dos de ellas capaces de cambiar el resultado del proyecto:
  1. 🧨 **Cuestionó la premisa del caso:** las fichas son listas de atributos separadas por comas, casi seguro **generadas desde campos estructurados** del PIM. Si se confirma, el grueso no es un problema de traducción de texto: es una **tabla de localización por atributo** (más simple y 100% determinista). Nadie había hecho esa pregunta a IT.
  2. 🧨 **El bucle de aprendizaje era reactivo** (comercial detecta errores *ya publicados*). Propuso invertirlo: detector de términos no cubiertos **antes** de generar → la traductora valida 15 términos por lote en vez de revisar miles de textos.
  3. El *enforcement* estaba infraespecificado: sustituir en la salida del LLM exige alineación origen-destino (¿cómo sabes que `lazo para llevar` vino de `hand strap`?). Propuso el mecanismo estándar CAT: masking pre-LLM + verificación con reintento.
  4. **Temp 0 ≠ determinismo** (no-determinismo de GPU, updates de modelo). La garantía real es la caché versionada + realimentar la TM con las salidas aceptadas.
  5. Placeholders de números/unidades en la normalización (multiplica la cobertura de match exacto).
  6. Política de conflictos en la TM (el mismo origen tendrá varios destinos: la propia TM hereda el bug que se quiere curar).
  7. ⚠️ Riesgo de proceso: 3 sesiones y 0 código. La incógnita decisiva (% de cobertura de la TM) no se responde con cuestionarios, se responde con un PoC de un día.
  8. Buy-vs-build ausente: **Azure Custom Translator** acepta TMX y fuerza terminología de forma nativa, dentro del Azure corporativo. Hay que evaluarlo aunque sea para descartarlo.

---

## 📊 Métricas (datos reales del repo)

```
                          Opus 4.8                  Fable 5
─────────────────────────────────────────────────────────────────────
Rol                       Construir el plan         Auditar el plan
Sesiones / commits        2 sesiones / 3 commits    1 sesión / 0 commits
Output                    PROYECTO.md (~3.000       Análisis razonado en
                          palabras, 280 líneas      8 hallazgos, 1 turno
                          netas) + repo + docs
Decisiones cerradas       2 (enfoque determinista   0 (no era su tarea)
                          y Opción 2 híbrida)
Errores factuales         0 detectados por el       0 (la auditoría
                          auditor a posteriori      confirmó la lógica)
Premisas cuestionadas     1 (RAG ≠ solución)        2 (¿es siquiera un
                                                    problema de traducción?
                                                    ¿el bucle debe ser
                                                    reactivo?)
Evidencia consultada      6/6 imágenes + caso       Doc + 4/6 imágenes
Hallazgos que cambian     —                         2 de 8 (los 🧨)
el resultado
Hallazgos de detalle      —                         5 (mecanismos) + 1
técnico                                             (proceso)
```

---

## 🧪 La lectura honesta (importante)

Esto **no es un A/B controlado** y sería trampa venderlo como tal:

- **Tareas distintas:** construir ≠ criticar. El auditor siempre juega con ventaja — llega con el mapa ya dibujado y solo tiene que encontrar los huecos.
- **Contexto asimétrico:** Fable 5 leyó todo el trabajo de Opus como input. Opus partió de cero con 5 imágenes y un caso verbal.
- Lo que **sí** se puede concluir: Fable 5 no encontró ni un error en el razonamiento de Opus (señal de calidad del primero), y aun así aportó 2 hallazgos que ningún humano del proyecto (ni el profesor, cuyo esquema validamos) había planteado — en particular el de "quizá no es un problema de traducción".

**Para compararlos de verdad** habría que darles el mismo prompt sobre el mismo estado del repo y medir:
1. Nº de hallazgos accionables nuevos (no presentes ya en el doc)
2. Nº de afirmaciones falsas / alucinadas (verificables contra el repo)
3. ¿Cuestiona premisas o solo optimiza dentro del marco dado?
4. ¿Consulta la evidencia primaria (imágenes) o solo el documento?
5. Especificidad: ¿nombra mecanismos concretos (masking, caché versionada, Azure Custom Translator) o da consejos genéricos ("añade tests", "valida con el usuario")?

La métrica 3 es la que más separó a los modelos en este caso: el valor diferencial de Fable 5 no estuvo en optimizar el plan, sino en preguntar si el problema estaba bien planteado.
