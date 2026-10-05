# Web-Apps, JSON-APIs, HtmlService und Picker

Doku: https://developers.google.com/apps-script/guides/web · https://developers.google.com/apps-script/guides/html

## 1. Web-App-Grundlagen

```javascript
function doGet(e) {
  // e.parameter = { key: "first value" }, e.parameters = { key: ["v1","v2"] }, e.pathInfo
  const action = e.parameter.action ?? "list";
  return jsonResponse_({ ok: true, action });
}

function doPost(e) {
  // e.postData.contents = raw body, e.postData.type = Content-Type
  let body;
  try {
    body = JSON.parse(e.postData?.contents ?? "{}");
  } catch (err) {
    return jsonResponse_({ ok: false, error: "Invalid JSON" });
  }
  const lock = LockService.getScriptLock();
  lock.waitLock(20000);
  try {
    SpreadsheetApp.openById(SHEET_ID).getSheetByName("Inbox")
      .appendRow([new Date(), body.name, body.email]);
  } finally {
    lock.releaseLock();
  }
  return jsonResponse_({ ok: true });
}

function jsonResponse_(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

### Bereitstellen
Editor → „Bereitstellen“ → „Neue Bereitstellung“ → Typ „Web-App“ → „Ausführen als“ + „Zugriff“ → URL `https://script.google.com/macros/s/DEPLOYMENT_ID/exec`.
- **`/exec`** = feste Version (nach Codeänderungen: „Bereitstellungen verwalten“ → Bearbeiten → **Neue Version**, dann bleibt die URL gleich). **`/dev`** = HEAD-Code, nur für Bearbeiter, mit Login.
- Für öffentliche JSON-APIs: Ausführen als **„Ich“**, Zugriff **„Jeder“** (in der Manifest-Sprache `ANYONE_ANONYMOUS`).
- In Workspace-Domains kann der Admin anonymen Zugriff verbieten.

### Wichtige Eigenheiten (JSON an eine Web-App senden)
1. **Redirect**: Antworten kommen über einen **302-Redirect** von `script.googleusercontent.com`. Clients müssen Redirects folgen: `curl -L`, in `fetch` Standard. Bei POST wandelt der Redirect den Request in ein GET um. Die Antwort enthält trotzdem das Ergebnis von `doPost`.
2. **Keine eigenen Statuscodes und Header**: `ContentService` liefert immer 200. Fehler im JSON-Body signalisieren (`{ ok: false, error }`). Eigene CORS-Header lassen sich nicht setzen.
3. **CORS aus dem Browser**: Einfache Requests ohne Preflight verwenden. `fetch(url, { method: "POST", body: JSON.stringify(data), headers: { "Content-Type": "text/plain;charset=utf-8" } })`. Mit `application/json` würde der Browser einen OPTIONS-Preflight senden, und den kann Apps Script nicht beantworten. Die Antwort von `/exec` ist dann per CORS lesbar.
4. **Formulardaten**: Bei `application/x-www-form-urlencoded` landen die Felder in `e.parameter`. Bei JSON muss man `JSON.parse(e.postData.contents)` selbst aufrufen.
5. **Authentifizierung**: Es gibt keine Header-basierte Auth für anonyme Web-Apps. Workaround: ein geheimes Token als Query-Parameter oder im Body, verglichen mit den Script Properties (Rotation einplanen). Alternativ Zugriff „Jeder mit Google-Konto“ bzw. „Domain“ + OAuth-Bearer-Token des Aufrufers im Header `Authorization: Bearer …`. Das Token braucht in der Praxis einen Drive-Scope, also vorher testen.
6. **Latenz**: Kaltstart 1–3 s. Nicht für Hochfrequenz-APIs gedacht; Quotas für gleichzeitige Ausführungen beachten (30 pro Nutzer).
7. **Webhooks von Drittanbietern** (Stripe, GitHub …): Signaturprüfung geht nur, wenn die Signatur im Body oder in der URL steht. Header sind in `e` **nicht verfügbar**. Gegebenenfalls einen Proxy (Cloud Function) davorschalten.
8. **Pub/Sub-Push** an die Web-App: Body enthält `message.data` (Base64) → `Utilities.newBlob(Utilities.base64Decode(data)).getDataAsString()`.

Test mit curl:
```bash
curl -L -H "Content-Type: text/plain" -d '{"name":"Ada","email":"ada@example.com"}' \
  "https://script.google.com/macros/s/DEPLOYMENT_ID/exec"
```

## 2. HtmlService – UI (Sidebar, Dialog, Web-App)

```javascript
function onOpen() {
  SpreadsheetApp.getUi().createMenu("Tools")
    .addItem("Sidebar öffnen", "showSidebar")
    .addToUi();
}

function showSidebar() {
  const tpl = HtmlService.createTemplateFromFile("Sidebar");
  tpl.user = Session.getActiveUser().getEmail();
  SpreadsheetApp.getUi().showSidebar(tpl.evaluate().setTitle("Tools"));
}

// Partial includes: <?!= include('Styles') ?>
function include(filename) {
  return HtmlService.createHtmlOutputFromFile(filename).getContent();
}

function getRows() { // callable from client
  return SpreadsheetApp.getActiveSheet().getDataRange().getDisplayValues();
}
```

`Sidebar.html`:
```html
<!DOCTYPE html>
<html>
  <head><base target="_top"><?!= include('Styles') ?></head>
  <body>
    <p>Hallo <?= user ?></p>
    <button id="load">Laden</button>
    <pre id="out"></pre>
    <script>
      document.getElementById("load").addEventListener("click", () => {
        google.script.run
          .withSuccessHandler((rows) => { document.getElementById("out").textContent = JSON.stringify(rows, null, 2); })
          .withFailureHandler((err) => { alert(err.message); })
          .getRows();
      });
    </script>
  </body>
</html>
```

- **Scriptlets**: `<? code ?>`, `<?= escaped output ?>`, `<?!= unescaped ?>` (nur für vertrauenswürdiges HTML wie Includes).
- **`google.script.run`**: asynchron. Übergeben werden können nur Primitive, Arrays, einfache Objekte und `form`-Elemente, **keine `Date`-Objekte** (vorher in einen ISO-String umwandeln). Funktionen mit `_` am Ende sind nicht aufrufbar.
- `google.script.host.close()`, `google.script.host.editor.focus()`, `google.script.url.getLocation(cb)` (Web-App-Parameter im Client), `google.script.history` (Client-Routing).
- Dialoge: `showModalDialog(html, title)`, `showModelessDialog`. Größe mit `setWidth`/`setHeight`.
- Web-App mit HTML: `doGet` gibt `HtmlService.createTemplateFromFile('Index').evaluate().setTitle('App').addMetaTag('viewport','width=device-width, initial-scale=1')` zurück. Einbetten in Google Sites oder iFrame: `.setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL)`.
- CSS-Frameworks und JS-Bibliotheken per CDN einbinden ist erlaubt (läuft im Sandbox-iFrame `IFRAME`-Modus).
- Datei-Upload aus HTML-Formular: `google.script.run.upload(formElement)`. Auf dem Server kommt ein Blob an → `DriveApp.createFile(blob)`. Alternativ FileReader → Base64 → Server → `Utilities.base64Decode`.

## 3. Google Picker in einem Dialog (Sample `picker/` im Repo)

Server:
```javascript
function showPicker() {
  const html = HtmlService.createHtmlOutputFromFile("dialog")
    .setWidth(600).setHeight(425).setSandboxMode(HtmlService.SandboxMode.IFRAME);
  SpreadsheetApp.getUi().showModalDialog(html, "Datei auswählen");
}
function getOAuthToken() {
  DriveApp.getRootFolder(); // forces Drive scope during authorization
  return ScriptApp.getOAuthToken();
}
```
Client (Kern):
```javascript
const view = new google.picker.DocsView(google.picker.ViewId.SPREADSHEETS).setIncludeFolders(true);
const picker = new google.picker.PickerBuilder()
  .addView(view)
  .setOAuthToken(token)
  .setDeveloperKey(DEVELOPER_KEY)    // API key from GCP project
  .setAppId(CLOUD_PROJECT_NUMBER)    // project number, required for drive.file
  .setOrigin(google.script.host.origin)
  .setCallback((data) => {
    if (data.action === google.picker.Action.PICKED) {
      const fileId = data.docs[0].id;
      google.script.run.processFile(fileId);
    }
  })
  .build();
picker.setVisible(true);
```
Manifest-Scopes: `script.container.ui`, `drive.file`. Das Skript muss mit einem GCP-Projekt verknüpft sein, in dem die Picker API aktiviert ist.

## 4. Externe APIs aufrufen (UrlFetchApp)

```javascript
function fetchJson_(url, options = {}) {
  const res = UrlFetchApp.fetch(url, { muteHttpExceptions: true, ...options });
  const code = res.getResponseCode();
  const text = res.getContentText();
  if (code < 200 || code >= 300) throw new Error(`HTTP ${code}: ${text.slice(0, 500)}`);
  return JSON.parse(text);
}

// Parallel requests
const responses = UrlFetchApp.fetchAll(urls.map((url) => ({ url, muteHttpExceptions: true })));
```
- Optionen: `method`, `headers`, `payload` (String → roher Body; Objekt → Formular; Blob → Multipart), `contentType`, `followRedirects`, `validateHttpsCertificates`, `escaping`.
- Maximal 50 MB Antwort, Timeout etwa 60 s. Basic Auth: `Authorization: 'Basic ' + Utilities.base64Encode('user:pass')`.
- OAuth2 gegen Drittanbieter (Salesforce, Microsoft, Spotify …): Bibliothek **apps-script-oauth2**. Callback-URL `https://script.google.com/macros/d/{SCRIPT_ID}/usercallback`.
- HTML-Scraping: `cheeriogs`-Bibliothek, oder Regex für einfache Fälle. `XmlService` funktioniert nur für wohlgeformtes XML.
