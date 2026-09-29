# Transcriptions API (Jais-ASR)

Source: https://docs.ifm.ai/#/transcription

`POST https://api.ifm.ai/v1/audio/transcriptions` — compatible with the OpenAI audio transcription API. Auth, quotas, and error envelopes match chat completions.

## Request

`multipart/form-data`: the audio file plus fields below.

```bash
curl https://api.ifm.ai/v1/audio/transcriptions \
  -H "Authorization: Bearer $IFM_API_KEY" \
  -F file=@meeting.wav \
  -F model="IFM/Jais-ASR"
```

```python
import os
from openai import OpenAI

client = OpenAI(base_url="https://api.ifm.ai/v1", api_key=os.environ["IFM_API_KEY"])
with open("meeting.wav", "rb") as audio:
    result = client.audio.transcriptions.create(model="IFM/Jais-ASR", file=audio)
print(result.text)
```

## Parameters

| Parameter | Type | Status | Notes |
| --- | --- | --- | --- |
| file | file | Required | Audio file (multipart). |
| model | string | Required | e.g. `IFM/Jais-ASR`. |
| language | string | Optional | Language hint. **Omitting it is recommended.** |
| prompt | string | Optional | Short hint for spelling of names, terminology, register. |
| response_format | string | Optional | `json` (default), `text`, `verbose_json`, `srt`, `vtt`. |
| timestamp_granularities | string[] | Optional | `segment` and/or `word`. Requires `verbose_json`. |
| temperature | number | Optional | 0–1, default 0. |

## Response (`verbose_json`)

```json
{
  "task": "transcribe",
  "tag": "ar",
  "duration": 12.48,
  "text": "نبدأ الاجتماع بمراجعة أرقام الربع الثالث.",
  "segments": [{ "id": 0, "start": 0.0, "end": 4.12, "text": "نبدأ الاجتماع" }],
  "usage": { "audio_seconds": 12.48 }
}
```

Default `json` returns `text` and the detected-language `tag`; `segments` and word timings require `verbose_json`.

## Language detection

**Leave `language` unset** — the model detects the language (including dialects) and returns it in `tag`. Set `language` only when certain: the right dialect (e.g. `language=eg` for Egyptian Arabic) can improve accuracy, but a wrong hint degrades it (English audio with `language=eg` transcribes worse, even though `tag` still returns `en`).

## Limits and formats

- **Max 30 seconds of audio per request.** Split longer recordings on silence and transcribe parts in sequence (long-audio support is coming).
- Containers: FLAC (`flac`, lossless), MP3 (`mp3`), MP4 (`mp4`, AAC), MPEG (`mpeg`), MPGA (`mpga`), M4A (`m4a`, AAC), OGG (`ogg`, `oga`; Vorbis/Opus), WAV (`wav`; PCM 16/24-bit), WEBM (`webm`; Opus).
