# Rezepte: geprüfte Code-Muster für Apps Script (V8)

Alle Muster sind eigenständig. Konfiguration steht jeweils oben als `const`.

## 1. Sheet als Tabelle: Header-Mapping, Objekte, Upsert

```javascript
/**
 * Reads a sheet into an array of objects keyed by header row.
 * @param {GoogleAppsScript.Spreadsheet.Sheet} sheet
 * @return {Object[]}
 */
function readTable_(sheet) {
  const [headers, ...rows] = sheet.getDataRange().getValues();
  return rows
    .filter((r) => r.some((c) => c !== ""))
    .map((r) => Object.fromEntries(headers.map((h, i) => [h, r[i]])));
}

/**
 * Inserts or updates records by key column in one write.
 */
function upsert_(sheet, records, keyField) {
  const data = sheet.getDataRange().getValues();
  const headers = data[0];
  const keyIdx = headers.indexOf(keyField);
  const index = new Map(data.slice(1).map((r, i) => [String(r[keyIdx]), i + 1]));
  for (const rec of records) {
    const row = headers.map((h) => rec[h] ?? "");
    const pos = index.get(String(rec[keyField]));
    if (pos) data[pos] = row; else { data.push(row); index.set(String(rec[keyField]), data.length - 1); }
  }
  sheet.getRange(1, 1, data.length, headers.length).setValues(data);
}
```

## 2. Retry mit Exponential Backoff

```javascript
/**
 * Runs fn with retries on transient errors.
 * @param {Function} fn
 * @param {number} [maxRetries=5]
 */
function withRetry_(fn, maxRetries = 5) {
  for (let attempt = 0; ; attempt++) {
    try {
      return fn();
    } catch (err) {
      const msg = String(err?.message ?? err);
      const transient = /429|500|502|503|504|Service invoked too many times|timed out|Rate Limit|Internal error/i.test(msg);
      if (!transient || attempt >= maxRetries) throw err;
      const delay = Math.min(2 ** attempt * 1000 + Math.random() * 1000, 32000);
      console.warn(`Retry ${attempt + 1} after ${Math.round(delay)} ms: ${msg}`);
      Utilities.sleep(delay);
    }
  }
}
```
For `UrlFetchApp` with `muteHttpExceptions: true`, check the response code inside `fn` and `throw` on 429/5xx.

## 3. Lange Jobs: Chunking und Fortsetzung über Trigger (6-min-Limit)

```javascript
const MAX_RUNTIME_MS = 4.5 * 60 * 1000;
const STATE_KEY = "JOB_CURSOR";

function processLargeJob() {
  const start = Date.now();
  const props = PropertiesService.getScriptProperties();
  let cursor = Number(props.getProperty(STATE_KEY) ?? 0);
  const items = getWorkItems_(); // e.g. rows or file IDs

  while (cursor < items.length) {
    if (Date.now() - start > MAX_RUNTIME_MS) {
      props.setProperty(STATE_KEY, String(cursor));
      scheduleContinuation_();
      return;
    }
    handleItem_(items[cursor]);
    cursor++;
  }
  props.deleteProperty(STATE_KEY);
  deleteContinuation_();
  console.log("Job finished");
}

function scheduleContinuation_() {
  deleteContinuation_();
  ScriptApp.newTrigger("processLargeJob").timeBased().after(60 * 1000).create();
}

function deleteContinuation_() {
  ScriptApp.getProjectTriggers()
    .filter((t) => t.getHandlerFunction() === "processLargeJob")
    .forEach((t) => ScriptApp.deleteTrigger(t));
}
```

## 4. Serienbrief (Mail Merge) aus Sheet mit Gmail-Entwurf als Vorlage

```javascript
const SUBJECT_OF_TEMPLATE_DRAFT = "Vorlage: Einladung";
const STATUS_COL = "Status";

function sendMailMerge() {
  const sheet = SpreadsheetApp.getActiveSheet();
  const data = sheet.getDataRange().getDisplayValues();
  const headers = data[0];
  const statusIdx = headers.indexOf(STATUS_COL);
  const draft = GmailApp.getDrafts().find((d) => d.getMessage().getSubject() === SUBJECT_OF_TEMPLATE_DRAFT);
  if (!draft) throw new Error("Template draft not found");
  const msg = draft.getMessage();
  const fill = (tpl, row) => tpl.replace(/{{\s*([^}]+?)\s*}}/g, (_, key) => row[headers.indexOf(key)] ?? "");

  const statuses = data.slice(1).map((row) => {
    if (row[statusIdx]) return [row[statusIdx]];
    if (MailApp.getRemainingDailyQuota() < 1) return ["QUOTA"];
    try {
      GmailApp.sendEmail(row[headers.indexOf("Email")], fill(msg.getSubject(), row), fill(msg.getPlainBody(), row), {
        htmlBody: fill(msg.getBody(), row),
        attachments: msg.getAttachments({ includeInlineImages: false }),
        name: "Team",
      });
      return [`Gesendet ${new Date().toLocaleString("de-DE")}`];
    } catch (e) {
      return [`Fehler: ${e.message}`];
    }
  });
  sheet.getRange(2, statusIdx + 1, statuses.length, 1).setValues(statuses);
}
```
(Vollständiges offizielles Sample: `solutions/automations/mail-merge`.)

## 5. PDF aus Docs-Vorlage erzeugen (pro Zeile)

```javascript
const TEMPLATE_DOC_ID = "DOC_ID";
const OUTPUT_FOLDER_ID = "FOLDER_ID";

function createPdf_(record) {
  const folder = DriveApp.getFolderById(OUTPUT_FOLDER_ID);
  const copy = DriveApp.getFileById(TEMPLATE_DOC_ID).makeCopy(`Tmp ${record.Name}`, folder);
  const doc = DocumentApp.openById(copy.getId());
  const body = doc.getBody();
  for (const [key, value] of Object.entries(record)) {
    body.replaceText(`{{${key}}}`, String(value)); // key must not contain regex chars
  }
  doc.saveAndClose();
  const pdf = folder.createFile(copy.getAs(MimeType.PDF)).setName(`${record.Name}.pdf`);
  copy.setTrashed(true);
  return pdf;
}
```
- PDF aus einem Sheet-Bereich mit Optionen: Export-URL `https://docs.google.com/spreadsheets/d/{id}/export?format=pdf&gid={sheetId}&portrait=true&size=A4&fitw=true&gridlines=false` per `UrlFetchApp` mit `Authorization: Bearer ScriptApp.getOAuthToken()`.
- Offizielle Samples: `solutions/automations/generate-pdfs`, `employee-certificate`.

## 6. Bilder aus einem Google Doc extrahieren und in Drive speichern

Variante A – DocumentApp (Inline-Bilder plus positionierte Bilder):
```javascript
const DOC_ID = "DOC_ID";
const TARGET_FOLDER_ID = "FOLDER_ID";

function extractImagesFromDoc() {
  const doc = DocumentApp.openById(DOC_ID);
  const folder = DriveApp.getFolderById(TARGET_FOLDER_ID);
  const body = doc.getBody();
  let n = 0;

  for (const img of body.getImages()) { // InlineImage
    const blob = img.getBlob();
    folder.createFile(blob.setName(`${doc.getName()}_img_${++n}.${extension_(blob)}`));
  }
  // Positioned (floating) images live on paragraphs
  const paragraphs = body.getParagraphs();
  for (const p of paragraphs) {
    for (const pImg of p.getPositionedImages()) {
      const blob = pImg.getBlob();
      folder.createFile(blob.setName(`${doc.getName()}_pos_${++n}.${extension_(blob)}`));
    }
  }
  console.log(`${n} images saved`);
}

function extension_(blob) {
  return ({ "image/png": "png", "image/jpeg": "jpg", "image/gif": "gif" })[blob.getContentType()] ?? "img";
}
```
Variante B – Export als HTML-ZIP, in dem alle Bilder in Originalqualität liegen:
```javascript
function extractImagesViaZip() {
  const url = `https://docs.google.com/feeds/download/documents/export/Export?id=${DOC_ID}&exportFormat=zip`;
  const zip = UrlFetchApp.fetch(url, { headers: { Authorization: `Bearer ${ScriptApp.getOAuthToken()}` } }).getBlob();
  const folder = DriveApp.getFolderById(TARGET_FOLDER_ID);
  Utilities.unzip(zip.setContentType("application/zip"))
    .filter((b) => b.getName().startsWith("images/"))
    .forEach((b) => folder.createFile(b.setName(b.getName().replace("images/", ""))));
}
```
Scopes: `documents.readonly` (A), `drive` (Export und Ordner). Alternativ zur Export-URL: `Drive.Files.export(id, 'application/zip')`. Bei Docs mit Tabs: über `doc.getTabs()` iterieren.

## 7. Drive: alle Dateien in einem Ordner rekursiv auflisten (schnell, mit Paginierung)

```javascript
function listFilesRecursive_(folderId, path = "") {
  const out = [];
  let pageToken;
  do {
    const res = Drive.Files.list({
      q: `'${folderId}' in parents and trashed = false`,
      fields: "nextPageToken, files(id, name, mimeType, modifiedTime, size)",
      pageSize: 1000,
      pageToken,
      supportsAllDrives: true,
      includeItemsFromAllDrives: true,
    });
    for (const f of res.files ?? []) {
      if (f.mimeType === MimeType.FOLDER) out.push(...listFilesRecursive_(f.id, `${path}/${f.name}`));
      else out.push({ ...f, path: `${path}/${f.name}` });
    }
    pageToken = res.nextPageToken;
  } while (pageToken);
  return out;
}
```

## 8. onEdit: Zeitstempel und Statuslogik

```javascript
const WATCH_SHEET = "Aufgaben";
const STATUS_COL = 3;   // C
const STAMP_COL = 4;    // D

function onEdit(e) {
  const range = e?.range;
  if (!range) return; // manual run guard
  const sheet = range.getSheet();
  if (sheet.getName() !== WATCH_SHEET || range.getColumn() !== STATUS_COL || range.getRow() < 2) return;
  const value = range.getValue();
  sheet.getRange(range.getRow(), STAMP_COL).setValue(value === "Erledigt" ? new Date() : "");
}
```

## 9. Formular-Eingang → E-Mail-Benachrichtigung + Kalendereintrag (installierbarer Trigger)

```javascript
const CALENDAR_ID = "primary";

function onFormSubmitHandler(e) { // trigger: From spreadsheet → On form submit
  const v = e.namedValues; // { "Name": ["Ada"], "Datum": ["05.10.2026"], ... }
  const name = v.Name?.[0];
  const start = parseGermanDate_(v.Datum?.[0]);
  const end = new Date(start.getTime() + 60 * 60 * 1000);
  const calendar = CALENDAR_ID === "primary" ? CalendarApp.getDefaultCalendar() : CalendarApp.getCalendarById(CALENDAR_ID);
  calendar.createEvent(`Termin: ${name}`, start, end, { description: JSON.stringify(v, null, 2) });
  MailApp.sendEmail(Session.getEffectiveUser().getEmail(), `Neue Anmeldung: ${name}`, JSON.stringify(v, null, 2));
}

function parseGermanDate_(s) {
  const [d, m, y] = s.split(".").map(Number);
  return new Date(y, m - 1, d, 9, 0);
}
```

## 10. Kalender ↔ Sheet synchronisieren (Termine exportieren)

```javascript
function exportEvents() {
  const cal = CalendarApp.getDefaultCalendar();
  const now = new Date();
  const in30 = new Date(now.getTime() + 30 * 24 * 3600 * 1000);
  const rows = cal.getEvents(now, in30).map((ev) => [
    ev.getTitle(), ev.getStartTime(), ev.getEndTime(), ev.getLocation(), ev.getGuestList().map((g) => g.getEmail()).join(", "),
  ]);
  const sheet = SpreadsheetApp.getActive().getSheetByName("Termine") ?? SpreadsheetApp.getActive().insertSheet("Termine");
  sheet.clearContents();
  sheet.getRange(1, 1, 1, 5).setValues([["Titel", "Start", "Ende", "Ort", "Gäste"]]);
  if (rows.length) sheet.getRange(2, 1, rows.length, 5).setValues(rows);
}
```

## 11. CSV importieren (aus Drive oder URL)

```javascript
function importCsv(url) {
  const csv = UrlFetchApp.fetch(url).getContentText("UTF-8");
  const data = Utilities.parseCsv(csv, ","); // use ";" for German Excel exports
  const sheet = SpreadsheetApp.getActive().getSheetByName("Import");
  sheet.clearContents().getRange(1, 1, data.length, data[0].length).setValues(data);
}
```

## 12. Gmail: Anhänge automatisch in Drive ablegen

```javascript
const QUERY = "has:attachment filename:pdf -label:archiviert newer_than:7d";
const FOLDER_ID = "FOLDER_ID";
const DONE_LABEL = "archiviert";

function saveAttachments() {
  const folder = DriveApp.getFolderById(FOLDER_ID);
  const label = GmailApp.getUserLabelByName(DONE_LABEL) ?? GmailApp.createLabel(DONE_LABEL);
  for (const thread of GmailApp.search(QUERY, 0, 50)) {
    for (const msg of thread.getMessages()) {
      for (const att of msg.getAttachments({ includeInlineImages: false })) {
        const date = Utilities.formatDate(msg.getDate(), Session.getScriptTimeZone(), "yyyy-MM-dd");
        folder.createFile(att.copyBlob()).setName(`${date}_${att.getName()}`);
      }
    }
    thread.addLabel(label);
  }
}
```

## 13. Chat-Benachrichtigung über Webhook (z. B. bei neuen Sheet-Zeilen)

```javascript
function notifyChat_(text) {
  const url = PropertiesService.getScriptProperties().getProperty("CHAT_WEBHOOK_URL");
  UrlFetchApp.fetch(url, {
    method: "post",
    contentType: "application/json; charset=UTF-8",
    payload: JSON.stringify({ text }),
    muteHttpExceptions: true,
  });
}
```

## 14. Ordnerstruktur aus Sheet anlegen (idempotent)

```javascript
function getOrCreateFolder_(parent, name) {
  const it = parent.getFoldersByName(name);
  return it.hasNext() ? it.next() : parent.createFolder(name);
}
```

## 15. Sheets API: große Formatierung in einem Request

```javascript
function formatHeader_(spreadsheetId, sheetId) {
  Sheets.Spreadsheets.batchUpdate({
    requests: [
      { repeatCell: {
          range: { sheetId, startRowIndex: 0, endRowIndex: 1 },
          cell: { userEnteredFormat: { textFormat: { bold: true }, backgroundColor: { red: 0.9, green: 0.93, blue: 1 } } },
          fields: "userEnteredFormat(textFormat,backgroundColor)" } },
      { updateSheetProperties: { properties: { sheetId, gridProperties: { frozenRowCount: 1 } }, fields: "gridProperties.frozenRowCount" } },
      { autoResizeDimensions: { dimensions: { sheetId, dimension: "COLUMNS", startIndex: 0, endIndex: 26 } } },
    ],
  }, spreadsheetId);
}
```

## 16. Docs API: Platzhalter ersetzen und Tabelle befüllen

```javascript
function fillDoc_(docId, values) {
  const requests = Object.entries(values).map(([k, v]) => ({
    replaceAllText: { containsText: { text: `{{${k}}}`, matchCase: true }, replaceText: String(v) },
  }));
  Docs.Documents.batchUpdate({ requests }, docId);
}
```

## 17. Slides: eine Folie pro Datensatz aus Vorlage

```javascript
function buildDeck_(presentationId, records) {
  const pres = SlidesApp.openById(presentationId);
  const template = pres.getSlides()[0];
  for (const rec of records) {
    const slide = template.duplicate();
    for (const [k, v] of Object.entries(rec)) slide.replaceAllText(`{{${k}}}`, String(v));
  }
  template.remove();
}
```

## 18. Tasks: Aufgaben aus Sheet anlegen

```javascript
function createTasksFromSheet() {
  const list = Tasks.Tasklists.list().items[0];
  const [, ...rows] = SpreadsheetApp.getActiveSheet().getDataRange().getValues();
  for (const [title, due, notes] of rows) {
    if (!title) continue;
    Tasks.Tasks.insert({ title, notes, due: due ? new Date(due).toISOString() : undefined }, list.id);
  }
}
```

## 19. Menü + Bestätigungsdialog + Toast

```javascript
function onOpen() {
  SpreadsheetApp.getUi().createMenu("🚀 Automationen")
    .addItem("Serienbrief senden", "confirmAndSend")
    .addSeparator()
    .addItem("Trigger installieren", "installTriggers")
    .addToUi();
}

function confirmAndSend() {
  const ui = SpreadsheetApp.getUi();
  if (ui.alert("Wirklich senden?", ui.ButtonSet.YES_NO) !== ui.Button.YES) return;
  sendMailMerge();
  SpreadsheetApp.getActive().toast("Fertig ✅", "Serienbrief", 5);
}
```

## 20. HMAC-Signatur, UUID, Hash

```javascript
const sig = Utilities.computeHmacSha256Signature(message, secret)
  .map((b) => (b < 0 ? b + 256 : b).toString(16).padStart(2, "0")).join("");
const id = Utilities.getUuid();
const md5 = Utilities.computeDigest(Utilities.DigestAlgorithm.MD5, text)
  .map((b) => (b & 0xff).toString(16).padStart(2, "0")).join("");
```
