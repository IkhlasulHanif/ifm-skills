# Custom function calling

Source: https://docs.ifm.ai/#/custom-function-calling

Tool calling lets K2 Horizon use functions implemented by your application. Interfaces use the standard OpenAI tool schema, so existing tool definitions work unchanged.

1. **The application** implements and executes the function.
2. **The function interface** (name, description, JSON Schema parameters) is defined in the `tools` array.
3. **K2 Horizon** selects the function and generates arguments but does not execute it.

## K2 Horizon format parameters

Set both to `"xml"` (they are IFM extensions, so pass via `extra_body`). Response parsing is unchanged — standard `tool_calls` with `finish_reason: "tool_calls"`.

```python
resp = client.chat.completions.create(
    model=os.environ["IFM_MODEL"],
    messages=messages,
    tools=tools,
    extra_body={"tool_presentation_format": "xml", "tool_call_format": "xml"},
)
```

## Declare the interface

```json
{
  "type": "function",
  "function": {
    "name": "lookup_order_status",
    "description": "Current status of a customer order.",
    "parameters": {
      "type": "object",
      "properties": { "order_id": { "type": "string" } },
      "required": ["order_id"]
    }
  }
}
```

## The tool loop

The model answers with `finish_reason: "tool_calls"` and an empty `content`. Execute the function, append the result, re-send — loop until `finish_reason: "stop"`. The assistant tool turn keeps both `tool_calls` and its reasoning.

```python
messages = [
  {"role": "user", "content": "Where is my order A-10432?"},
  {
    "role": "assistant",
    "content": "",
    "reasoning_content": "User wants live order status; call lookup_order_status with the order id.",
    "tool_calls": [{"id": "call_1", "type": "function",
      "function": {"name": "lookup_order_status", "arguments": "{\"order_id\":\"A-10432\"}"}}]
  },
  {"role": "tool", "tool_call_id": "call_1", "content": "{\"status\": \"shipped\", \"eta\": \"2026-09-17\"}"},
  # the next assistant message answers using the tool result
]
```

The model may return several entries in `tool_calls` in one turn — each needs its own `tool` message (matching `tool_call_id`).

## tool_choice

| Value | Behavior |
| --- | --- |
| `"auto"` | Default when tools are present. The model decides whether to call a tool or answer directly. |
| `"none"` | Tools stay visible but are never called. Useful for a final summarizing turn. |
| `"required"` | The model must call at least one tool. Use when the flow cannot proceed without data. |
| `{"type": "function", "function": {"name": "..."}}` | Forces one named function; the model still chooses arguments. |

Malformed tool schemas are rejected with `422 validation_error`.

For streaming tool calls see [streaming.md](streaming.md).
