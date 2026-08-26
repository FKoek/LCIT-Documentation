# Frends-processen analyseren voor het Niveau 3 sequence diagram

Dit document beschrijft hoe je elementen uit een Frends-proces (JSON-export van een Process, of een C# Code Task) herkent en vertaalt naar het Niveau 3 Mermaid sequence diagram. Gebruik dit samen met `../../shared/standards.md`. Raadpleeg bij twijfel over Frends-specifieke concepten (Tasks, Agents, Environment Variables, expressies) ook de `fc-integration:frends-ipaas-developer`-skill, indien beschikbaar — een deel van de kennis in dit document (exportformaten, shape type-codes, trigger-taxonomie, het exclusive/inclusive-decision-onderscheid) is daaruit overgenomen omdat die skill dit rechtstreeks tegen echte Frends 6.2-exports heeft bevestigd.

## 0. Eerst: welk exportformaat is aangeleverd?

Frends kent **twee volledig verschillende exportformaten** — controleer dit vóórdat je begint met analyseren, anders werk je mogelijk met onvolledige informatie zonder dat te merken:

- **BPMN XML-export (`.bpmn`)** — bevat alleen het diagram: shapes, namen, sequence flows, lay-out. **Geen** Task-parameters, geen C#, geen expressies, geen trigger-configuratie. Dit is een documentatie-export.
- **Proprietary JSON-export (`.json`)** — bevat het volledige proces: het BPMN-diagram plús de C#-code en alle per-shape parameters. Dit is de enige variant met genoeg detail voor een betrouwbaar Niveau 3-diagram.

**Als je alleen een `.bpmn`-bestand krijgt aangeleverd**: meld expliciet aan de gebruiker dat dit alleen het diagramskelet bevat (geen endpoints, geen expressies, geen trigger-configuratie), en vraag om de JSON-export in plaats daarvan, of laat de gebruiker de ontbrekende functionele details zelf aanvullen. Ga niet door alsof de BPMN-XML evenveel informatie bevat als de JSON.

## 1. Waar te beginnen: de trigger

Zoek in de JSON-export naar `TriggersJson` (of, bij een BPMN-XML, het `startEvent`-element) om het triggertype te bepalen. Frends kent 11 triggertypes, elk met een eigen `$type`:

| Trigger | `$type` | Sync/async | Sequence diagram-vertaling |
| --- | --- | --- | --- |
| Manual | `ManualTrigger` | — (handmatig gestart) | Zelf-call zoals in sjabloon 5.1: `note over <proces>: Manual trigger` |
| Schedule | `ScheduleTrigger` | — (tijdgestuurd) | Self-call zoals in sjabloon 5.1: `note over <proces>: Scheduler` gevolgd door `<proces> ->>+ <proces>: Start processing` ... `<proces> ->>- <proces>: End processing` |
| File | `FileWatchTrigger` | asynchroon (event) | `<file-source> -) <frends-proces>: file detected` |
| Conditional | `ConditionalTrigger` | — (polling, vaak via een Subprocess) | Self-call, met een `note` die vermeldt welke conditie gepolld wordt |
| HTTP | `HttpTrigger` | synchroon | `<bron-systeem> ->>+ <frends-proces>: <METHOD> <path>` |
| API (OpenAPI-backed) | `HttpApiTrigger` | synchroon | Zelfde als HTTP, dit is de volledig API-managed variant — functioneel identiek voor het diagram, wel vermeldenswaardig in de functionele beschrijving als het onderscheid relevant is |
| AMQP / Queue | `QueueTrigger` | asynchroon | `<bron-systeem> -) <frends-proces>: <event>` |
| Service Bus | `ServiceBusTrigger` | asynchroon | `<bron-systeem> -) <frends-proces>: <event>` |
| RabbitMQ | `RabbitMQTrigger` | asynchroon | `<bron-systeem> -) <frends-proces>: <event>` |
| Azure Event Hub | `AzureEventHubTrigger` | asynchroon | `<bron-systeem> -) <frends-proces>: <event>` |
| TCP | `TcpTrigger` | — (raw connectie) | `<bron-systeem> ->>+ <frends-proces>: TCP connection` |

Alle vier de message-gebaseerde triggers (AMQP/Queue, Service Bus, RabbitMQ, Azure Event Hub) zijn asynchroon en worden hetzelfde gemodelleerd (`-)`), maar benoem in de functionele beschrijving wél welk specifiek platform het is — dat is relevant voor wie de integratie later moet onderhouden.

## 2. Participants herkennen

In Frends is elk **Process** (of duidelijk afgebakende Subprocess die naar een ander extern systeem praat) een participant. Herken:

- Het aanroepende/bron-systeem (van de Trigger).
- Het Frends-proces zelf (of de relevante Subprocess als het proces uit meerdere logisch gescheiden Subprocessen bestaat — behandel elke Subprocess als eigen participant als dat de leesbaarheid verbetert, anders als interne stap binnen dezelfde participant).
- Elk extern doelsysteem waarmee een Task communiceert (SAP, een REST-API, een database, een file-share, etc.).

Gebruik dezelfde afkortingsconventie als bij Mulesoft waar van toepassing (bijv. `-api` suffix voor een API-laag), of de bestaande naamgeving uit de Frends Environment als die er is.

## 3. Frends-elementen → sequence diagram-concepten

| Frends-element | Sequence diagram-vertaling |
| --- | --- |
| **HTTP Request Task** naar extern systeem/API | Synchrone call: `A ->>+ B: <METHOD> <path>` gevolgd door `B -->>- A: <status>` |
| **Subprocess-aanroep (Call Subprocess)** die zelf externe calls doet | Eigen participant-lijn, of geneste `rect`-groepering binnen de aanroepende stap (zie het geneste voorbeeld in de standards, sectie 8) |
| **Exclusive Decision** (precies één tak vuurt — de klassieke if/else) | `alt <conditie 1> ... else <conditie 2> ... end`. Dit is de enige Decision-vorm die 1-op-1 op `alt`/`else` past. |
| **Inclusive Decision** (elke tak met een eigen `True`-conditie vuurt — dus mogelijk meerdere tegelijk) | **Niet** als `alt`/`else` modelleren — dat suggereert ten onrechte precies één pad. Modelleer in plaats daarvan elke tak als eigen `opt <conditie>`-blok, zodat het diagram klopt met het feit dat 0, 1, of meerdere takken kunnen uitvoeren. |
| **Loop / Foreach / While-element** | `loop <omschrijving iteratie> ... end` |
| **Parallel/Concurrent execution** | `par <flow 1> ... and <flow 2> ... end` |
| **Scope-and-Catch** (try/catch rond een deel van de flow) | `alt succes ... else fout (<statuscode>) ... end` |
| **Queue Task / message-trigger-vervolg** (verzenden naar/lezen van een message queue) | Asynchrone message: `A -) B: <event>` |
| **Code Task (C#)** die alleen data transformeert/mapt (geen externe call) | **Niet modelleren** als interactie — interne verwerking. Alleen relevant als het een functionele beslissing/validatie betreft; modelleer dat dan als `opt` of `note`, niet als call naar een participant. |
| **Code Task (C#)** die wél een externe library/API aanroept (bijv. een custom NuGet Task die naar een extern systeem praat) | Wel als synchrone/asynchrone call modelleren, zoals hierboven — behandel de aanroep zoals een normale Task-call |

## 4. Shape Type-codes voor betrouwbare herkenning (alleen bij JSON-export)

In de JSON-export heeft elke `ElementParameters`-entry een integer `Type`-veld — gebruik dit voor betrouwbare, programmatische herkenning in plaats van alleen op naam/omschrijving te gokken:

| `Type` | Shape | Relevant voor het diagram |
| --- | --- | --- |
| 0 | Trigger | Zie de trigger-tabel in sectie 1 (`SelectedTypeId` = het trigger-`$type`) |
| 1 | Task | Meestal een externe call — zie sectie 3 |
| 2 | Exclusive Decision | → `alt`/`else` |
| 7 | Call Subprocess | Eigen participant-lijn of geneste `rect` |
| 8 | Scope (embedded, zonder foreach/while) | Groepering, geen apart diagram-concept tenzij gecombineerd met Catch (14) → dan `alt succes/fout` |
| 10 | Foreach | → `loop` |
| 11 | While | → `loop` |
| 12 | Code Task | Zie sectie 3 (alleen modelleren bij externe call) |
| 14 | Catch (hoort bij een Scope, Type 8) | Samen met de bijbehorende Scope → `alt succes ... else fout ... end` |

**Let op:** de `Type`-code voor Inclusive Decision (`inclusiveGateway`) is niet bevestigd in de brondata van deze tabel — herken die daarom op het BPMN-element `inclusiveGateway` zelf, niet op een specifiek `Type`-nummer, en valideer bij twijfel tegen de export.

## 5. Endpoints en labels

Gebruik de daadwerkelijke HTTP-method + path (of het queue-/topic-naam voor async) uit de Task-configuratie:

```
frends_proces ->>+ sap_api: POST /api/v1/orders
sap_api -->>- frends_proces: 200 OK
```

## 6. Wat je NIET in het diagram zet

- Environment Variable-lookups en configuratie.
- Logging-Tasks, tenzij expliciet gevraagd.
- Pure data-mapping/transform Code Tasks zonder externe communicatie.

## 7. Onduidelijke of ontbrekende informatie

Als een Task een dynamisch URL-adres gebruikt (samengesteld uit variabelen) of de doel-API niet eenduidig uit de export blijkt, benoem dit als aanname in de functionele beschrijving en vraag de gebruiker om bevestiging in plaats van te gokken. Hetzelfde geldt als je een Inclusive Decision (`inclusiveGateway`) tegenkomt waarvan de `Type`-code niet met zekerheid uit sectie 4 is af te leiden — herken 'm dan op het BPMN-element, en meld in de functionele beschrijving dat dit een Inclusive (niet Exclusive) Decision is, zodat de lezer weet dat meerdere takken tegelijk kunnen vuren.
