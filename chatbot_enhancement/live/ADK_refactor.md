# ADK Compliance Report — `adk_live_service` vs Google ADK Official Documentation

> **Source**: ADK Gemini Live API Toolkit Development Guide (5 parts)  
> **Target**: `easyconf-live` package in `adk_live_service/src/easyconf_live/`  
> **Date**: 2026-06-06  

---

## Part 1: Intro to Streaming (`https://adk.dev/streaming/dev-guide/part1/`)

### Official guidance
- ADK defines a **4-phase lifecycle**: Application Init → Session Init → Bidi-streaming → Terminate
- `Runner` is created once at startup with `app_name`, `agent`, `session_service`
- `LiveRequestQueue`, `RunConfig` are created per session in Phase 2
- Platform switching via `GOOGLE_GENAI_USE_VERTEXAI` env var
- Recommended transport: FastAPI WebSocket server

### Actual usage in `adk_live_service`

| Item | Status | Details |
|------|--------|---------|
| 4-phase lifecycle | ✅ Correct | `app.py` implements all 4 phases clearly |
| `Agent` from `google.adk` | ✅ Correct | Used in all agent files |
| `Runner` from `google.adk` | ✅ Correct | Created in `LiveAgentApp.start()` at line 551 |
| `InMemorySessionService` | ✅ Correct | Used at line 51 |
| `LiveRequestQueue` | ✅ Correct | Created per session in `_ensure_live_runtime()` |
| `RunConfig` | ✅ Correct | Built in `_build_run_config()` |
| `GOOGLE_GENAI_USE_VERTEXAI` | ✅ Optional | Uses Gemini Live API via API key (development path). Vertex AI not required — both platforms share the same Live API. |
| Platform switching | ✅ Optional | Not needed if staying on Gemini Live API via API key. Only needed for Vertex AI production deployment. |
| FastAPI WebSocket | ❌ Different approach | Uses custom stdin/stdout IPC + `MediaTransport` instead of FastAPI |

### Recommendation
- Add `GOOGLE_GENAI_USE_VERTEXAI` env var support for production Vertex AI deployment
- Document the platform-switching strategy in config

---

## Part 2: Sending Messages (`https://adk.dev/streaming/dev-guide/part2/`)

### Official guidance
- `send_content(Content)`: text with turn-by-turn
- `send_realtime(Blob)`: audio/video realtime streaming
- `send_activity_start()` / `send_activity_end()`: **only when VAD is explicitly disabled**
- `close()`: graceful termination (mandatory in BIDI mode)
- Queue must be created in async context
- Sync send methods (non-blocking, uses `put_nowait`)

### Actual usage

| Item | Status | Details |
|------|--------|---------|
| `send_content()` | ✅ Correct | Used in `_on_json_message` for text_input |
| `send_realtime()` with `Blob` | ✅ Correct | Used for audio and video forwarding |
| `close()` in finally | ✅ Correct | Called in `_close_live_runtime()` |
| Activity signals (`send_activity_start/end`) | ❌ **BUG** | Used **without disabling VAD** | Activity signals should ONLY be sent when `realtime_input_config` disables automatic VAD. Current code sends `send_activity_start/end` while VAD remains enabled (default), causing conflicting turn detection signals. |
| Activity signals for text input | ❌ Non-standard | Wraps text content in activity_start/end. Docs specify `send_content()` alone is sufficient for text turns. |
| Queue creation in async context | ✅ Correct | Created inside `_ensure_live_runtime()` which is async |

### Impact
Using activity signals without disabling VAD creates conflicting turn signals. The model may ignore manual signals or misinterpret user audio. This is the most critical issue from Part 2.

---

## Part 3: Event Handling (`https://adk.dev/streaming/dev-guide/part3/`)

### Official guidance
- `run_live()` yields `Event` objects (Pydantic models)
- Fields: `turn_complete`, `interrupted`, `partial`, `input_transcription`, `output_transcription`, `content`, `error_code`, `error_message`, `usage_metadata`
- Python field names are **snake_case** (e.g., `event.turn_complete`)
- Serialize with `event.model_dump_json(exclude_none=True, by_alias=True)`
- Server-side access uses dot notation, not `getattr`

### Actual usage

| Item | Status | Details |
|------|--------|---------|
| `async for event in runner.run_live()` | ✅ Correct | Line 250 |
| Turn complete handling | ✅ Correct | `getattr(event, "turnComplete", False)` — works but non-idiomatic |
| Interrupted handling | ✅ Correct | `getattr(event, "interrupted", False)` — works but non-idiomatic |
| Transcription handling | ✅ Correct | Both input and output transcription handled |
| Audio inline data | ✅ Correct | Extracts and forwards audio PCM data |
| **Field access pattern** | ❌ **WRONG** | Uses `getattr(event, "turnComplete")` instead of `event.turn_complete` | The Event class is a Pydantic model; accessing fields by camelCase aliases via `getattr` works only because Pydantic aliases are applied to fields. This is fragile and bypasses type safety. Should use `event.turn_complete` (snake_case Python fields). |
| `errorMessage` access | ❌ **WRONG** | `getattr(event, "errorMessage", None)` at line 274 — should be `event.error_message` |
| `model_dump_json()` | ❌ Not used | Does NOT serialize events with `model_dump_json()`; manually extracts fields |
| `partial` flag | ❌ Not handled | No streaming merge logic for partial text chunks |
| `usage_metadata` | ❌ Not handled | No token tracking or cost monitoring |
| Error handling with `error_code` | ❌ Not used | Only checks `errorMessage` text, not structured error codes |
| Proper run_live() cleanup in error | ⚠️ Partial | `close()` called in `_close_live_runtime()`, but only when Node.js sends `session.close` |

### Event Field Access — Mapping

| Doc snake_case | Current `getattr()` call | Correct way |
|---|---|---|
| `event.turn_complete` | `getattr(event, "turnComplete", False)` | `event.turn_complete` |
| `event.interrupted` | `getattr(event, "interrupted", False)` | `event.interrupted` |
| `event.input_transcription` | `getattr(event, "inputTranscription", None)` | `event.input_transcription` |
| `event.output_transcription` | `getattr(event, "outputTranscription", None)` | `event.output_transcription` |
| `event.error_message` | `getattr(event, "errorMessage", None)` | `event.error_message` |

> **Note**: The code works because `Event.model_dump_json(by_alias=True)` maps Python snake_case → camelCase in JSON serialization, but `getattr(event, "camelCase")` relies on Pydantic's `model_config` `alias_generator` or field aliases. In ADK's `Event` class (Pydantic v2), field aliases are generated automatically, so `getattr(event, "turnComplete")` resolves to the `turn_complete` field. However, this is an undocumented internal behavior — the public API uses snake_case.

---

## Part 4: RunConfig (`https://adk.dev/streaming/dev-guide/part4/`)

### Official guidance
- `StreamingMode.BIDI` (WebSocket Live API) vs `StreamingMode.SSE` (HTTP)
- `response_modalities`: `["TEXT"]` or `["AUDIO"]` (default `["AUDIO"]`)
- `session_resumption`: Enable for automatic reconnection beyond ~10min connection limit
- `context_window_compression`: Enable for unlimited session duration
- `save_live_blob`: Persist audio to session artifacts
- `max_llm_calls`: Cost control
- `speech_config`: Voice configuration
- `input_audio_transcription` / `output_audio_transcription`: Transcription

### Actual usage

| Item | Status | Details |
|------|--------|---------|
| `response_modalities` | ✅ Correct | Configurable via Node.js modality setting |
| `speech_config` | ✅ Correct | Configurable via Node.js voice setting |
| Input/output transcription | ✅ Correct | Configurable via Node.js |
| `custom_metadata` | ✅ Correct | Attaches session/user/locale metadata |
| **`streaming_mode`** | ❌ **MISSING** | Not explicitly set. Defaults to BIDI, but should be explicit. |
| **`session_resumption`** | ❌ **MISSING** | Without this, Live API connection drops after ~10 minutes; no auto-reconnect. |
| **`context_window_compression`** | ❌ **MISSING** | Without this, sessions are duration-limited (15min audio-only, 2min video on Gemini Live API). |
| **`save_live_blob`** | ❌ **MISSING** | Audio is not persisted to ADK Session; only streamed ephemerally. |
| `max_llm_calls` | ❌ Not used | No cost control limit on LLM calls per session |

### Recommended RunConfig

```python
from google.genai import types
from google.adk.agents.run_config import RunConfig, StreamingMode

RunConfig(
    streaming_mode=StreamingMode.BIDI,          # explicit
    response_modalities=["AUDIO"],               # or ["TEXT"]
    speech_config=types.SpeechConfig(...),       # voice selection
    input_audio_transcription=types.AudioTranscriptionConfig(),
    output_audio_transcription=types.AudioTranscriptionConfig(),
    session_resumption=types.SessionResumptionConfig(),        # auto-reconnect
    context_window_compression=types.ContextWindowCompressionConfig(
        compression_strategy="MAX_TOKEN_LOSS",   # unlimited sessions
        max_loss=50,
    ),
    save_live_blob=True,                         # persist audio
    max_llm_calls=50,                            # cost control
    realtime_input_config=types.RealtimeInputConfig(
        # Disable VAD only if using manual activity signals
        # automatic_vad_filter=types.AutomaticVADFilterConfig(...)
    ),
)
```

---

## Part 5: Audio, Images, Video (`https://adk.dev/streaming/dev-guide/part5/`)

### Official guidance
- Audio input: 16-bit PCM @ 16kHz, mono
- Audio output: 16-bit PCM @ 24kHz, mono
- Video: JPEG, 1 FPS max, 768×768 recommended
- VAD enabled by default; activity signals need disabled VAD
- `SpeechConfig` with `PrebuiltVoiceConfig`

### Actual usage

| Item | Status | Details |
|------|--------|---------|
| Audio input format (PCM16 @ 16kHz) | ✅ Correct | `config.pcm_input_rate = 16000` |
| Audio output format (PCM @ 24kHz) | ✅ Correct | `config.pcm_output_rate = 24000` |
| Video format (JPEG) | ✅ Correct | `mime_type="image/jpeg"` |
| Video FPS (1 FPS) | ✅ Correct | `config.screen_max_fps = 1.0` |
| Video resolution | ⚠️ Partial | Height is 432px (doc recommends 768×768) |
| `SpeechConfig` with voice | ✅ Correct | Uses `PrebuiltVoiceConfig` with `voice_name` |
| **VAD + activity signals conflict** | ❌ **CRITICAL** | Activity signals (`send_activity_start/end`) are used without disabling automatic VAD. This creates conflicting turn signals. |
| `realtime_input_config` | ❌ Missing | Should be configured if manual VAD/activity signals are used |
| Audio chunk size | ⚠️ Acceptable | 4096-byte stdin reads; not explicitly chunked for latency |

### VAD Conflict Detail

In `_on_media_message` (audio path) and `_on_json_message` (text_input/audio_stream paths):
```python
# Current (WRONG):
runtime.queue.send_activity_start()  # VAD is still ON!
# ... send content ...
runtime.queue.send_activity_end()

# Required pattern when using activity signals:
# 1. Disable VAD in RunConfig:
#    realtime_input_config=types.RealtimeInputConfig(
#        automatic_vad_filter=types.AutomaticVADFilterConfig(disabled=True)
#    )
```

---

## Cross-Cutting Issues Summary

### 🚨 Critical

| # | Issue | File:Line | Impact |
|---|-------|-----------|--------|
| C1 | Activity signals used without disabling VAD | `app.py:422-428, 443-450, 484-492` | Conflicting turn detection; model may misinterpret user speech |
| C2 | Event fields accessed via `getattr` with camelCase | `app.py:274-329` | Bypasses type safety; fragile to ADK internals changes |

### ⚠️ High

| # | Issue | File:Line | Impact |
|---|-------|-----------|--------|
| H1 | Missing `session_resumption` | `app.py:109-119` | Connection drops after ~10min; no auto-reconnect |
| H2 | Missing `context_window_compression` | `app.py:109-119` | Sessions limited: 15min audio, 2min video |
| H3 | Missing `streaming_mode` explicit setting | `app.py:109-119` | Works via default but not self-documenting |
| H4 | Sub-agents using live model instead of text model | `agents/conference.py:48`, `navigation.py:29`, etc. | Unnecessary cost for text-only sub-agents |

### 🔶 Medium

| # | Issue | File:Line | Impact |
|---|-------|-----------|--------|
| M1 | N/A — Valid choice | `config.py:27-28` | Gemini Live API via API key is a valid production path. Vertex AI is optional. |
| M2 | No `save_live_blob` | `app.py:109-119` | Audio not persisted to session |
| M3 | No `max_llm_calls` limit | - | No cost control safeguard |
| M4 | `InMemorySessionService` in production | `app.py:51` | Session state lost on restart; not suitable for multi-replica |

### 🔹 Low

| # | Issue | File:Line | Impact |
|---|-------|-----------|--------|
| L1 | Activity signals wrapped around text turns | `app.py:443-450` | Non-standard; `send_content()` alone is sufficient for text |
| L2 | Manual event serialization instead of `model_dump_json` | `app.py:256` | Extra code; bypasses serialization helpers |
| L3 | `errorMessage` not `error_code` | `app.py:274` | Cannot distinguish error types programmatically |

---

## Refactoring Priority Order

### Phase 1 — Critical Fixes

1. **Fix VAD conflict**: Add `realtime_input_config` with VAD disabled (or remove manual activity signals and rely on VAD)
2. **Fix event field access**: Replace all `getattr(event, "camelCase")` with `event.snake_case`
3. **Add explicit `streaming_mode=BIDI`** in `RunConfig`

### Phase 2 — Production Readiness

4. **Enable `session_resumption`** for auto-reconnect beyond 10min
5. **Enable `context_window_compression`** for unlimited session duration
6. **Switch sub-agents to text model** where appropriate
7. **Add `save_live_blob`** for audio persistence

### Phase 3 — Production Hardening

8. **Add persistent `SessionService`** (PostgreSQL/MySQL) for production
9. **Add cost controls**: `max_llm_calls`, `usage_metadata` tracking

---

*Generated by comparing `adk_live_service` source code against ADK documentation at https://adk.dev/streaming/dev-guide/part1/ through part5/*
