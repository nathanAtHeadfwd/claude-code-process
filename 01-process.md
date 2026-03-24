# Het Ontwikkelproces

## Overzicht

We hebben gewerkt met Claude Code als primaire ontwikkelomgeving, in een **iteratief verticaal-slice model**: elke iteratie leverde een werkend stuk end-to-end functionaliteit op, met backend en frontend samen.

---

## Fase 0 — Plan

Vóór de eerste regel code is een masterplan opgesteld in **plan mode** (`/plan`). Dit plan bevatte:

- Technologiekeuzes (stack, libraries)
- Een iteratielijst met doel en eindresultaat per iteratie
- Non-obvious implementatiedetails afgeleid uit de bronbestanden
- Kritieke bronbestanden om naar te verwijzen tijdens implementatie

Het plan werd opgeslagen als persistent bestand (`.claude/plans/`) en was gedurende het hele project beschikbaar als context.

### Wat het plan mogelijk maakte

Door het plan vooraf op te stellen konden we in elke vervolgconversatie direct beginnen zonder de projectcontext opnieuw op te bouwen. Claude laadt het plan automatisch en vervolgt waar het gebleven was.

---

## Iteratiestructuur

Elke iteratie volgde dit patroon:

```
1. Iteratiespec schrijven (development/iteration-N-naam.md)
   - Doel en eindresultaat
   - Backend-taken (genummerd, e.g. A1, A2)
   - Frontend-taken
   - Verificatiestappen

2. Implementeren
   - Backend en frontend samen in één iteratie
   - Zo min mogelijk afhankelijkheden tussen taken

3. Verifiëren
   - Backend: directe API-calls (curl/Python)
   - Frontend: TypeScript-check + visuele inspectie

4. Committen en PR
   - Per iteratie één feature branch
   - PR met verificatiechecklists
```

### Iteratieoverzicht (VDDview)

| # | Naam | Eindresultaat |
|---|---|---|
| 1 | Scaffold + Schema | Beide projecten draaien; DuckDB-schema bestaat |
| 2 | Upload → Pipeline → PxQ Chart | Volledige pipeline op dummydata; PxQ-grafiek zichtbaar |
| 3 | Q + P Charts + Filters | Alle 3 metrics; gemeente/categorie/regeling-filters werkend |
| 4 | KPI Scorecard + Sparklines | 4 KPI-kaarten met sparklines; scenario-selector |
| 5 | Hierarchical Table | Productcategorietabel met onzekerheidsbalk |
| 6 | Comparison View | T1 vs T2 KPI-kaarten + waterfall-decomposities |
| 7 | Settings + Export + Polish | Instellingenview; PDF/Excel-export; skeletonloaders |
| 8 | Audit & Correctness | Systematische correctheidscontrole t.o.v. specificaties |
| 9 | Docker Demo + Railway | Eén commando lokale demo; Railway-deployment |

---

## Vereiste Inputs

### Per project (eenmalig)

| Input | Doel |
|---|---|
| **Bronnotebooks / referentieimplementatie** | Exacte algoritmelogica overnemen (ARIMA, indexatie, CI-formules) |
| **API-documentatie** | Endpoint-contracts vastleggen vóór implementatie |
| **Datamodel-documentatie** | Tabelschema's (DuckDB golden layer) |
| **Frontend-documentatie** | Componentspecificaties, kleurschema, ECharts-configuratie |
| **Testdata** | Representatief CSV-bestand voor end-to-end verificatie |

### Per iteratie

| Input | Doel |
|---|---|
| **Iteratiespec** | Geeft Claude een scherp afgebakende taak met expliciete verificatiestappen |
| **Verwijzingen naar bronbestanden** | Voorkomt dat Claude logica opnieuw uitvindt of afwijkt van de referentie |
| **Verificatiescripts / voorbeeldoutput** | Maakt geautomatiseerde correctheidscontrole mogelijk |

---

## Gebruik van Agents

### Parallelle agents (iteratie 8)

Voor de audit-iteratie werden **8 parallelle background-agents** gestart, elk met een disjunct bestandsset:

```
Agent A: upload.py + pipeline/__init__.py + ingest.py   (prognoselabel, aanleveringsmoment)
Agent B: postprocess.py                                  (P price level step-change fix)
Agent C: data.py + settings.py + api.md                 (PxQ_afgerond, error shapes)
Agent D: arima.py + preprocess.py                       (ARIMA parameters verificatie)
Agent E: hooks.ts                                       (TimeseriesPoint interface)
Agent F: TimeseriesChart.tsx + KostenoverzichtView.tsx  (ECharts crosshair sync)
Agent G: FilterPanel.tsx + filterStore.ts + Views       (productcategorie single-select)
Agent H: Breadcrumb.tsx + HierarchicalTable.tsx         (TODO-documentatie)
```

**Waarom disjuncte bestandssets?** Elke agent schrijft alleen naar bestanden die geen andere agent aanraakt. Dit elimineert merge-conflicten volledig zonder dat worktree-isolatie nodig is.

### Beperkingen die we tegenkwamen

- **Rate limiting**: Bij 8 gelijktijdige agents raakten 6 van de 8 het gebruik-limiet. Oplossing: taken in kleinere golven uitvoeren (2–3 agents tegelijk).
- **Git-detectie**: De `isolation: "worktree"` optie werkte niet betrouwbaar omdat de git-repo niet werd herkend in de omgeving. Omzeild door disjuncte bestandssets te gebruiken op de werkdirectory.

---

## Commit- en PR-discipline

- Eén commit per logische wijziging (niet per bestand)
- Feature branch per iteratie, PR met verificatiechecklists
- Geen `Co-Authored-By` trailers in commit messages
- Grote binaire bestanden (`.duckdb`, uploads) nooit committen

---

## Wat goed werkte

1. **Plan-first**: Het masterplan voorkomt dat je halverwege de architectuur moet herdenken.
2. **Iteratiespec als taakafbakening**: Geeft Claude een concreet eindpunt per iteratie — geen scope-creep.
3. **Bronbestanden als grondwaarheid**: Door altijd naar de referentienotebooks te verwijzen bleef de algoritmelogica correct.
4. **Verificatie vóór commit**: Directe API-calls na elke pipeline-run geven zekerheid over correctheid.
5. **Docker als afsluiter**: Eén deployable artifact als eindresultaat van de buildfase.
