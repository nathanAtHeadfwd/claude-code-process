# AI-Assisted Development — Process Documentation

Documentatie over het ontwikkelproces dat we hebben gebruikt bij het bouwen van **VDDview**, een webapplicatie voor gemeentelijke zorgkostenprognoses. Ontwikkeld door Van Dam Datapartners met Claude Code als primaire ontwikkelomgeving.

## Documenten

| Document | Inhoud |
|---|---|
| [01-process.md](01-process.md) | Hoe we hebben gewerkt: structuur, inputs, iteraties, agents |
| [02-improvements.md](02-improvements.md) | Verbetermogelijkheden: audits, parallelisatie, agent-strategieën |

## Context

Het project betrof een greenfield full-stack applicatie:
- **Backend**: FastAPI + DuckDB + pandas + pmdarima
- **Frontend**: React 18 + TypeScript + Vite + ECharts + Zustand
- **Tijdspanne**: 9 iteraties van scaffold tot productie-ready Docker deployment
- **Aanpak**: verticale slices, één werkende feature per iteratie
