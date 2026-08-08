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
5. File metadata recorded in `_v2_files`
6. If PDF or image: `doc-processor.js` queues text extraction (processing_status = `pending`)

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

- **PDF** — text extraction via `pdf-parse`; per-page texts saved to `page_texts` (JSON array)
- **Images** — OCR via a three-step fallback chain:
  1. **Vision LLM** (`vision-llm`) — single-call OCR + classify + field extraction; if successful, skips stages 2 and 3 entirely. `ocr_engine = 'vision-llm'`
  2. **Mistral OCR API** (`mistral`) — fallback 1 when Vision LLM returns no text. `ocr_engine = 'mistral'`
  3. **PaddleOCR** (`paddle-ocr.js`) — fallback 2, local microservice at `PADDLE_OCR_URL` (air-gap safe). `ocr_engine = 'paddle'`
- **Audio/video** — transcription (see section below)
- Extracted text stored in `_v2_files.extracted_text`
- After extraction: AI classifies document type and extracts structured fields (`classified_type_id`, `extracted_fields`)

### PaddleOCR microservice

`paddle-ocr.js` sends the image to an external PaddleOCR HTTP service (`POST PADDLE_OCR_URL/ocr`). Activated only when `PADDLE_OCR_URL` env variable is set. Returns `null` (silently) if the service is unavailable, so the file processing status becomes `skipped` with `processing_error = 'OCR API недоступен'`.

## Audio/Video Transcription (`audio-transcriber.js`)

Audio and video files are detected by MIME type or extension and processed separately from images/PDFs.

**Supported formats:** mp3, mp4, m4a, wav, ogg, oga, opus, webm, flac, aac, mov, mkv, mpeg, mpga (and corresponding video MIME types: `video/mp4`, `video/webm`, `video/quicktime`, `video/x-matroska`). Telegram voice notes arrive as `.oga`.

**Transcription API:** OpenAI-compatible Whisper endpoint. Configured via env variables:
- `WHISPER_API_URL` — base URL (default: `https://api.polza.ai/v1`)
- `WHISPER_MODEL` — model name (default: `whisper-1`)
- `WHISPER_API_KEY` / `AI_API_KEY` / `GROQ_API_KEY` — API key (checked in this order)
- `WHISPER_LANGUAGE` — ISO code of the spoken language (e.g. `ru`); empty means auto-detect.
  **Worth setting.** With auto-detect some providers *translate* a Russian recording into
  English instead of transcribing it.

The request asks for `response_format=json`, which every OpenAI-compatible provider
accepts; `text` is rejected outright by some (routerai). A response that is not JSON is
still read as a plain transcript, so older providers keep working.

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

Files stored on filesystem in `backend/uploads/{db}/` directory (optionally in subdirectories).

## DB Tables (per-workspace, lazy-init)

- `_v2_files` — `id`, `filename`, `original_name`, `mime_type`, `size`, `uploaded_by`, `object_id`, `processing_status` (VARCHAR 20), `processing_error` (TEXT), `extracted_text` (TEXT), `ocr_applied` (BOOLEAN, default FALSE), `ocr_engine` (VARCHAR 50), `page_texts` (TEXT — JSON array of per-page strings, PDF only), `classified_type_id`, `classified_name`, `classification_confidence`, `extracted_fields` (JSONB), `fields_confirmed`, `processed_at`, `imported_as`, `imported_id`, `imported_at`, `created_at`
