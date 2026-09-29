---
name: ifm-api
description: Build with the IFM hosted API (api.ifm.ai) and IFM models — K2 Horizon, K2 V2, Jais 2, Jais-ASR. Use when code or the user mentions IFM, K2 Horizon, K2-Think, K2-V2, Jais, IFM_API_KEY, or api.ifm.ai; when writing chat completions, reasoning, tool calling, streaming, structured output, multi-turn history, session routing, or audio transcription against IFM; when handling IFM errors/rate limits; or when migrating from K2 V2 to K2 Horizon.
license: MIT
metadata:
  author: Ikhlasul Akmal Hanif
  source: https://docs.ifm.ai
---

# IFM API

IFM's hosted API is **OpenAI-compatible**. Use the official OpenAI SDK (or plain HTTP) with IFM's base URL and key — existing OpenAI code works with a few IFM-specific additions.

## Setup

```bash
export IFM_API_KEY="IFM-..."              # create at https://platform.ifm.ai/api-keys
export IFM_BASE_URL="https://api.ifm.ai/v1"
export IFM_MODEL="IFM/K2-Horizon-375B-A23B"
```

```python
# pip install openai
import os
from openai import OpenAI

client = OpenAI(base_url=os.environ["IFM_BASE_URL"], api_key=os.environ["IFM_API_KEY"])
resp = client.chat.completions.create(
    model=os.environ["IFM_MODEL"],
    messages=[{"role": "user", "content": "hello"}],
)
print(resp.choices[0].message.content)
```

## Hosted models

| Model id | Use | Endpoint |
| --- | --- | --- |
| `IFM/K2-Horizon-375B-A23B` | Frontier reasoning, agents, tool use. 512K context. **Default choice.** | `/v1/chat/completions` |
| `IFM/K2-Think-V2` | Older reasoning model, 128K. **Deprecating** — migrate to K2 Horizon. | `/v1/chat/completions` |
| `IFM/Jais-ASR` | Speech-to-text (Arabic dialects, English, auto language detection). | `/v1/audio/transcriptions` |

All other K2 Horizon sizes, K2 V2, and Jais 2 are open weights only (Hugging Face), not on the hosted API. Full catalog: [references/models.md](references/models.md).

## Rules that differ from plain OpenAI usage

1. **Replay reasoning every turn.** Assistant messages return `reasoning_content` alongside `content`. Append both back into `messages` verbatim. An empty string `""` is valid — keep it; never drop the field or send `null`. (`reasoning` is a mirror field for older clients.)
2. **Reasoning effort**: `"low" | "medium" | "high"` (default `"high"`, recommended), passed via `chat_template_kwargs` — in the SDK: `extra_body={"chat_template_kwargs": {"reasoning_effort": "high"}}`.
3. **Tool calling on K2 Horizon**: pass `extra_body={"tool_presentation_format": "xml", "tool_call_format": "xml"}`. Tool definitions and `tool_calls` responses are standard OpenAI shape. Assistant tool turns keep `tool_calls` *and* reasoning.
4. **Sampling**: recommended `temperature=1.0`, `top_p=0.95` for K2 Horizon.
5. **Session routing**: send header `X-Session-ID` (random, stable per conversation/agent run, never shared across users, never PII) for KV-cache reuse. SDK: `extra_headers={"X-Session-ID": sid}`. Best-effort latency optimization only — still send full history.
6. **Text-only** message content. `frequency_penalty` and `logprobs` are accepted but ignored.
7. **Structured output**: `response_format` supports `json_object` and (K2 Horizon) `json_schema` with `strict`. Applies to `content` only — `reasoning_content` is never JSON.
8. **Budget**: 10M tokens/day per key (prompt + completion, reset 00:00 UTC). Replayed reasoning counts as prompt tokens.

## Full agent turn (reasoning + tools + session routing)

```python
import json, os, uuid
session_id = f"conv_{uuid.uuid4().hex}"

def call(messages, tools):
    return client.chat.completions.create(
        model=os.environ["IFM_MODEL"],
        messages=messages,
        tools=tools,
        temperature=1.0,
        top_p=0.95,
        extra_headers={"X-Session-ID": session_id},
        extra_body={
            "chat_template_kwargs": {"reasoning_effort": "high"},
            "tool_presentation_format": "xml",
            "tool_call_format": "xml",
        },
    )

while True:
    choice = call(messages, tools).choices[0]
    msg = choice.message
    turn = {"role": "assistant", "content": msg.content or "",
            "reasoning_content": getattr(msg, "reasoning_content", "") or ""}
    if msg.tool_calls:
        turn["tool_calls"] = [tc.model_dump() for tc in msg.tool_calls]
    messages.append(turn)
    if choice.finish_reason != "tool_calls":
        break
    for tc in msg.tool_calls:  # one tool message per call
        result = run_tool(tc.function.name, json.loads(tc.function.arguments))
        messages.append({"role": "tool", "tool_call_id": tc.id, "content": json.dumps(result)})
```

## Reference files — read the one you need

| Topic | File |
| --- | --- |
| Model catalog, architectures, context sizes, Hugging Face links | [references/models.md](references/models.md) |
| Chat completions parameters & response schema; reasoning effort & tokens | [references/chat-completions.md](references/chat-completions.md) |
| Tool calling, `tool_choice`, tool loop | [references/function-calling.md](references/function-calling.md) |
| Streaming (content, reasoning, tool-call deltas, truncation) | [references/streaming.md](references/streaming.md) |
| JSON mode / JSON Schema, failure modes | [references/structured-output.md](references/structured-output.md) |
| Multi-turn history and `X-Session-ID` session routing | [references/multi-turn-and-sessions.md](references/multi-turn-and-sessions.md) |
| Audio transcription (Jais-ASR) | [references/transcription.md](references/transcription.md) |
| Limits, rate-limit headers, error codes & retry policy | [references/limits-and-errors.md](references/limits-and-errors.md) |
| Migrating K2 V2 → K2 Horizon | [references/migration.md](references/migration.md) |

Official docs: https://docs.ifm.ai
