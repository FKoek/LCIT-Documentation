---
name: frends-documentation-skill
description: Genereer een Niveau 3 Integratieproces sequence diagram (Mermaid) en bijbehorende functionele beschrijving vanuit een bestaande Frends-integratie. Gebruik deze skill altijd wanneer de gebruiker vraagt om een sequence diagram, integratiediagram, functionele beschrijving van een Frends-integratie, of documentatie voor een Frends-proces te genereren — ook als ze alleen "diagram voor deze integratie" of "documenteer deze flow" zeggen zonder het woord "Mermaid" of "Niveau 3" te noemen. Trigger ook wanneer de gebruiker Frends process-exports/JSON of C# Code Tasks uploadt en vraagt om deze te analyseren, te visualiseren of te documenteren. Deze skill implementeert het door het Low Code Integration Team vastgestelde Niveau 3-diagramstandaard (sequence diagrams, geen flowcharts). Gebruik `mulesoft-documentation-skill` in plaats hiervan voor Mulesoft-integraties.
metadata:
  version: "1.2.0"
---

# Frends — Niveau 3: Integratieproces sequence diagram generator

**Versie:** 1.2.0

## Taal

**Alle output van deze skill — het Mermaid-diagram (labels, notes, comments) én de functionele beschrijving — wordt altijd in het Engels geschreven**, ongeacht de taal van de aangeleverde input, de flow-JSON, of van deze skill-instructies zelf (die blijven in het Nederlands, voor het team). Dit geldt ook bij het corrigeren of aanvullen van een bestaand diagram: lever het resultaat in het Engels op, ook als het origineel Nederlandstalige labels bevat.

Uitzondering: technische identifiers die letterlijk uit de code/config komen (endpoint-paths, systeemnamen, veldnamen) worden niet vertaald — alleen de beschrijvende tekst eromheen (notes, functionele beschrijving, commentaar) is Engelstalig.

## Doel

Deze skill analyseert een bestaande **Frends**-integratie-implementatie en genereert automatisch:

1. Een **Mermaid sequence diagram** dat voldoet aan de standards voor Niveau 3 (zie `../../shared/standards.md`).
2. Een **functionele beschrijving** van het integratieproces (zie `../../shared/functional-description-template.md`).

Dit is de geautomatiseerde versie van het handmatige stappenplan uit de standards. De skill vervangt de standards niet — die blijven de bron van waarheid. Deze skill past ze toe.

Voor Mulesoft-integraties: gebruik de losse `mulesoft-documentation-skill`. Deze twee skills zijn bewust gesplitst (elk platform heeft zijn eigen analyse-logica en triggerwoorden), maar delen dezelfde standards — zie "Architectuur" hieronder.

## Wanneer gebruiken

- De gebruiker uploadt een Frends process-export (JSON) of C# Code Tasks en wil een sequence diagram en/of functionele beschrijving.
- De gebruiker beschrijft een Frends-integratieproces in eigen woorden en wil dit gedocumenteerd zien volgens de standards.
- De gebruiker vraagt om een bestaand Niveau 3-diagram van een Frends-proces te controleren, corrigeren of aan te vullen volgens de standards.

Als er geen bestand is geüpload maar de gebruiker wel over "de integratie" praat, vraag om de flow-configuratie/code, of laat de gebruiker de stappen in de tekst beschrijven (bron-systeem, doel-systeem(en), endpoints, tussenliggende API's/services, foutafhandeling). Als onduidelijk is of het om Mulesoft of Frends gaat, vraag dit na — gok niet, en verwijs zo nodig naar `mulesoft-documentation-skill`.

## Werkwijze (stappenplan)

### Stap 1 — Verzamel input

Lees alle geüploade bestanden. Verwacht Frends: JSON process-exports of C# Code Tasks, vaak met `Trigger`, `Elements`/`Tasks`, `HTTP Request`, `Loop`, `Router`/`Condition`, `Exception Handler`. Zie `references/frends-analyse.md` voor de volledige mapping van flow-elementen naar sequence diagram-concepten.

**Controleer eerst het exportformaat** (zie sectie 0 van `references/frends-analyse.md`): een `.bpmn`-bestand bevat alleen het diagramskelet, geen functionele details. Als alleen een BPMN-XML is aangeleverd, meld dit expliciet aan de gebruiker en vraag om de JSON-export voordat je verdergaat.

Als het aangeleverde bestand duidelijk geen Frends is (bijv. een Mulesoft XML-flow), meld dit en verwijs naar `mulesoft-documentation-skill` in plaats van door te gaan.

### Stap 2 — Analyseer de flow

Doorloop de volledige flow vanaf het startpunt (trigger) tot en met de uiteindelijke response naar het initiërende systeem. Identificeer expliciet:

- **Participants**: alle systemen die met elkaar communiceren (bron-systeem, Frends-proces/subprocess, externe systemen). Gebruik korte, consistente afkortingen — zie sectie 3.4 en 7.2 van de standards.
- **Interacties**: elke call tussen participants — method + endpoint (bijv. `POST /v1/orders`), en de bijbehorende response (statuscode, bijv. `200 OK`, `400 Bad Request`).
- **Synchroon vs asynchroon**: request/response calls zijn synchroon (`->>+` / `-->>-`); fire-and-forget events (queues, topics, pub/sub) zijn asynchroon (`-)`).
- **Logische blokken**: conditionele paden (Exclusive Decision → `alt`/`else`; **let op de Inclusive Decision** — daar kunnen meerdere takken tegelijk vuren, dus die modelleer je als losse `opt`-blokken per tak, niet als `alt`/`else`, zie sectie 3 van `references/frends-analyse.md`), optionele stappen zonder alternatief (`opt`), herhalingen over een lijst (loop/foreach/while → `loop`), parallelle verwerking (parallel execution → `par`).
- **Foutafhandeling**: exception handlers → modelleer als `alt succes` / `else fout` met het bijbehorende statuscode-pad.
- **Groeperingen**: elke logische processtap (bijv. "ophalen OAuth-token", "aanroepen SAP") wordt een eigen `rect`-blok met een `note over` erboven, conform sectie 4 van de standards.

Sla géén interne transformatiestappen (mapping-only Code Tasks) op als aparte interacties — die zijn geen communicatie tussen systemen en horen niet in een sequence diagram thuis (zie "Veelgemaakte fouten", sectie 9 van de standards).

### Stap 3 — Genereer het Mermaid-diagram

Bouw het diagram exact volgens `../../shared/standards.md`:

1. Begin met de vaste openingsregels uit de standards (theme/themeVariables config, sectie 3.1) — kopieer deze **letterlijk**, wijzig nooit de kleuren.
2. `sequenceDiagram` + `title Main Flow – <naam>` + `autonumber`.
3. Declareer alle `participant`s expliciet, vóór de eerste interactie.
4. Bouw de flow chronologisch op, boven naar beneden, met de juiste pijlnotatie (sectie 3.5 en 6):
   - `A ->>+ B: <actie>` voor start van een synchrone call met activatie
   - `B -->>- A: <statuscode>` voor het bijbehorende antwoord
   - `A -) B: <event>` voor asynchrone fire-and-forget berichten
5. Groepeer elke processtap in `rect rgb(235, 245, 255)` (main flow) of `rect rgb(235, 255, 235)` (uitstapjes/sub-calls naar externe systemen), met een `note over` erboven die de stap in one line samenvat.
6. Gebruik `alt`/`else`, `opt`, `loop`, `par` waar van toepassing — nooit `alt` zonder echt alternatief pad (gebruik dan `opt`), nooit `loop` voor iets dat slechts eventueel gebeurt.
7. Sluit af met de response helemaal terug naar het initiërende systeem.
8. Lijn de code uit in kolommen voor leesbaarheid.
9. Controleer tegen sectie 9 ("Veelgemaakte fouten") vóórdat je het diagram oplevert: geen flowchart-concepten (nodes/shapes/classDef), geen dubbele aanhalingstekens `"` in labels, consistente naamgeving.

Render het diagram in een artifact (`.mmd` of als mermaid-codeblok) zodat de gebruiker het direct kan controleren en in VS Code kan plakken.

### Stap 4 — Genereer de functionele beschrijving

Gebruik `../../shared/functional-description-template.md` als structuur. De beschrijving is voor developers, testers en functioneel betrokkenen — leg uit *wat* de integratie doet en *waarom*, niet alleen de technische stappen.

### Stap 5 — Lever op

Maak een output-bestand (`.md`) met:

1. De functionele beschrijving.
2. Het Mermaid-diagram (codeblok, klaar om te kopiëren naar VS Code — zie de standards, sectie 2, stap 1: geen online Mermaid editor gebruiken i.v.m. dataveiligheid).

Wijs de gebruiker erop dat ze de code lokaal in VS Code moeten bewerken/bewaren (met de Mermaid + Mermaid Preview extensies, LF line endings) en in hun solution design moeten opslaan — dit is een afspraak in de standards, geen keuze van de tool.

## Kwaliteitscontrole vóór oplevering

Loop altijd deze checklist af voordat je het resultaat presenteert:

- [ ] Openingsregels (theme-config) letterlijk overgenomen, ongewijzigd
- [ ] `autonumber` aanwezig direct na `sequenceDiagram`/`title`
- [ ] Alle participants vooraf gedeclareerd met korte, logische afkortingen
- [ ] Elke synchrone call heeft een bijbehorende response met statuscode
- [ ] Activatie (`+`/`-`) consistent gebruikt bij synchrone calls
- [ ] Elke processtap heeft een `note over` + `rect`-groepering
- [ ] `alt` alleen bij een echt alternatief pad; anders `opt`
- [ ] `loop` alleen bij echte iteratie over een lijst/collectie
- [ ] Geen interne transformatie/mapping-stappen als aparte interacties gemodelleerd
- [ ] Geen flowchart-concepten, geen `"` in labels
- [ ] Flow volledig: van trigger tot en met de finale response naar het bron-systeem
- [ ] Alle notes, labels, commentaar en de functionele beschrijving zijn in het Engels (technische identifiers zoals endpoint-paths uitgezonderd)

## Versiebeheer

Deze skill houdt zijn eigen versienummer bij in de frontmatter (`metadata.version`) en in de leesbare `**Versie:**`-regel bovenaan dit document. Dit versienummer gaat over wijzigingen aan déze skill specifiek (Frends-analyse, template, stappenplan) en is vooral **informatief** — het laat mensen die dit bestand lezen zien hoe volwassen/stabiel de skill is.

**Let op — dit is niet wat de marktplaats gebruikt om updates aan te bieden.** Deze skill wordt gedistribueerd als onderdeel van één plugin (`integration-diagram-tools`, samen met `mulesoft-documentation-skill` en `create-confluence-documentation`). De marktplaats kijkt naar het versienummer in `plugins/integration-diagram-tools/.claude-plugin/plugin.json` — dat is de enige plek die daadwerkelijk bepaalt of gebruikers een update aangeboden krijgen. Zie "Architectuur" hieronder voor het volledige releaseproces.

De gedeelde standards (`../../shared/standards.md`) hebben **geen eigen versienummer per skill** — zie "Architectuur" hieronder voor hoe een wijziging daaraan wordt doorgevoerd.

**Bij elke aanpassing aan deze skill wordt het versienummer verplicht opgehoogd**, ook als daar niet expliciet om gevraagd wordt. Bepaal zelf, op basis van de aard van de wijziging, of het een patch, minor of major betreft (semver):

- **Patch** (bijv. 1.0.0 → 1.0.1): tekstcorrecties, verduidelijkingen, kleine bugfixes die het gedrag niet wezenlijk veranderen.
- **Minor** (bijv. 1.0.0 → 1.1.0): een nieuwe stap, sectie, referentiebestand, of uitbreiding die achterwaarts compatibel is — bestaand gebruik blijft werken.
- **Major** (bijv. 1.0.0 → 2.0.0): een wijziging die het stappenplan, de output-structuur, of de manier waarop de skill wordt aangeroepen wezenlijk verandert, waardoor eerdere aannames over de skill niet meer kloppen.

Werk bij elke wijziging beide plekken bij (frontmatter én de leesbare regel) zodat ze nooit uit sync raken.

## Architectuur: gedeelde standaardbestanden tussen twee skills

Deze skill en `mulesoft-documentation-skill` zijn bewust **gesplitst** (elk platform heeft eigen analyse-logica en eigen triggerwoorden, dus een losse, gerichte skill werkt betrouwbaarder dan één skill die eerst het platform moet raden), maar delen alles wat platform-onafhankelijk is: de standards, de functionele-beschrijving-template, en het lege diagram-skeleton. Om te voorkomen dat die drie in twee kopieën uit elkaar gaan lopen, leven ze op **plugin-niveau**, niet in de map van deze skill zelf:

```
plugins/integration-diagram-tools/
├── shared/
│   ├── standards.md                          ← één centraal exemplaar
│   ├── functional-description-template.md    ← idem
│   └── assets/example-skeleton.mmd           ← idem
└── skills/
    ├── mulesoft-documentation-skill/   (verwijst naar dezelfde bestanden)
    └── frends-documentation-skill/     (deze skill, verwijst naar ../../shared/...)
```

Alleen wat écht platform-specifiek is — `frends-analyse.md` hier, `mulesoft-analyse.md` bij de andere skill — blijft los per skill.

Er is bewust **geen live koppeling met Confluence** — `shared/standards.md` is een hardcoded bestand dat gewoon gelezen wordt, geen aparte check, geen automatische sync.

**Bijwerken van de standards is een bewuste, handmatige actie door één persoon, ongeveer eens per maand** (of eerder, bij een relevante Confluence-wijziging):

1. Kopieer de actuele standards vanaf Confluence naar `plugins/integration-diagram-tools/shared/standards.md` — **één keer, dit werkt automatisch door voor beide skills** omdat ze naar hetzelfde bestand verwijzen.
2. Werk zo nodig ook de platform-specifieke referentiebestanden bij (`frends-analyse.md` in deze skill, `mulesoft-analyse.md` in de andere) als de wijziging daar doorwerkt.
3. Hoog het versienummer op van **beide** skills (`mulesoft-documentation-skill` én `frends-documentation-skill`) en van de plugin zelf (`plugin.json`) — ook als er verder niets aan een van beide skills is gewijzigd, want de effectieve inhoud (via de gedeelde standards) is voor beide veranderd. Dit is wat de marktplaats gebruikt om de update aan te bieden.
4. Publiceer de nieuwe versie naar de marktplaats-repository.

Gebruikers hoeven zelf niets te doen behalve op "update" klikken wanneer die beschikbaar is.

## Referentiebestanden

- `../../shared/standards.md` — het volledige standards-document (bron van waarheid voor alle syntax- en stijlregels), gedeeld met `mulesoft-documentation-skill`, periodiek handmatig bijgewerkt vanaf Confluence door één persoon.
- `references/frends-analyse.md` — hoe Frends-process-elementen mappen naar sequence diagram-concepten, inclusief exportformaat-detectie (BPMN-XML vs JSON), de volledige trigger-taxonomie, het Exclusive/Inclusive Decision-onderscheid, en shape Type-codes voor betrouwbare herkenning. Grotendeels gebaseerd op kennis uit de `fc-integration:frends-ipaas-developer`-skill.
- `../../shared/functional-description-template.md` — structuur voor de functionele beschrijving, gedeeld met `mulesoft-documentation-skill`.
- `../../shared/assets/example-skeleton.mmd` — leeg startpunt met de verplichte openingsregels, klaar om in te vullen.

Wil de gebruiker deze documentatie in Confluence hebben? Gebruik daarvoor de losse skill `create-confluence-documentation` — dat is bewust geen onderdeel van deze skill.

Lees `../../shared/standards.md` altijd volledig door vóór het genereren van een diagram — dit bestand bevat de exacte sjablonen (sectie 5) en het volledige uitgewerkte voorbeeld (sectie 8) waar je de output tegen moet spiegelen.
