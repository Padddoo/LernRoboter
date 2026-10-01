# Projektplan: Lernstudio (LernRoboter)

> Grundlage: `SPEC_15.md`. Dieses Dokument fasst die Spec zusammen, ergänzt Architekturdetails, benennt Lücken und Risiken und legt die konkrete Vorgehensweise fest (Phasen, Aufgaben, Definition of Done).

---

## 1. Projektübersicht

### 1.1 Worum geht es?

Eine **private, familieninterne Web-App**. Die Kinder geben einen konkreten Lernstoff ein (Text, PDF, Foto, oder ein Kapitel aus der Admin-Bibliothek). Daraus erzeugt die App **einmal** ein strukturiertes **Lernpaket** (JSON). Alle Lernformate lesen dieses Lernpaket:

| # | Produkt | Charakter | Kostentreiber | Meilenstein |
|---|---|---|---|---|
| 1 | Karteikarten (FSRS + Anki/CSV-Export, Selbst-Modus) | Festigung | keiner (nach Generierung) | M2 |
| 2 | Abfrage-Spiel (Text, später Sprache) | Aktives Abrufen | schnelles LLM, später STT/TTS | M2 / M3 |
| 3 | Interaktive Präsentation (Folien + Narration, unterbrechbar) | Verstehen, Fragen stellen | TTS (einmalig), LLM + STT/TTS pro Frage | M3 |
| 4 | Video-Overview (MP4, Folien + Stimme) | Überblick, offline | TTS-Zeichen | M4 |
| 5 | PDF-Export | Nachschlagen/Drucken | keiner | M5 |

Empfohlene Lernreihenfolge in der UI: **Video → Interaktiv → Abfrage → Karten**. Die *Bau*-Reihenfolge ist umgekehrt (billig/textbasiert zuerst, Voice und Video später), weil Text-Lernen schnell Nutzen bringt und Voice/Video die größten Risiken tragen.

### 1.2 Nutzer und Rollen

- **Admin (Toby)**: Bibliothek (Markdown mit Frontmatter), Profile, Guardrails, Budgets, Render-Freigaben, Nutzungsübersicht.
- **Kind (Fred 11, Sophia 16, weitere)**: Lernpakete erzeugen und nutzen, Bibliothek lesen (gefiltert über `visible_to`).

### 1.3 Leitprinzipien

1. **Einmal generieren, viele Ausgabeformate.** Das starke Modell wird pro Quelle einmal aufgerufen; Dialoge laufen über das schnelle Modell.
2. **Grounding auf die Quelle** ist die wichtigste Guardrail. Inhalte müssen über `sourceRefs` belegbar sein.
3. **Alles Konfigurierbare ist Konfiguration** (Provider, Modelle, Guardrails, Budgets, Preise) – nichts hart verdrahtet.
4. **Privat by default**: Login-Pflicht, keine öffentlichen Datei-URLs, Uploads sind Daten, keine Befehle.
5. **Latenz ist messbar**: Jede Voice-Teilstrecke wird geloggt.

### 1.4 Übergreifendes Akzeptanzkriterium

Foto eines Arbeitsblatts → erstes Abfrage-Spiel in **< 2 Minuten**. Video darf im Hintergrund länger brauchen.

---

## 2. Analyse der Spec: Lücken, Widersprüche, Präzisierungen

Diese Punkte sollten vor bzw. während M1 geklärt/umgesetzt werden. Vorschläge sind so gewählt, dass die Spec im Kern unverändert bleibt.

### 2.1 Das 2-Minuten-Kriterium ist mit *einem* Generierungsaufruf gefährdet

Ein vollständiges Lernpaket (bis 20 Folien à 30–90 s Narration ≈ bis zu 3.000+ Wörter, dazu 40 Karten, 25 Quizfragen) bedeutet grob **10–20k Output-Tokens**. Zusammen mit Foto-Transkription und Korrekturschritt wird das knapp oder reißt die 2 Minuten.

**Vorschlag: zweistufige Generierung**, ohne das Datenmodell zu ändern:

1. **Stufe A (schnell, blockierend):** `meta`, `learningGoals`, `keyTerms`, `flashcards`, `quizItems`, `misconceptions`. → Karten und Abfrage-Spiel sind sofort nutzbar.
2. **Stufe B (asynchron, im Worker):** `slides` inkl. `narration`, danach TTS und Video.

Das Lernpaket bekommt dafür einen Status pro Teil (`core: ready`, `slides: pending|ready|failed`). Prompt-Caching der Quelle macht den zweiten Aufruf günstig. Alternativ/zusätzlich: Streaming der Ausgabe.

### 2.2 Duo-Modus passt nicht zum Datenmodell

`speaker` hängt laut Spec an der **Folie**, der Duo-Modus braucht aber Sprecherwechsel **innerhalb** einer Folie („pro Narrationsabschnitt“).

**Vorschlag:** `narration` bleibt ein String (Solo-Modus), zusätzlich optional

```ts
narrationSegments?: { speaker: "A" | "B"; text: string }[];
```

Das TTS-/Video-Modul verwendet `narrationSegments`, wenn vorhanden, sonst `narration` mit Stimme A. Dadurch muss M3/M4 nicht umgebaut werden, wenn M5 den Duo-Modus bringt.

### 2.3 Weitere Präzisierungen

| Thema | Problem | Vorschlag |
|---|---|---|
| Modell-IDs | Spec nennt `claude-sonnet-5`; IDs ändern sich. | Nur per `.env` (`AI_MODEL_STRONG`, `AI_MODEL_FAST`, `AI_MODEL_VISION`), vor Implementierung gegen aktuelle Anbieter-Doku prüfen (Spec §5 fordert das ohnehin). |
| JSON-Ausgabe | Freies JSON großer Pakete ist fehleranfällig. | Structured Output / Tool-Use mit JSON-Schema (aus Zod generiert, `zod-to-json-schema`), danach Zod-Validierung + ein Reparaturversuch (Spec §6.1). |
| Diagramme | LLM-generiertes Mermaid/SVG rendert oft nicht. | Serverseitig parsen (Mermaid) bzw. SVG sanitizen (DOMPurify); bei Fehler Folie auf `bullets` zurückfallen lassen. SVG **nie** ungefiltert einbetten (XSS). |
| `sourceRefs` | „Absätze/Zitate“ ist unscharf. | Quelle beim Normalisieren in nummerierte Absätze zerlegen (`P1…Pn`); `sourceRefs` referenziert IDs. Prüfbar, kürzer, und das UI kann die Stelle hervorheben. |
| Urheberrecht | Export-Sperre/-Hinweis für Bibliotheksinhalte braucht Herkunft. | `sources.origin = "library" | "kid_upload"`; Lernpaket erbt das Flag, Export-Routen prüfen es. |
| `visible_to` | Nutzt Namen. | Auf Profil-`slug` mappen; unbekannte Slugs beim Import als Warnung loggen. |
| Quellenlänge | „15–20 Seiten“ ist kein messbares Limit. | Limit in Zeichen/Tokens konfigurieren (z. B. `SOURCE_MAX_CHARS=60000`), UI zeigt Füllstand. |
| PIN-Auth | 4-stellige PINs sind leicht zu raten. | Rate-Limit + Sperre nach Fehlversuchen; Cloudflare Access davor (Spec-Empfehlung) als eigentliche Schutzschicht. |
| Krisensignale | Erkennung nicht spezifiziert. | Im System-Prompt des Dialogmodells als Verhaltensregel; zusätzlich strukturiertes Flag in der Antwort (`safety_event: true`) → Event in DB, Admin-Benachrichtigung nur bei `parentLogAccess` (Spec §9.1). |
| Foto-Löschung | 30-Tage-Regel braucht einen Job. | Täglicher Cleanup-Job im Worker (`RETENTION_DAYS_PHOTOS`). |
| Anki-Export | `.apkg` ist ein ZIP mit SQLite-DB; vorhandene npm-Pakete sind alt. | Eigene kleine Implementierung mit `better-sqlite3` + `archiver` (Anki-Schema „collection.anki2“), CSV als einfacher Fallback zuerst. |

---

## 3. Zielarchitektur

### 3.1 Komponenten

```
                ┌───────────────── Docker Compose (Heimserver) ─────────────────┐
 Browser ──► Cloudflare Tunnel/Access ──► app (Next.js)          worker (Node)    │
 (Handy,                                   ├─ UI (App Router)      ├─ Job-Queue (SQLite)
  Tablet)                                  ├─ Route-Handler/API    ├─ Lernpaket Stufe B
                                           ├─ Auth (Session)       ├─ TTS-Batch + Cache
                                           ├─ lib/ai (Provider)    ├─ Video-Render (Playwright+ffmpeg)
                                           └─ Voice-Endpoints      ├─ Library-Watcher
                                                (SSE/WebSocket)    └─ Cleanup/Retention
                                                     │                    │
                                                     └──── data/ (Volume): db.sqlite, uploads/, audio/, video/
```

### 3.2 Repo-Struktur (Vorschlag)

```
app/                      Next.js App Router
  (auth)/login            Profilauswahl, PIN / Admin-Passwort
  (kid)/                  Start, Neues Thema, Bibliothek, Meine Themen
  (kid)/paket/[id]/       Lernpaket-Seite + video | interaktiv | abfrage | karten
  admin/                  Bibliothek, Profile, Budgets, Freigaben, Nutzung
  api/                    Route-Handler (Upload, Generierung, Media, Voice, Export)
components/
  slides/                 Folien-Templates (geteilt: Browser + Video-Render)
lib/
  ai/                     Provider-Abstraktion: llm.ts, vision.ts, tts.ts, stt.ts + Adapter
  prompts/                Generierungs-, Dialog-, Bewertungs-Prompts (versioniert)
  schema/                 Zod-Schemas (Lernpaket, Guardrails, Frontmatter)
  db/                     Drizzle-Schema, Migrations, Queries
  ingest/                 Text-, PDF-, Foto-, Markdown-Normalisierung
  guardrails/             Profil-Konfiguration → Prompt-Bausteine
  usage/                  Kosten-Tracking, Budget-Prüfung
  srs/                    FSRS-Anbindung (ts-fsrs)
  export/                 Anki (.apkg), CSV, später PDF
  notify/                 Telegram
worker/                   Separater Node-Prozess (Queue-Loop, Jobs)
library/                  Admin-Markdown (per Watcher importiert, nicht im Repo eingecheckt)
data/                     Laufzeitdaten (gitignored)
tests/                    Unit (vitest), E2E (Playwright)
docker-compose.yml, Dockerfile.app, Dockerfile.worker, .env.example
```

### 3.3 Provider-Abstraktion (`lib/ai/`)

Schmale Interfaces, damit die offenen Entscheidungen (TTS, STT, Realtime) austauschbar bleiben:

```ts
interface LLMProvider   { generateStructured<T>(req, schema): Promise<T>; streamText(req): AsyncIterable<string>; }
interface VisionProvider{ transcribe(images: Buffer[], hints?): Promise<string>; }
interface TTSProvider   { synthesize(text, voice): Promise<AudioResult>; stream(text, voice): AsyncIterable<Uint8Array>; }
interface STTProvider   { transcribe(audio, { keyTerms }): Promise<string>; stream?(...): ... }
```

Jeder Aufruf meldet Verbrauch (Tokens, Zeichen, Sekunden) an `lib/usage`. Preise pro Einheit in einer Konfigurationstabelle, nicht im Code.

### 3.4 Datenbank (Drizzle + SQLite, WAL-Modus)

| Tabelle | Wichtige Spalten |
|---|---|
| `profiles` | id, slug, name, role, pinHash, grade, guardrails (JSON), monthlyBudgetEur, requireVideoApproval, voicePrefs |
| `sources` | id, profileId, origin (`library`/`kid_upload`), kind (text/pdf/photo/md), title, subject, grade, markdown, paragraphs (JSON), libraryPath, createdAt |
| `uploads` | id, sourceId, path, mime, deleteAfter |
| `lernpakete` | id, sourceId, profileId, meta (JSON), data (JSON), coreStatus, slidesStatus, promptVersion, model |
| `media` | id, lernpaketId, kind (slide_audio/video/vtt/pdf), slideId, voice, textHash, path, durationMs |
| `render_jobs` | id, type, lernpaketId, status (`queued|tts|rendering|done|failed|awaiting_approval`), progress, error, attempts |
| `flashcard_reviews` | profileId, cardId, lernpaketId, fsrsState (JSON), due, lastReview |
| `own_flashcards` | Selbst-Modus-Karten des Kindes + KI-Feedback |
| `quiz_sessions` | id, profileId, lernpaketId, mode, answers (JSON), score, weakSpots |
| `chat_logs` | id, profileId, lernpaketId, context (präsentation/quiz), role, text, latencies (JSON), safetyEvent |
| `usage` | profileId, month, category (llm/tts/stt/vision), units, costEur |

TTS-Cache-Schlüssel: `hash(provider + voice + text)` → wird von Video **und** interaktiver Präsentation wiederverwendet (Spec §10).

---

## 4. Vorgehensweise

### Phase 0 – Setup und Spikes (vor M1, ca. 2–3 Tage)

Ziel: Projektgerüst steht, die größten Unsicherheiten sind mit Messwerten belegt.

**Setup**
- [ ] Next.js (App Router, TS strict), Tailwind (Dark Mode, mobil zuerst), ESLint/Prettier, vitest, Playwright.
- [ ] Drizzle + better-sqlite3, erste Migration, `data/` gitignored.
- [ ] `.env.example` mit allen Schlüsseln (Platzhalter), Zod-validierte Env-Konfiguration.
- [ ] Dockerfile(s) + `docker-compose.yml` (`app`, `worker`, Volume `data/`), lokal lauffähig.
- [ ] CI (GitHub Actions): Lint, Typecheck, Unit-Tests.

**Spikes (Wegwerf-Skripte unter `spikes/`)**
- [ ] **Generierungsdauer**: Eine echte Quelle (z. B. 2 Seiten Biologie) → Lernpaket mit aktuellem Sonnet-Modell, Zeit und Tokens messen. Entscheidet über die zweistufige Generierung (§2.1).
- [ ] **Foto-Transkription**: 3–5 echte Fotos (Handschrift, Formeln, Tafelbild) → Qualität und Dauer.
- [ ] **TTS-A/B-Test** (Offene Entscheidung 2): dieselbe Narration mit ElevenLabs, Grok Voice, OpenAI TTS; Fred hört. Kosten pro 1.000 Zeichen notieren.
- [ ] **Voice-Latenz grob**: STT → Haiku-Stream → satzweise TTS, `stt_ms / llm_first_token_ms / tts_first_byte_ms` messen.

**Ergebnis**: Kurze Notiz `docs/entscheidungen.md` mit Messwerten und getroffenen Entscheidungen (ADR-Stil).

### M1 – Fundament

| Bereich | Aufgaben |
|---|---|
| Auth & Profile | Profilauswahl, PIN pro Kind (gehasht, Rate-Limit), Admin-Passwort, Session-Cookie (httpOnly, `iron-session` o. ä.), Middleware schützt alles außer Login. Seed für Fred/Sophia mit Guardrail-Defaults aus Spec §9. |
| Eingabe | Formular mit Fach, Klassenstufe (vorbelegt), Anlass, Testdatum. Text-Paste; PDF-Upload mit `pdfjs-dist` (leer → Seiten rasterisieren → Foto-Pfad); Foto-Upload (mehrere, `capture`), Vision-Transkription, **Korrekturschritt** (editierbarer Markdown-Text). |
| Normalisierung | `normalizeSource()` → Markdown + nummerierte Absätze, Längenlimit mit verständlichem Hinweis. |
| Bibliothek | MD-Upload im Admin + Watcher auf `library/` (chokidar im Worker), Frontmatter per Zod validiert, Filter nach Fach/Klasse, `visible_to`. |
| Generierung | Prompt (§6.2 + Guardrail-Bausteine), Structured Output, Zod-Validierung, 1 Reparaturversuch; ggf. zweistufig (§2.1). Uploads im Prompt klar als Daten markiert (Injection-Schutz). |
| Lernpaket-Seite | Vier Kacheln in Lernreihenfolge mit Status-Badges; Ansicht von Lernzielen, Begriffen, Folien (statisch), Karten, Fragen. |
| Infrastruktur | Media-Route-Handler mit Auth-Prüfung (Besitz/Rolle), Usage-Tracking ab dem ersten LLM-Aufruf. |

**DoD M1:** Kind loggt sich ein, fotografiert ein Arbeitsblatt, korrigiert den Text, erhält ein validiertes Lernpaket; Admin importiert ein MD-Kapitel, das nur für die in `visible_to` genannten Kinder sichtbar ist. Unit-Tests für Schemas, Normalisierung, Frontmatter; ein E2E-Test für den Foto-Flow mit gemocktem Provider.

### M2 – Text-Lernen

| Bereich | Aufgaben |
|---|---|
| Karteikarten | Lernansicht mit `ts-fsrs` (Again/Hard/Good/Easy, Tasten + Swipe), Fälligkeiten pro Profil, Fortschritt. |
| Export | CSV zuerst, dann `.apkg`; Export aus Bibliotheksquellen nur eingeloggt + Hinweis „nur für den privaten Gebrauch“. |
| Selbst-Modus | Kind schreibt Karten; schnelles Modell gibt **nur** Qualitäts-Feedback (atomar? Antwort in Frage?), schreibt nicht um. Generierte Karten erst nach eigenem Entwurf sichtbar. |
| Abfrage-Spiel (Text) | Rundenlogik als reine, testbare State-Machine: Frage → Antwort → Bewertung (`richtig/teilweise/falsch` + Feedback, strukturiert) → bei „teilweise“ Hinweis + 2. Versuch → Auflösung. Falsche Fragen kommen später wieder, `misconceptions` gezielt, Boss-Frage, Punkte/Streak, Ergebnis mit Schwachstellen. |
| Guardrails | Dialog-System-Prompt aus `GuardrailProfile` zusammengesetzt (Sprache, Satzlimit, Off-Topic-Policy, Socratic Hints, Grounding), Krisen-Regel, `safety_event`. |
| Budget | 80 %-Hinweis, 100 %-Sperre (Voice/Video), Text bleibt verfügbar. |

**DoD M2:** Akzeptanzkriterium „< 2 Minuten bis zum ersten Abfrage-Spiel“ messbar erfüllt (Text-Modus). Unit-Tests für Quiz-State-Machine, FSRS-Anbindung, Anki-Export (Datei in Anki importierbar, manuell geprüft). Kleines Eval-Set (10–20 Antworten mit erwarteter Bewertung) für den Bewertungs-Prompt.

### M3 – Voice

| Bereich | Aufgaben |
|---|---|
| TTS | Provider-Adapter (Gewinner aus Spike), Narration aller Folien im Worker vorproduzieren, Cache über `textHash`. |
| Interaktive Präsentation | Folien-Komponenten (`components/slides`, 16:9, auch Basis für Video), Audio-Player mit Fortschritt, Navigation; „Frage stellen“ per Button und Push-to-Talk. |
| Voice-Pipeline | Audio-Aufnahme (MediaRecorder) → STT mit `keyTerms` als Hinweis → Haiku-Stream (Prompt-Caching: Quelle + Lernpaket, kurze `max_tokens`) → **satzweise** an Streaming-TTS → Wiedergabe per MediaSource/Web Audio. Transport: SSE oder WebSocket. Transkript-Spalte. |
| Abfrage-Spiel (Sprache) | Gleiche State-Machine wie M2, nur Ein-/Ausgabe über Voice. |
| Messung | `stt_ms`, `llm_first_token_ms`, `tts_first_byte_ms`, Gesamt pro Turn in `chat_logs`; Admin-Ansicht mit Perzentilen (p50/p95). |

**DoD M3:** p50 „Ende der Frage → erste Silbe“ < 1,5 s auf dem Heimserver über Handy-WLAN; Fallback auf Web Speech API konfigurierbar. Danach Entscheidung zu Realtime-S2S (Offene Entscheidung 4) anhand der Messwerte.

### M4 – Video

| Bereich | Aufgaben |
|---|---|
| Worker | Queue-Loop auf `render_jobs` (atomares Claimen, Retry mit Backoff, Abbruch-sicher), Status `queued → tts → rendering → done/failed`. |
| Rendering | Playwright öffnet interne, tokengeschützte Render-Route je Folie → PNG 1920×1080; ffmpeg: Bild + Audio pro Folie mit exakter Dauer, Crossfade, Concat; Untertitel als `.vtt` (aus Narration + Audiodauern), optional eingebrannt. Längen-Presets kurz/normal/ausführlich steuern die Narration bei der Generierung. |
| UI | Fortschritt per SSE/Polling, Player mit Untertitel-Toggle, Download (MP4) für Offline. |
| Freigaben & Benachrichtigung | `requireVideoApproval` → Status `awaiting_approval`, Admin-Freigabe; Telegram-Bot für Freigaben, Budget-Schwellen und (nur bei `parentLogAccess`) Safety-Events. |
| Retention | Täglicher Cleanup (Fotos nach 30 Tagen, verwaiste Medien). |

**DoD M4:** 10-Minuten-Video aus 12 Folien rendert stabil im Docker-Container; Abbruch/Neustart des Workers verliert keine Jobs; Kosten des Renders erscheinen in `usage`.

### M5 – Feinschliff

- Duo-Modus (`narrationSegments`, zwei Stimmen, Toggle beim Erzeugen).
- Barge-in/VAD (z. B. `@ricky0123/vad-web`), Narration pausiert bei Sprachbeginn.
- Prüfer-Modus (mündliche Prüfungssimulation mit Nachfragen).
- PDF-Export (Folien + Sprechernotizen, druckbare Karten) über Playwright `page.pdf()`.

---

## 5. Querschnittsthemen

### 5.1 Teststrategie

- **Unit (vitest):** Zod-Schemas, Normalisierung, Frontmatter, Guardrail-Prompt-Bau, Quiz-State-Machine, FSRS, Budget-Logik, Anki-Export.
- **Provider-Mocks:** Alle `lib/ai`-Interfaces haben Fake-Implementierungen mit festen Fixtures → Tests und CI ohne API-Kosten.
- **E2E (Playwright):** Login → Eingabe → Lernpaket → Abfrage-Spiel (gemockt), mobile Viewports.
- **Prompt-Evals:** Kleine, versionierte Sets (Generierung: Grounding/Atomarität; Bewertung: richtig/teilweise/falsch; Guardrails: Off-Topic, Prompt-Injection im Foto, Krisensignal). Bei Prompt-Änderungen manuell/halbautomatisch ausführen.

### 5.2 Sicherheit & Datenschutz (Checkliste)

- [ ] Middleware: jede Route außer Login authentifiziert; Media-Handler prüft Besitz bzw. Sichtbarkeit.
- [ ] Dateien unter `data/`, nie unter `public/`; Pfade nie aus User-Input zusammensetzen (IDs → DB → Pfad).
- [ ] Upload-Limits (Größe, MIME-Sniffing), SVG/Mermaid sanitizen.
- [ ] Uploads im Prompt als Daten gekapselt; Injection-Evals.
- [ ] `.env` gitignored, `.env.example` gepflegt; Anbieter mit Zero-Retention/ohne Training konfigurieren.
- [ ] Chat-Logs nur für Admin sichtbar, wenn `parentLogAccess` aktiv; Statistiken immer.

### 5.3 Betrieb

- Docker Compose auf dem Heimserver hinter Cloudflare Tunnel + Access.
- Backup von `data/` (SQLite per `.backup`/Litestream, Medien per rsync/restic).
- Logs strukturiert (pino), Healthcheck-Endpunkte für `app` und `worker`.

---

## 6. Risiken

| Risiko | Wahrscheinlichkeit | Auswirkung | Gegenmaßnahme |
|---|---|---|---|
| Generierung zu langsam für 2-Minuten-Ziel | hoch | hoch | Zweistufige Generierung (§2.1), Streaming, Spike in Phase 0 |
| Voice-Latenz > 1,5 s | mittel | hoch | Satzweises TTS-Streaming, Prompt-Caching, Messung ab Tag 1, Realtime-Option nach M3 |
| Ungültiges/halluziniertes JSON | mittel | mittel | Structured Output, Zod, Reparaturversuch, `sourceRefs` auf Absatz-IDs |
| Schlechte Foto-Transkription (Handschrift) | mittel | mittel | Pflicht-Korrekturschritt, Spike mit echten Fotos |
| TTS-Kosten laufen davon | mittel | mittel | Cache, Budgets, Video-Freigabe, Kosten pro Render anzeigen |
| Video-Rendering instabil im Container | mittel | niedrig | Isolierter Worker, Retries, Video ist „nice to have“ gegenüber Text-Lernen |
| Kaputte Diagramme | hoch | niedrig | Validierung + Fallback auf Bullets |
| Prompt-Injection über Uploads | niedrig | mittel | Daten-Kapselung, klare Rolle, Evals |

---

## 7. Offene Entscheidungen – Empfehlung und Zeitpunkt

| # | Entscheidung | Empfehlung | Spätestens |
|---|---|---|---|
| 1 | Hosting | Heimserver (Docker Compose + Cloudflare Tunnel). Vercel würde externen Worker, Blob-Storage und gehostete DB erfordern. | Phase 0 |
| 2 | TTS-Anbieter | A/B-Test im Spike; Adapter-Interface macht späteren Wechsel billig. | Ende Phase 0 |
| 3 | STT | Serverseitig als Default (Fachbegriffe via `keyTerms`), Web Speech API als konfigurierbarer Fallback. | Beginn M3 |
| 4 | Realtime-S2S | Erst nach M3-Latenzmessungen; nur falls p50 deutlich über 1,5 s bleibt. | Ende M3 |
| 5 | Budget & Video-Freigabe | Start z. B. 10 €/Monat/Kind, Freigabe aus, über Admin-UI anpassbar. | M2 (Budget), M4 (Freigabe) |
| 6 | `parentLogAccess` | Fred `true`, Sophia `false` (wie Spec), mit beiden Kindern transparent besprechen. | vor Produktivnutzung M2 |

---

## 8. Nächste konkrete Schritte

1. Offene Entscheidung 1 (Hosting) bestätigen.
2. Phase 0 starten: Projektgerüst, Docker Compose, CI.
3. Spikes „Generierungsdauer“ und „Foto-Transkription“ mit echtem Material durchführen (braucht API-Key und 2–3 Beispielquellen/-fotos).
4. Auf Basis der Messwerte über die zweistufige Generierung entscheiden und M1 beginnen.
