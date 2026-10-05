# Ressourcen, Linksammlung und Sample-Index

## Offizielle Dokumentation (Deutsch: `?hl=de` anhängen)

| Thema | URL |
|---|---|
| Apps Script Startseite | https://developers.google.com/apps-script |
| Referenz aller Dienste | https://developers.google.com/apps-script/reference |
| Quotas | https://developers.google.com/apps-script/guides/services/quotas |
| Trigger | https://developers.google.com/apps-script/guides/triggers |
| Web-Apps | https://developers.google.com/apps-script/guides/web |
| HTML Service | https://developers.google.com/apps-script/guides/html |
| Manifest | https://developers.google.com/apps-script/manifest |
| Erweiterte Dienste | https://developers.google.com/apps-script/guides/services/advanced |
| Best Practices | https://developers.google.com/apps-script/guides/support/best-practices |
| clasp | https://developers.google.com/apps-script/guides/clasp |
| Gmail API | https://developers.google.com/workspace/gmail/api/guides |
| Calendar API | https://developers.google.com/workspace/calendar/api/guides/overview |
| CalDAV | https://developers.google.com/workspace/calendar/caldav/v2/guide |
| Chat Überblick | https://developers.google.com/workspace/chat/overview |
| Chat REST-Referenz | https://developers.google.com/workspace/chat/api/reference/rest |
| Chat Samples | https://developers.google.com/workspace/chat/samples |
| Docs | https://developers.google.com/workspace/docs · API: /workspace/docs/api/how-tos/overview |
| Drive | https://developers.google.com/workspace/drive · API: /workspace/drive/api/guides/about-sdk |
| Drive Activity v2 | https://developers.google.com/workspace/drive/activity/v2 |
| Drive Labels | https://developers.google.com/workspace/drive/labels/guides/overview |
| Google Picker | https://developers.google.com/workspace/drive/picker/guides/overview |
| Keep API | https://developers.google.com/workspace/keep/api/guides · REST: /workspace/keep/api/reference/rest |
| Forms | https://developers.google.com/workspace/forms · API: /workspace/forms/api/guides |
| Meet REST API | https://developers.google.com/workspace/meet/api/guides/overview |
| Meet Media API | https://developers.google.com/workspace/meet/media-api/guides/overview |
| Meet-Add-ons | https://developers.google.com/workspace/meet/add-ons/guides/overview |
| Sheets | https://developers.google.com/workspace/sheets · Konzepte: /workspace/sheets/api/guides/concepts |
| Slides | https://developers.google.com/workspace/slides · API: /workspace/slides/api/guides/overview |
| Tasks | https://developers.google.com/workspace/tasks/overview |
| Vault | https://developers.google.com/workspace/vault |
| Add-on-Samples | https://developers.google.com/workspace/add-ons/samples (Filter `product=googledocs`, `googleslides` …) |
| Travel Concierge (KI-Add-on) | https://developers.google.com/workspace/add-ons/samples/travel-concierge |
| Fact-Check Custom Function | https://developers.google.com/apps-script/samples/custom-functions/fact-check |
| OAuth-Scopes (alle Google-APIs) | https://developers.google.com/identity/protocols/oauth2/scopes |
| APIs Explorer (Requests testen) | im jeweiligen REST-Referenz-Eintrag „Try it“ |

## GitHub

- **Offizielle Samples**: https://github.com/googleworkspace/apps-script-samples
- **OAuth2-Bibliothek**: https://github.com/googleworkspace/apps-script-oauth2 (Library-ID `1B7FSrk5Zi6L1rSxxTDgDEUsPzlukDsi4KGuTMorsTQHhGBzBkMun4iDF`)
- **clasp**: https://github.com/google/clasp
- **Justin Poehnelt (Google DevRel) – Monorepo**: https://github.com/jpoehnelt/apps-script (`mjml-apps-script` für E-Mail-Templates, Gmail-KI-Agent mit Vertex AI, pnpm + Turbo + Biome + Changesets). Blog: https://justin.poehnelt.com (viele Apps-Script-Tipps: Performance, TypeScript, Bundling, Vertex AI)
- **Awesome-Liste (Amit Agarwal / labnol)**: https://github.com/labnol/google-apps-script-awesome
- **Topics**: https://github.com/topics/google-apps-script · /topics/google-script · /topics/google-app-scripts
- Typen: `@types/google-apps-script` (DefinitelyTyped)

## Community-Bibliotheken (Auswahl aus der Awesome-Liste)

| Kategorie | Bibliothek | Zweck |
|---|---|---|
| Auth | apps-script-oauth2 / oauth1, cGoa | OAuth für Drittanbieter-APIs und Dienstkonten |
| Sheets-ORM | Sheetfu, Tamotsu, Goodel | Sheet wie eine Datenbanktabelle nutzen |
| Utilities | lodashgs, gas-underscore | Lodash/Underscore |
| Web | Gexpress | Express-ähnliches Routing für Web-Apps |
| Parsing | cheeriogs, htmlparser2-Port | HTML parsen (jQuery-Syntax) |
| Bilder | ImgApp | Bildgröße ermitteln, zuschneiden |
| Datenbank | FirebaseApp | Firebase Realtime DB |
| Logging | BetterLog, BBLog | Logs in Sheet oder Firebase |
| Testing | gast, GSUnit, QUnitGS2 | Tests in Apps Script |
| Tooling | clasp, gas-local, gas-webpack-plugin, apps-script-starter, gas-minimal-boilerplate | lokale Entwicklung, Build |
| Editor | AppsScriptColor | Dark Theme, Ordner im Editor |

Weitere Lern- und Hilfe-Quellen: Digital Inspiration / labnol.org (Amit Agarwal), Desktop Liberation (Bruce Mcpherson), Ben Collins (Sheets/Apps Script), Spreadsheet Dev, GreenFlux Blog (z. B. „So you want to send JSON to a Google Apps Script Web App“, „Extract images from a Google Doc and save to Drive folder“), Stack Overflow Tag `google-apps-script`, Google-Groups-Forum „Google Apps Script Community“, Google Developers Discord.

## Index: offizielles Samples-Repo (googleworkspace/apps-script-samples)

| Ordner | Inhalt |
|---|---|
| `advanced/` | Snippets für alle erweiterten Dienste: adminSDK, analytics, bigquery, calendar, chat, classroom, docs, drive, driveActivity, driveLabels, gmail, people, sheets, slides, tagManager, tasks, youtube (+ `test_*.gs`) |
| `ai/` | autosummarize, custom-func-ai-agent, custom-func-ai-studio, custom_func_vertex, devdocs-link-preview, drive-rename, email-classifier, gmail-sentiment-analysis, standup-chat-app |
| `solutions/automations/` | agenda-maker, aggregate-document-content, bracket-maker, calendar-timesheet, content-signup, course-feedback-response, employee-certificate, equipment-requests, event-session-signup, feedback-sentiment-analysis, folder-creation, generate-pdfs, import-csv-sheets, mail-merge, news-sentiment, offsite-activity-signup, tax-loss-harvest-alerts, timesheets, upload-files, vacation-calendar, youtube-tracker |
| `solutions/custom-functions/` | calculate-driving-distance, summarize-sheets-data, tier-pricing |
| `solutions/add-on/` | book-smartchip (Smart Chips), share-macro |
| `solutions/editor-add-on/` | clean-sheet |
| `solutions/ooo-assistant/` | Abwesenheits-Assistent (Chat) |
| `solutions/webhook-chat-app/` | Chat-Webhook, Thread-Antworten |
| `sheets/` | api, customFunctions, dateAddAndSubtract, forms, maps, removingDuplicates, quickstart |
| `docs/` | cursorInspector, dialog2sidebar, translate, quickstart |
| `slides/` | api, imageSlides, progress (Fortschrittsbalken), selection, SpeakerNotesScript, style, translate |
| `gmail/` | add-ons, inlineimage, markup, sendingEmails, quickstart |
| `drive/` | activity, activity-v2, quickstart |
| `forms/`, `forms-api/` | Benachrichtigungen; Forms-REST-API-Demos und Snippets |
| `calendar/`, `tasks/`, `people/`, `classroom/`, `chat/` | Quickstarts und Snippets |
| `adminSDK/` | directory, reports, reseller |
| `templates/` | Startvorlagen: custom-functions, docs-addon, forms-addon, sheets-addon, sheets-import, standalone, web-app |
| `triggers/` | Trigger-Beispiele |
| `ui/` | communication, dialogs, forms, html, sidebar, user, webapp |
| `mashups/` | sheets2calendar, sheets2chat, sheets2contacts, sheets2docs, sheets2drive, sheets2forms, sheets2gmail, sheets2maps, sheets2slides, sheets2translate |
| `service/` | jdbc, propertyService |
| `picker/` | Google Picker in einem Dialog |
| `wasm/` | WebAssembly in Apps Script (hello-world, image-add-on, python) |
| `data-studio/` | Community Connectors für Looker Studio |
| `apps-script/execute` | Apps Script API: Funktionen remote ausführen |
| `utils/` | Logging-Helfer |

Code-Richtlinien des Repos (`GEMINI.md`): V8-Syntax, `const`/`let`, Arrow Functions, Destructuring, Template Literals, JSDoc-Typen, Biome für Lint und Format (`pnpm lint`, `pnpm format`), Snippet-Tags `[START …]`/`[END …]` nicht verändern, keine Namenskollisionen zwischen Samples.
