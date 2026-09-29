# Migration: K2 V2 → K2 Horizon

Source: https://docs.ifm.ai/#/migration

## What changes

| Area | Required change |
| --- | --- |
| Model name | Replace `MBZUAI-IFM/K2-V2-Instruct` or `MBZUAI-IFM/K2-Think-V2` with `IFM/K2-Horizon-375B-A23B`. |
| Context window | 128K → 512K tokens. |
| Conversation state | Assistant turns carry `reasoning_content` that must be replayed verbatim. |
| Function calling | Set `tool_presentation_format` and `tool_call_format` to `"xml"`; assistant tool turns keep both `tool_calls` and reasoning. |
| Sampling | Recommended `temperature` 1.0 and `top_p` 0.95. |
| Token budget | Replayed reasoning counts as prompt tokens — long threads use the daily cap faster. |

## Multi-turn

```python
# K2 V2 — content only
messages.append({"role": "assistant", "content": msg.content})

# K2 Horizon — replay reasoning too
msg = resp.choices[0].message
messages.append({
    "role": "assistant",
    "content": msg.content,
    "reasoning_content": msg.reasoning_content,   # keep "" as-is
})
```

Code that rebuilds history from `content` alone still runs but drops the model's reasoning context between turns. Streaming: buffer reasoning and content deltas separately, append the finished turn at stream end.

## Context window

Existing requests are unaffected; client-side history trimming written for 128K can usually be loosened or removed.

## Function calling

```python
resp = client.chat.completions.create(
    model=os.environ["IFM_MODEL"],
    messages=messages,
    tools=tools,
    extra_body={"tool_presentation_format": "xml", "tool_call_format": "xml"},
)
```

Responses keep the OpenAI shape (`tool_calls`, `finish_reason: "tool_calls"`) — parsing code doesn't change.

## Sampling

Requests passing `temperature`/`top_p` explicitly are unaffected; requests that omit them may sample differently — re-check outputs that depended on K2 V2 behavior.

## Checklist

- [ ] Model name in an env var (`IFM_MODEL`) so cutover is config, not code.
- [ ] History replay includes `reasoning_content` (keep `""`, never `null`).
- [ ] Both tool-format params set to `"xml"` before benchmarking agentic workloads.
- [ ] Sampling set to `temperature=1.0`, `top_p=0.95` (or explicitly chosen).
- [ ] Re-check prompts that relied on K2 V2 phrasing — K2 Horizon reasons first and may need less scaffolding.
- [ ] Log `usage` per call during the first week to size the new token profile.
- [ ] Handle `410 deprecated_model_error` for any remaining K2 V2 callers.
