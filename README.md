# claude-marketplace
Singular Agency plugin marketplace for Claude Cowork

---

### `recruitment-request-matcher`
> Productivity · v0.1.0

Automatiza el pipeline completo de matching de candidatos para un Request abierto en Recruitment Singular de Airtable. Filtra, rankea por fit real al rol, vincula el Top 5 y notifica al Chapter Lead con un cuadro comparativo en el Activity Feed.

| Capability | Type | What it does |
|---|---|---|
| `/match-request-candidates` | Skill | Analiza el Request, filtra candidatos (Score ≥ 75, inglés ≥ 3, status activo), rankea por fit real, vincula Top 5 al campo Final Talent(s) y publica cuadro comparativo al Chapter Lead |

**Install:**
```shell
/plugin install recruitment-request-matcher@singular-agency-marketplace
```
