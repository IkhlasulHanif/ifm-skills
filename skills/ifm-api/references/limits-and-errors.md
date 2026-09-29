# Limits and errors

Sources: https://docs.ifm.ai/#/limits, https://docs.ifm.ai/#/errors

## Daily token cap

| | |
| --- | --- |
| Daily token cap | 10,000,000 (prompt + completion) |
| Reset | Daily, 00:00 UTC |
| Scope | Per API key |

Past the cap: `429 rate_limit_error` until reset. Unpublished per-minute request/token guards also exist — build against the daily cap and handle `429` via `Retry-After`. Monitor usage at https://platform.ifm.ai.

## Rate-limit headers

| Header | Meaning |
| --- | --- |
| `x-ratelimit-limit-tokens` | Current platform-guard allowance (informational). |
| `x-ratelimit-remaining-tokens` | Tokens left against that guard in the current window. |
| `x-ratelimit-limit-tokens-daily` | Daily cap (10,000,000). |
| `x-ratelimit-remaining-tokens-daily` | Tokens left today. |
| `x-ratelimit-reset-tokens` | Seconds until the guard window refills. |
| `Retry-After` | Seconds to wait before retrying (on 429s). |

## Errors

Every error has an HTTP status and JSON body with a stable `error.type`. **Branch on `type`, not message text.**

| Status | Type | Notes | Retry |
| --- | --- | --- | --- |
| 400 | invalid_request_error | Malformed JSON, missing field, invalid value. Fix payload. | No |
| 401 | authentication_error | Missing/invalid key. Check `Authorization: Bearer`. | No |
| 403 | permission_error | Key not entitled to this model, endpoint, or region. | No |
| 404 | not_found_error | Unknown model ID or endpoint path. | No |
| 408 | timeout_error | Server time limit exceeded. Shorten prompt or reduce `max_tokens`. | Backoff |
| 410 | deprecated_model_error | Model past sunset. Body carries migration-guide URL. | No |
| 413 | request_too_large | Body too large or prompt exceeds context window. | No |
| 422 | validation_error | Semantically invalid (e.g. malformed tool schema, unsupported param combo). | No |
| 429 | rate_limit_error | Rate/quota exceeded. Wait `Retry-After`, then exponential backoff. | Backoff |
| 499 | request_cancelled | Client disconnected. Not billed. | Yes |
| 500 | api_error | Internal error. Exponential backoff + jitter. | Backoff |
| 502 / 504 | upstream_error | Gateway/inference node failure. Transient. | Backoff |
| 503 | service_unavailable | Deploy or capacity event. | Backoff |
| 529 | overloaded_error | At capacity. Back off aggressively with jitter. | Backoff |

Example bodies:

```json
// 400
{ "error": { "type": "invalid_request_error", "code": "missing_required_parameter",
             "param": "messages", "message": "Field 'messages' is required." } }

// 429, header Retry-After: 37
{ "error": { "type": "rate_limit_error", "code": "token_rate_limit",
             "message": "Per-minute token budget exhausted." } }
```

## Retry helper (Python, raw HTTP)

```python
import random, time, requests

RETRYABLE = {408, 429, 499, 500, 502, 503, 504, 529}

def post_with_retry(url, headers, body, max_attempts=6):
    for attempt in range(max_attempts):
        r = requests.post(url, headers=headers, json=body, timeout=600)
        if r.status_code < 400:
            return r.json()
        if r.status_code not in RETRYABLE or attempt == max_attempts - 1:
            r.raise_for_status()
        wait = float(r.headers.get("Retry-After", 0)) or min(60, 2 ** attempt)
        cap = 2.0 if r.status_code == 529 else 1.0  # back off harder when overloaded
        time.sleep(wait * cap + random.uniform(0, 1))
```

With the OpenAI SDK, set `max_retries` on the client (it honors `Retry-After`) and branch on `err.body["error"]["type"]` for non-retryable errors.
