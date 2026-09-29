# Streaming

Source: https://docs.ifm.ai/#/streaming

Server-sent events in the OpenAI-compatible chunk schema; existing SDK streaming code works unchanged.

## Content and reasoning

Set `stream=True`. Reasoning and content arrive as **separate deltas** — buffer each, then append the finished turn when the stream ends.

```python
stream = client.chat.completions.create(
    model=os.environ["IFM_MODEL"], messages=messages, stream=True,
)

content, reasoning = "", ""
for chunk in stream:
    delta = chunk.choices[0].delta
    reasoning += getattr(delta, "reasoning_content", None) or ""
    content   += getattr(delta, "content", None) or ""

messages.append({"role": "assistant", "content": content, "reasoning_content": reasoning})
```

## Tool calls

`arguments` arrives in fragments across many chunks. Accumulate per `tool_calls[i].index` and parse JSON **only after the stream ends** — a partial fragment is almost never valid JSON.

```python
calls = {}
for chunk in stream:
    for tc in (chunk.choices[0].delta.tool_calls or []):
        buf = calls.setdefault(tc.index, {"id": "", "name": "", "args": ""})
        if tc.id:                 buf["id"] = tc.id
        if tc.function.name:      buf["name"] += tc.function.name
        if tc.function.arguments: buf["args"] += tc.function.arguments

# parse only after the stream ends
args = json.loads(calls[0]["args"])
```

## Ending a stream

- The last content chunk carries `finish_reason` — `"stop"`, `"length"`, or `"tool_calls"` — followed by the `[DONE]` sentinel.
- A stream that ends **without** a `finish_reason` is truncated: retry it, and do not append the half-finished turn to history.
- Streams cannot be resumed. If the connection drops, re-send the request from the last complete turn. Reuse the same `X-Session-ID` so the retry lands on the node holding the prefix (see [multi-turn-and-sessions.md](multi-turn-and-sessions.md)).
