# Module: tts

**Path:** `src/api/v2/modules/tts/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/:db/tts/...`
**Auth:** JWT required.

## Purpose

Text-to-speech synthesis via Piper TTS (local neural engine). Converts arbitrary text into WAV audio. Supports multiple voices and adjustable speed. Piper runs as a local binary (`PIPER_BIN` env), models stored in `PIPER_MODELS_DIR` (`/opt/piper-voices` by default).

## Endpoints

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/tts/status` | viewer | Check if Piper TTS is available (returns `{ ok, available, engine }`) |
| GET | `/tts/voices` | viewer | List installed voice models (`{ ok, voices: [{ id, name, language, gender }] }`) |
| POST | `/tts/synthesize` | viewer | Synthesize text to WAV audio (returns `audio/wav` binary) |

### POST `/tts/synthesize` body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `text` | string | yes | Text to synthesize (1–10000 chars) |
| `voice` | string | no | Voice model ID (default: `ru_RU-irina-medium`) |
| `speed` | number | no | Playback speed 0.25–4.0 (default: 1.0) |

Response: raw WAV binary (`Content-Type: audio/wav`).

## AI Tools

| Tool | Risk Tier | Description |
|------|-----------|-------------|
| `speak_text` | TIER_LOW | Synthesize text via Piper TTS, returns base64 audio |
| `list_tts_voices` | TIER_LOW | List available voice models |

## Service (`service.js`)

- `isAvailable()` — checks Piper binary presence, caches result
- `synthesize(text, { voice, speed })` — runs Piper CLI, returns `{ pcm, sampleRate }`
- `listVoices()` — reads `.onnx` files from models dir, parses `.onnx.json` metadata
- Concurrency limited to 2 parallel synthesis calls (`p-limit`)
- Default sample rate: 22050 Hz (overridden by model config)

## Frontend

| File | Description |
|------|-------------|
| `services/ttsService.js` | HTTP client: `synthesize()`, `voices()`, `status()` |
| `components/TtsButton.vue` | Play/stop button for TTS playback |
| `components/TtsSettingsModal.vue` | Voice and speed settings UI |
| `composables/useTts.js` | Shared composable: audio state, voice selection, playback control |

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PIPER_BIN` | _(empty)_ | Path to Piper binary. If empty, TTS is disabled. |
| `PIPER_MODELS_DIR` | `/opt/piper-voices` | Directory containing `.onnx` voice models |
| `PIPER_DEFAULT_VOICE` | `ru_RU-irina-medium` | Default voice model ID |
