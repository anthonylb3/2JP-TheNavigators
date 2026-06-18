# README — E-mail AI Pipeline (basisflow)

Privacy-first n8n-workflow die binnenkomende e-mails automatisch classificeert, een conceptantwoord schrijft op basis van een lokale kennisbank, en dat concept als Gmail-draft klaarzet voor menselijke review. Alle AI-verwerking gebeurt lokaal via Ollama — er gaat geen inhoud van mails naar een externe API.

**Scope van dit document:** de basisflow zoals in de flowchart. Clickup, Salesforce en de Audit trail-uitbreiding (Google Sheets logging) staan los van deze flow en worden hier niet behandeld.

---

## 1. Wat doet deze workflow

Er zijn twee afzonderlijke n8n-workflows:

### Workflow A — Kennisbank laden
Leest `kennisbank.txt`, knipt de tekst in stukken (chunks), zet die om naar vectoren (embeddings) en slaat ze op in Qdrant. Dit is de "geheugen"-laag waar Agent 2 straks in zoekt om relevante informatie te vinden bij het schrijven van een antwoord.

### Workflow B — Classificatie en draft (de hoofdflow)
```
Gmail Trigger
  → AI Agent 1 (classificatie)
    → Structured Output Parser
      → Formatting (Code in JavaScript)
        → AI Agent 2 (schrijft concept-antwoord, zoekt in kennisbank via Qdrant)
          → Create a Draft (Gmail)
```

Een binnenkomende mail wordt automatisch geclassificeerd, Agent 2 zoekt relevante context in de kennisbank en schrijft een conceptantwoord, en dat concept verschijnt als draft in Gmail — klaar om door een mens gecontroleerd en verstuurd te worden. Er is bewust geen automatisch verzenden: **de Gmail-draft is de menselijke reviewlaag.**

---

## 2. Voorwaarden

Voordat je begint, moet aanwezig zijn:

- **Docker** met n8n, Ollama en Qdrant draaiend (lokaal of op een server — deze README gaat over de workflow zelf, niet over waar hij draait)
- Een **Google-account** dat als afzender/postvak fungeert (bijv. `tproject095@gmail.com`), met Gmail API, Google Drive API en Google Sheets API enabled in Google Cloud
- Een **OAuth2-credential** in n8n voor dat Google-account
- De Ollama-modellen lokaal gepulled:
  - `gemma4:latest` (classificatie, Agent 1)
  - `llama3.1:8b` (draft, Agent 2)
  - `nomic-embed-text` (embeddings, gebruikt door beide workflows)
- Een `kennisbank.txt`-bestand met de inhoud waar Agent 2 op moet kunnen terugvallen

---

## 3. Workflow A — Kennisbank laden: node voor node

| Node | Wat het doet | Instellingen |
|---|---|---|
| **Execute workflow** (trigger) | Handmatige trigger om de kennisbank opnieuw in te laden, bijvoorbeeld na een wijziging in `kennisbank.txt` | Geen extra instellingen; klik op "Execute Workflow" om te starten |
| **Read/Write Files from Disk** | Leest `kennisbank.txt` van de lokale schijf en geeft de inhoud door | Pad naar het bestand, bijv. `/home/node/.n8n-files/kennisbank.txt` |
| **Default Data Loader** | Splitst de ruwe tekst op in kleinere stukken (chunks) die Qdrant als losse vectoren kan opslaan | Standaardinstellingen werken meestal goed; chunk-grootte kun je aanpassen als stukken te groot/klein blijken |
| **Embeddings Ollama** | Zet de tekstinhoud om naar vectoren via het `nomic-embed-text`-model, zodat Qdrant erop kan zoeken | Base URL naar je Ollama-instantie; model = `nomic-embed-text` |
| **Qdrant Vector Store** | Slaat de vectoren op in een collectie. Bij elke uitvoering wordt de bestaande collectie overschreven met de nieuwe inhoud van de kennisbank | Collectienaam invoeren via **"By ID"** als de collectie nog niet bestaat; daarna kun je 'm selecteren |

**Wanneer opnieuw draaien:** elke keer dat `kennisbank.txt` wijzigt. De workflow overschrijft de bestaande collectie volledig, dus oude content verdwijnt automatisch mee.

---

## 4. Workflow B — Classificatie en draft: node voor node

### Gmail Trigger
Vangt nieuwe binnenkomende mails op in het gekoppelde Gmail-account.
- **Credential:** Gmail OAuth2 API, gekoppeld aan het dedicated Google-account
- **Poll-interval:** instelbaar; hoe vaak n8n checkt op nieuwe mail

### AI Agent 1 — classificatie
Leest de inhoud van de binnenkomende mail en bepaalt de categorie (bijv. type vraag, urgentie, vervolgactie).
- **Chat Model:** Ollama Chat Model, gekoppeld aan `gemma4:latest`
- **Memory:** indien gebruikt, voor context tussen berichten in dezelfde conversatie
- **Output Parser:** Structured Output Parser — dwingt de output van Agent 1 in een vast formaat (JSON-achtige structuur) zodat de vervolgstappen daar betrouwbaar mee kunnen werken

### Structured Output Parser
Zet de vrije tekst-output van Agent 1 om naar een gestructureerd format (bijv. velden als categorie, samenvatting, urgentie) dat de Formatting-node verder kan verwerken.

### Formatting — Code in JavaScript
Verwerkt de classificatie-output naar het formaat dat Agent 2 nodig heeft als input. Hier kun je bijvoorbeeld velden hernoemen, samenvoegen, of de structuur aanpassen aan wat de prompt van Agent 2 verwacht.
- **Type:** Code-node, JavaScript
- Aanpassen als je de output van Agent 1 wijzigt of als Agent 2's prompt andere velden verwacht

### AI Agent 2 — conceptantwoord schrijven
Schrijft een conceptmail op basis van de classificatie én relevante informatie uit de kennisbank.
- **Ollama Chat Model:** gekoppeld aan `llama3.1:8b`, stuurt Agent 2 aan om tekst te genereren
- **Qdrant Vector Store (zoeken):** zoekt relevante informatie in de kennisbank op basis van de inhoud van de binnenkomende mail
- **Embeddings Ollama:** zet de zoekopdracht om naar vectoren zodat Qdrant ermee kan zoeken (`nomic-embed-text`)
- **Systeemprompt:** let op natuurlijke taal — bij dit project is een taalkwaliteitsfix toegepast in de systeemprompt om onnatuurlijke (te letterlijk vertaalde) formuleringen te voorkomen. Pas dit aan op de gewenste schrijfstijl en taal.

### Create a Draft (Gmail)
Slaat het gegenereerde antwoord op als concept in Gmail, klaar voor menselijke review voordat het verzonden wordt.
- **Credential:** dezelfde Gmail OAuth2-credential als de Trigger
- **Belangrijk:** dit is bewust het eindpunt van de basisflow. Er is geen aparte reviewstap nodig naast deze draft — het concept-mechanisme van Gmail vervult die rol al.

---

## 5. Bekende valkuilen en oplossingen

| Probleem | Oorzaak | Oplossing |
|---|---|---|
| n8n kan Ollama/Qdrant niet bereiken via `localhost` | In Docker draait n8n in een eigen netwerk-namespace | Gebruik `http://host.docker.internal:11434` (Ollama) en `http://host.docker.internal:6333` (Qdrant) als Ollama/Qdrant op de Docker-host draaien; gebruik de servicenaam (bijv. `http://ollama:11434`) als ze in dezelfde Docker Compose-stack draaien |
| "OAuth client was not found" | Client ID in n8n bevat een `http://`-prefix | Voer de Client ID als bare string in, zonder protocol-prefix |
| Gmail-redirect werkt niet | Redirect URI in Google Cloud komt niet exact overeen met wat n8n gebruikt | Redirect URI blijft `http://localhost:5678/rest/oauth2-credential/callback` vanuit Google's perspectief — de browser handelt de redirect af, niet de container |
| Qdrant Vector Store-node geeft een foutmelding bij eerste gebruik | Collectie bestaat nog niet | Gebruik **"By ID"**-mode om de collectienaam handmatig in te voeren |
| Ollama-model niet gevonden | Model is op de hostmachine gepulled, niet in de container | Pull modellen altijd via `docker exec -it ollama ollama pull [model]` |
| Gmail API / Drive API / Sheets API errors | Eén of meer API's niet enabled in Google Cloud | Zorg dat Gmail API, Google Drive API én Google Sheets API allemaal expliciet enabled staan, ook al gebruik je Sheets nog niet — scheelt later |

---

## 6. Testen

1. Draai **Workflow A** (kennisbank) handmatig en controleer of er vectoren in Qdrant terechtkomen.
2. Activeer **Workflow B** en stuur een testmail naar het gekoppelde Gmail-account.
3. Controleer in de n8n-executielog:
   - Komt de trigger binnen?
   - Classificeert Agent 1 zoals verwacht? (bekijk de output van de Structured Output Parser)
   - Vindt Agent 2 relevante kennisbank-context?
   - Staat er een concept-draft in Gmail?

---

## 7. Wat hier niet in zit

Buiten scope van deze README, en losstaand van deze basisflow:
- Clickup-integratie (task routing, urgent/normaal)
- Salesforce-integratie (contacten, cases, account-updates)
- Audit trail / logging naar Google Sheets
- Productie-deployment (server, Docker Compose-stack, uptime)
