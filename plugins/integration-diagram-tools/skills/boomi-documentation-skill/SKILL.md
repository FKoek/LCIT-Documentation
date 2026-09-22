---
name: boomi-documentation-skill
description: Genereer een Niveau 3 Integratieproces sequence diagram (Mermaid) en bijbehorende functionele beschrijving vanuit een bestaand Boomi-proces. Gebruik deze skill altijd wanneer de gebruiker vraagt om een sequence diagram, integratiediagram, functionele beschrijving van een Boomi-integratie, of documentatie voor een Boomi-proces te genereren — ook als ze alleen "diagram voor deze integratie" of "documenteer deze flow" zeggen zonder het woord "Mermaid" of "Niveau 3" te noemen. Trigger ook wanneer de gebruiker een Boomi-procescomponent (XML), een component-ID, een platform-URL, of een procesnaam op het Boomi-platform noemt en vraagt om deze te analyseren, te visualiseren of te documenteren. Deze skill implementeert het door het Low Code Integration Team vastgestelde Niveau 3-diagramstandaard (sequence diagrams, geen flowcharts) en leest het Boomi-platform uitsluitend read-only. Gebruik `mulesoft-documentation-skill` of `frends-documentation-skill` in plaats hiervan voor Mulesoft- resp. Frends-integraties.
metadata:
  version: "1.0.0"
---

# Boomi — Niveau 3: Integratieproces sequence diagram generator

**Versie:** 1.0.0

## Taal

**Alle output van deze skill — het Mermaid-diagram (labels, notes, comments) én de functionele beschrijving — wordt altijd in het Engels geschreven**, ongeacht de taal van de aangeleverde input, de proces-XML, of van deze skill-instructies zelf (die blijven in het Nederlands, voor het team). Dit geldt ook bij het corrigeren of aanvullen van een bestaand diagram: lever het resultaat in het Engels op, ook als het origineel Nederlandstalige labels bevat.

Uitzondering: technische identifiers die letterlijk uit de code/config komen (endpoint-paths, systeemnamen, veldnamen) worden niet vertaald — alleen de beschrijvende tekst eromheen (notes, functionele beschrijving, commentaar) is Engelstalig.

## Alleen lezen — nooit wijzigen op het platform

Deze skill **documenteert** een bestaand Boomi-proces. Ze wijzigt, deployt, undeployt, of voert nooit iets uit op het Boomi-platform, ook niet als de gebruiker daarom lijkt te vragen terwijl ze deze skill aanroept. Als het platform rechtstreeks geraadpleegd wordt (zie Stap 1), gebeurt dat uitsluitend met de read-only scripts van de `bc-integration:boomi-integration`-skill — zie `references/boomi-analyse.md` sectie 1 voor de precieze lijst van wat wel en niet gebruikt mag worden. Bij twijfel: lezen mag, schrijven nooit.

## Doel

Deze skill analyseert een bestaand **Boomi**-integratieproces en genereert automatisch:

1. Een **Mermaid sequence diagram** dat voldoet aan de standards voor Niveau 3 (zie `../../shared/standards.md`).
2. Een **functionele beschrijving** van het integratieproces (zie `../../shared/functional-description-template.md`).

Dit is de geautomatiseerde versie van het handmatige stappenplan uit de standards. De skill vervangt de standards niet — die blijven de bron van waarheid. Deze skill past ze toe.

Voor Mulesoft- of Frends-integraties: gebruik de losse `mulesoft-documentation-skill` resp. `frends-documentation-skill`. Deze drie skills zijn bewust gesplitst (elk platform heeft zijn eigen analyse-logica en triggerwoorden), maar delen dezelfde standards — zie "Architectuur" hieronder.

## Wanneer gebruiken

- De gebruiker uploadt een Boomi-procescomponent (XML, `type="process"`) en wil een sequence diagram en/of functionele beschrijving.
- De gebruiker noemt een procesnaam, component-ID, folder of platform-URL op een Boomi-omgeving waar deze skill toegang toe heeft, en wil dat proces gedocumenteerd zien.
- De gebruiker beschrijft een Boomi-integratieproces in eigen woorden en wil dit gedocumenteerd zien volgens de standards.
- De gebruiker vraagt om een bestaand Niveau 3-diagram van een Boomi-proces te controleren, corrigeren of aan te vullen volgens de standards.

Als er geen bestand is geüpload, geen procesnaam/ID/URL is genoemd, en de gebruiker wel over "de integratie" praat, vraag om de proces-XML, een component-ID/platform-URL, of laat de gebruiker de stappen in de tekst beschrijven (bron-systeem, doel-systeem(en), endpoints, tussenliggende connecties, foutafhandeling). Als onduidelijk is of het om Boomi, Mulesoft of Frends gaat, vraag dit na — gok niet, en verwijs zo nodig naar `mulesoft-documentation-skill` of `frends-documentation-skill`.

## Werkwijze (stappenplan)

### Stap 1 — Verzamel input

Er zijn twee manieren om aan de proces-XML te komen (zie `references/boomi-analyse.md` sectie 1 voor het volledige, read-only werkwijze):

1. **De gebruiker levert de XML direct aan** (geüpload bestand, of geplakt in de chat). Werk hiermee zoals met een geüploade Mulesoft-XML of Frends-JSON.
2. **Rechtstreeks van het platform lezen**, wanneer de gebruiker een procesnaam, component-ID, folder of platform-URL noemt. Dit kan alleen als de `bc-integration:boomi-integration`-skill geïnstalleerd en geconfigureerd is op de machine van de gebruiker (controleer met `boomi-env-check.sh`). Gebruik dan uitsluitend de read-only scripts (`boomi-component-search.sh`, `boomi-component-pull.sh`, eventueel `boomi-version-history.sh`/`boomi-component-diff.sh`) — nooit een script dat iets aanmaakt, wijzigt, deployt of uitvoert. Als die skill niet beschikbaar of niet geconfigureerd is, meld dit en vraag om de proces-XML als bestand.

Haal in beide gevallen ook de componenten op (of vraag de gebruiker erom) waarnaar het proces verwijst en die nodig zijn om systemen en endpoints correct te benoemen: Connection- en Operation-componenten achter elke `connectoraction`, en subprocess-componenten achter een `processcall`/`processroute` als die functioneel relevant zijn. Zie `references/boomi-analyse.md` sectie 1, "Afhankelijkheden meelezen".

Als het aangeleverde bestand duidelijk geen Boomi-procescomponent is (bijv. een Mulesoft XML-flow of een Frends JSON-export), meld dit en verwijs naar de bijpassende skill in plaats van door te gaan.

### Stap 2 — Analyseer het proces

Een Boomi-proces is een platte lijst van shapes verbonden via dragpoints (een graaf), geen geneste XML zoals Mulesoft en geen BPMN-diagram zoals Frends — zie `references/boomi-analyse.md` sectie 0 voor hoe je de daadwerkelijke uitvoeringsvolgorde reconstrueert vanaf de `start`-shape. Doorloop de volledige graaf vanaf de start-shape tot en met elk terminaal pad (`stop`, `exception`, `returndocuments`). Identificeer expliciet:

- **Participants**: het bron-/aanroepende systeem, dit Boomi-proces zelf, en elk extern systeem (per onderscheiden Connection-component) — zie sectie 2 van `references/boomi-analyse.md`.
- **Trigger**: het type start-shape (scheduled/manual, WSS-listener, Event Streams Listen, file watch, Trading Partner Listen, Flow Services, MCP Server) en de bijbehorende sync/async-vertaling — zie sectie 3 van `references/boomi-analyse.md`.
- **Interacties**: elke `connectoraction`-call — actie/methode + resource/pad (uit de Operation-component), en de bijbehorende response.
- **Synchroon vs asynchroon**: connector-calls en WSS/FSS/MCP-triggers zijn synchroon (`->>+`/`-->>-`); Event Streams Listen/Produce, file-watch triggers en Trading Partner Listen/Send zijn asynchroon (`-)`).
- **Logische blokken**: `decision` → `alt`/`else`; `route` → `alt`/`else`-keten per `routevalue` plus Default; `branch` → **opeenvolgende** `rect`-blokken (nooit `par` — branches lopen nooit gelijktijdig); `flowcontrol` met `chunks >= 2` → `par` (dit is wél echte parallelliteit); `flowcontrol` met `forEachCount >= 1` → `loop`.
- **Foutafhandeling**: `catcherrors` → `alt succes`/`else fout`; `businessrules` → `alt Accepted`/`else Rejected`; `exception` → foutresponse of terminale note.
- **Subprocess-aanroepen**: `processcall`/`processroute` — inline de subprocess-logica genest in dezelfde participant-lijn als het puur intern is, of geef het een eigen participant-lijn als het subproces echt een ander extern systeem representeert. Zie sectie 4 van `references/boomi-analyse.md`.
- **Groeperingen**: elke logische processtap wordt een eigen `rect`-blok met een `note over` erboven, conform sectie 4 van de standards.

Sla géén interne stappen op als aparte interacties: Set Properties, Map, Data Process (behalve een externe call vanuit Custom Scripting), Notify, Document Cache, en Stop-shapes zijn geen communicatie tussen systemen en horen niet in een sequence diagram thuis — zie sectie 6 van `references/boomi-analyse.md` en sectie 9 ("Veelgemaakte fouten") van de standards.

### Stap 3 — Genereer het Mermaid-diagram

Bouw het diagram exact volgens `../../shared/standards.md`:

1. Begin met de vaste openingsregels uit de standards (theme/themeVariables config, sectie 3.1) — kopieer deze **letterlijk**, wijzig nooit de kleuren.
2. `sequenceDiagram` + `title Main Flow – <naam>` + `autonumber`.
3. Declareer alle `participant`s expliciet, vóór de eerste interactie.
4. Bouw de flow chronologisch op, boven naar beneden, met de juiste pijlnotatie (sectie 3.5 en 6):
   - `A ->>+ B: <actie>` voor start van een synchrone call met activatie
   - `B -->>- A: <statuscode/resultaat>` voor het bijbehorende antwoord
   - `A -) B: <event>` voor asynchrone fire-and-forget berichten (Event Streams Produce, Trading Partner Send, file-watch/EDI-triggers)
5. Groepeer elke processtap in `rect rgb(235, 245, 255)` (main flow) of `rect rgb(235, 255, 235)` (uitstapjes/sub-calls naar externe systemen), met een `note over` erboven die de stap in one line samenvat.
6. Gebruik `alt`/`else`, `opt`, `loop`, `par` waar van toepassing volgens de vertaaltabel in `references/boomi-analyse.md` sectie 4 — let specifiek op het onderscheid Branch (sequentieel, geen `par`) versus Flow Control met `chunks >= 2` (wél `par`).
7. Sluit af met de response helemaal terug naar het initiërende systeem (`returndocuments`), of een terminale note bij een `exception`/`stop`.
8. Lijn de code uit in kolommen voor leesbaarheid.
9. Controleer tegen sectie 9 ("Veelgemaakte fouten") vóórdat je het diagram oplevert: geen flowchart-concepten (nodes/shapes/classDef), geen dubbele aanhalingstekens `"` in labels, consistente naamgeving.

Render het diagram in een artifact (`.mmd` of als mermaid-codeblok) zodat de gebruiker het direct kan controleren en in VS Code kan plakken.

### Stap 4 — Genereer de functionele beschrijving

Gebruik `../../shared/functional-description-template.md` als structuur. De beschrijving is voor developers, testers en functioneel betrokkenen — leg uit *wat* de integratie doet en *waarom*, niet alleen de technische stappen. Benoem hier ook expliciet elke aanname die je in Stap 1/2 hebt moeten maken (bijv. een niet-opgehaalde Connection of een subprocess dat niet gedocumenteerd kon worden), conform sectie 7 van `references/boomi-analyse.md`.

### Stap 5 — Lever op

Maak een output-bestand (`.md`) met:

1. De functionele beschrijving.
2. Het Mermaid-diagram (codeblok, klaar om te kopiëren naar VS Code — zie de standards, sectie 2, stap 1: geen online Mermaid editor gebruiken i.v.m. dataveiligheid).

Wijs de gebruiker erop dat ze de code lokaal in VS Code moeten bewerken/bewaren (met de Mermaid + Mermaid Preview extensies, LF line endings) en in hun solution design moeten opslaan — dit is een afspraak in de standards, geen keuze van de tool.

## Kwaliteitscontrole vóór oplevering

Loop altijd deze checklist af voordat je het resultaat presenteert:

- [ ] Geen enkele wijziging, deploy, of uitvoering op het Boomi-platform gedaan — alleen gelezen
- [ ] Openingsregels (theme-config) letterlijk overgenomen, ongewijzigd
- [ ] `autonumber` aanwezig direct na `sequenceDiagram`/`title`
- [ ] Alle participants vooraf gedeclareerd met korte, logische namen (per onderscheiden Connection-component, niet per Operation)
- [ ] Elke synchrone call heeft een bijbehorende response met statuscode/resultaat
- [ ] Activatie (`+`/`-`) consistent gebruikt bij synchrone calls
- [ ] Elke processtap heeft een `note over` + `rect`-groepering
- [ ] `alt` alleen bij een echt alternatief pad (Decision/Route/Try-Catch/Business Rules); anders `opt`
- [ ] `loop` alleen bij een Flow Control-stap met `forEachCount >= 1`, nooit als vertaling van Branch
- [ ] `par` alleen bij een Flow Control-stap met `chunks >= 2`; Branch-shapes zijn altijd sequentiële `rect`-blokken, nooit `par`
- [ ] Geen interne stappen (Set Properties, Map, Data Process zonder externe call, Notify, Document Cache, Stop) als aparte interacties gemodelleerd
- [ ] Geen flowchart-concepten, geen `"` in labels
- [ ] Flow volledig: van trigger tot en met de finale response naar het bron-systeem, of tot elk terminaal pad
- [ ] Alle notes, labels, commentaar en de functionele beschrijving zijn in het Engels (technische identifiers zoals endpoint-paths uitgezonderd)
- [ ] Elke aanname over een niet-opgehaalde Connection/Operation/subprocess staat expliciet benoemd in de functionele beschrijving

## Versiebeheer

Deze skill houdt zijn eigen versienummer bij in de frontmatter (`metadata.version`) en in de leesbare `**Versie:**`-regel bovenaan dit document. Dit versienummer gaat over wijzigingen aan déze skill specifiek (Boomi-analyse, template, stappenplan) en is vooral **informatief** — het laat mensen die dit bestand lezen zien hoe volwassen/stabiel de skill is.

**Let op — dit is niet wat de marktplaats gebruikt om updates aan te bieden.** Deze skill wordt gedistribueerd als onderdeel van één plugin (`integration-diagram-tools`, samen met `mulesoft-documentation-skill`, `frends-documentation-skill` en `create-confluence-documentation`). De marktplaats kijkt naar het versienummer in `plugins/integration-diagram-tools/.claude-plugin/plugin.json` — dat is de enige plek die daadwerkelijk bepaalt of gebruikers een update aangeboden krijgen. Zie "Architectuur" hieronder voor het volledige releaseproces.

De gedeelde standards (`../../shared/standards.md`) hebben **geen eigen versienummer per skill** — zie "Architectuur" hieronder voor hoe een wijziging daaraan wordt doorgevoerd.

**Bij elke aanpassing aan deze skill wordt het versienummer verplicht opgehoogd**, ook als daar niet expliciet om gevraagd wordt. Bepaal zelf, op basis van de aard van de wijziging, of het een patch, minor of major betreft (semver):

- **Patch** (bijv. 1.0.0 → 1.0.1): tekstcorrecties, verduidelijkingen, kleine bugfixes die het gedrag niet wezenlijk veranderen.
- **Minor** (bijv. 1.0.0 → 1.1.0): een nieuwe stap, sectie, referentiebestand, of uitbreiding die achterwaarts compatibel is — bestaand gebruik blijft werken.
- **Major** (bijv. 1.0.0 → 2.0.0): een wijziging die het stappenplan, de output-structuur, of de manier waarop de skill wordt aangeroepen wezenlijk verandert, waardoor eerdere aannames over de skill niet meer kloppen.

Werk bij elke wijziging beide plekken bij (frontmatter én de leesbare regel) zodat ze nooit uit sync raken.

## Architectuur: gedeelde standaardbestanden tussen drie skills

Deze skill, `mulesoft-documentation-skill` en `frends-documentation-skill` zijn bewust **gesplitst** (elk platform heeft eigen analyse-logica en eigen triggerwoorden, dus een losse, gerichte skill werkt betrouwbaarder dan één skill die eerst het platform moet raden), maar delen alles wat platform-onafhankelijk is: de standards, de functionele-beschrijving-template, en het lege diagram-skeleton. Om te voorkomen dat die drie in meerdere kopieën uit elkaar gaan lopen, leven ze op **plugin-niveau**, niet in de map van deze skill zelf:

```
plugins/integration-diagram-tools/
├── shared/
│   ├── standards.md                          ← één centraal exemplaar
│   ├── functional-description-template.md    ← idem
│   └── assets/example-skeleton.mmd           ← idem
└── skills/
    ├── mulesoft-documentation-skill/   (verwijst naar dezelfde bestanden)
    ├── frends-documentation-skill/     (verwijst naar dezelfde bestanden)
    └── boomi-documentation-skill/      (deze skill, verwijst naar ../../shared/...)
```

Alleen wat écht platform-specifiek is — `boomi-analyse.md` hier, `mulesoft-analyse.md` resp. `frends-analyse.md` bij de andere twee skills — blijft los per skill.

Het gedeelde skelet (`../../shared/assets/example-skeleton.mmd`) gebruikt bewust platform-neutrale placeholders (`<intermediate-system-1>`, `<intermediate-system-2>`), omdat het door alle drie de skills wordt gebruikt. Vul die placeholders voor een Boomi-integratie als volgt: `<source-system>` → het bron-/aanroepende systeem uit de trigger; `<intermediate-system-1>` → dit Boomi-proces zelf; `<intermediate-system-2>` (en verder) → elk onderscheiden extern systeem (per Connection-component); `<target-system>` → het laatste systeem in de keten.

Er is bewust **geen live koppeling met Confluence** — `shared/standards.md` is een hardcoded bestand dat gewoon gelezen wordt, geen aparte check, geen automatische sync.

**Bijwerken van de standards is een bewuste, handmatige actie door één persoon, ongeveer eens per maand** (of eerder, bij een relevante Confluence-wijziging):

1. Kopieer de actuele standards vanaf Confluence naar `plugins/integration-diagram-tools/shared/standards.md` — **één keer, dit werkt automatisch door voor alle drie de skills** omdat ze naar hetzelfde bestand verwijzen.
2. Werk zo nodig ook de platform-specifieke referentiebestanden bij (`boomi-analyse.md` in deze skill, `mulesoft-analyse.md`/`frends-analyse.md` bij de andere twee) als de wijziging daar doorwerkt.
3. Hoog het versienummer op van **alle drie** de documentatie-skills (`mulesoft-documentation-skill`, `frends-documentation-skill`, `boomi-documentation-skill`) en van de plugin zelf (`plugin.json`) — ook als er verder niets aan een van de skills is gewijzigd, want de effectieve inhoud (via de gedeelde standards) is voor alle drie veranderd. Dit is wat de marktplaats gebruikt om de update aan te bieden.
4. Publiceer de nieuwe versie naar de marktplaats-repository.

Gebruikers hoeven zelf niets te doen behalve op "update" klikken wanneer die beschikbaar is.

## Referentiebestanden

- `../../shared/standards.md` — het volledige standards-document (bron van waarheid voor alle syntax- en stijlregels), gedeeld met `mulesoft-documentation-skill` en `frends-documentation-skill`, periodiek handmatig bijgewerkt vanaf Confluence door één persoon.
- `references/boomi-analyse.md` — hoe je een Boomi-procesexport structureel leest (shapes + dragpoints, geen geneste XML), hoe je read-only aan de proces-XML komt (incl. wanneer/hoe de `bc-integration:boomi-integration`-skill te gebruiken), de volledige trigger-taxonomie, de shape-naar-diagram-vertaaltabel, en het Branch-versus-Flow-Control-onderscheid (sequentieel vs. écht parallel).
- `../../shared/functional-description-template.md` — structuur voor de functionele beschrijving, gedeeld met de andere twee documentatie-skills.
- `../../shared/assets/example-skeleton.mmd` — leeg startpunt met de verplichte openingsregels, klaar om in te vullen.

Wil de gebruiker deze documentatie in Confluence hebben? Gebruik daarvoor de losse skill `create-confluence-documentation` — dat is bewust geen onderdeel van deze skill.

Lees `../../shared/standards.md` altijd volledig door vóór het genereren van een diagram — dit bestand bevat de exacte sjablonen (sectie 5) en het volledige uitgewerkte voorbeeld (sectie 8) waar je de output tegen moet spiegelen.
