# Google Workspace APIs – Zugriff aus Apps Script

Für jede API: **(A) integrierter Dienst**, **(B) erweiterter Dienst**, **(C) REST über `UrlFetchApp`**. Dazu Scopes, Kernkonzepte und Fallstricke.

Allgemeines Muster für (C):
```javascript
function callGoogleApi_(url, method = "get", payload) {
  const options = {
    method,
    contentType: "application/json",
    headers: { Authorization: `Bearer ${ScriptApp.getOAuthToken()}` },
    muteHttpExceptions: true,
  };
  if (payload) options.payload = JSON.stringify(payload);
  const res = UrlFetchApp.fetch(url, options);
  const code = res.getResponseCode();
  if (code >= 300) throw new Error(`API ${code}: ${res.getContentText()}`);
  return res.getContentText() ? JSON.parse(res.getContentText()) : {};
}
```
Voraussetzungen für (C):
1. Die benötigten Scopes stehen in `appsscript.json` unter `oauthScopes`, plus `script.external_request`.
2. Das Skript ist mit einem **eigenen (Standard-)GCP-Projekt** verknüpft, und die API ist dort aktiviert.
3. Der OAuth-Zustimmungsbildschirm ist konfiguriert (intern für Workspace, extern mit Testnutzern für Privatkonten).

Paginierung (alle Google-REST-APIs): `pageToken` und `nextPageToken` in einer Schleife abfragen.

---

## Gmail
Doku: https://developers.google.com/workspace/gmail/api/guides
- **A `GmailApp`**: `search(query, start, max)`, `getInboxThreads()`, `sendEmail(to, subject, body, { htmlBody, attachments, cc, bcc, name, replyTo, inlineImages })`, `createDraft`, Labels (`getUserLabelByName`, `createLabel`), `thread.addLabel`, `markRead`, `moveToArchive`.
- **A `MailApp`**: nur Senden (Scope `script.send_mail`), sinnvoll wenn kein Postfachzugriff nötig ist.
- **B `Gmail`** (Advanced): `Gmail.Users.Messages.list('me', { q, maxResults })`, `.get('me', id, { format: 'full'|'metadata'|'raw' })`, `Gmail.Users.Messages.send({ raw }, 'me')` (raw = base64url-kodierte RFC-2822-Nachricht), `Users.Settings.Filters`, `Users.Settings.SendAs` (Signaturen), `Users.Labels`, `Users.History.list` (inkrementelle Synchronisation), `Users.watch` (Push über Pub/Sub).
- Suchoperatoren wie im Gmail-UI: `from:`, `to:`, `subject:`, `has:attachment`, `filename:pdf`, `label:`, `is:unread`, `newer_than:2d`, `after:2026/01/01`, `in:anywhere`, `-label:verarbeitet`.
- Fallstricke: 100 bzw. 1 500 Empfänger pro Tag; `GmailApp.search` liefert maximal 500 Threads pro Aufruf (paginieren!). Ein Label „verarbeitet“ setzen, damit nichts doppelt verarbeitet wird. Für Inline-Bilder `inlineImages: { logo: blob }` + `<img src="cid:logo">`. Gmail-Add-ons (kontextuell, Compose) siehe Datei 04.
- Scopes: `gmail.readonly`, `gmail.modify`, `gmail.send`, `gmail.compose`, `gmail.labels`, `gmail.settings.basic`, `https://mail.google.com/` (voll).

## Google Calendar
Doku: https://developers.google.com/workspace/calendar/api/guides/overview
- **A `CalendarApp`**: `getDefaultCalendar()`, `getCalendarById(id)`, `getEvents(start, end, { search })`, `createEvent(title, start, end, { description, location, guests, sendInvites })`, `createAllDayEvent`, `createEventSeries` mit `CalendarApp.newRecurrence()`, `event.setColor(CalendarApp.EventColor.RED)`.
- **B `Calendar`** (Advanced): Konferenzlinks (Meet) mit `conferenceData.createRequest` + `{ conferenceDataVersion: 1 }`, `Calendar.Events.list(calId, { syncToken })` für inkrementelle Synchronisation, `Calendar.Events.patch`, Event-Typen (`focusTime`, `outOfOffice`, `workingLocation`), Erweiterte Properties (`extendedProperties.private`), `Calendar.Freebusy.query`, `Calendar.Acl`.
- Fallstricke: Zeitzonen explizit setzen (`timeZone` im Event bzw. `Session.getScriptTimeZone()`). Ganztägige Events: `date` statt `dateTime`, das Ende ist exklusiv. Max. ca. 5 000 bzw. 10 000 erstellte Events pro Tag. Trigger `onEventUpdated` meldet nur, *dass* sich etwas geändert hat. Die Details holt man über `syncToken`.
- **CalDAV** (https://developers.google.com/workspace/calendar/caldav/v2/guide): Protokoll für externe Clients (Thunderbird, iOS, eigene Server). Endpunkt `https://apidata.googleusercontent.com/caldav/v2/{calendarId}/events`, OAuth 2.0 Pflicht. **In Apps Script unüblich**: Lieber Calendar API nutzen. CalDAV nur, wenn ein bestehender CalDAV-Client angebunden werden muss (Cloud-Projekt mit aktivierter CalDAV API).

## Google Chat
Doku: https://developers.google.com/workspace/chat/overview · REST: https://developers.google.com/workspace/chat/api/reference/rest · Samples: https://developers.google.com/workspace/chat/samples
- **Chat-Apps** (Bots) lassen sich direkt in Apps Script bauen: Sie reagieren auf Interaktionsereignisse (Nachricht, Hinzufügen zu einem Space, Slash-Befehle, Kartenklicks, Dialoge). Konfiguration in der GCP-Konsole unter „Google Chat API → Konfiguration“ (Apps-Script-Bereitstellungs-ID). Neuere Chat-Apps werden als **Google Workspace-Add-on** konfiguriert (Event-Struktur `e.chat.messagePayload`). Ältere Apps nutzen `onMessage(e)`, `onAddToSpace(e)`, `onRemoveFromSpace(e)`, `onCardClick(e)`. Siehe Datei 04.
- **B `Chat`** (Advanced Service): `Chat.Spaces.list()`, `Chat.Spaces.Messages.create(message, spaceName)`, `Chat.Spaces.Messages.list(spaceName, { filter })`, `Chat.Spaces.Members.list`, `Chat.Spaces.setup` (Space mit Mitgliedern anlegen), Reaktionen, Anhänge, Nachrichtensuche (neue Samples im Repo unter `chat/advanced-service`).
- Authentifizierung: **Nutzer-Auth** (Scopes `chat.messages`, `chat.spaces`, `chat.memberships` …) funktioniert direkt mit dem Advanced Service. **App-Auth** (als Bot posten, Scope `chat.bot`) braucht ein Dienstkonto. In Apps Script geht das über die OAuth2-Bibliothek mit Dienstkonto-Schlüssel aus den Script Properties.
- **Webhooks** (einfachste Variante, nur Senden): `UrlFetchApp.fetch(webhookUrl, { method: 'post', contentType: 'application/json', payload: JSON.stringify({ text: 'Hallo' }) })`. Thread-Antworten mit `?messageReplyOption=REPLY_MESSAGE_FALLBACK_TO_NEW_THREAD` + `thread.threadKey`.
- Nachrichtenformat: `text` mit Chat-Markup (`*fett*`, `_kursiv_`, `<users/123>` für Erwähnungen) oder `cardsV2`.

## Google Docs
Doku: https://developers.google.com/workspace/docs · API: https://developers.google.com/workspace/docs/api/how-tos/overview
- **A `DocumentApp`**: `create`, `openById`, `getBody()`, `body.appendParagraph`, `appendTable`, `appendListItem`, `replaceText(regex, text)` (RE2-Regex!), `findText`, `getImages()`, Header und Footer, `setAttributes`, `Document.saveAndClose()` (nötig, bevor man z. B. ein PDF exportiert). **Tabs**: `doc.getTabs()`, `tab.getChildTabs()`, `tab.asDocumentTab().getBody()`.
- **B `Docs`** (Advanced): `Docs.Documents.get(id, { includeTabsContent: true })`, `Docs.Documents.batchUpdate({ requests }, id)`. Requests: `insertText`, `replaceAllText`, `deleteContentRange`, `insertTable`, `insertInlineImage`, `updateTextStyle`, `updateParagraphStyle`, `createNamedRange`, `replaceNamedRangeContent`.
- Konzepte: Strukturelemente mit **`startIndex`/`endIndex`** (UTF-16). Bei mehreren Einfügungen **von hinten nach vorne** arbeiten, damit sich die Indizes nicht verschieben. Platzhalter wie `{{name}}` + `replaceAllText` sind die robusteste Templating-Methode.
- Vorlagen-Workflow: `DriveApp.getFileById(templateId).makeCopy(name, folder)` → `DocumentApp.openById(copy.getId())` → `replaceText` → `saveAndClose()` → PDF: `copy.getAs(MimeType.PDF)`.
- Add-on-Samples für Docs: https://developers.google.com/workspace/add-ons/samples?product=googledocs

## Google Drive
Doku: https://developers.google.com/workspace/drive · API: https://developers.google.com/workspace/drive/api/guides/about-sdk
- **A `DriveApp`**: `getFileById`, `getFolderById`, `searchFiles('title contains "x" and trashed = false')` (v2-Syntax: `title`), `createFile(blob)`, `folder.createFolder`, `file.makeCopy`, `setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW)`, `addEditor`, `moveTo(folder)`, `setTrashed(true)`. Iteratoren: `while (it.hasNext()) { const f = it.next(); }`. Für große Mengen den Fortsetzungstoken `it.getContinuationToken()` + `DriveApp.continueFileIterator(token)` nutzen.
- **B `Drive`** (Advanced, Standard: **v3**): `Drive.Files.list({ q: "name contains 'x' and trashed = false", fields: 'nextPageToken, files(id,name,mimeType,modifiedTime)', pageSize: 1000, supportsAllDrives: true, includeItemsFromAllDrives: true })`, `Drive.Files.create(resource, blob)`, `Drive.Files.update`, `Drive.Files.copy`, `Drive.Permissions.create(perm, fileId, { sendNotificationEmail: false })`, `Drive.Files.export(id, mimeType)`, `Drive.Changes.list`, Shortcuts, **Shared Drives** (`Drive.Drives.list`). In v3 heißt es `name` statt `title`, `create` statt `insert`, und `fields` ist wichtig für die Performance.
- Konvertieren: Beim `create` eine Ziel-`mimeType` angeben, z. B. `application/vnd.google-apps.document`, um aus DOCX ein Google Doc zu machen. OCR: Bild/PDF → Google Doc.
- **Drive Activity API v2** (https://developers.google.com/workspace/drive/activity/v2): **B `DriveActivity`**: `DriveActivity.Activity.query({ itemName: 'items/FILE_ID', pageSize })` oder `ancestorName: 'items/FOLDER_ID'`. Liefert, wer was wann gemacht hat (edit, create, move, rename, permissionChange, comment). Scope `drive.activity.readonly`.
- **Drive Labels API** (https://developers.google.com/workspace/drive/labels/guides/overview): **B `DriveLabels`** zum Lesen und Verwalten von Label-Definitionen (`DriveLabels.Labels.list({ view: 'LABEL_VIEW_FULL' })`). Labels auf Dateien setzen: `Drive.Files.modifyLabels({ labelModifications: [...] }, fileId)`, auslesen: `Drive.Files.listLabels(fileId)`. Labels sind eine Workspace-Funktion (Admin muss sie aktivieren).
- **Google Picker** (https://developers.google.com/workspace/drive/picker/guides/overview): Datei-Auswahldialog in HtmlService (siehe Datei 03). Er braucht OAuth-Token (`ScriptApp.getOAuthToken()`), Developer Key (API-Key aus dem GCP-Projekt), App-ID (Projektnummer) und `setOrigin(google.script.host.origin)`. Mit Scope `drive.file` bekommt die App Zugriff genau auf die gewählten Dateien. Sample im Repo: `picker/`.

## Google Keep
Doku: https://developers.google.com/workspace/keep/api/guides · REST: https://developers.google.com/workspace/keep/api/reference/rest
- **Kein integrierter und kein erweiterter Dienst.** Die Keep API ist für **Workspace-Unternehmenskunden** gedacht (Admin, Compliance), nicht für Privatkonten.
- Ressourcen: `notes` (`create`, `get`, `list`, `delete`), `notes.permissions` (`batchCreate`, `batchDelete`), `media.download` (Anhänge). Notizen haben `title` und `body` (`text` oder `list` mit `listItems`, auch verschachtelt). Ein Update existiert nicht: löschen und neu anlegen.
- Scopes: `https://www.googleapis.com/auth/keep` bzw. `keep.readonly`. Typischerweise mit **Dienstkonto + domainweiter Delegierung** (OAuth2-Bibliothek, `setSubject(userEmail)`). Endpunkt: `https://keep.googleapis.com/v1/notes`.

## Google Forms
Doku: https://developers.google.com/workspace/forms · API: https://developers.google.com/workspace/forms/api/guides
- **A `FormApp`**: `FormApp.create(title)`, `addTextItem`, `addMultipleChoiceItem().setChoiceValues([...])`, `addCheckboxItem`, `addGridItem`, `addPageBreakItem`, `setDestination(FormApp.DestinationType.SPREADSHEET, ssId)`, `getResponses()`, `response.getItemResponses()`, Quiz-Modus `setIsQuiz(true)`, Vorbefüllte Links `response.toPrefilledUrl()`.
- Trigger: installierbarer `onFormSubmit` (an der Form: `e.response`; am Sheet: `e.values`, `e.namedValues`).
- **C Forms REST API** (`https://forms.googleapis.com/v1/forms`): `forms.create` (nur Titel), dann `forms.batchUpdate` (Items anlegen), `forms.responses.list`, **Watches** (Push-Benachrichtigungen über Pub/Sub bei neuen Antworten oder Schemaänderungen). Neuere Formulare müssen ggf. über `setPublishSettings` veröffentlicht werden (Doku prüfen). Scopes: `forms.body`, `forms.responses.readonly`. Samples: `forms-api/` im Repo.

## Google Meet
- **Meet REST API** (https://developers.google.com/workspace/meet/api/guides/overview), nur über **C UrlFetchApp**: `https://meet.googleapis.com/v2/spaces` (`create`, `get`, `patch`, `endActiveConference`), `conferenceRecords` (`list`), `.participants`, `.participantSessions`, `.recordings`, `.transcripts`, `.transcripts.entries`. Scopes: `meetings.space.created`, `meetings.space.readonly`. Typischer Use-Case: Meeting-Raum erzeugen, nach dem Meeting Transkript holen, mit Gemini zusammenfassen und in einem Doc ablegen.
- Meet-Links für Kalendertermine: einfacher über die Calendar API (`conferenceData`).
- **Meet Media API** (https://developers.google.com/workspace/meet/media-api/guides/overview): Echtzeit-Audio/Video über **WebRTC**, Developer Preview. **Nicht in Apps Script umsetzbar**: Dafür Node.js, Python oder C++ auf einem Server verwenden.
- **Meet-Add-ons** (https://developers.google.com/workspace/meet/add-ons/guides/overview): Werden mit dem **Meet Add-ons SDK für das Web** (JavaScript, eigenes Hosting) gebaut, **nicht** mit Apps Script/CardService. Apps Script kann höchstens als Backend (Web-App) dienen.

## Google Sheets
Doku: https://developers.google.com/workspace/sheets · API-Konzepte: https://developers.google.com/workspace/sheets/api/guides/concepts
- **A `SpreadsheetApp`**: `getActiveSpreadsheet()`, `openById`, `getSheetByName`, `getRange('A1:C10')` / `getRange(row, col, numRows, numCols)`, `getDataRange()`, `getValues()`/`setValues()`, `getDisplayValues()`, `appendRow` (langsam bei vielen Zeilen), `getLastRow()`, `insertSheet`, `copyTo`, `sort`, `createFilter`, `newConditionalFormatRule()`, `newDataValidation()`, `protect()`, `getUi().createMenu()`, Sidebar/Dialog, `TextFinder` (`createTextFinder('x').findAll()`), Named Ranges, `RangeList`, Diagramme (`newChart()`).
- **B `Sheets`** (Advanced): `Sheets.Spreadsheets.Values.get(ssId, 'Tab!A1:Z')`, `.batchGet(ssId, { ranges })`, `.update({ values }, ssId, range, { valueInputOption: 'USER_ENTERED' })`, `.append`, `.batchUpdate({ data, valueInputOption }, ssId)`, `Sheets.Spreadsheets.batchUpdate({ requests }, ssId)` (Formatierung, Sheets anlegen, Merge, AutoResize, Pivot, bedingte Formatierung, Filter-Views).
- Konzepte: A1-Notation vs. `GridRange` (0-basiert, Ende exklusiv). `sheetId` (numerisch) ist nicht dasselbe wie der Sheet-Name. `valueInputOption`: `RAW` (unverändert) vs. `USER_ENTERED` (wird geparst wie eine Eingabe). `ValueRenderOption`: `FORMATTED_VALUE`, `UNFORMATTED_VALUE`, `FORMULA`.
- **Custom Functions** (`=MEINEFUNKTION(A1:A10)`): 30 s, **keine Dienste mit Nutzer-Auth** (z. B. kein `GmailApp`). `UrlFetchApp` und `CacheService` gehen. Bereiche kommen als 2D-Array an. Gibt man ein 2D-Array zurück, wird es „gespillt“. Siehe Datei 04.
- Datum: Sheets liefert JS-`Date`-Objekte. Zahlen bleiben Zahlen, leere Zellen sind `""`.

## Google Slides
Doku: https://developers.google.com/workspace/slides · API: https://developers.google.com/workspace/slides/api/guides/overview
- **A `SlidesApp`**: `SlidesApp.create`, `openById`, `getSlides()`, `appendSlide(SlidesApp.PredefinedLayout.TITLE_AND_BODY)`, `slide.insertTextBox`, `insertImage(blob|url)`, `insertTable`, `replaceAllText('{{x}}', 'y')`, `slide.getShapes()`, Speaker Notes (`slide.getNotesPage().getSpeakerNotesShape()`), `insertSheetsChart(chart)` (verknüpftes Diagramm, mit `refresh()`), Thumbnails über die API.
- **B `Slides`** (Advanced): `Slides.Presentations.batchUpdate({ requests }, presId)` mit `createSlide`, `insertText`, `replaceAllText`, `replaceAllShapesWithImage`, `createImage`, `updateShapeProperties`, `duplicateObject`. Eigene `objectId`s vergeben (5–50 Zeichen, eindeutig).
- Einheiten: EMU (1 pt = 12 700 EMU). Seitengröße abfragen: `presentation.getPageWidth()`.
- Workflow „Sheet → Präsentation“: Vorlage kopieren, Folie pro Datenzeile duplizieren, Platzhalter ersetzen (Samples: `slides/`, `mashups/sheets2slides.gs`).
- Add-on-Samples für Slides: https://developers.google.com/workspace/add-ons/samples?product=googleslides

## Google Tasks
Doku: https://developers.google.com/workspace/tasks/overview
- **B `Tasks`** (Advanced): `Tasks.Tasklists.list()`, `Tasks.Tasks.list(taskListId, { showCompleted: false, dueMin })`, `Tasks.Tasks.insert({ title, notes, due }, taskListId)`, `.patch`, `.move` (Unteraufgaben über `parent`), `.clear`.
- `due` ist RFC-3339, aber **nur das Datum zählt** (die Uhrzeit wird ignoriert). Status `needsAction` oder `completed`. Samples: `tasks/`.

## Google Vault
Doku: https://developers.google.com/workspace/vault
- eDiscovery und Aufbewahrung für **Workspace** (Lizenz erforderlich). Nur **C REST**: `https://vault.googleapis.com/v1/matters` (Matters anlegen, schließen, löschen), `matters.holds` (Legal Holds für Gmail, Drive, Chat, Groups, Voice, Meet), `matters.exports` (Export erstellen, Status abfragen, Download über Cloud Storage), `matters.savedQueries`, `matters.permissions`.
- Scope: `https://www.googleapis.com/auth/ediscovery` (bzw. `.readonly`). Der Nutzer braucht Vault-Berechtigungen. Typisch: zeitgesteuerter Report über offene Holds in ein Sheet.

## Admin SDK, People, Classroom (Bonus)
- **AdminDirectory**: Nutzer, Gruppen, OUs, Geräte (`AdminDirectory.Users.list({ customer: 'my_customer' })`). **AdminReports**: Audit-Logs und Nutzungsberichte. Nur für Admins.
- **People** (ersetzt `ContactsApp`): `People.People.Connections.list('people/me', { personFields: 'names,emailAddresses' })`, `People.People.createContact`, `searchContacts`, `People.OtherContacts`.
- **Classroom**: Kurse, Teilnehmende, Aufgaben (`Classroom.Courses.list()`).

## Google Workspace Events API
- Abonnements auf Änderungen in Chat, Meet und Drive mit Zustellung über Pub/Sub (`WorkspaceEvents.Subscriptions.create`). Für eine Reaktion in Apps Script: Pub/Sub-Push → Web-App `doPost`, oder Pull per Zeittrigger.
