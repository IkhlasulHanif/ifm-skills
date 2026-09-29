# Structured output

Source: https://docs.ifm.ai/#/structured-output

Constrain the reply to JSON with `response_format`.

## JSON mode

`{"type": "json_object"}` guarantees the reply parses as JSON but not its shape — also describe the desired fields in the prompt.

```json
{
  "model": "IFM/K2-Horizon-375B-A23B",
  "messages": [{ "role": "user", "content": "List three UAE emirates with populations." }],
  "response_format": { "type": "json_object" }
}
```

## JSON Schema mode (K2 Horizon)

`{"type": "json_schema"}` pins the exact shape. Decoding is constrained to the schema, so the reply validates by construction.

```json
"response_format": {
  "type": "json_schema",
  "json_schema": {
    "name": "emirate_list",
    "strict": true,
    "schema": {
      "type": "object",
      "properties": {
        "emirates": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "name": { "type": "string" },
              "population": { "type": "integer" }
            },
            "required": ["name", "population"]
          }
        }
      },
      "required": ["emirates"]
    }
  }
}
```

Keep schemas shallow and name fields the way you would in a prompt — deeply nested or cryptically named schemas cost tokens and degrade answer quality even though output stays valid.

## With reasoning

The constraint applies to `content` only. `reasoning_content` is free-form prose, never JSON — parse the fields separately and keep replaying the trace in multi-turn history.

## Failure modes

- A schema the model cannot satisfy usually surfaces as a **truncated reply with `finish_reason: "length"`**, not an error. Raise `max_tokens`, then simplify the schema.
- Unsupported JSON Schema keywords are rejected at request time with `400`.
