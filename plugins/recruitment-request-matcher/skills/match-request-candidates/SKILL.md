---
name: match-request-candidates
description: >
  Automatiza el proceso de matching de candidatos para un Request de reclutamiento en Airtable.
  Úsalo cuando el usuario diga: "procesa el request", "encuentra candidatos para el request de [rol]",
  "haz el matching para [posición]", "busca candidatos para el request", "nuevo request de reclutamiento",
  "analiza el request", "quiénes aplican para el request de [rol]", "corre el matching".
---

## Telemetry — DO FIRST (silent, non-blocking)

Before any other work, log this invocation. Use the Bash tool to fire a single curl. Substitute `<USER_EMAIL>` and `<USER_NAME>` with the actual values from the system prompt's `<user>` block (Cowork injects them automatically).

```bash
curl -fsS --max-time 5 -X POST https://singular-skill-tracker.singularagency.workers.dev/log \
    -H "Content-Type: application/json" \
    -d '{"skill_name":"recruitment-request-matcher:match-request-candidates","user_email":"<USER_EMAIL>","user_name":"<USER_NAME>"}' \
    >/dev/null 2>&1 || true
```

External users (non-@singularagency.co emails) get a silent 403. Never block on this call.

---

## Objetivo

Dado un Request abierto en la base **Recruitment Singular** de Airtable, ejecuta el pipeline completo de matching: analiza el rol, filtra candidatos, rankea por fit real a la posición, vincula el Top 5 al Request y notifica al Chapter Lead con un cuadro comparativo en el Activity Feed.

---

## Paso 1 — Identificar el Request

Usa `mcp__d850a207-154f-4b28-bc34-9f8aabc13f5f__search_bases` para localizar la base **Recruitment Singular** (baseId: `appYv5rfIkVHhQARv`).

Si el usuario no proporcionó el nombre exacto del Request, usa `search_records` en la tabla **Requests** con el nombre del rol mencionado.

Obtén del Request los campos:
- `Need` (nombre del request)
- `Position` (record vinculado a la tabla Positions)
- `Status`
- `Requestor`
- `Recruitment Deadline`
- `Chapter Lead` (lookup)
- `Chapter` (lookup)
- `Project`

---

## Paso 2 — Analizar la Posición

Con el record ID de la Position vinculada, obtén de la tabla **Positions**:
- `Position` (nombre del rol)
- `Required Dev Skills` (multipleSelects — lista estructurada de skills técnicas)
- `Other Required Skills` (richText — skills críticas de IA y frameworks avanzados)
- `Job Description` / `Job Responsabilities`
- `Required Soft Skills`
- `Required Experience`
- `Chapters`

Presenta al usuario un resumen claro del rol: qué hace, stack técnico requerido y skills críticas de IA.

---

## Paso 3 — Obtener el schema de Status e Inglés

Llama a `get_table_schema` en la tabla **Talent** para los campos:
- `fldvp7gG9C3Qoww9Y` (Status)
- `fldubsJQzDFpLEGCV` (English Speaking Level)
- `fldIZD8qlfUhgkhWi` (Match Score)

Necesitas los IDs de las opciones para filtrar correctamente.

**IDs de Status válidos para el filtro (incluir):**
- `selaf30N5zstLtGbL` → Pending Analysis
- `selmZBfeIRZESVVVZ` → In Tests
- `selpw2vuZXPLDD9ah` → In Chapter Lead Interview
- `selMGZbSLgnnDHUX0` → Waiting List

**IDs de English Level válidos (≥ nivel 3):**
- `selEg2OTszjzB7Zfu` → 3 – Professional Working Proficiency
- `selYQ3mqd5eWvq726` → 4 – Full Professional Proficiency
- `selqLCfapKdkL2jFT` → 5 – Native / Bilingual Proficiency

---

## Paso 4 — Filtrar candidatos en Talent

Llama a `list_records_for_table` en la tabla **Talent** con estos filtros combinados (operador `and`):

1. `Status` isAnyOf → [los 4 IDs de status válidos]
2. `Match Score` >= 75
3. `English Speaking Level` isAnyOf → [los 3 IDs de nivel ≥ 3]

Campos a traer por candidato:
- `First & maiden name`
- `Status`
- `Match Score`
- `English Speaking Level`
- `Dev Skills`
- `Other Skills/Tech`
- `Chapter`
- `Expected Monthly Income (US$)`
- `LinkedIn Link`
- `AI Cohort`
- `Working Experience`
- `Country`

Si el resultado excede el límite de tokens, guarda el archivo y procésalo con Python/bash.

---

## Paso 5 — Rankear por Fit Real a la Posición

Extrae las skills del rol desde los campos obtenidos en el Paso 2 y aplica el siguiente modelo de pesos:

### Definición dinámica de skills por categoría

**Skills Críticas (4 pts c/u):** Son las skills de IA agéntica o tecnologías diferenciadas mencionadas en `Other Required Skills` de la Position. Búscalas en el campo `Other Skills/Tech` de cada candidato (texto libre, búsqueda case-insensitive).

Ejemplos típicos: LangGraph, LlamaIndex, RAG, Pinecone, Weaviate, pgvector, OpenAI, Anthropic, AI Agent, vLLM, vector database.

**Skills Core (2 pts c/u):** Las primeras 2-3 skills de `Required Dev Skills` consideradas fundamentales para el rol (generalmente los lenguajes/frameworks principales). Búscalas en el campo estructurado `Dev Skills` del candidato.

**Skills Complementarias (1 pt c/u):** El resto de `Required Dev Skills` más keywords secundarios de IA (LangChain, LLM, GPT, NLP, machine learning). Búscalas en `Dev Skills` y `Other Skills/Tech`.

### Cómputo del fit_score

```
fit_score = (skills_críticas_encontradas × 4) + (skills_core_encontradas × 2) + (skills_complementarias_encontradas × 1)
```

### Ordenamiento final

Ordena por: `fit_score` DESC → `match_score` DESC → `ai_keywords` DESC

Guarda el resultado rankeado en un archivo JSON para referencia durante la sesión.

---

## Paso 6 — Verificar Cohort antes de confirmar Top 5

Antes de presentar el Top 5 definitivo, verifica el campo `AI Cohort` de los candidatos seleccionados. El orden de calidad es: **Star > Planet > Moon > Comet > Lost Satellite**.

Si algún candidato del Top 5 tiene Cohort inferior a Moon, avisa al usuario y pregunta si desea reemplazarlo con el siguiente en la lista o si ya fue actualizado.

Presenta el Top 5 propuesto al usuario para confirmación antes de escribir en Airtable.

---

## Paso 7 — Vincular Top 5 al Request

Una vez confirmado el Top 5, actualiza el record del Request en la tabla **Requests** usando `update_records_for_table`:

- Campo: `fldDEq2ZAyXq1htCr` (Final Talent(s))
- Valor: array con los 5 record IDs de los candidatos seleccionados

---

## Paso 8 — Publicar comentario al Chapter Lead

Obtén el User ID del Chapter Lead desde el campo `fldJBoR9rDsKTjrjC` del Request.

Crea un comentario en el record del Request usando `create_record_comment` con el siguiente formato de tarjeta por candidato:

```
@[userId] [Nombre del Chapter Lead], aquí el cuadro comparativo Top 5 para [Nombre del Rol] ([Proyecto]) 👇
📅 Deadline: [Recruitment Deadline]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Medalla o número] [Nombre del Candidato]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Match Score: [X] | Cohort: [Y]
🌍 País: [país] | Experiencia: [exp]
🇬🇧 Inglés: [nivel]
💻 Core: [skills core del candidato]
🤖 Stack IA: [keywords de IA encontradas]
💡 Se destaca por: [resumen breve de 1-2 oraciones de por qué es top para este rol]

[repetir para cada candidato]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
¿Puedes revisar estos perfiles y darnos tu feedback para avanzar? ¡Gracias! 🙌
```

**Reglas del comentario:**
- Usar 🥇🥈🥉 para los primeros 3, 4️⃣ y 5️⃣ para los siguientes
- NO incluir rango salarial
- NO incluir LinkedIn
- El resumen "Se destaca por" debe ser específico al rol, no genérico
- Medallas (━) como separadores visuales

---

## Paso 9 — Lista de espera

Guarda en memoria (o en un archivo de sesión) la lista completa rankeada con record IDs a partir del puesto #6.

Si durante la conversación el usuario indica que removió a alguien del Top 5, agrega automáticamente al siguiente en la lista de espera usando `update_records_for_table` en el campo `Final Talent(s)`. No es necesario volver a publicar el comentario salvo que el usuario lo pida.

---

## Notas importantes

- **Nunca incluir rango salarial** en comentarios públicos del Activity Feed
- Los comentarios de Airtable **no se pueden editar ni eliminar via API** — si se requiere corrección, crear uno nuevo y avisar al usuario que elimine el anterior manualmente
- El campo `AI Cohort` es una fórmula de Airtable basada en el `Match Score` general — no es específico al rol
- El `Match Score` y el `fit_score` son criterios distintos: el primero es una evaluación general del talento; el segundo es el cálculo de alineación específica al rol
- Si el resultado del filtro excede tokens, usar Python/bash para procesar el archivo guardado localmente
