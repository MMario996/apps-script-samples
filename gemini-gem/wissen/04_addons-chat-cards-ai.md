# Add-ons, Chat-Apps, Karten-UI, Custom Functions und KI (Gemini/Vertex)

## 1. Add-on-Typen im Überblick

| Typ | UI | Läuft in | Doku |
|---|---|---|---|
| **Google Workspace-Add-on** | `CardService` (Karten) | Gmail, Calendar, Drive, Docs, Sheets, Slides, Chat (gleiche Codebasis) | developers.google.com/workspace/add-ons |
| **Editor-Add-on** | HtmlService (Menü, Sidebar, Dialog) | Docs, Sheets, Slides, Forms | developers.google.com/workspace/add-ons/editors |
| **Meet-Add-on** | Web-SDK (eigenes Hosting) | Meet | Kein Apps Script |

Samples: https://developers.google.com/workspace/add-ons/samples (u. a. **Travel Concierge**: ein KI-Agent als Workspace-Add-on, der Kontext aus Gmail, Calendar und Docs nutzt, mit Gemini/Vertex AI und Agent Development Kit (ADK) im Backend; Muster: Add-on-UI in Apps Script, Agent auf Vertex AI Agent Engine, Aufruf per `UrlFetchApp`).

## 2. Workspace-Add-on: Manifest und Einstiegspunkte

```json
{
  "timeZone": "Europe/Berlin",
  "runtimeVersion": "V8",
  "oauthScopes": [
    "https://www.googleapis.com/auth/gmail.addons.execute",
    "https://www.googleapis.com/auth/gmail.addons.current.message.readonly",
    "https://www.googleapis.com/auth/script.locale"
  ],
  "addOns": {
    "common": {
      "name": "Mein Helfer",
      "logoUrl": "https://www.gstatic.com/images/icons/material/system/1x/label_googblue_48dp.png",
      "homepageTrigger": { "runFunction": "onHomepage" },
      "universalActions": [{ "label": "Hilfe", "openLink": "https://example.com/hilfe" }]
    },
    "gmail": {
      "contextualTriggers": [{ "unconditional": {}, "onTriggerFunction": "onGmailMessage" }],
      "composeTrigger": { "selectActions": [{ "text": "Einfügen", "runFunction": "onCompose" }], "draftAccess": "METADATA" }
    },
    "calendar": { "eventOpenTrigger": { "runFunction": "onEventOpen" } },
    "drive": { "onItemsSelectedTrigger": { "runFunction": "onDriveItemsSelected" } },
    "sheets": { "homepageTrigger": { "runFunction": "onHomepage" } },
    "docs": {
      "homepageTrigger": { "runFunction": "onHomepage" },
      "linkPreviewTriggers": [{
        "runFunction": "onLinkPreview",
        "patterns": [{ "hostPattern": "example.com", "pathPrefix": "items" }],
        "labelText": "Item", "logoUrl": "https://example.com/logo.png"
      }]
    }
  }
}
```
- **Smart Chips / Link-Vorschau** (`linkPreviewTriggers`) und das **Erstellen von Ressourcen** (`createActionTriggers`) in Docs, Sheets und Slides. Sample im Repo: `solutions/add-on/book-smartchip`, `ai/devdocs-link-preview`.
- Event-Objekt: `e.commonEventObject` (`hostApp`, `platform`, `formInputs`, `parameters`), `e.gmail.messageId` + `e.gmail.accessToken` (Gmail: `GmailApp.setCurrentMessageAccessToken(e.gmail.accessToken)`), `e.calendar`, `e.drive.activeCursorItem`, `e.docs`/`e.sheets`.
- Testen: „Bereitstellen“ → „Bereitstellungen testen“ → „Installieren“. Veröffentlichen: GCP-Projekt, OAuth-Prüfung, Marketplace SDK.

## 3. CardService – Muster

```javascript
function onHomepage(e) {
  return buildCard_("Willkommen!");
}

function buildCard_(message) {
  const input = CardService.newTextInput().setFieldName("query").setTitle("Suchbegriff");
  const action = CardService.newAction().setFunctionName("onSearch").setParameters({ source: "home" });
  const button = CardService.newTextButton().setText("Suchen").setOnClickAction(action)
    .setTextButtonStyle(CardService.TextButtonStyle.FILLED);

  const section = CardService.newCardSection()
    .addWidget(CardService.newDecoratedText().setText(message).setWrapText(true))
    .addWidget(input)
    .addWidget(CardService.newButtonSet().addButton(button));

  return CardService.newCardBuilder()
    .setHeader(CardService.newCardHeader().setTitle("Mein Helfer"))
    .addSection(section)
    .build();
}

function onSearch(e) {
  const query = e.commonEventObject.formInputs?.query?.stringInputs?.value?.[0] ?? "";
  const card = buildCard_(`Ergebnis für „${query}“`);
  return CardService.newActionResponseBuilder()
    .setNavigation(CardService.newNavigation().pushCard(card)) // or updateCard / popCard
    .setNotification(CardService.newNotification().setText("Fertig"))
    .build();
}
```
- Widgets: `TextParagraph` (einfaches HTML wie `<b>`, `<a>`, `<font color>`), `DecoratedText`, `TextInput`, `SelectionInput` (Dropdown, Checkbox, Radio, Switch, Multi-Select), `DateTimePicker`, `Image`, `Grid`, `Columns`, `ButtonSet`, `Divider`, `CollapseControl`, `Chip`.
- Limits: etwa 100 Widgets pro Karte. Antwortzeit unter 30 s. Kein eigenes CSS oder JS.
- Prototypen per Drag & Drop: **Card Builder** (https://addons.gsuite.google.com/uikit/builder).

## 4. Chat-Apps mit Apps Script

Doku: https://developers.google.com/workspace/chat/overview · Quickstart: https://developers.google.com/workspace/chat/quickstart/apps-script-app

Variante **„als Workspace-Add-on“** (aktueller Weg). Manifest:
```json
{ "addOns": { "common": { "name": "Mein Bot", "logoUrl": "https://…" }, "chat": {} } }
```
```javascript
function onMessage(e) {
  const { message } = e.chat.messagePayload;
  return {
    hostAppDataAction: { chatDataAction: { createMessageAction: { message: {
      text: `Du hast geschrieben: ${message.text}`,
    } } } },
  };
}
function onAddedToSpace(e) { /* greeting */ }
function onAppCommand(e) { /* slash/quick commands: e.chat.appCommandPayload.appCommandMetadata.appCommandId */ }
```
Variante **„klassisch“** (Chat-API-Konfiguration → Apps-Script-Projekt): `onMessage(event)` gibt `{ text }` oder `{ cardsV2: [{ cardId, card }] }` zurück; dazu `onAddToSpace`, `onRemoveFromSpace`, `onCardClick`. Slash-Befehle: `event.message.slashCommand.commandId`.

- Konfiguration in der GCP-Konsole: **Google Chat API** aktivieren → Konfiguration → App-Name, Avatar, Funktionen (1:1, Spaces), Verbindung „Apps Script“ + **Bereitstellungs-ID** (Head-Deployment-ID unter „Bereitstellungen testen“), Slash-Befehle, Sichtbarkeit.
- Asynchrone Nachrichten (z. B. per Zeittrigger) laufen über den Advanced Service `Chat` (Nutzer-Auth) oder per App-Auth mit Dienstkonto.
- Cards v2 in Chat als JSON: `{ header: { title }, sections: [{ widgets: [{ textParagraph: { text } }, { buttonList: { buttons: [{ text, onClick: { action: { function: "fn" } } }] } }] }] }`.
- Dialoge: `actionResponse: { type: "DIALOG", dialogAction: { dialog: { body: card } } }`.
- Samples: `chat/`, `ai/standup-chat-app`, `solutions/ooo-assistant`, `solutions/webhook-chat-app`.

## 5. Editor-Add-ons (klassisch)
- `onOpen(e)` + `onInstall(e) { onOpen(e); }`. Menü über `SpreadsheetApp.getUi().createAddonMenu()`.
- `e.authMode`: `NONE` (vor der Autorisierung – nur Menü bauen, nichts lesen!), `LIMITED`, `FULL`.
- UI über HtmlService (siehe Datei 03). Sample: `solutions/editor-add-on/clean-sheet`, `templates/sheets-addon`, `templates/docs-addon`, `templates/forms-addon`.

## 6. Custom Functions (Sheets)

```javascript
/**
 * Converts a net amount to gross.
 * @param {number|number[][]} net Net amount or range.
 * @param {number} [rate=0.19] VAT rate.
 * @return {number|number[][]} Gross amount.
 * @customfunction
 */
function BRUTTO(net, rate = 0.19) {
  const calc = (v) => (typeof v === "number" ? Math.round(v * (1 + rate) * 100) / 100 : "");
  return Array.isArray(net) ? net.map((row) => row.map(calc)) : calc(net);
}
```
- Name in Großbuchstaben (Konvention). Keine Funktionen mit `_` am Ende (nicht sichtbar). **30 s Limit**, **keine Dienste mit Nutzer-Auth**, keine Seiteneffekte (nicht in andere Zellen schreiben).
- Werden neu berechnet, wenn sich die Argumente ändern. Bei `UrlFetchApp` daher `CacheService` nutzen.
- Ganze Spalten effizient verarbeiten: Bereich übergeben und ein 2D-Array zurückgeben, statt die Funktion in 10 000 Zellen zu kopieren.
- Samples: `solutions/custom-functions/*`, `sheets/customFunctions`, `ai/custom-func-ai-studio`, `ai/custom_func_vertex`, `ai/custom-func-ai-agent`. **Fact-Check-Sample** (https://developers.google.com/apps-script/samples/custom-functions/fact-check): `=FACT_CHECK(aussage)` ruft einen Gemini-basierten ADK-Agenten auf Vertex AI auf und gibt eine Bewertung zurück.

## 7. KI in Apps Script: Gemini API und Vertex AI

### A) Gemini API (Google AI Studio, API-Key) – am einfachsten
```javascript
const GEMINI_MODEL = "gemini-2.5-flash"; // check current model names in the docs

function askGemini(prompt) {
  const apiKey = PropertiesService.getScriptProperties().getProperty("GEMINI_API_KEY");
  const url = `https://generativelanguage.googleapis.com/v1beta/models/${GEMINI_MODEL}:generateContent`;
  const res = UrlFetchApp.fetch(url, {
    method: "post",
    contentType: "application/json",
    headers: { "x-goog-api-key": apiKey },
    payload: JSON.stringify({
      contents: [{ role: "user", parts: [{ text: prompt }] }],
      generationConfig: { temperature: 0.2, responseMimeType: "application/json" },
    }),
    muteHttpExceptions: true,
  });
  if (res.getResponseCode() !== 200) throw new Error(res.getContentText());
  return JSON.parse(res.getContentText()).candidates[0].content.parts[0].text;
}
```
- Strukturierte Ausgabe: `responseMimeType: "application/json"` + `responseSchema`.
- Dateien und Bilder: `parts: [{ inlineData: { mimeType, data: Utilities.base64Encode(blob.getBytes()) } }]`.

### B) Vertex AI (GCP-Projekt, ohne API-Key)
- Endpunkt: `https://{LOCATION}-aiplatform.googleapis.com/v1/projects/{PROJECT}/locations/{LOCATION}/publishers/google/models/{MODEL}:generateContent` (für `global` lautet der Host `aiplatform.googleapis.com`).
- Auth: `ScriptApp.getOAuthToken()` mit Scope `https://www.googleapis.com/auth/cloud-platform`. Das Skript ist mit dem GCP-Projekt verknüpft, die Vertex AI API ist aktiviert, und der Nutzer hat die Rolle „Vertex AI User“. Alternativ ein Dienstkonto über die OAuth2-Bibliothek (wie im Repo-Sample `ai/custom_func_vertex`, das in Custom Functions nötig ist, weil diese kein Nutzer-OAuth haben).
- Repo-Samples: `ai/autosummarize`, `ai/email-classifier`, `ai/gmail-sentiment-analysis`, `ai/drive-rename`, `ai/standup-chat-app`, `ai/devdocs-link-preview`, `solutions/automations/feedback-sentiment-analysis`, `solutions/automations/news-sentiment`.

### Muster „Gmail-Klassifizierer“
Zeittrigger → `GmailApp.search('is:unread -label:ai-processed newer_than:1d')` → pro Mail den Prompt mit Betreff und Snippet an Gemini schicken (JSON-Antwort mit Kategorie) → Label setzen, ggf. Entwurf erstellen → Label `ai-processed`. Laufzeit und Kosten über eine maximale Anzahl Mails pro Lauf begrenzen.
