# Mulesoft-flows analyseren voor het Niveau 3 sequence diagram

Dit document beschrijft hoe je elementen uit een Mulesoft-flow (XML) herkent en vertaalt naar het Niveau 3 Mermaid sequence diagram. Gebruik dit samen met `../../shared/standards.md`.

## 1. Waar te beginnen

Zoek naar het startpunt van de flow:

- `<http:listener .../>` binnen een `<flow>` → dit is de **start interactie**. De `path` en HTTP-method (`config-ref`/`allowedMethods`) worden het endpoint in de eerste message.
- `<scheduler>` of `<ee:transform>` triggers zonder inkomend HTTP-verzoek → modelleer als self-call: `note over X: Scheduler` gevolgd door `X ->>+ X: Start processing` ... `X ->>- X: End processing` (zie sjabloon 5.1 in de standards).
- `<jms:listener>`, `<amqp:listener>`, `<vm:listener>` → asynchrone trigger, gebruik `-)` notatie.

## 2. Participants herkennen

Mulesoft-projecten volgen meestal het drie-lagen API-model (zie Niveau 2-diagram):

| Laag | Herkenning in code | Afkorting |
| --- | --- | --- |
| Experience API | project-naam eindigt vaak op `-ea` of heeft consument-gerichte endpoints | `<naam>-ea` |
| Process API | bevat business-logica, orchestreert meerdere System APIs | `<naam>-pa` |
| System API | rechtstreeks contact met bron-/doelsysteem (SAP, Salesforce, DB) | `<naam>-sa` |

Elke aparte Mulesoft-applicatie (elk `.xml`-project of elke duidelijk gescheiden API) wordt een eigen `participant`. Het achterliggende systeem waar de System API mee praat (SAP, WMS, SugarCRM, etc.) is ook een eigen participant.

Het gedeelde skelet (`../../shared/assets/example-skeleton.mmd`) gebruikt bewust platform-neutrale placeholders (`<intermediate-system-1>`, `<intermediate-system-2>`) omdat het ook door `frends-documentation-skill` wordt gebruikt. Vul die placeholders voor een Mulesoft-integratie in volgens de laag-conventie hierboven, bijv. `<intermediate-system-1>` → `sap_sa` (System API), `<intermediate-system-2>` → `sap_pa` (Process API).

### Meerdere instanties van dezelfde laag samenvatten in één participant

Bij een groot landschap kunnen meerdere, verschillende API's van dezelfde laag naar hetzelfde component roepen — bijv. drie losse Experience API's (`sap-ea`, `crm-ea`, `wms-ea`) die allemaal dezelfde Process API aanroepen, en het voor de leesbaarheid van het diagram beter is om die drie samen te vatten tot één participant.

**Verzin in dat geval geen samengevoegde naam die suggereert dat het één specifieke, benoembare API is** (bijv. `calling-ea`) — dat wekt ten onrechte de indruk dat er één concrete Experience API bestaat met die naam. Noem de participant in plaats daarvan naar het **type/de laag** die hij representeert: `experience-api` (of leesbaar: "Experience API"). Zo is voor iedereen die het diagram leest meteen duidelijk dat dit een generieke, samengevatte weergave van de laag is — geen specifieke, echt bestaande component. Vermeld in een `note` of in de functionele beschrijving welke concrete API's precies samengevat zijn, zodat die informatie niet verloren gaat.

Dit geldt evengoed voor Process API's (`process-api`) en System API's (`system-api`) als die op dezelfde manier worden samengevat.

## 3. Flow-elementen → sequence diagram-concepten

| Mulesoft-element | Sequence diagram-vertaling |
| --- | --- |
| `<http:request>` naar een andere Mulesoft-API of extern systeem | Synchrone call: `A ->>+ B: <METHOD> <path>` gevolgd door `B -->>- A: <status>` |
| `<flow-ref>` naar een sub-flow binnen dezelfde API | **Niet** als aparte participant-interactie tonen, tenzij de sub-flow zelf externe calls doet — dan de calls van de sub-flow gewoon in dezelfde participant-lijn tonen |
| `<choice>` met meerdere `<when>` + `<otherwise>` | `alt <conditie 1> ... else <conditie 2> ... end` |
| `<foreach>` | `loop <omschrijving iteratie> ... end` |
| `<scatter-gather>` | `par <flow 1> ... and <flow 2> ... end` |
| `<try>` + `<error-handler>` / `<on-error-continue>` / `<on-error-propagate>` | `alt succes ... else fout (<statuscode>) ... end` |
| `<async>` scope | Asynchrone message: `A -) B: <event>` (geen activatie, geen response-pijl) |
| `<ee:transform>` / DataWeave / `<set-variable>` / `<set-payload>` | **Niet modelleren** als interactie — dit is interne verwerking, geen communicatie tussen systemen. Eventueel toelichten in een `note` als het functioneel relevant is. |
| `raise-error` / expliciete foutresponses | Response-pijl met het statuscode-label, bijv. `B -->>- A: 400 Bad Request` |

## 4. Endpoints en labels

Gebruik de daadwerkelijke HTTP-method + path uit de `<http:listener>`/`<http:request>`-configuratie als label, bijv.:

```
sugarcrm_ea ->>+ customer_pa: POST /internal/v1/customers/create-customer
```

Voor SOAP-based System APIs (bijv. SAP RFC/BAPI via SOAP) gebruik het volledige service-pad zoals in het originele voorbeeld in de standards (sectie 8).

## 5. Wat je NIET in het diagram zet

- Logging-componenten (Splunk, CloudHub logging) — tenzij expliciet gevraagd door de gebruiker.
- Interne caching/object-store lookups, tenzij deze een functioneel relevante beslissing beïnvloeden (dan als `opt` of `note`).
- Configuratie- en property-bestanden.

## 6. Onduidelijke of ontbrekende informatie

Als de flow-XML een `<http:request>` bevat zonder duidelijke doel-API (bijv. een dynamische URL uit een variabele), benoem dit expliciet in de functionele beschrijving als aanname en vraag de gebruiker om bevestiging in plaats van te gokken.
