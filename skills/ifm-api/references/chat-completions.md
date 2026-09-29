# Chat completions API

`POST https://api.ifm.ai/v1/chat/completions` — fully compatible with the OpenAI Chat Completions API. Auth: `Authorization: Bearer $IFM_API_KEY`.

Source: https://docs.ifm.ai/#/chat-completions, https://docs.ifm.ai/#/reasoning

## Parameters

| Parameter | Type | Status | Notes |
| --- | --- | --- | --- |
| model | string | Required | e.g. `IFM/K2-Horizon-375B-A23B` |
| messages | array | Required | system / user / assistant / tool roles. **Text content only.** |
| temperature | float | Supported | 0–2, default 0.7. Docs suggest ≤0.3 for agentic workloads; the K2 Horizon migration guide recommends 1.0 (with top_p 0.95). |
| top_p | float | Supported | 0–1, default 1. |
| max_tokens | int | Supported | Cap on completion length. Defaults to remaining context. |
| stop | string \| array | Supported | Up to 4 stop sequences. |
| seed | int | Supported | Best-effort determinism for identical requests. |
| stream | bool | Supported | SSE streaming — see [streaming.md](streaming.md). |
| tools / tool_choice | array \| str | Supported | Standard OpenAI function schema — see [function-calling.md](function-calling.md). |
| frequency_penalty | float | Supported | Accepted for compatibility but **has no effect**. |
| logprobs | bool | Supported | Accepted for compatibility; **not returned**. |
| reasoning_effort | string | Extended | `"low"` · `"medium"` · `"high"`, default `"high"`. Per-family behavior differs. |
| response_format | object | Extended | `{"type": "json_object"}`; K2 Horizon also accepts `json_schema` — see [structured-output.md](structured-output.md). |

IFM-specific body fields (send via `extra_body` in the OpenAI SDK):

- `chat_template_kwargs: {"reasoning_effort": "low"|"medium"|"high"}`
- `tool_presentation_format: "xml"`, `tool_call_format: "xml"` (K2 Horizon tool calling)

## Reasoning

K2 Horizon offers multiple reasoning modes. Set the thinking budget with `reasoning_effort` passed through `chat_template_kwargs`:

```json
{
  "model": "IFM/K2-Horizon-375B-A23B",
  "messages": [{ "role": "user", "content": "hello" }],
  "chat_template_kwargs": { "reasoning_effort": "high" }
}
```

```python
client.chat.completions.create(
    model="IFM/K2-Horizon-375B-A23B", messages=messages,
    extra_body={"chat_template_kwargs": {"reasoning_effort": "high"}},
)
```

The trace comes back in `reasoning_content`, separate from `content`. Each effort level closes the trace with its own token:

| Effort | Closing token |
| --- | --- |
| low | `<ifm\|think_faster>` |
| medium | `<ifm\|think_fast>` |
| high (recommended) | `<ifm\|think>` |

Every assistant turn sent back must replay its reasoning trace verbatim — see [multi-turn-and-sessions.md](multi-turn-and-sessions.md).

## Response

```json
{
  "id": "chatcmpl-acd8436fe38e4520",
  "model": "IFM/K2-Horizon-375B-A23B",
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "\nHello! Is there anything I can help you with today?",
      "reasoning_content": "The user just says hello, with no request. Respond warmly and…",
      "reasoning": "The user just says hello, with no request. Respond warmly and…",
      "tool_calls": []
    },
    "finish_reason": "stop"
  }],
  "usage": {
    "prompt_tokens": 11,
    "completion_tokens": 214,
    "total_tokens": 225,
    "prompt_tokens_details": { "cached_tokens": 0 },
    "completion_tokens_details": { "reasoning_tokens": 184 }
  }
}
```

- Reply text: `choices[0].message.content` (may start with a newline — strip for display if needed).
- `reasoning` mirrors `reasoning_content`.
- `finish_reason`: `"stop"`, `"length"`, or `"tool_calls"`.
- `usage.completion_tokens_details.reasoning_tokens` counts thinking tokens; `prompt_tokens_details.cached_tokens` shows prefix cache hits (see session routing).

## curl

```bash
curl https://api.ifm.ai/v1/chat/completions \
  -H "Authorization: Bearer $IFM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "IFM/K2-Horizon-375B-A23B", "messages": [{"role": "user", "content": "hello"}]}'
```
