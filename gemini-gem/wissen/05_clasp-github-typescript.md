# Professioneller Workflow: clasp, Git/GitHub, TypeScript, CI/CD, Tests

Quellen: https://github.com/google/clasp · https://developers.google.com/apps-script/guides/clasp · Monorepo-Beispiel https://github.com/jpoehnelt/apps-script (pnpm, Turbo, Biome, Changesets, GitHub Actions) · Offizielle Samples: https://github.com/googleworkspace/apps-script-samples (Biome, pnpm, `GEMINI.md` mit Code-Richtlinien)

## 1. clasp einrichten

```bash
npm install -g @google/clasp
clasp login                       # öffnet Browser, speichert Token in ~/.clasprc.json
# Einmalig: https://script.google.com/home/usersettings → "Google Apps Script API" AKTIVIEREN
clasp create --type sheets --title "Mein Projekt" --rootDir ./src   # neues Projekt (+ neues Sheet)
clasp clone <SCRIPT_ID> --rootDir ./src                             # bestehendes Projekt holen
clasp pull        # Cloud → lokal
clasp push        # lokal → Cloud (überschreibt!)   | clasp push --watch
clasp open        # Editor im Browser öffnen        (clasp 3: clasp open-script)
clasp deploy -d "v1.2 Fix"                # neue Version + Bereitstellung
clasp deploy -i <DEPLOYMENT_ID> -d "…"    # bestehende Bereitstellung (Web-App-URL bleibt gleich)
clasp deployments                         # Liste
clasp logs                                # Cloud-Logs (GCP-Projekt nötig)
clasp run myFunction                      # Funktion remote ausführen (API-Executable-Setup nötig)
```
- **clasp 3.x** hat die Befehle umbenannt (z. B. `create-script`, `clone-script`, `open-script`, `create-deployment`, `list-deployments`, `tail-logs`). Die alten Namen funktionieren meist als Alias. Bei Problemen `clasp --help` prüfen.
- `.clasp.json`: `{ "scriptId": "…", "rootDir": "src", "fileExtension": "js" }`. Dateien: `.js`/`.gs`, `.html`, `appsscript.json`.
- `.claspignore` (wie `.gitignore`): z. B. `**/*.test.js`, `node_modules/**`.
- Unterordner werden in Dateinamen mit `/` umgewandelt (`utils/date.js` → `utils/date`). Die Reihenfolge der Dateien lässt sich in `.clasp.json` über `filePushOrder` steuern.
- Mehrere Umgebungen: zwei Skript-IDs (dev, prod) und `.clasp.dev.json`/`.clasp.prod.json`, vor dem Push kopieren (`cp .clasp.prod.json .clasp.json && clasp push`). Mit clasp 3 alternativ `clasp --project .clasp.prod.json push`.

## 2. Git und GitHub

Empfohlene Struktur:
```
mein-projekt/
├─ src/
│  ├─ appsscript.json
│  ├─ Code.js
│  ├─ services/sheetRepo.js
│  └─ ui/Sidebar.html
├─ test/            # Node-Tests (nicht gepusht)
├─ .clasp.json      # scriptId ist kein Geheimnis, kann committed werden
├─ .claspignore
├─ biome.json
├─ package.json
└─ .github/workflows/deploy.yml
```
- **Niemals committen**: `~/.clasprc.json` (OAuth-Refresh-Token), Dienstkonto-Schlüssel, API-Keys.
- Branch-Strategie: `main` = Produktion. Pull Requests mit Lint und Tests in CI. Merge löst Deploy aus.
- Versionierung: Git-Tag ↔ `clasp deploy -d "$(git describe --tags)"`.

## 3. CI/CD mit GitHub Actions

1. Lokal `clasp login` ausführen → Inhalt von `~/.clasprc.json` als GitHub-Secret `CLASPRC_JSON` speichern.
2. Skript-ID als Secret oder Variable `SCRIPT_ID`, Deployment-ID als `DEPLOYMENT_ID`.

```yaml
name: Deploy Apps Script
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npm run lint && npm test
      - run: npm install -g @google/clasp
      - name: Write clasp credentials
        run: echo '${{ secrets.CLASPRC_JSON }}' > ~/.clasprc.json
      - name: Write .clasp.json
        run: echo '{"scriptId":"${{ secrets.SCRIPT_ID }}","rootDir":"dist"}' > .clasp.json
      - run: npm run build   # optional (TypeScript/Bundler)
      - run: clasp push -f
      - run: clasp deploy -i "${{ secrets.DEPLOYMENT_ID }}" -d "CI ${{ github.sha }}"
```
- Der Refresh-Token kann verfallen, wenn der OAuth-Client im Testmodus ist (7 Tage) oder das Passwort geändert wird. Dann neu einloggen und das Secret aktualisieren. Robuster: eigener OAuth-Client (`clasp login --creds creds.json`) im Produktionsmodus.
- Alternative ohne clasp: die Apps Script API direkt (`projects.updateContent`, `projects.versions.create`, `projects.deployments.update`) mit Dienstkonto bzw. Workload Identity. Dienstkonten können Apps-Script-Projekte allerdings nur eingeschränkt verwalten.

## 4. TypeScript

- `npm i -D @types/google-apps-script` → Autovervollständigung für alle Dienste (auch in reinem JS mit `// @ts-check` und `jsconfig.json`).
- **clasp ab Version 3 transpiliert kein TypeScript mehr.** Stattdessen bündeln: **esbuild**, **Rollup** (`rollup-plugin-gas` o. ä.), **Vite** (`vite-plugin-google-apps-script`) oder Webpack (`gas-webpack-plugin`) → Ausgabe nach `dist/`, dann `clasp push` aus `dist/`.
- Wichtig beim Bündeln: Funktionen, die als Trigger, Menüeintrag, `google.script.run` oder Web-App-Handler dienen, müssen als **globale Funktionsdeklarationen** im Output landen (z. B. `globalThis.onOpen = onOpen` bzw. Plugin-Option). Kein ES-Module-Output (`format: 'iife'` oder `'esm'` ohne Export-Statements).
- `tsconfig.json`: `"target": "ES2020"`, `"lib": ["ES2020"]`, `"types": ["google-apps-script"]`, `"module": "ESNext"`, `"moduleResolution": "Bundler"`.
- npm-Pakete funktionieren nur, wenn sie **reines JS ohne Node-APIs** sind (kein `fs`, kein `http`, kein `Buffer` ohne Polyfill).

## 5. Linting und Formatierung

- **Biome** (wie im offiziellen Samples-Repo): `npx @biomejs/biome check --write .`. In `biome.json` Globals wie `SpreadsheetApp` erlauben oder `noUndeclaredVariables` abschalten und auf TS-Typen setzen.
- Alternativ ESLint mit `eslint-plugin-googleappsscript` (Globals) + Prettier.

## 6. Testen

- **Lokal mit Node (Vitest/Jest)**: reine Logik in Funktionen ohne Dienstaufrufe kapseln (z. B. `transformRows(rows)`) und lokal testen. Dienste per Mock injizieren (`function syncData(sheetApp = SpreadsheetApp) {…}`) oder globale Mocks (`globalThis.SpreadsheetApp = { … }`). Tools: **gas-local**, **gas-mock-globals**.
- **In Apps Script**: Test-Bibliotheken wie **gast** (TAP) oder **QUnitGS2**. Eigene `test_*`-Funktionen (wie im Samples-Repo: `advanced/test_*.gs`).
- Trigger testen mit simulierten Event-Objekten:
```javascript
function test_onEdit() {
  const sheet = SpreadsheetApp.getActive().getSheetByName("Data");
  onEdit({ range: sheet.getRange("B2"), value: "done", oldValue: "", source: SpreadsheetApp.getActive() });
}
```
- Eine Kopie der Produktivdaten (Test-Sheet-ID in Script Properties) verwenden, niemals gegen Produktionsdaten testen.

## 7. Weitere Werkzeuge
- **Apps Script API** (`script.googleapis.com`): Projekte per Code anlegen und ändern, Funktionen remote ausführen (`scripts.run`, erfordert API-Executable-Deployment und dasselbe GCP-Projekt).
- **Apps Script-Dashboard**: https://script.google.com/home (Projekte, Ausführungen, Trigger aller Skripte).
- Editor-Erweiterungen der Community: AppsScriptColor (Dark Theme), Apps Script GitHub Assistant (Push/Pull aus dem Browser-Editor direkt nach GitHub, ohne clasp).
- **Gemini im Apps Script-Editor bzw. Gemini Code Assist** und **Gemini CLI** mit `GEMINI.md`-Kontextdatei im Repo (wie im offiziellen Samples-Repo).
