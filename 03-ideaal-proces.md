# Het Ideale Proces

Dit document beschrijft het ideale werkproces voor AI-ondersteunde softwareontwikkeling met Claude Code, gebaseerd op de lessen uit `01-process.md` en de verbeteringen uit `02-improvements.md`.

---

## Fase 0 — Voorbereiding

### 1. Inputs verzamelen

Verzamel de volgende inputs vóór de eerste conversatie:

| Input | Doel |
|---|---|
| Bronnotebooks / referentieimplementatie | Exacte algoritmelogica (formules, indexatie, CI) |
| API-documentatie | Endpoint-contracts vastleggen vóór implementatie |
| Datamodel-documentatie | Tabelschema's en veldnamen |
| Frontend-documentatie | Componentspecificaties, kleurschema |
| Testdata | Representatief bestand voor end-to-end verificatie |

### 2. Repository inrichten

```bash
# Basis scaffolding
git init && git commit --allow-empty -m "init"

# Branch-protection instellen (via GitHub UI of gh CLI)
gh api repos/{owner}/{repo}/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["test"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews=null \
  --field restrictions=null
```

Voeg direct een `ci.yml` toe (zie Fase 1) zodat branch-protection al geldt vanaf de eerste PR.

---

## Fase 1 — Plan

### In plan mode (`/plan`)

Stel een masterplan op dat bevat:

- Technologiekeuzes (stack, libraries, buildtool)
- Een **iteratielijst** met voor elke iteratie: doel, eindresultaat en kritieke bronbestanden
- Non-obvious implementatiedetails afgeleid uit de bronbestanden
- Een `docs/decisions.md` als startpunt voor het beslissingenlogboek
- Een **security baseline** (zie hieronder)

Sla het plan op als `.claude/plans/masterplan.md`. Dit bestand wordt automatisch geladen in elke vervolgconversatie.

### Security baseline: Defense in Depth

Het **Defense in Depth** principe is het uitgangspunt voor alle implementatiebeslissingen: meerdere onafhankelijke beveiligingslagen, zodat het falen van één laag niet direct tot een compromis leidt.

Leg bij het masterplan per laag expliciet een beslissing vast — ook als het antwoord "niet van toepassing voor deze versie" is:

| Laag | Minimale eis | Voorbeeldmaatregelen |
|---|---|---|
| Input validatie | Verplicht | Saniteer alle externe input; valideer types, lengtes en formaten aan de serverkant |
| Authenticatie & autorisatie | Beoordeel per scope | Least privilege; geen onnodige endpoints publiek bereikbaar |
| Transport | Verplicht | HTTPS in productie; geen gevoelige data in URL-parameters of logs |
| Data & secrets | Verplicht | Geen secrets in code of versiebeheer; `.env`-bestanden in `.gitignore` |
| Logging & monitoring | Verplicht | Log security-relevante events; log nooit gevoelige data (wachtwoorden, tokens, BSN) |
| Afhankelijkheden | Verplicht | Geen bekende CVE's; pin versies; voer `npm audit` / `pip-audit` uit in CI |
| Netwerk & CORS | Beoordeel per scope | Beperk origins, methoden en headers tot wat daadwerkelijk nodig is |

**Koppeling aan het twee-niveaus model**: elke keuze die een van deze lagen verzwakt of omzeilt is automatisch **Niveau 2** (stoppen en vragen), ongeacht hoe klein de keuze verder lijkt.

### GitHub Actions direct in iteratie 1

Voeg bij de scaffold-iteratie meteen een werkende CI-pipeline toe. Dit zorgt dat buildfouten direct zichtbaar zijn, niet pas bij deployment.

#### `.github/workflows/ci.yml`

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run typecheck       # tsc -b --noEmit (met noUnusedLocals)
      - run: npm run lint
      - run: npm run test
      - run: docker build .
```

#### `.github/workflows/deploy.yml`

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/deploy.sh
```

### TypeScript direct aanzetten op maximale strictheid

```json
{
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "strictNullChecks": true
  }
}
```

---

## Fase 2 — Iteratiecyclus

Elke iteratie volgt dit vaste patroon:

```
1. Iteratiespec schrijven
2. Implementeren (sequentieel of parallel — zie Fase 3)
3. Mini-audit uitvoeren
4. PR aanmaken → CI groen → mergen
```

### Iteratiespec-format

Sla elke spec op als `development/iteration-N-naam.md`:

```markdown
## Doel
<één zin>

## Eindresultaat
<wat er zichtbaar/werkend moet zijn na deze iteratie>

## UI-metriekdefinities
<!-- Verplicht voor elke waarde die getoond wordt aan de gebruiker -->
| Metriek | Wat de gebruiker ziet | Formule / bron | Verwacht bereik |
|---|---|---|---|
| Gem. Indexatie | Jaarlijkse prijsstijging T2-jaar | indexatiefactoren[T2_jaar] − 1 | 0% – 10% |

## Keuzes vastgelegd
<!-- Niveau 1 (laag risico): beslissen, vastleggen in deze tabel, kort vermelden bij oplevering. -->
<!-- Niveau 2 (hoog risico of moeilijk omkeerbaar): stoppen en vragen vóór implementatie. -->
| Keuze | Beslissing | Reden |
|---|---|---|
| CORS-policy | allow_origins=["*"] | Interne tool, geen auth, alle origins toegestaan |
| DB-locatie in Docker | /data/vddview.duckdb via DB_PATH env var | Volume op /data voor persistentie |

## Taken
- [ ] A1: ...
- [ ] A2: ...

## Bronbestanden
- `docs/api.md` sectie X
- `notebooks/model.py` regel 847

## Acceptance criteria
- [ ] `GET /api/v1/settings` geeft `aanleveringsmoment_data` terug als integer
- [ ] `tsc -b --noEmit` geeft 0 fouten
- [ ] `docker build` slaagt zonder fouten
- [ ] Alle unit tests slagen
```

### Executieve beslissingen: het patroon dat extra werk veroorzaakt

Elke keuze die Claude maakt en die **niet expliciet in de spec staat**, is een executieve beslissing. Die zijn onzichtbaar — ze staan niet in de code, niet in de commit message, nergens — totdat ze later een probleem veroorzaken.

**Voorbeelden uit VDDview:**

| Wat Claude deed | Wat er niet vastgelegd was | Gevolg |
|---|---|---|
| `indexatie`-veld gebruiken voor de KPI-kaart | Wat "Gem. Indexatie" voor de gebruiker betekent | Kaart toonde −5.5% i.p.v. +5.8% |
| `allow_origins=["*"]` bij Docker-integratie | Of CORS verruimd mocht worden | Stille securitykeuze, niet opgemerkt |
| DB-pad via env var oplossen | Hoe data-persistentie in Docker werkte | Correcte keuze, maar niet afleidbaar uit de spec |
| SPA-fallback route toevoegen aan FastAPI | Of de frontend via FastAPI of apart geserved werd | Correcte keuze, niet gedocumenteerd |

Het probleem is niet dat Claude de verkeerde keuze maakte — soms was de keuze prima. Het probleem is dat de keuze **onzichtbaar** was. Niemand wist dat die keuze gemaakt was, dus niemand kon hem reviewen, betwisten of erop bouwen.

**De twee niveaus:**

**Niveau 1 — Beslissen en notificeren** (standaard):
Lage impact, goed omkeerbaar. Claude maakt de keuze, legt hem vast in de tabel **Keuzes vastgelegd**, en noemt hem kort bij oplevering.
Voorbeelden: CORS-policy, env var namen, hoe een SPA-fallback geïmplementeerd wordt, DB-bestandspad in Docker.

**Niveau 2 — Stoppen en vragen**:
Hoge impact of moeilijk omkeerbaar. Claude stopt vóór implementatie.
Voorbeelden: datamodelwijziging, security-impacterende keuze, fundamentele architectuurkeuze die meerdere iteraties beïnvloedt, keuze die bronbestanden of specs tegenspreekt.

> **Security-uitzondering**: elke keuze die een laag uit de Defense in Depth baseline verzwakt of omzeilt is altijd Niveau 2 — ook als de keuze verder laag-impact lijkt. Zie de security baseline in Fase 1.

**Hoe je dit afdwingt in de planfase:**

Voeg aan het einde van elke iteratiespec een expliciete stap toe:

```
Vóór implementatie: loop de tabel 'Keuzes vastgelegd' door.
Ontbreekt er een keuze? Voeg hem toe (niveau 1) of stel hem als vraag (niveau 2).
Begin pas als de tabel compleet is.
```

### Mini-audit na elke iteratie

Vraag Claude na implementatie expliciet:

> "Controleer de geïmplementeerde outputs tegen de API-documentatie en het datamodel. Zijn alle veldnamen, typen en formules correct?"

Of gebruik de herbruikbare audit-agent prompt:

```
Lees docs/api.md, docs/data-model.md en de bronnotebooks.
Vergelijk elke endpoint-response, elk tabelschema en elke formule
met de huidige implementatie. Rapporteer afwijkingen als genummerde
bevindingen met bestandsnaam en regelnummer.
```

---

## Fase 3 — Parallelle implementatie

Er zijn twee strategieën voor parallelisatie, afhankelijk van beschikbare capaciteit.

---

### Strategie A — Golven (bij rate-limiting)

Gebruik dit wanneer je API-limieten hebt of voorzichtig wilt zijn.

Verdeel taken in golven van **2–3 agents** met **disjuncte bestandssets**:

```
Golf 1 (2–3 agents): meest kritieke / meest afhankelijke taken
  → wacht op voltooiing + mini-audit

Golf 2 (2–3 agents): onafhankelijke taken zonder golf-1-afhankelijkheid
  → wacht op voltooiing + mini-audit

Golf 3 (resterende taken)
```

**Regel voor elke agent-prompt:**
> "Schrijf alleen naar de bestanden die in jouw taakomschrijving staan. Lees andere bestanden alleen ter referentie."

Taakomvang-richtlijn:

| Omvang | Advies |
|---|---|
| 1–2 bestanden | Ideaal — gefocust en makkelijk te reviewen |
| 3–5 bestanden | Acceptabel |
| 6+ bestanden | Vermijden — agent verliest overzicht |

---

### Strategie B — Git worktrees (zonder rate-limiting)

Gebruik dit wanneer je geen limieten hebt en maximale parallelisatie wilt. Elke agent krijgt een **geïsoleerde kopie van de repository** via `git worktree`. Hiermee kunnen agents onafhankelijk werken aan overlappende bestanden zonder merge-conflicten te riskeren.

#### Hoe het werkt

```bash
# Maak een worktree per agent-taak
git worktree add ../project-agent-A feature/agent-A
git worktree add ../project-agent-B feature/agent-B
git worktree add ../project-agent-C feature/agent-C

# Elke directory is een volledige checkout op een aparte branch
# Agents kunnen parallel werken — zelfs in dezelfde bestanden
```

Na voltooiing:

```bash
# Review en merge elke branch
git checkout main
git merge feature/agent-A   # of via PR
git merge feature/agent-B
git merge feature/agent-C

# Ruim worktrees op
git worktree remove ../project-agent-A
git worktree remove ../project-agent-B
git worktree remove ../project-agent-C
```

#### Agent-instructie bij worktree-gebruik

Geef elke agent het pad naar zijn eigen worktree mee:

```
Jouw werkdirectory is: ../project-agent-A
Branch: feature/agent-A
Taak: [taakomschrijving]

Commit je wijzigingen aan het einde met een beschrijvende commit message.
Je mag alle bestanden in de repo lezen en schrijven — andere agents werken
in hun eigen worktrees en kunnen jouw bestanden niet overschrijven.
```

#### Vergelijking van de twee strategieën

| Aspect | Golven (A) | Worktrees (B) |
|---|---|---|
| Vereiste capaciteit | Laag | Hoog (geen rate-limiting) |
| Merge-conflicten | Geen (disjuncte sets) | Mogelijk bij merge — oplossen na afloop |
| Taakverdeling | Strikt disjunct | Vrij — agents mogen zelfde bestanden aanraken |
| Overhead | Laag | Iets meer (worktree setup + merge) |
| Maximale parallelisatie | Beperkt door limieten | Onbeperkt |

#### Wanneer welke strategie?

- **Golven**: dagelijks werk, iteraties met duidelijk afgebakende taken, of wanneer API-limieten van toepassing zijn.
- **Worktrees**: grote refactors, audit-iteraties, of taken waarbij meerdere agents in dezelfde bestanden moeten werken.

---

## Fase 4 — PR en merge

1. Maak een PR aan per iteratie (of per agent-branch bij worktrees)
2. Wacht tot CI groen is (typecheck + lint + tests + docker build)
3. Loop de acceptance criteria in de iteratiespec door
4. Merge naar `main`

---

## Fase 5 — Beslissingenlogboek bijhouden

Voeg na elke iteratie non-obvious beslissingen toe aan `docs/decisions.md`:

```markdown
## [Onderwerp] (iteratie N)

**Beslissing**: [wat er gedaan is]

**Reden**: [waarom — vaak afgeleid uit bronbestanden of constraints]

**Alternatief overwogen**: [wat afgewezen is en waarom]
```

Dit bespaart tijd bij audits en bij het starten van nieuwe Claude-sessies.

---

## Samenvatting: het ideale proces in één oogopslag

```
Fase 0 — Voorbereiding
  └─ Inputs verzamelen, repo inrichten, branch-protection instellen

Fase 1 — Plan
  └─ Masterplan in /plan, iteratielijst, CI-pipeline + Docker in iteratie 1

Fase 2 — Per iteratie
  ├─ Spec schrijven
  │   ├─ UI-metriekdefinities (wat ziet de gebruiker, welke formule)
  │   ├─ Keuzes vastgelegd (elke implementatiekeuze die niet uit bronbestanden volgt)
  │   └─ Acceptance criteria
  ├─ Implementeren (niveau 1: beslissen en notificeren; niveau 2: stoppen en vragen)
  ├─ Mini-audit → PR → CI groen → merge
  └─ Beslissingenlogboek bijwerken

Fase 3 — Parallelisatie
  ├─ Golven van 2–3 agents (bij rate-limiting, disjuncte bestandssets)
  └─ Git worktrees (zonder rate-limiting, volledige isolatie per agent)
```

### Prioriteiten bij opzet

| Prioriteit | Actie | Wanneer |
|---|---|---|
| ★★★ | CI-pipeline + branch-protection | Iteratie 1 |
| ★★★ | Mini-audit prompt per iteratie | Vanaf iteratie 1 |
| ★★★ | Docker-build in CI | Iteratie 1 |
| ★★ | Acceptance criteria in iteratiespec | Vanaf iteratie 1 |
| ★★ | TypeScript strictheid maximaal | Iteratie 1 |
| ★★ | Git worktrees bij grote parallelisatie | Wanneer limieten geen issue zijn |
| ★ | Beslissingenlogboek | Doorlopend |
