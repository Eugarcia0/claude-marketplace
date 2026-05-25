# recruitment-request-matcher

Automatiza el pipeline completo de matching de candidatos para un Request abierto en la base **Recruitment Singular** de Airtable. Filtra, rankea por fit real al rol, vincula el Top 5 y notifica al Chapter Lead con un cuadro comparativo directamente en el Activity Feed.

## Install

```shell
/plugin install recruitment-request-matcher@singular-agency-marketplace
```

## Skills

| Skill | Frases disparadoras | Qué hace |
|-------|---------------------|----------|
| `/match-request-candidates` | "procesa el request", "encuentra candidatos para el request de [rol]", "haz el matching para [posición]", "nuevo request de reclutamiento" | Analiza el Request y Position en Airtable, filtra candidatos (Score ≥ 75, inglés ≥ 3, status activo), rankea por fit real, vincula Top 5 al campo Final Talent(s) y publica cuadro comparativo al Chapter Lead |

## Flujo completo

1. **Análisis del Request** — Lee el Request y su Position vinculada en Airtable
2. **Filtro de candidatos** — Status activo + Match Score ≥ 75 + Inglés ≥ nivel 3
3. **Ranking por fit** — Skills críticas de IA (4 pts) + Core backend (2 pts) + Complementarias (1 pt)
4. **Verificación de Cohort** — Confirma calidad del Top 5 antes de escribir
5. **Vinculación** — Actualiza el campo Final Talent(s) del Request
6. **Notificación** — Publica cuadro comparativo al Chapter Lead en el Activity Feed
7. **Lista de espera** — Mantiene ranking completo para sustituciones automáticas

## Notas

- Compatible con la base `Recruitment Singular` (appYv5rfIkVHhQARv)
- Requiere conector de Airtable activo en Cowork
- Los comentarios del Activity Feed no se pueden editar vía API; si se necesita corrección, el skill crea uno nuevo y avisa al usuario que elimine el anterior
