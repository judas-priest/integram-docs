# Module: files

**Path:** `src/api/v2/modules/files/`
**Files:** `router.js`, `service.js`, `doc-processor.js`, `audio-transcriber.js`, `paddle-ocr.js`
**Base URL:** `/api/v2/:db/files/...`
**Auth:** JWT required. Upload/mkdir: `editor`. Delete: `editor`. List/download: any authenticated user.

## Purpose

File upload, storage, download, and indexing. Files can be attached to EAV objects via `file`-type columns. Uploaded files are parsed (PDF/image text extraction via OCR) and metadata is recorded in `_v2_files` for search and review.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/files` | List files for workspace (`?subdir=path`) |
| POST | `/files` | Upload file (multipart/form-data, max 50 MB) |
| POST | `/files/mkdir` | Create directory (`name`, optional `subdir`) |
| GET | `/files/meta` | List file metadata from `_v2_files` (`?objectId=`, `?limit=`, `?offset=`) |
| GET | `/files/meta/:id/extracted` | Get extracted fields for a file (review dialog) |
| POST | `/files/meta/:id/confirm` | Confirm extracted fields and create EAV object (`typeId`, `fields`) |
| POST | `/files/meta/:id/reprocess` | Reset and re-queue file for processing |
| POST | `/files/meta/:id/mark-imported` | Mark file as imported (`importedAs`: `table`\|`document`, `importedId`) |
| GET | `/files/:filename` | Download file binary (`?subdir=`) |
| DELETE | `/files/:filename` | Delete file (`?subdir=`) |

## Upload Flow

1. `multer` receives the multipart upload (in-memory buffer up to 50 MB)
1a. Filename decoded from latin1 back to UTF-8 via `fixMulterFilename` — see *Filename encoding* below
2. MIME type detected from magic bytes; extension corrected if mismatched
3. MIME type validated before writing to disk
4. File written to workspace upload directory (respects optional `subdir`)
5. File metadata recorded in `_v2_files` with `processing_status = 'pending'`
6. `doc-processor.js` picks pending rows up and extracts text

The status is set by `recordFileMetadata` itself, at INSERT time, and that is deliberate.
`getPendingFiles` selects strictly `WHERE processing_status = 'pending'`, so a row inserted
without a status is invisible to the worker forever — the file sits on disk, appears in
`_v2_files`, and shows up in neither the queue nor the error list. Thirty files across nine
workspaces had been lying that way in production. Callers that extract text themselves (the
AI-chat upload route) overwrite the status with `done`; callers whose extraction fails leave
`pending`, and the worker then retries with its own branch set, which is wider than
`extractFileText`'s.

## Filename encoding

`multer` hands over the `Content-Disposition` filename decoded as latin1, while browsers
send UTF-8. `fixMulterFilename` (`service.js`) recovers the original name and is
**idempotent** — it returns the name unchanged once it contains code points above U+00FF,
or when the bytes are not valid UTF-8 (a genuine latin1 name).

That idempotency matters: `sanitizeFilename` calls it too, so any caller that decodes the
name itself before sanitizing would otherwise decode twice. A double decode keeps only the
low byte of each code point — Cyrillic U+0410..U+044F collapses into 0x10..0x4F, and
`sanitizeFilename` then strips everything below 0x20, so "Устройство дерева
спецификаций.docx" used to land on disk as "#AB@>9AB2> 45@520 A?5F8D8:0F88.docx" with
capitals А–П gone entirely.

`_v2_files.original_name` keeps the decoded name; `filename` is the sanitized name on disk.
`GET /files` lists the directory itself, so it shows `filename`, not `original_name`.

Names damaged before this was fixed are repaired by
`backend/scripts/fix-mangled-upload-filenames.js` (dry run by default, `--apply` to rename).

## Text Extraction (`doc-processor.js`)

- **PDF** — text extraction via `pdf-parse`; per-page texts saved to `page_texts` (JSON array).
  A PDF with no text layer is a scan and goes to the OCR chain below
- **Images** — the OCR chain, same one
- **Audio/video** — transcription (see section below)
- **Text files** (`.md/.txt/.csv/.json/…`) — read as UTF-8, no OCR
- **Office** (`.docx/.xlsx/.xls/.pptx/.odt/.ods/.odp`) — `extractOfficeText`
- **Archives** (`.zip/.rar/.7z/.tar/.tgz/.gz/.bz2/.xz`) — members are walked and their text
  aggregated. `.zip` is read in memory; the rest are unpacked through a CLI, so `.rar` and
  `.7z` need `7z` on the host (`p7zip-full` plus `p7zip-rar` for the RAR codec — installed on
  production 12.08.2026, without them the file ends as `error` with the reason named)
- Extracted text stored in `_v2_files.extracted_text`

The branches after PDF/images are chosen by **file extension**, not by MIME. That is why files
uploaded as `application/octet-stream` — 47 of them in one production workspace — are still
read correctly. What they hit instead was the final `else`, which used to write `skipped` with
no `processing_error` at all; a silent skip is indistinguishable from a bug, so it now records
the MIME and the filename that found no handler.
- After extraction: AI classifies document type and extracts structured fields (`classified_type_id`, `extracted_fields`).
  When a vision engine wins the chain it returns classification and fields in the same call, so
  those stages are skipped — both branches check `ocr.meta` for that, images and scanned PDFs
  alike. The PDF branch used to go to `classifying` unconditionally, paying for two more LLM
  calls to re-derive what the vision answer already contained

### OCR chain (`ocr-chain.js`)

`ocrChain(filePath, mimeType, opts)` is the only way to get text out of a scan. Every
caller — both branches of `doc-processor`, `normalizer/file-parser.js`,
`ai/file-context.js` — goes through it. Before it existed each of them carried its own
copy of the sequence, and they had drifted: four orders of engines, four notions of
"engine available", four wordings of failure, no shared timeout. PaddleOCR sat imported
and uncalled for six weeks because adding it meant editing four places.

An engine is declared once, in the `ENGINES` registry:

| Engine | Accepts | Available when | Timeout |
|---|---|---|---|
| `vision-llm` | images only | alias `vision` resolves to a provider (`hasAlias`) | 120 s |
| `unlimited-ocr` | images, PDF | `UNLIMITED_OCR_URL` | 120 s |
| `mistral` | images, PDF | `OCR_API_URL` **and** `OCR_API_KEY`/`AI_API_KEY` | 120 s |
| `paddle` | images only | `PADDLE_OCR_URL` | 30 s |
| `vision-pdf` | PDF only | alias `vision_pdf` resolves (`LLM_ALIAS_VISION_PDF`) | 120 s |

`vision-llm` goes first on images: it is the only engine needing no OCR env at all, and
the only one returning classification and fields together with the text. `vision-pdf` is the
same call — `ocrAndExtractViaVision` with a different `alias` — and it closes the PDF order,
last, because it is the only per-page paid engine there.

**Two aliases, not one — and why the earlier explanation here was wrong.** This section
used to claim that a PDF cannot travel in the `image_url` field at all, that Anthropic has no
such type and OpenAI-compatible endpoints answer 400 to it. Measured against routerai on
2026-08-12 with a scan that has no text layer, that is false:

| Model | Field | Result | prompt/compl tokens |
|---|---|---|---|
| `google/gemini-2.5-flash-lite` | `image_url` (`data:application/pdf;base64,…`) | full text, correct | 1303 / 73 |
| `google/gemini-2.5-flash-lite` | `file` content part | same text, same token count | 1303 / 73 |
| `anthropic/claude-haiku-4.5` | `file` | PDF metadata instead of text | — |
| `anthropic/claude-haiku-4.5` | `image_url` | response without `choices` | — |

The request field is not what decides it — the model is. A three-page scan through the same
call returns all three pages plus classification. So the reason a scanned PDF used to be
unreadable in production is the model behind the `vision` alias: `LLM_ALIAS_VISION` points at
Anthropic, which returns a *plausible wrong answer* (a description of the file) rather than an
error, and a chain cannot tell that apart from success.

That is why PDFs get their own alias instead of `application/pdf` being added to
`vision-llm.accepts`. One alias for both would force one model on both jobs: swapping it for
the sake of scans would silently change how images are read, and leaving Anthropic there would
have the chain store file metadata as if it were document text. `LLM_ALIAS_VISION_PDF` must
name exactly **one** model — `routedFetch` walks the candidates of an alias in order, so a
second, PDF-blind model behind the first would quietly answer with garbage. Verify a model
before putting it there:

```
node scripts/probe-pdf-vision.mjs /path/to/scan.pdf
```

The probe prints the whole answer rather than a verdict, because a wrong model produces text,
not an error — no mock can catch this, only reading the output can.

`paddle` stays out of the PDF order for a different reason: `paddle-ocr.js` is documented
for image files and port 8866 is PaddleHub serving.

The call returns `{ text, engine, meta, attempted, available }`. `attempted` lists what
was actually invoked, so the failure message describes what happened rather than what the
environment looks like — `ocrFailureMessage()` tells "no engine configured" apart from
"nobody returned text (tried …)". An engine that throws does not break the chain; the
next one runs.

### `failures` — why each engine gave up

The chain also returns `failures: [{ engine, reason }]` — one entry per engine that
threw. It exists because "this document is unreadable" and "the provider has nothing left
to bill" used to look identical in the file card: both ended as *nobody returned text
(tried …)*, while the log held the actual reason nobody reads. The two are fixed by
different people — one re-scans the document, the other tops up the account — so the card
has to tell them apart.

For that to work, `ocrAndExtractViaVision` must stop turning a provider refusal into
`null`. It rethrows an error carrying either of **two** marks:

- `code` from `PROVIDER_REFUSAL_CODES` — `LLM_CREDITS_EXHAUSTED`, `LLM_RATE_LIMIT`,
  `LLM_TIMEOUT`, `QUOTA_EXCEEDED`, `DLP_BLOCKED`, `CLOSED_CONTOUR`, `LLM_PROVIDER_REFUSED`;
- `_status` from `PROVIDER_REFUSAL_STATUSES` — `402`, `429`.

`LLM_PROVIDER_REFUSED` covers the case the first two marks miss: an HTTP answer the router
does **not** intercept. It handles 402, 429 and 5xx itself and hands everything else back
untouched, so a `400` used to reach `if (!resp.ok) return null` and be filed as "the
document has no text". Measured on ai2o.online, 15.08.2026: `auth2api/claude-sonnet-5`
answered ``400 `temperature` is deprecated for this model``, and the file card said *nobody
returned text (tried vision-llm)* — a message that names neither the fault nor where to
look. Now the provider's own words are read out of the body (`error.message`, or the raw
body when it is not JSON), trimmed to 200 characters and thrown as
`<provider>/<model> ответил <status>: <text>`.

One mark is not enough, and the second one is not belt-and-braces. A 429 arrives with no
code at all: `llm-router.js` routes it through `makeCooldownError`, which sets only
`_status`; the code is stamped at the very end of the router and only for 402. Match on
`code` alone and rate limiting — the most common refusal in production — stays
indistinguishable from an unreadable scan.

Errors without either mark still return `null`, deliberately: a network failure
(`socket hang up`), a file the worker cannot read (`ENOENT`, `EACCES`), a `5xx` from the
provider. "The provider's server broke" and "the file is not on disk" are not verdicts
about *recognition*, and an administrator looking at a file card cannot act on them — so
they stay in the log, not in `processing_error`. Note also that a generic "the error has a
`code`" test would not do: `ENOENT` and `EACCES` have one too, and a read failure would
then advertise itself as a refused model call.

`ocrFailureMessage` appends the collected reasons to the previous sentence after a period,
rather than replacing it — *tried vision-pdf* answers "what was done", the reason answers
"why it did not work", and both are needed. The whole string is then cut at 500 characters,
the same limit the other `doc-processor` branches apply (`err.message.slice(0, 500)`), so
one column does not hold two different notions of "too long". The result in
`processing_error` reads:

```
Скан-PDF: ни один движок не вернул текст (пробовали unlimited-ocr, mistral). mistral: ENOENT: no such file or directory
```

Throwing does not shorten the walk. The chain catches the exception, records the reason in
`failures` and moves on to the next candidate — a refusal from one engine says nothing
about the rest, and the local engines in particular do not depend on an external
provider's balance. `scripts/probe-pdf-vision.mjs` catches the rethrown refusal too and
prints `ОТКАЗ ПРОВАЙДЕРА: <code or status> — <message>`, because 402 and "the model did
not understand the PDF" are fixed differently and the probe exists to tell them apart.

`available()` lives next to each engine (`isUnlimitedOcrAvailable`, `isOcrViaLlmAvailable`
in `files/service.js`, `isPaddleAvailable` in `paddle-ocr.js`), and each reads its env at
call time — read at import, the check and the request would disagree and the message
would lie.

### Workspace context: billing, quota, closed contour

`ocrChain(filePath, mimeType, { workspaceId, workspaceSettings })` carries the workspace down
to `vision-llm` and `vision-pdf`, the engines in the chain that spend LLM tokens. Both fields
matter and for different reasons:

- `workspaceId` is what `logLlmCall` records in `llm_calls.workspace_id` (a TEXT column holding
  the workspace **db name**). Without it the spend lands with `workspace_id = null` and is
  invisible to both the quota query and the spend report.
- `workspaceSettings` is what `routedFetch` needs before it will apply the token quota
  (`ai/service.js: checkQuota`) or refuse an external provider in a closed contour. Passing
  neither is not "no limits configured" — it is "limits skipped".

Callers supply them like this: `doc-processor` passes `workspaceId: db` (a background worker
has no `req`, so `ocrAndExtractViaVision` fills the settings itself via
`getWorkspaceSettings(db)`, cached for a minute); the AI-chat upload route passes both from
`req.workspace`; `normalizer/file-parser.js` takes `{ workspaceId }` from its callers
(`classifyBatch` and the extractor dispatcher both already carry `db`).

`ocrAndExtractViaVision` refuses files over 20 MB before reading them (`MAX_VISION_BYTES`).
Uploads are allowed up to 100 MB and the document travels as a base64 string in the request
body, so without the guard a worker would hold a 130 MB string for a request the provider
rejects anyway.

DLP is a different matter and its limits should not be overstated. Rules are matched against
the *text* of the outgoing request, and the content of an image is base64 in that request —
nothing in it for a keyword or regex to match. What actually protects a scan from leaving a
restricted workspace is the closed contour (no external provider is selected at all), not the
DLP rules. See `dlp.md`.

## Audio/Video Transcription (`audio-transcriber.js`)

Audio and video files are detected by MIME type or extension and processed separately from images/PDFs.

**Supported formats:** mp3, mp4, m4a, wav, ogg, oga, opus, webm, flac, aac, mov, mkv, mpeg, mpga (and corresponding video MIME types: `video/mp4`, `video/webm`, `video/quicktime`, `video/x-matroska`). Telegram voice notes arrive as `.oga`.

**Transcription API:** OpenAI-compatible Whisper endpoint. Configured via env variables:
- `WHISPER_API_URL` — base URL (default: `https://api.polza.ai/v1`)
- `WHISPER_MODEL` — model name (default: `openai/whisper-large-v3-turbo`, exported as
  `DEFAULT_WHISPER_MODEL`)
- `WHISPER_API_KEY` / `AI_API_KEY` / `GROQ_API_KEY` — API key (checked in this order)
- `WHISPER_LANGUAGE` — ISO code of the spoken language (e.g. `ru`); empty means auto-detect.
  **Worth setting.** With auto-detect some providers *translate* a Russian recording into
  English instead of transcribing it.

The request asks for `response_format=json`, which every OpenAI-compatible provider
accepts; `text` is rejected outright by some (routerai). A response that is not JSON is
still read as a plain transcript, so older providers keep working.

**Choosing the model on routerai.** Production points `WHISPER_API_URL` at
`https://routerai.ru/api/v1`. Measured against `POST /audio/transcriptions` on 2026-08-12
with a one-second WAV:

| Model | Result | Rate per second |
|---|---|---|
| `openai/whisper-1` | 200 | 0.0107 |
| `openai/whisper-large-v3` | 200 | 0.00268 |
| `openai/whisper-large-v3-turbo` | 200 | 0.00119 |
| `whisper-large-v3` (no vendor prefix) | 400 `Model not found` | — |

Two things follow, and the default now reflects both. The `openai/` prefix is mandatory —
routerai rejects a bare model id. And `whisper-1`, which used to be the default, is the most
expensive of the three per second: turbo is nine times cheaper and newer. The caveat visible in
the responses is billing granularity — turbo charged ten seconds for a one-second file while
`whisper-1` charged one, so the saving only appears on clips longer than about ten seconds.

**File size limit:** 25 MB. Files exceeding this limit fail with an error stored in `processing_error`.

**Processing flow:**
1. `processing_status` set to `extracting`
2. File POSTed to Whisper endpoint; the transcript is read from `{"text": …}` or from a bare-text body
3. Transcript saved to `extracted_text`, `processing_status` set to `done`
4. If the file is linked to an EAV object (`object_id` is set): transcript auto-written to a memo column aliased `"Транскрипт"` on that object (if such a column exists in the type definition)
5. Embedding scheduled for the file record

Note: audio/video files skip the classification/field-extraction stages (stages 2–3 of the doc-processor pipeline).

## Storing a file from another module

`service.storeFileBuffer(pool, db, { buffer, originalName, mimeType, uploadedBy, objectId, subdir, user })`
is the entry point **new code is required to use** for putting bytes into workspace files:
it checks the write grant, sanitizes the name, corrects extension and MIME by magic bytes,
validates against the MIME whitelist, writes to disk under a name that does not collide,
records metadata and routes the file into the processing pipeline. `POST /files` is a thin
wrapper over it, and so is the `telegram_download_file` automation action — a file must
reach OCR/transcription the same way no matter how it arrived.

**It is not yet the only path.** Five places still repeat the sequence by hand, and they
have already drifted apart — treat this list as work outstanding, not as a set of
alternatives:

| Place | How it differs |
|---|---|
| `portal/router.js:2818` | takes the upload dir from `process.env.UPLOAD_DIR`, not `getUploadDir`, and omits `isAudioFile` from `shouldProcess` — **audio uploaded through the portal is never transcribed** |
| `teamchat/router.js:551` | records metadata, queues no processing — chat attachments stay out of OCR and out of search |
| `portal/teamchat/router.js:608` | same |
| `ai/router.js:829` | own copy of the sequence |
| `ai/agent/tools/excel.js:37` | own copy of the sequence |

Names never overwrite: an existing name gets `-1`, `-2`, … before the extension
(`uniqueFilename`). Without it a second `file_12.oga` — the name Telegram gives every
voice note — would leave two metadata rows, one file, and an older row whose stored
transcript describes bytes that are gone. The caller gets back the name actually used.

Losing the metadata row does not lose the file: the call returns a `warning` instead of
throwing once the bytes are on disk.

`user` is required — the write grant is checked explicitly at the start of the call, and a
caller without one gets a 403 before any work happens. Routes that read a request body
should call `service.requireWriteGrant(user, db)` first, so a forbidden upload is refused
before megabytes are buffered. Resolving the directory itself lives in `files/paths.js`
(`uploadDirFor`), apart from the permission check, so the upload root can be substituted
in tests.

## Object Attachment

File columns in EAV store the file path/name as the requisite value. Use `GET /files/meta` with `?objectId=` to list files attached to a specific object.

## Storage Backend

Files live on the filesystem, one directory per workspace. The only place that resolves it is
`uploadDirFor(db)` in `paths.js` — nobody else may derive the path:

```
<root>/<db>/
```

`<root>` comes from `UPLOAD_DIR`; without it the module falls back to a path computed from its own
location (`__dirname`, seven levels up, `integram-server/download`). Both give
`/opt/integram-server/download` on production — the variable does not change the path, it states it
in the server config instead of deriving it from how deeply the code is nested.

Why it matters: `create-file.js` used to compute the root itself with `path.resolve('../../…')`,
which is relative to the **process working directory**, not to the file. On production that resolved
to `/integram-server/download` while this module wrote to `/opt/integram-server/download` — the
agent wrote files into one tree and the download route read from another (fixed 12.08.2026).

## DB Tables (per-workspace, lazy-init)

Workspaces created under the earlier schema store the object link in a column named `obj_id`.
`ensureFilesTable()` renames it to `object_id` (rename, not add — an added column would leave the
existing links orphaned in `obj_id`). Before that rename existed, `recordFileMetadata` failed with
`column "object_id" ... does not exist` and every upload answered 500 (23 workspaces on production,
12.08.2026).

- `_v2_files` — `id`, `filename`, `original_name`, `mime_type`, `size`, `uploaded_by`, `object_id`, `processing_status` (VARCHAR 20), `processing_error` (TEXT), `extracted_text` (TEXT), `ocr_applied` (BOOLEAN, default FALSE), `ocr_engine` (VARCHAR 50), `page_texts` (TEXT — JSON array of per-page strings, PDF only), `classified_type_id`, `classified_name`, `classification_confidence`, `extracted_fields` (JSONB), `fields_confirmed`, `processed_at`, `imported_as`, `imported_id`, `imported_at`, `created_at`
