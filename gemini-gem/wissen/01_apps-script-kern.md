# Apps Script – Kernwissen

Quelle: https://developers.google.com/apps-script · Referenz: https://developers.google.com/apps-script/reference
Stand dieser Notiz: 2026. Quotas und Preview-Features vor dem Einsatz in der offiziellen Doku gegenprüfen.

## 1. Was ist Apps Script?

- Cloudbasierte JavaScript-Plattform (V8-Runtime, ECMAScript 2015+ inkl. Klassen, async-Syntax wird geparst, aber **es gibt keine echte Parallelität und kein Event-Loop-I/O**: alle Dienstaufrufe sind synchron; `Promise`/`async` bringen keinen Geschwindigkeitsvorteil).
- Kein Node.js: **kein `require`, kein `fetch`, kein `setTimeout`, kein `window`/`document`** auf dem Server. Stattdessen `UrlFetchApp`, `Utilities.sleep`, `HtmlService` für die UI.
- Dateitypen: `.gs` (Server-Code) und `.html` (Client-Templates). Alle `.gs`-Dateien teilen **einen globalen Namensraum**. Die Ladereihenfolge entspricht der Dateireihenfolge im Editor; Code auf oberster Ebene läuft **bei jedem** Aufruf.
- Projektarten:
  - **Container-gebunden** (an Sheet, Doc, Slides, Form): `SpreadsheetApp.getActiveSpreadsheet()`, `getUi()`, einfache Trigger wie `onOpen` und `onEdit`.
  - **Standalone** (eigene Datei in Drive): für Web-Apps, Bibliotheken, Workspace-Add-ons, Chat-Apps und Automationen.
- Bereitstellungsarten: Web-App, API-Ausführbare (Apps Script API `scripts.run`), Add-on (Editor / Workspace), Bibliothek, Chat-App.

## 2. Manifest `appsscript.json`

Sichtbar machen: Editor → Projekteinstellungen → „Manifestdatei im Editor anzeigen“.

```json
{
  "timeZone": "Europe/Berlin",
  "runtimeVersion": "V8",
  "exceptionLogging": "STACKDRIVER",
  "oauthScopes": [
    "https://www.googleapis.com/auth/spreadsheets.currentonly",
    "https://www.googleapis.com/auth/script.external_request",
    "https://www.googleapis.com/auth/script.scriptapp"
  ],
  "dependencies": {
    "enabledAdvancedServices": [
      { "userSymbol": "Drive", "serviceId": "drive", "version": "v3" },
      { "userSymbol": "Sheets", "serviceId": "sheets", "version": "v4" }
    ],
    "libraries": [
      { "userSymbol": "OAuth2", "libraryId": "1B7FSrk5Zi6L1rSxxTDgDEUsPzlukDsi4KGuTMorsTQHhGBzBkMun4iDF", "version": "43" }
    ]
  },
  "webapp": { "executeAs": "USER_DEPLOYING", "access": "ANYONE_ANONYMOUS" },
  "urlFetchWhitelist": ["https://api.example.com/"]
}
```

- **Scopes minimieren**: z. B. `spreadsheets.currentonly` statt `spreadsheets`, `drive.file` statt `drive`, `gmail.send` statt `mail.google.com`. Sind `oauthScopes` gesetzt, werden sie nicht mehr automatisch ermittelt, also **alle** benötigten eintragen.
- `@OnlyCurrentDoc` als JSDoc-Kommentar im Code beschränkt Docs-, Sheets-, Slides- und Forms-Zugriffe auf die aktuelle Datei.
- Seit 2025 gibt es **granulare OAuth-Zustimmung**: Nutzer können einzelne Scopes ablehnen. Mit `ScriptApp.requireScopes(authMode, scopes)` bzw. `ScriptApp.getAuthorizationInfo(...)` prüfen und sauber reagieren.
- `webapp.executeAs`: `USER_DEPLOYING` (läuft als Entwickler) oder `USER_ACCESSING` (läuft als Besucher, der sich anmelden muss).
- `webapp.access`: `MYSELF`, `DOMAIN`, `ANYONE` (Google-Login nötig), `ANYONE_ANONYMOUS`.

## 3. Dienste-Überblick

### Integrierte Dienste (ohne Aktivierung)
| Dienst | Zweck |
|---|---|
| `SpreadsheetApp` | Google Sheets |
| `DocumentApp` | Google Docs (inkl. Tabs: `doc.getTabs()`, `tab.asDocumentTab().getBody()`) |
| `SlidesApp` | Google Slides |
| `FormApp` | Google Forms |
| `GmailApp` / `MailApp` | Mail lesen und senden / nur senden (kleinerer Scope) |
| `CalendarApp` | Kalender |
| `DriveApp` | Dateien und Ordner (Iteratoren!) |
| `ContactsApp` | **abgekündigt**, stattdessen den erweiterten Dienst People API nutzen |
| `UrlFetchApp` | HTTP-Requests (`fetch`, `fetchAll`) |
| `HtmlService` / `ContentService` | UI und Web-Apps / Text-, JSON- und CSV-Ausgabe |
| `CardService` | Karten-UI für Workspace-Add-ons |
| `PropertiesService` | Key-Value-Speicher (Script, User, Document) |
| `CacheService` | Cache (Script, User, Document) |
| `LockService` | Sperren (Script, User, Document) |
| `ScriptApp` | Trigger, OAuth-Token, Projektinfos |
| `Utilities` | Datum formatieren, Base64, Hash/HMAC, `sleep`, Zip, `newBlob`, `parseCsv`, UUID |
| `Session` | aktueller Nutzer, Zeitzone |
| `Jdbc` | MySQL, SQL Server, Oracle, Cloud SQL |
| `XmlService` | XML parsen und erzeugen |
| `Charts`, `Maps`, `LanguageApp` | Diagramme, Karten, Übersetzung |

### Erweiterte Dienste (im Editor unter „Dienste +“ aktivieren)
Dünne Wrapper um die REST-APIs, inklusive automatischem OAuth. Wichtige: `Drive` (v3), `Sheets`, `Docs`, `Slides`, `Gmail`, `Calendar`, `Tasks`, `Chat`, `DriveActivity`, `DriveLabels`, `People`, `AdminDirectory`, `AdminReports`, `AdminLicenseManager`, `Classroom`, `BigQuery`, `YouTube`, `AnalyticsData`, `AnalyticsAdmin`, `WorkspaceEvents`. Je nach Verfügbarkeit gibt es auch einen Vertex-AI-Dienst; das bitte in der Doku prüfen.

Signatur-Muster: `Service.Resource.method(requestBody?, pathParams..., optionalArgs)`. Beispiel: `Drive.Files.list({ q: "...", pageSize: 100, fields: "nextPageToken, files(id,name)" })`.

## 4. Trigger

### Einfache Trigger (reservierte Funktionsnamen)
`onOpen(e)`, `onEdit(e)`, `onSelectionChange(e)`, `onInstall(e)`, `doGet(e)`, `doPost(e)`
- Laufen ohne Autorisierung, also **keine Dienste, die Auth brauchen** (z. B. kein `GmailApp`, kein `UrlFetchApp`). Maximal **30 Sekunden**.
- `onEdit` wird **nicht** durch Skriptänderungen ausgelöst und nicht durch Formelneuberechnung.
- `onEdit`-Event: `e.range`, `e.value`, `e.oldValue`, `e.source`, `e.user` (eingeschränkt). Achtung: Bei Mehrfachauswahl oder beim Einfügen ist `e.value` undefiniert.

### Installierbare Trigger
Zeitgesteuert, `onOpen`, `onEdit`, `onChange`, `onFormSubmit` (Sheet oder Form), Kalender-Update (`onEventUpdated`). Sie laufen **als der Nutzer, der sie erstellt hat**, mit dessen Rechten. Die Laufzeit entspricht dem normalen Limit.

```javascript
function installTriggers() {
  // Avoid duplicates
  ScriptApp.getProjectTriggers()
    .filter((t) => t.getHandlerFunction() === "hourlyJob")
    .forEach((t) => ScriptApp.deleteTrigger(t));
  ScriptApp.newTrigger("hourlyJob").timeBased().everyHours(1).create();
  ScriptApp.newTrigger("onFormSubmitHandler")
    .forSpreadsheet(SpreadsheetApp.getActive())
    .onFormSubmit()
    .create();
}
```
- Zeitgesteuerte Trigger haben eine Ausführungsungenauigkeit von etwa ±15 Minuten. `atHour(h).nearMinute(m)` ist ungefähr.
- Limit: **20 Trigger pro Nutzer und Skript**.
- Event-Objekte testen: Eine Testfunktion schreiben, die ein Fake-`e` übergibt.

## 5. Quotas und Limits (wichtigste, Richtwerte)

| Limit | Privat (gmail.com) | Workspace |
|---|---|---|
| Laufzeit pro Ausführung | 6 min | 6 min |
| Custom Function Laufzeit | 30 s | 30 s |
| Gesamtlaufzeit Trigger pro Tag | 90 min | 6 h |
| Gleichzeitige Ausführungen pro Nutzer | 30 | 30 |
| E-Mail-Empfänger pro Tag | 100 | 1 500 |
| UrlFetch-Aufrufe pro Tag | 20 000 | 100 000 |
| UrlFetch Antwortgröße | 50 MB | 50 MB |
| Properties: Wertgröße / Gesamt | 9 KB / 500 KB | 9 KB / 500 KB |
| Cache: Wertgröße / max. TTL | 100 KB / 6 h (21 600 s) | gleich |
| Trigger pro Nutzer und Skript | 20 | 20 |
| Dokumente erstellen pro Tag | 250 | 1 500 |
| Kalender-Events erstellen pro Tag | 5 000 | 10 000 |

Restquota für Mails: `MailApp.getRemainingDailyQuota()`.
Quelle: https://developers.google.com/apps-script/guides/services/quotas

## 6. Speicher, Cache und Locks

```javascript
const props = PropertiesService.getScriptProperties();
props.setProperty("API_KEY", "…"); // einmalig oder über Projekteinstellungen > Skripteigenschaften
const apiKey = props.getProperty("API_KEY");

const cache = CacheService.getScriptCache();
const cached = cache.get("rates");
if (!cached) cache.put("rates", JSON.stringify(data), 3600);

const lock = LockService.getScriptLock();
if (!lock.tryLock(30000)) throw new Error("Could not obtain lock");
try { /* critical section */ } finally { lock.releaseLock(); }
```
- `ScriptProperties` gelten für alle Nutzer. `UserProperties` gelten pro Nutzer. `DocumentProperties` gelten pro Datei (für Add-ons).
- Große Daten gehören in Drive-Dateien (JSON), ein Sheet oder eine Datenbank (Firestore über REST, Cloud SQL über JDBC). Properties sind dafür nicht geeignet.

## 7. Performance-Grundregeln

1. **Aufrufe minimieren**: Jeder Dienstaufruf ist ein Netzwerk-Roundtrip. Einmal `getDataRange().getValues()`, dann in JS verarbeiten, dann einmal `setValues()`.
2. Formate, Notizen und Validierungen ebenfalls im Batch setzen: `setBackgrounds()`, `setNumberFormats()`, `setNotes()`.
3. Sehr große Sheets: Sheets API `Sheets.Spreadsheets.Values.batchGet/batchUpdate` ist oft deutlich schneller.
4. Mehrere HTTP-Requests: `UrlFetchApp.fetchAll(requests)` lädt parallel.
5. Teure Daten cachen (`CacheService`).
6. Lange Jobs: Fortschritt speichern und neu triggern (siehe Rezepte).
7. `DriveApp`-Iteratoren sind langsam. `Drive.Files.list` mit `q` und `fields` ist schneller.
8. Keine `SpreadsheetApp.flush()`-Aufrufe in Schleifen.

## 8. Fehlerbehandlung und Logging

- `console.*` schreibt nach Cloud Logging. Ausführungen siehe Editor → „Ausführungen“.
- `exceptionLogging: "STACKDRIVER"` im Manifest.
- Eigenes GCP-Projekt verknüpfen (Projekteinstellungen → GCP-Projekt): Das ist nötig für Cloud Logging im Detail, eigene APIs, OAuth-Zustimmungsbildschirm, Add-on-Veröffentlichung und Vertex AI.
- Fehlerbenachrichtigungen für Trigger: per E-Mail (täglich oder sofort) im Trigger-Dialog einstellen.

## 9. Bibliotheken

- Eigene Bibliothek: Skript-ID teilen → Im Zielprojekt „Bibliotheken +“ → Version wählen. Achtung: Bibliotheken verlangsamen die Ausführung etwas; für Performance-kritische Teile Code lieber kopieren.
- Bekannte Bibliotheken: **OAuth2 for Apps Script** (googleworkspace/apps-script-oauth2), **OAuth1**, **cheeriogs** (HTML parsen), **Sheetfu** und **Tamotsu** (Sheet als ORM), **lodashgs**, **BetterLog**, **gast** und **GSUnit** (Tests), **cGoa** (OAuth-Helfer), **mjml-apps-script** (MJML-E-Mail-Templates, jpoehnelt).

## 10. V8-Spezifika

- Klassen, `static`, Getter und Setter funktionieren. **Klassen und `const`/`let` werden nicht „gehoistet“**: Wird eine Klasse aus Datei B schon auf oberster Ebene von Datei A benutzt (z. B. `const x = new Foo()`), zählt die Dateireihenfolge. Innerhalb von Funktionen ist das unkritisch. Funktionen (`function foo(){}`) sind global verfügbar.
- Nur **Funktionsdeklarationen** erscheinen im Ausführen-Menü und sind als Trigger oder per `google.script.run` aufrufbar, keine Arrow Functions in `const`.
- `Intl`, `Array.prototype.flat`, `Object.fromEntries`, `String.prototype.replaceAll`, `structuredClone` (prüfen) stehen weitgehend zur Verfügung.
- WebAssembly wird unterstützt (siehe `wasm/`-Samples im Repo, z. B. Python oder Rust nach WASM).
