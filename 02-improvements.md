# Verbetermogelijkheden

## 1. Vaker auditen op basis van technische documentatie

### Wat er nu mis ging

We deden één grote audit (iteratie 8) aan het einde van de implementatiefase. Daarin vonden we acht correctheidsproblemen — waarvan sommige al in iteratie 2 waren geïntroduceerd. Hoe later je fouten vindt, hoe meer werk het kost om ze te herstellen.

### Voorstel: mini-audit per iteratie

Voeg aan elke iteratiespec een **spec-vs-implementatie checklist** toe. Na elke iteratie vraagt Claude expliciet:

> "Controleer de geïmplementeerde outputs tegen de API-documentatie en het datamodel. Zijn alle veldnamen, typen en formules correct?"

Concrete checkpunten per iteratie:

| Domein | Wat te controleren |
|---|---|
| API-responses | Veldnamen en typen exact zoals gedocumenteerd in `docs/api.md` |
| Algoritmelogica | Formules overeenkomen met bronnotebooks (indexatie, CI, months-ahead) |
| Datamodel | Kolomnamen en typen overeenkomen met `docs/data-model.md` |
| TypeScript-interfaces | Alle velden aanwezig in zowel interface als gebruik |

### Automatiseer de audit als agent-taak

Maak een herbruikbare **audit-agent prompt**:

```
Lees docs/api.md, docs/data-model.md en de bronnotebooks.
Vergelijk elke endpoint-response, elk tabelschema en elke formule
met de huidige implementatie. Rapporteer afwijkingen als genummerde
bevindingen met bestandsnaam en regelnummer.
```

Draai deze agent aan het einde van elke iteratie als onderdeel van de standaard workflow.

---

## 2. TypeScript-strictheid verhogen

### Wat er mis ging

`npx tsc --noEmit` gaf 0 fouten, maar `tsc -b` (de productie-build in Docker) gaf 8 fouten — waaronder ongebruikte variabelen en ontbrekende velden in fallback-objecten. Hierdoor werden fouten pas ontdekt bij de Docker-build.

### Oplossing: tsconfig aanscherpen

```json
{
  "compilerOptions": {
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "strictNullChecks": true
  }
}
```

Zorg dat `npm run typecheck` (alias voor `tsc -b --noEmit`) in de pre-commit workflow zit, zodat dezelfde controle draait als in de Docker-build.

---

## 3. Parallelisatie van agents verbeteren

### Huidige aanpak en beperkingen

We lanceerden 8 agents tegelijk. Resultaat: 6 van de 8 raakten het gebruik-limiet en moesten handmatig worden overgenomen. De tijdwinst van parallelisatie werd deels tenietgedaan door de rate-limit-afhandeling.

### Verbeterde strategie: golven van 2–3 agents

```
Golf 1 (2-3 agents): meest kritieke / meest afhankelijke taken
  → wacht op voltooiing

Golf 2 (2-3 agents): onafhankelijke taken die geen golf-1-output nodig hebben
  → wacht op voltooiing

Golf 3 (resterende taken)
```

Dit houdt het gelijktijdige gebruik onder de rate-limit drempel en geeft je tussentijdse controle.

### Taakomvang: kleiner is beter

Een agent-taak die één of twee bestanden aanpakt is betrouwbaarder dan een taak die vijf bestanden aanpakt:

| Taakomvang | Voordelen | Nadelen |
|---|---|---|
| 1–2 bestanden | Snel, gefocust, makkelijk te reviewen | Meer taken om te coördineren |
| 3–5 bestanden | Minder overhead | Groter risico op contextdrift |
| 6+ bestanden | — | Agent verliest overzicht, hogere kans op fouten |

### Disjuncte bestandssets als harde eis

Formuleer agent-taken altijd zo dat geen twee agents hetzelfde bestand schrijven. Voeg aan elke agent-prompt toe:

> "Schrijf alleen naar de bestanden die in jouw taakomschrijving staan. Lees andere bestanden alleen ter referentie."

---

## 4. Iteratiespecs als machine-leesbare checklists

### Huidige situatie

Iteratiespecs zijn Markdown-documenten met een mix van proza en takenlijsten. Verificatiestappen zijn beschrijvend.

### Voorstel: gestructureerde acceptance criteria

Voeg aan elke iteratiespec een `## Acceptance criteria` sectie toe met expliciete, automatisch verifieerbare checks:

```markdown
## Acceptance criteria

- [ ] `GET /api/v1/settings` geeft `aanleveringsmoment_data` terug als integer (niet string)
- [ ] `P_voorspeld` in FactVoorspelling is op index-jaar prijsniveau (ratio t.o.v. laatste werkelijke P < 1.05)
- [ ] `aantal_maanden_vooruit` = `jaren_vooruit × 12 − start_forecast_month + 1`
- [ ] `tsc -b --noEmit` geeft 0 fouten
- [ ] `docker build` slaagt zonder fouten
```

Claude kan deze criteria na implementatie automatisch doorlopen als een verificatie-agent.

---

## 5. Gelaagde documentatie bijhouden

### Wat ontbrak

Gedurende het project ontdekten we non-obvious implementatiedetails (bijv. de PxQ CI-formule, de indexatiefallback-logica) die nergens expliciet gedocumenteerd stonden. We moesten deze herleiden uit de bronnotebooks.

### Voorstel: beslissingenlogboek

Houd een `docs/decisions.md` bij met entries zoals:

```markdown
## PxQ CI-formule (iteratie 2)

**Beslissing**: Gebruik midpoint-interpolatie:
  PxQ_low = ((P_low - P)/2 + P) × ((Q_low - Q)/2 + Q)

**Reden**: Directe vermenigvuldiging van P_low × Q_low onderschat de onzekerheid.
Afgeleid uit NB_DE_Silver_Forecast.py regel 847.

**Alternatief overwogen**: P_low × Q_low (afgekeurd)
```

Dit bespaart tijd bij audits en bij het inwerken van nieuwe ontwikkelaars (of nieuwe Claude-sessies).

---

## 6. Deploymentpijplijn eerder opzetten

### Wat er nu gebeurde

Docker en Railway werden pas in iteratie 9 toegevoegd. Tot die tijd was er geen geautomatiseerde build-validatie.

### Voorstel: Docker-build in iteratie 1

Voeg aan de scaffold-iteratie toe:
- Een basis `Dockerfile` die de frontend bouwt en de backend start
- Een `docker build` stap in de PR-verificatiechecklists

Voordeel: TypeScript-buildfouten (zoals `noUnusedLocals`) worden al bij de eerste PR gevonden, niet pas in iteratie 9.

---

## Samenvatting prioriteiten

| Prioriteit | Verbetering | Inspanning |
|---|---|---|
| ★★★ | Mini-audit per iteratie (herbruikbare audit-agent) | Laag — één prompttemplate |
| ★★★ | Docker-build als verificatiestap vanaf iteratie 1 | Laag — Dockerfile kopiëren |
| ★★ | TypeScript `noUnusedLocals` aanzetten | Laag — één tsconfig-wijziging |
| ★★ | Acceptance criteria als machine-leesbare checklist | Middel — spec-format aanpassen |
| ★★ | Agent-golven i.p.v. alles tegelijk | Middel — coördinatie aanpassen |
| ★ | Beslissingenlogboek bijhouden | Hoog — disciplineert schrijven |
