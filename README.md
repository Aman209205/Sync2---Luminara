# Luminara - Sync2

### Understand the lecture. Learn your way.

**BOB Hacks'26 — Problem Statement 1: The Smart Classroom**
Native **Android** application (Kotlin + Jetpack Compose) with a **Python FastAPI** backend.

> A lecture is not only speech. Luminara understands what the teacher *says*, what they *write* and
> what they *draw*, fuses those into one lecture object, and turns it into multilingual study
> material plus an AI agent that can only answer from that lecture.

---

## Contents

1. [The problem](#1-the-problem)
2. [The Luminara solution](#2-the-luminara-solution)
3. [Key features](#3-key-features)
4. [Student and teacher flows](#4-student-and-teacher-flows)
5. [Recorded lectures](#5-recorded-lectures)
6. [Live Class](#6-live-class)
7. [The multimodal AI pipeline](#7-the-multimodal-ai-pipeline)
8. [IBM BOB integration](#8-ibm-bob-integration)
9. [Architecture](#9-architecture)
10. [Data model](#10-data-model)
11. [API surface](#11-api-surface)
12. [Setup](#12-setup)
13. [Configuration](#13-configuration)
14. [Demo](#14-demo)
15. [Deployment](#15-deployment)
16. [Testing and verification](#16-testing-and-verification)
17. [Honesty: what is real](#17-honesty-what-is-real)
18. [Limitations](#18-limitations)
19. [Future scope](#19-future-scope)
20. [Repository map](#20-repository-map)
21. [Third-party software](#21-third-party-software)
22. [Requirements coverage](#22-requirements-coverage)

---

## 1. The problem

A professor teaches in English. She explains ideas aloud, draws a binary search tree on the board,
writes a recurrence relation, and points at a graph. Some students in the room follow Hindi, Bangla
or Arabic far more comfortably.

Speech translation alone does not solve this, because **a lecture is not only speech.** In our demo
lecture the professor never says the recurrence relation out loud. She says:

> "I have also written the recurrence relation on the board. Please copy it into your notes."

A transcribe-and-translate product loses `T(n) = T(n/2) + O(1)` entirely. The student receives a
fluent translation of a lecture they still cannot follow.

## 2. The Luminara solution

Luminara is **multimodal lecture intelligence**.

```
Lecture input (audio / video / live class + board image or camera capture)
        │
   ┌────┴─────────────────────┐
Speech recognition      Board OCR + diagram understanding    ← two independent evidence streams
   └────┬─────────────────────┘
        ▼
   Multimodal fusion
        ▼
  LECTURE KNOWLEDGE
  (summary · concepts · terms · formulas · visual observations ·
   cross-modal links · per-claim sources)
        │
   ┌────┴──────┬──────────────┬───────────────┬─────────────┐
Structured   Translation   Lecture script  Study pack   BOB agent
  notes      (en/hi/bn/ar)   + search        (PDF)    (grounded Q&A)
```

The design rule that makes this structural rather than cosmetic: **speech and vision are analysed
separately, and the vision pass never receives the transcript.** When the app says a formula came
from the whiteboard rather than from the professor's words, that is true by construction — not a
label applied afterwards.

Delete the fusion stage and the product collapses into a transcript viewer. That is the test of
whether it is real.

## 3. Key features

| Capability | Where you see it |
|---|---|
| Speech recognition | Processing stages · Script tab · timestamped transcript |
| Classroom OCR | Visuals tab — verbatim board text |
| Diagram and graph understanding | "50 is the root node, 25 is the left child of 50, 75 is the right child" |
| Formula preservation | `T(n) = T(n/2) + O(1)` intact inside Hindi prose |
| Structured notes | Eight sectioned cards, including "What the Board Added" |
| Translation | Study material in the chosen language, technical terms kept in English |
| Lecture-grounded Q&A | Ask BOB, with tappable source chips (`Speech · 00:59`, `Whiteboard`) |
| Lecture script | Timestamped account of the class, searchable, linked to board moments |
| Study pack | A4 PDF: summary, concepts, formulas, board image, terms, full script |
| Lecture search | "where was the formula written" → the exact evidence and its source |
| My Lectures | Library with thumbnail, status, language, duration, formula count, date |
| **Live Class** | Near-real-time bilingual transcript, board capture, live BOB — then saved as an ordinary lecture |
| Classes | Teacher creates a class and a join code; students join and receive published lectures |
| Demo lecture | Bundled audio + whiteboard, always available, no account needed |

## 4. Student and teacher flows

Onboarding asks for a **role**, a name and a preferred language, and the copy follows the role
("Start learning" / "Start teaching"). Signing in is **optional** — the demo lecture, personal
uploads and live sessions all work signed out. An account is needed only to join or teach a class.

**Student**

```
Home ──▶ Live class ──▶ My classes ──▶ My lectures ──▶ Ask BOB
```

Home leads with two primary tiles — **Live class** and **Ask BOB** — then My Classes (with a Join
action), the demo lecture, and the library. Joining a class with its code makes published lectures
appear; opening one uses the same Lecture Detail everything else uses.

**Teacher**

```
Home ──▶ Start live class ──▶ My classes ──▶ Upload lecture ──▶ My lectures
```

```
Sign in → Create class (name + subject) → join code issued
        → Upload lecture (video / audio / image, into a class)
        → same processing pipeline → review in Lecture Detail → Publish
```

Publishing is the gate: a lecture starts unpublished and students cannot see it until the teacher
publishes it. Teacher-only routes reject students, non-members receive `403`, and anonymous callers
receive `401` on class lectures.

Accounts are email + password, hashed with **PBKDF2-SHA256 (240,000 rounds, per-user salt)**, with a
signed, expiring bearer token. Passwords are never returned by any endpoint.

## 5. Recorded lectures

Upload audio, video or a board image — or open the bundled demo lecture. Video and compressed audio
are normalised at the door to 16 kHz mono WAV; the pipeline itself is unchanged. If no board photo
accompanies a video, three frames are sampled and the most board-like is used, labelled as a frame
rather than a photograph.

Lecture Detail is one screen with seven tabs — **Overview, Script, Notes, Visuals, Formulas,
Ask BOB, Sources** — and every source chip is navigation: tap `Speech · 00:59` on a note or a BOB
answer and you land on that moment of the script.

## 6. Live Class

The phone records the class and posts **9-second chunks** to the same local speech recognition and
the same translation path the recorded pipeline uses. The screen shows the original, the translation
and the current delay.

### Near real time, and it says so

A 9-second chunk cannot be transcribed before it has been spoken, so the student is always at least
one chunk behind, plus processing. The backend returns a measured `behind_ms`, `/api/live/config`
reports `realtime: false`, and the app displays that number while recording.

**Measured: ~9–10 s shown on screen, 11.3–12.0 s server-side.** Nothing claims zero latency.

### The board, during class

An optional camera preview sits above the transcript. **Capture Board** photographs one frame and
runs it through the *existing* vision pass, pinning the result to the moment of the class:

```
00:54 — Chart: Grand Finale Schedule Table
```

* Captures land on the same timeline as the speech and can also be sampled every 12 s (opt-in); a
  manual tap always takes priority.
* **No video is ever recorded or streamed** — one JPEG per capture, rotated upright on the device.
* At the end every useful capture is merged into a single vision result, so a live lecture reaches
  fusion with exactly the shape an uploaded lecture has.

### Live BOB

Questions can be asked **during** the class. Luminara builds a LectureKnowledge-shaped view of what
has been captured so far and passes it to the ordinary agent, so citations still point at real
moments. After the class, BOB uses the complete lecture knowledge as it always has.

### End Class

**End class** hands everything to the identical reasoning step the recorded path uses. A live
session becomes an ordinary lecture — notes, script, formulas, translation, study pack, BOB — and
appears in My Lectures.

### One modality never kills another

| Failure | Behaviour |
|---|---|
| Camera fails or is declined | Surfaces on the capture; the class keeps recording |
| Vision provider fails | The transcript continues; the capture is marked unreadable |
| Translation fails | The original transcript remains |
| BOB fails | The lecture continues; the message offers a retry |
| No audible speech | Luminara **refuses** to create a lecture rather than inventing one |

Silent chunks are gated before transcription, and a transcript that is one phrase repeated is
discarded as a recognition artefact.

## 7. The multimodal AI pipeline

Eight stages, each a database row with a real start time, end time and the engine that did the work.
The Processing screen renders those rows directly — there is no simulated progress, and a skipped
stage records why.

| # | Stage | Engine |
|---|---|---|
| 1 | Lecture audio decoded | stdlib `wave` + NumPy |
| 2 | Teacher speech recognised | local speech recognition (`base`, CPU) |
| 3 | Classroom text extracted | IBM BOB `premium` (vision) |
| 4 | Visual content analysed | IBM BOB `premium` + a local geometry pass |
| 5 | Lecture understood (fusion) | IBM BOB `premium` |
| 6 | Learning material generated | deterministic projection, no model call |
| 7 | Translated for the student | IBM BOB `fast` |
| 8 | BOB ready | context compilation |

**Measured end to end on the deployed backend: 93 s.** Results are cached in SQLite, so re-opening a
lecture is instant.

Some things are deliberately *not* model calls — notes are a projection of the lecture knowledge,
the script reuses the stored transcript, and search is lexical. Keeping them deterministic makes
them instant, free, and incapable of drifting away from the evidence the rest of the app cites.

Formula preservation is enforced by construction: `latex` and `plain` are never sent to the
translator and are re-asserted from the source after merging, so a formula is structurally incapable
of returning as translated words.

## 8. IBM BOB integration

BOB is not a chat window bolted onto the side. **IBM BOB is the reasoning engine for the entire
pipeline** — it reads the whiteboard, interprets the diagram, fuses the modalities, writes the
notes, translates them, and answers the student's questions.

* Gateway: `https://api.us-east.bob.ibm.com/inference/v1`
* Auth: `Authorization: apikey <key>` — not Bearer
* Models: `premium` (vision-capable) for OCR, vision, fusion and the agent; `fast` for translation
* Every response carries the engine that produced it (`bob:premium`, `bob:fast`) and the app displays
  that badge. Nothing is attributed to BOB that BOB did not generate.

Three things make BOB an *agent over this lecture* rather than a generic assistant:

1. **Context compilation** — the lecture knowledge is compiled into an evidence block in which every
   fact carries its origin (speech timestamp, whiteboard, diagram, formula).
2. **Intent routing** — `qa`, `explain_simple`, `diagram`, `formula`, `translate` and `quiz` change
   what BOB is asked to produce, not merely how it phrases the answer.
3. **Grounded output contract** — BOB returns structured JSON with citations and a `grounded` flag,
   and says plainly when the lecture does not contain an answer instead of inventing one.

The endpoint, auth scheme, WAF requirement and model catalogue were each read out of the shipped IBM
BOB client and then confirmed against the live service — see [ARCHITECTURE.md](ARCHITECTURE.md) §6.

## 9. Architecture

**Android** (`android/`) — Kotlin, Jetpack Compose (Material 3), MVVM with a single
`LuminaraViewModel`, OkHttp + kotlinx.serialization, Coil, Navigation-Compose, CameraX for board
capture. Eleven screens:

```
Onboarding · Auth · Home · Classes · ClassDetail · UploadLecture
Setup · Processing · LectureDetail · Live · Bob
```

**Backend** (`backend/`) — one FastAPI service, SQLite via SQLAlchemy, no microservices.

```
app/
  main.py            lecture REST API
  live.py            /api/live — start · chunk · board · timeline · ask · finish · discard
  accounts.py        /api — auth, classes, membership
  landing.py         /download page and /luminara.apk
  llm.py             provider router; every result carries its engine
  auth.py            password hashing and signed tokens
  config.py  db.py  models.py
  pipeline/
    asr.py           speech recognition, ffmpeg-free WAV decoding
    media.py         video / compressed audio → 16 kHz mono WAV; board-frame picking
    vision.py        board OCR + diagram interpretation (+ local geometry pass)
    understanding.py multimodal fusion → LectureKnowledge
    notes.py         deterministic projection into note sections
    script.py        timestamped lecture script from the stored transcript
    search.py        lexical search over speech / board / formulas / notes
    translate.py     formula-safe translation
    runner.py        staged orchestration with real timings
  export/studypack.py  print-designed HTML → PDF via the local headless browser
  agents/bob.py        the lecture-grounded agent
  agents/bob_client.py pluggable BOB transport
  demo/                demo lecture assets + manifest
```

Provider routing is three-tier — **IBM BOB → optional secondary provider → local engine** — and
every result carries the engine string that produced it. A degraded run looks degraded; it is never
dressed up as a live one. The local engine is not a mock: speech is still transcribed for real and
the geometry pass still measures the diagram, so the lecture object is built from genuine data,
coarser and labelled `local`.

Full detail in [ARCHITECTURE.md](ARCHITECTURE.md).

## 10. Data model

`User` · `SchoolClass` · `Membership` · `Lecture` → `TranscriptSegment` · `VisualObservation` ·
`Formula` · `Note` (one per language) · `QAExchange` · `StageEvent` · `BoardCapture`, plus a
`Preference` key/value table.

The full knowledge document is also cached on `Lecture.knowledge_json`, so re-opening a lecture
never re-derives anything.

## 11. API surface

Interactive docs at `/docs` when the backend is running.

**Lectures**

```
GET    /health                              service and engine status
GET    /api/config                          languages, engines, flags
GET    /api/lectures                        library
POST   /api/lectures/demo                   create the demo lecture
POST   /api/lectures/upload                 video / audio / board image (+ class_id)
POST   /api/lectures/{id}/process           run the pipeline
GET    /api/lectures/{id}/status            stage rows with real timings
GET    /api/lectures/{id}                   lecture + notes in a language
POST   /api/lectures/{id}/translate         add a language
GET    /api/lectures/{id}/script            timestamped script
GET    /api/lectures/{id}/search?q=         lexical search over the lecture
GET    /api/lectures/{id}/export.pdf        study pack (HTML fallback available)
GET    /api/lectures/{id}/image | /audio    source media
GET    /api/lectures/{id}/suggestions       starter questions
POST   /api/lectures/{id}/ask               grounded Q&A
GET    /api/lectures/{id}/chat              history
POST   /api/lectures/{id}/publish           teacher publishes to a class
DELETE /api/lectures/{id}                   remove a lecture
```

**Live Class**

```
GET    /api/live/config                     chunk length, realtime:false, board_capture
POST   /api/live/start                      open a session
POST   /api/live/chunk                      one audio chunk → transcript + translation + behind_ms
POST   /api/live/board                      one frame → vision → timecoded capture
GET    /api/live/{id}/timeline              speech + board on one axis
GET    /api/live/{id}/capture/{capture_id}  the stored frame
POST   /api/live/{id}/ask                   BOB, mid-class, from what has been captured
POST   /api/live/pause                      pause / resume
GET    /api/live/{id}/state                 running transcript + captures
POST   /api/live/finish                     finalise into an ordinary lecture
POST   /api/live/discard                    drop an abandoned session
```

**Accounts and classes**

```
POST   /api/auth/register | /api/auth/login | GET|POST /api/auth/me
GET    /api/classes  ·  POST /api/classes  ·  POST /api/classes/join
GET    /api/classes/{id}  ·  DELETE /api/classes/{id}
```

**Public**

```
GET    /download        install page with live service status
GET    /luminara.apk    stable download URL
```

## 12. Setup

Full instructions in [SETUP.md](SETUP.md). Short version:

```bash
# backend
cd backend
cp .env.example .env                 # add BOB_API_KEY
pip install -r requirements.txt
python scripts/make_demo_assets.py
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000

# android
cd android
./gradlew :app:assembleDebug
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

Optional extras, imported lazily and degrading cleanly when absent:

```bash
pip install imageio-ffmpeg          # video and compressed-audio upload
pip install opencv-python-headless  # geometry pass + board-frame picking
```

A debug build targets `http://10.0.2.2:8000` (the host as seen from the emulator) and falls back to
`127.0.0.1:8000` for a USB device via `adb reverse tcp:8000 tcp:8000`.

`ffmpeg` and `tesseract` are **not** system requirements. The study pack is rendered by whatever
headless Chrome or Edge is already installed; without one, the endpoint serves readable HTML and
says so.

## 13. Configuration

Everything lives in `backend/.env` (git-ignored). `backend/.env.example` is the authoritative list
of names and verified defaults, covering:

* the BOB gateway — base URL, key, model aliases, auth style, and the User-Agent its WAF requires
* an optional secondary provider, used only when BOB is unreachable
* `AUTH_SECRET` — signs classroom tokens; keep it stable or every session is invalidated on restart
* `WHISPER_MODEL` / `WHISPER_DEVICE`, `SUPPORTED_LANGUAGES`, `HOST`, `PORT`, `CORS_ORIGINS`
* `LUMINARA_DATA_DIR` — point the database and uploads at a mounted volume
* `FORCE_OFFLINE=1` — force the local deterministic engine even when keys are present

**No key ever reaches the client.** The Android app talks only to the Luminara backend; the release
APK is scanned for key material and provider hostnames before every publish.

## 14. Demo

See [DEMO.md](DEMO.md) for the 2–3 minute script. In brief: open Luminara → choose **Hindi** →
**Try the demo** → watch real pipeline stages → Hindi notes with `T(n) = T(n/2) + O(1)` intact →
**Visuals** shows the board OCR and the diagram reading → **Ask BOB**: *"What formula did the
professor write?"* — answered with source chips and a `bob:premium` badge.

The demo lecture is the deterministic fallback: bundled assets, no account, no network conditions
required beyond the backend itself.

## 15. Deployment

A judge can scan a QR code, install the APK, sign in and use the real pipeline with no cable, no
`adb reverse` and no LAN address.

* The backend runs behind a **Cloudflare Tunnel** on public HTTPS (`--protocol http2`; the default
  QUIC transport measured 0/36 successful requests on this network against 36/36 on HTTP/2).
* The release APK compiles the public URL into `BuildConfig` and compiles the local auto-discovery
  **out**, so a public build can never wander onto a localhost address.
* `GET /download` serves an install page with live service status; `GET /luminara.apk` is a stable
  path, so a printed QR survives a rebuild.

Hosted free tiers were assessed and rejected on measured RAM — the speech model needs 763 MB, and
Render / Fly / Koyeb free tiers offer 256–512 MB. Hugging Face Spaces returns **402 Payment
Required** for Docker Spaces on a free account. `backend/Dockerfile` is ready for any host with
≥2 GB RAM.

Provider choice, secret handling, measured constraints and the redeploy steps are in
[DEPLOYMENT.md](DEPLOYMENT.md).

## 16. Testing and verification

```bash
cd backend
python scripts/smoke_asr.py           # speech recognition
python scripts/smoke_live.py          # live chunking path
python scripts/smoke_live_board.py    # Live Class V2: capture, timeline, live BOB, finalise
python scripts/smoke_classroom.py     # accounts, classes, publishing, access control
python scripts/test_board_moments.py  # board-citation clustering
```

Verified from a physical device with **WiFi off, mobile data only and no `adb reverse`**:

| Check | Result |
|---|---|
| Registration and login | Account created against the public backend |
| Full pipeline | 8/8 stages in **93 s** |
| Hindi | Title → "Binary Search एल्गोरिथ्म और समय जटिलता", technical terms preserved |
| Ask BOB | Answered in Hindi citing `Speech · 00:26`, `Speech · 00:34`, `Whiteboard` (`bob:premium`) |
| Study pack | 383 KB PDF saved to Downloads, Devanagari filename intact |
| Live Class | 12 chunks over 100 s; **~9–10 s behind** shown, 11.3–12.0 s measured |
| Capture Board | `00:54 — Chart: Grand Finale Schedule Table` (`bob:premium`) |
| Audio during vision | Backend log interleaves `/chunk` around `/board` |
| End Class | Board stages **done**; retitled from the board content; in My Lectures |
| Download page | Renders over HTTPS; serves a byte-identical APK |

## 17. Honesty: what is real

Everything shown is produced at run time:

* the narration is genuinely transcribed;
* the whiteboard is genuinely read by a vision model that never sees the transcript;
* progress stages are database rows with real start/end timestamps — no simulated progress;
* if a stage is skipped or fails, the app says so and names the reason;
* a live session that captured no speech refuses to become a lecture.

Two things are **generated inputs**, not generated outputs, and are documented as such: the demo
lecture's narration is Windows text-to-speech, and its whiteboard is a rendered image
(`backend/scripts/make_demo_assets.py`). They are inputs to the pipeline exactly as a real recording
and a real photograph would be.

## 18. Limitations

* **The public URL is ephemeral** — it lives as long as the tunnel process; restarting it means
  rebuilding the APK and the QR code.
* **The laptop is the server.** No free hosted tier tested could run the local speech model.
* In-app camera capture for *recorded* lectures is not implemented; teachers attach an image at
  upload. (Live Class does capture from the camera.)
* Board-frame selection from video is a heuristic (edge density across three sampled frames);
  attaching a photo overrides it.
* Speech recognition runs on CPU, so a lecture takes ~93 s and a single worker serialises
  concurrent uploads.
* English and Hindi are verified end to end. Bangla and Arabic use the same code path and appear in
  the selector, but have not been reviewed by a native speaker.
* Live translation was verified through the deployed backend; the on-device V2 run happened in a
  silent room, so it is marked partial in the requirements matrix.
* Assignments, grading, attendance, analytics, institution administration, web and desktop clients
  and cross-device sync are out of scope.

## 19. Future scope

Incremental streaming transcription · in-app board capture for recorded lectures · per-student
personalisation from past questions · richer LaTeX rendering · on-device inference for
low-connectivity classrooms · more languages with native-speaker review · teacher analytics on which
concepts were re-asked most · a named tunnel or hosted deployment to remove the ephemeral URL.

## 20. Repository map

```
README.md               this file
SETUP.md                install and run
ARCHITECTURE.md         how it works and why
DEMO.md                 the 2–3 minute demo script
DEPLOYMENT.md           public deployment, measured constraints, redeploy
THIRD_PARTY.md          third-party software and attribution
REQUIREMENTS_MATRIX.md  every requirement mapped to evidence
android/                Kotlin + Jetpack Compose application
backend/                FastAPI service, pipeline, agent, exports
deploy/                 deploy script and the QR code judges scan
```

## 21. Third-party software

See [THIRD_PARTY.md](THIRD_PARTY.md). No third-party technology is claimed as original work.

## 22. Requirements coverage

See [REQUIREMENTS_MATRIX.md](REQUIREMENTS_MATRIX.md) — every requirement mapped to its
implementation and the evidence for it, including the public deployment and Live Class V2.
