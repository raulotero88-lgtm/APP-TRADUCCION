# APP-TRADUCCIÓN

Aplicación en Python para traducir de forma **consistente y determinista** descripciones técnicas de productos de hardware de un e-commerce (originales en inglés/alemán) a español, apoyándose en traducciones y terminología ya validadas por traductor humano.

## Estado

🟡 **Fase de auditoría / discovery** — aún sin código. El enfoque elegido es **Memoria de Traducción (TM) + glosario determinista**, no un RAG semántico puro (ver el porqué en el documento de control).

## Documentación

- 📋 **[PROYECTO.md](PROYECTO.md)** — documento de control maestro: objetivo, diagnóstico, decisiones, opciones de arquitectura, roadmap y registro de sesiones. **Empieza por aquí.**
- 📁 **[documentos/](documentos/)** — material aportado por el cliente (Vanesa): pruebas de TI, ejemplos de Trados, estructura TMX y corpus paralelo.

## Idea en una frase

Reutilizar las traducciones validadas por coincidencia exacta y aplicar un glosario con reglas "traducir / NO traducir", de modo que el mismo texto de entrada produzca **siempre** la misma traducción, eliminando el Spanglish, las mistraducciones y la inconsistencia entre productos similares.
