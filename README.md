# Despegar Internal Skills

Skills internos de análisis para equipos de Despegar/Decolar.

## nps-analyst

Skill de análisis de NPS para el data lake de Despegar. Permite hacer cortes de NPS por journey, segmento, región, agente, buy failures y más.

### Instalación

```bash
npx skills add lucastossolini-despe/skills --skill nps-analyst --agent '*' --yes --copy
```

### Requisitos
- Acceso al data lake de Despegar (tablas `data.lake.*`)
- Acceso a los Google Docs del Knowledge Base NPS (compartidos internamente)
