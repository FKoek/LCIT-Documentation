# Boomi-processen analyseren voor het Niveau 3 sequence diagram

Dit document beschrijft hoe je elementen uit een Boomi-procesexport (component-XML, `type="process"`) herkent en vertaalt naar het Niveau 3 Mermaid sequence diagram. Gebruik dit samen met `../../shared/standards.md`. Voor de exacte XML-structuur van elk shape-type (attributen, child-elementen, edge cases) is de `bc-integration:boomi-integration`-skill — indien beschikbaar op de machine van de gebruiker — de uitputtende bron (`references/steps/*.md`, `references/components/*.md`); dit document bevat alleen wat nodig is om te *lezen en te vertalen*, niet om te bouwen.

## 0. Eerst: hoe is een Boomi-proces opgebouwd?

Een Boomi-procescomponent is, anders dan een Mulesoft-flow (geneste XML) maar vergelijkbaar met een Frends-proces, een **platte lijst van shapes met een graaf van verwijzingen**, geen geneste boomstructuur:

```xml
<process ...>
  <shapes>
    <shape name="shape1" shapetype="start" ...>
      <configuration>...</configuration>
      <dragpoints>
        <dragpoint name="shape1.dragpoint1" toShape="shape2" .../>
      </dragpoints>
    </shape>
    <shape name="shape2" shapetype="connectoraction" ...>
      ...
      <dragpoints>
        <dragpoint name="shape2.dragpoint1" toShape="shape5" .../>
      </dragpoints>
    </shape>
    ...
  </shapes>
</process>
```

**Analysewerkwijze:**

1. Bouw eerst een lookup `shape name → shape element` van alle `<shape>`-elementen.
2. Zoek de shape met `shapetype="start"` — dat is het enige startpunt (elk proces heeft er precies één).
3. Volg vanaf daar recursief elke `<dragpoint toShape="...">` naar de volgende shape. Dit is de daadwerkelijke uitvoeringsvolgorde — **niet** de XML-volgorde van de `<shape>`-elementen in het bestand, die kan afwijken.
4. Sommige shapes hebben **meerdere** dragpoints (vertakking): `decision` (true/false), `route` (per `routevalue` + default), `branch` (per branch-nummer, sequentieel — zie sectie 3), `catcherrors` (default/error), `businessrules` (Accepted/Rejected), `processcall` (per return path), `tradingpartneraction` Start (Documents/Acknowledgments/Errors/Archive), `tradingpartneraction` Send (Errors/Archive), `find_changes`/CDC (Added/Updated/Deleted). Loop elke tak apart uit.
5. Meerdere dragpoints kunnen naar **dezelfde** `toShape` wijzen (paden die weer samenkomen). Dat is normaal — het diagram toont dan gewoon dat verschillende paden bij dezelfde volgende interactie uitkomen; geen speciale behandeling nodig, behalve dat je de gedeelde stap niet twee keer hoeft te tekenen als dat de leesbaarheid schaadt.
6. Terminal shapes (`stop`, `exception`, `returndocuments` — altijd `<dragpoints/>` leeg) beëindigen dat pad.

## 1. Alleen lezen: hoe kom je aan de procesexport?

Deze skill **wijzigt nooit iets op het Boomi-platform** — geen push, geen create, geen deploy/undeploy, geen extensions, geen branch/merge-acties, geen test-executions. Alleen lezen om te documenteren. Er zijn twee manieren om aan de proces-XML te komen:

**A. De gebruiker levert de XML direct aan** (geëxporteerde/gepulde component-XML, `type="process"`). Werk hiermee zoals met een geüploade Mulesoft-XML of Frends-JSON.

**B. Rechtstreeks van het platform lezen**, wanneer de gebruiker een procesnaam, component-ID, folder of platform-URL noemt in plaats van een bestand. Dit vereist dat de `bc-integration:boomi-integration`-skill geïnstalleerd en geconfigureerd is (eigen `.env` met platform-credentials in de actieve werkmap). Gebruik dan **uitsluitend** deze read-only scripts van die skill:

- `boomi-component-search.sh` — om het proces (en zijn componenten) te vinden op naam/folder/type.
- `boomi-component-pull.sh` — om de proces-XML en de componenten waarnaar het verwijst te downloaden.
- `boomi-version-history.sh` / `boomi-component-diff.sh` — alleen als de gebruiker expliciet een specifieke versie of vergelijking vraagt.

**Gebruik nooit** `boomi-component-create.sh`, `boomi-component-push.sh`, `boomi-deploy.sh`, `boomi-undeploy.sh`, `boomi-extensions.sh set`, `boomi-branch.sh create/delete/merge*`, `boomi-folder-create.sh`, `boomi-test-execute.sh`, `event-streams-setup.sh create-*`, of enige handmatige `curl` naar de platform-API. Als de gebruiker vraagt om iets op het platform te wijzigen ("kun je deze flow ook meteen opschonen/redeployen"), meld dat dit buiten het bereik van deze documentatie-skill valt en verwijs naar de `boomi-integration`-skill zelf.

Als `boomi-integration` niet beschikbaar of niet geconfigureerd is (`boomi-env-check.sh` faalt), meld dit aan de gebruiker en vraag om de proces-XML als bestand aan te leveren in plaats van door te gaan met gokken.

### Afhankelijkheden meelezen

Een proces-XML alleen vertelt zelden het hele verhaal — `connectionId`/`operationId`/`mapId`/`processId` zijn GUID's die naar aparte componenten verwijzen. Om de systemen, endpoints en eventuele subprocessen correct te benoemen, pull (of vraag de gebruiker om) in elk geval:

- **Connection-componenten** (`connector-settings`) waarnaar een `connectoraction` verwijst — geven de echte systeemnaam en base-URL/host.
- **Operation-componenten** (`connector-action`) — geven de daadwerkelijke resource path/methode/actie (bijv. de WSS `objectName`+`operationType`, de REST resource path, de SQL-actie).
- **Subprocess-componenten** waarnaar een `processcall` of `processroute`-stap verwijst, als het functioneel relevant is of te achterhalen (zie sectie 3, "processcall").
- **API Service Component** (`type="webservice"`), als het proces via een Advanced-atom API wordt ontsloten — dat verandert het externe pad t.o.v. het kale `/ws/simple/...`-pad van de WSS-operation.

Als een referentie niet op te halen is (verwijderd, geen toegang, of de gebruiker levert alleen de hoofd-XML aan), benoem dit expliciet als aanname in de functionele beschrijving (zie sectie 7) in plaats van te gokken naar een systeemnaam.

## 2. Participants herkennen

- **Bron-/aanroepend systeem**: het systeem of de partij die de trigger activeert (zie de trigger-tabel in sectie 3) — een HTTP-caller, een trading partner, een file-drop, of "Scheduler" bij een tijdgestuurde start.
- **Dit Boomi-proces zelf**: één participant-lijn voor de hoofdlogica van het proces. Subprocessen die puur interne verwerking/orkestratie zijn (geen eigen externe systemen) horen **in dezelfde lijn** als geneste `rect`-blokken — net als een Mulesoft sub-flow of een Frends-subprocess dat geen eigen externe calls doet. Een subprocess dat wél zelfstandig met een duidelijk ander extern systeem praat mag een eigen participant-lijn krijgen als dat de leesbaarheid van het diagram verbetert.
- **Elk extern systeem**: één participant per onderscheiden **Connection-component** (niet per Operation — meerdere operations op dezelfde connection zijn dezelfde participant). Gebruik de displaynaam van de Connection-component (kort gemaakt), niet de ruwe `connectorType`-identifier (bijv. "SAP" in plaats van `invixoconsultinggroupas-OZI90V-boomia-prod`).
- **Event Streams-topics en Trading Partners** zijn ook participants, benoemd naar het topic resp. de partner (of "Trading Partner (AS2)" als de specifieke partner niet is opgehaald).

Er is in Boomi geen ingebakken laag-conventie zoals Mulesoft's `-ea`/`-pa`/`-sa`. Gebruik gewoon korte, herkenbare namen; als meerdere, functioneel verschillende connections naar hetzelfde soort systeem wijzen (bijv. drie losse REST-connecties naar drie verschillende partner-API's), geef ze allemaal hun eigen naam — verzin geen verzamelnaam die suggereert dat het één systeem is.

## 3. Trigger (start-shape) → sequence diagram-vertaling

Elk proces heeft precies één `shapetype="start"`-shape. Het `<configuration>`-kind-element bepaalt het triggertype:

| Start-configuratie | Sync/async | Sequence diagram-vertaling |
| --- | --- | --- |
| `<noaction/>` ("No Data" — scheduled/manual) | — | Self-call zoals sjabloon 5.1: `note over <process>: Scheduler` (of "Manual trigger"), `<process> ->>+ <process>: Start processing` ... `<process> ->>- <process>: End processing` |
| `<passthroughaction/>` ("Data Passthrough" — subprocess) | — | **Geen eigen start-interactie tekenen** — dit proces wordt aangeroepen via een `processcall` vanuit een ander (ouder-)proces; de interactie die daar al gemodelleerd is, is de binnenkomst |
| `<connectoraction actionType="Listen" connectorType="wss">` (Web Services Server) | sync | `<caller> ->>+ <process>: <METHOD*> /ws/simple/<operationType><ObjectName>` (of het curated API-pad als een API Service Component het proces wrapt) — zie de Operation-component voor het echte pad en `inputType` (bepaalt of het GET of POST is) |
| `<connectoraction actionType="Listen" connectorType="officialboomi-X3979C-events-prod">` (Event Streams Listen) | async | `<Event Streams topic> -) <process>: message received` |
| `<connectoraction actionType="Consume" ...>` als start (on-demand pull) | sync-achtig | `<process> ->>+ <Event Streams topic>: consume batch` / `<Event Streams topic> -->>- <process>: N message(s)` |
| `<connectoraction actionType="LISTEN" connectorType="disk-sdk">` (Disk V2 file watch) | async | `<file source> -) <process>: file detected` |
| `<connectoraction actionType="Listen" connectorType="fss">` (Flow Services Server) | sync | `<Boomi Flow> ->>+ <process>: invoke <action>` |
| `<connectoraction actionType="Listen" connectorType="officialboomi-X3979C-mcp-prod">` (MCP Server) | sync | `<AI agent> ->>+ <process>: call tool <tool-name>` |
| `<tradingpartneraction actionType="Listen">` (Trading Partner Start, B2B/EDI) | async | `<trading partner> -) <process>: <standard> document received` (bijv. X12/EDIFACT/HL7 — noem het standaardtype in de note) |

`*` Bij WSS is de HTTP-methode niet expliciet in de start-shape, maar volgt uit de Operation-component se `inputType` (`none` → GET, elk ander type → POST) en `operationType` (het werkwoord in het pad).

## 4. Shape-elementen → sequence diagram-concepten

| `shapetype` | Boomi-element | Sequence diagram-vertaling |
| --- | --- | --- |
| `connectoraction` (niet de start-shape) | Connector-call naar extern systeem (REST, Database (Legacy/V2), Disk V2, MFT, Mail, Salesforce, Boomi for SAP, custom SDK-connector, HTTP Client, OpenAPI, Event Streams Produce/Consume) | Synchrone call: `A ->>+ B: <actie> <resource/pad>` gevolgd door `B -->>- A: <resultaat/status>`. **Uitzondering:** Event Streams **Produce** is fire-and-forget: `A -) <topic>: publish <bericht>` (geen activatie, geen response-pijl) |
| `map` | Map (transformatie tussen profielen) | **Niet modelleren** als interactie — interne transformatie, geen communicatie tussen systemen |
| `decision` | Decision (if/then, binair) | `alt <conditie is waar> ... else <conditie is onwaar> ... end` — leid de conditietekst af uit de `comparison` + de twee `decisionvalue`'s (bijv. `comparison="equals"` tussen een DDP en een statische waarde → "Status equals 'ACTIVE'") |
| `route` | Route (switch/case, 3+ paden) | Eén `alt`/`else`-keten met één tak per `routevalue` (in XML-volgorde — dat is ook de evaluatievolgorde) plus een laatste `else` voor het Default-pad |
| `branch` | Branch (**sequentiële** multi-pad-splitsing — branches lopen nooit gelijktijdig) | Model als **opeenvolgende** `rect`-blokken, in de volgorde van de branch-nummers (1, 2, 3, ...) — gebruik hier **nooit** `par`, dat suggereert onterecht gelijktijdigheid |
| `catcherrors` | Try/Catch | `alt succes (Try-pad) ... else fout (Catch-pad) ... end`; de foutmelding komt uit `meta.base.catcherrorsmessage` — vermeld dit in een `note` op het fout-pad |
| `businessrules` | Business Rules (naam-gebaseerde validatie) | `alt Accepted ... else Rejected ... end`; noem in een `note` op het Rejected-pad kort welke regel(s) kunnen falen (uit de `<rule name="...">`-elementen), niet elke individuele conditie in detail |
| `processcall` | Process Call (subprocess-aanroep) | Als het subproces zelf externe communicatie bevat: haal de subproces-XML op en verwerk de interacties **genest** binnen dezelfde participant-lijn (`rect` binnen `rect`), of als eigen participant-lijn als het subproces echt een ander systeem representeert. Als het subproces puur interne logica is (routing, transformatie): **niet apart modelleren** — behandel het alsof de aanroepende lijn het zelf deed. Elk return path (`<returnpaths childShapeName="...">`) is een apart vervolgpad — modelleer als `alt`/`else` als de paden functioneel verschillen |
| `processroute` (Process Route) | Dynamische subprocess-selectie op basis van een route key, geëvalueerd tijdens runtime | `alt <route key = waarde 1>: roept <subprocess 1> aan ... else <route key = waarde 2>: roept <subprocess 2> aan ... end` — behandel elke subprocess-tak zoals bij `processcall` hierboven; vermeld dat de keuze dynamisch is (niet uit de XML zelf af te leiden welke waarde in productie voorkomt) |
| `documentproperties` (Set Properties) | DDP/DPP/MIME/connector-parameter zetten | **Niet modelleren** als interactie — interne staat/configuratie. Alleen relevant als een `note` als de gezette waarde functioneel betekenisvol is (bijv. een dynamisch endpoint-pad opbouwen) |
| `dataprocess` | Data Process (Custom Scripting, Search/Replace, Split/Combine, Base64, Zip/Unzip) | **Niet modelleren** als interactie — interne documentbewerking. **Uitzondering**: een Custom Scripting-stap (`processtype="12"`) die zelf een externe library/API aanroept vanuit het script — behandel die aanroep dan als een gewone synchrone/asynchrone call zoals bij `connectoraction` |
| `notify` | Notify (logging) | **Niet modelleren** — puur logging, geen communicatie tussen systemen |
| `exception` | Exception (terminale fout met bericht) | Response-pijl met het foutbericht als label, bijv. `B -->>- A: 500 <exception title/bericht>`, of als er geen aanroeper meer is op dat pad: een `note` die vermeldt dat het proces hier faalt met dat bericht |
| `stop` | Stop (stil terminaal einde, geen fout) | **Niet modelleren** als interactie — het pad eindigt hier gewoon. Vermeld eventueel in de functionele beschrijving dat verwerking hier stopt zonder verdere actie |
| `returndocuments` | Return Documents (terminaal, stuurt data terug) | De response-pijl terug naar de aanroeper/het ouderproces: `<process> -->>- <caller>: <resultaat>` — dit is typisch de laatste stap van de hoofdflow |
| `flowcontrol` met `forEachCount >= 1` | Flow Control — batching per document/batch | `loop <per document/batch van N> ... end` om de stappen die het beïnvloedt |
| `flowcontrol` met `chunks >= 2` | Flow Control — parallelle fibers (échte gelijktijdigheid) | `par <fiber 1> ... and <fiber 2> ... end` om de stappen die het beïnvloedt |
| `flowcontrol` met alle defaults (`chunks="0" forEachCount="0"`) | Flow Control — pass-through | **Niet modelleren** — functioneel onzichtbaar, gedraagt zich alsof de shape er niet stond |
| `tradingpartneraction` (Send) | Trading Partner Send (B2B/EDI-uitgaand) | Asynchroon fire-and-forget: `<process> -) <trading partner>: send <standard>-document`. Als het Errors-pad naar echte afhandeling leidt (niet alleen een Stop): `opt <uitgaande validatie mislukt> ... end` |
| Trading Partner Start-paden (Documents/Acknowledgments/Errors/Archive) | Inkomende EDI-routering | Documents = het hoofdpad (zie triggertabel sectie 3); Acknowledgments/Errors/Archive modelleer je als aparte `opt`-blokken alleen als de gebruiker daar expliciet naar vraagt — voor de hoofdflow is het Documents-pad meestal voldoende |
| Document Cache-stappen (Add/Retrieve/Remove) | In-memory cache binnen dezelfde executie | **Niet modelleren** als aparte participant — het is geen extern systeem. Alleen noemen in een `note` als de cache functioneel een lookup-stap vervangt die de lezer anders zou missen |
| `changedatacapture` (Find Changes / CDC) | Wijzigingsdetectie t.o.v. een eerdere run, drie gelabelde outputs | `alt Added ... else Updated ... else Deleted ... end` |

## 5. Endpoints en labels

Gebruik voor elke `connectoraction` de **daadwerkelijke** actie en het pad/resource uit de bijbehorende Operation-component (en Connection-component voor de systeemnaam), niet de rauwe connector-identifier:

- **REST/OpenAPI/HTTP Client**: `actionType` (GET/POST/PUT/...) + resource path uit de Operation, bijv. `sap ->>+ rest-api: POST /v1/orders`.
- **Web Services Server** (dit proces als API): `operationType` + `objectName` → `/ws/simple/<operationType><ObjectName>` (let op de sentence-case van `objectName` in het echte pad), of het curated pad uit de API Service Component als die het proces wrapt.
- **Database (Legacy/V2)**: de SQL-actie (GET/INSERT/UPDATE/DELETE, of de configured statement) als label, bijv. `process ->>+ postgres: SELECT orders WHERE status = 'NEW'`.
- **Salesforce / Boomi for SAP / custom SDK-connectors**: de geconfigureerde operatienaam/actie, bijv. `process ->>+ salesforce: query Account`.
- **Event Streams**: `publish <topic>` (Produce), `consume <topic>` (Consume), of "message received" (Listen).
- **Trading Partner**: het EDI-standaardtype (X12/EDIFACT/HL7/ODETTE/TRADACOMS) als onderdeel van het label.

## 6. Wat je NIET in het diagram zet

- **Set Properties** (`documentproperties`) — interne DDP/DPP/MIME-configuratie.
- **Map en Data Process** (behalve een externe call vanuit Custom Scripting) — interne transformatie/documentbewerking.
- **Notify** — logging, geen communicatie tussen systemen.
- **Document Cache** — in-memory structuur binnen dezelfde executie, geen extern systeem.
- **Stop** — stil terminaal einde van een pad, geen interactie.
- **Flow Control op de standaardinstellingen** (`chunks="0" forEachCount="0"`) — functioneel een no-op.

## 7. Onduidelijke of ontbrekende informatie

- Als een `connectoraction`'s `connectionId`/`operationId` niet is op te halen (verwijderd, geen toegang, of alleen de hoofdproces-XML is aangeleverd zonder afhankelijkheden), benoem dit expliciet als aanname in de functionele beschrijving en gebruik een generieke naam (bijv. "extern systeem (connectie niet beschikbaar)") in plaats van te gokken naar een systeemnaam.
- Als een `processcall` of `processroute` verwijst naar een subprocess dat niet is opgehaald, meld dat de interne werking van dat subprocess niet gedocumenteerd kon worden en vraag de gebruiker om het aan te leveren of toegang te geven, in plaats van aan te nemen dat het puur intern is.
- Als een `route`- of `processroute`-sleutel pas tijdens runtime bekend is (bijv. afkomstig uit een externe lookup), benoem dat de exacte gekozen tak niet uit de XML zelf is af te leiden.
- Bij een `decision`/`route`/`businessrules` met een numerieke of regex-vergelijking waarvan de functionele betekenis niet vanzelfsprekend is uit de veldnaam, vraag de gebruiker om bevestiging van de bedoelde functionele conditie in plaats van een letterlijke technische vertaling van de vergelijkingsoperator te geven.
