# Multi-turn conversations and session routing

Sources: https://docs.ifm.ai/#/multi-turn, https://docs.ifm.ai/#/session-routing

## Multi-turn

The model is **stateless**: each request carries the full history. Append every assistant message back into `messages` exactly as returned, then the next user turn. Applies to plain replies, tool calls, and streamed responses.

Replay `reasoning_content` verbatim. An empty string `""` is valid — keep it; don't drop the field or send `null`.

```python
msg = resp.choices[0].message
messages.append({
    "role": "assistant",
    "content": msg.content,
    "reasoning_content": msg.reasoning_content,   # keep "" as-is
})
messages.append({"role": "user", "content": "Now double it."})
resp = client.chat.completions.create(model=os.environ["IFM_MODEL"], messages=messages)
```

| Field | Meaning |
| --- | --- |
| `reasoning_content` | Canonical field (matches vLLM and SGLang). Use this one. |
| `reasoning` | Mirror with the same value, for older clients. Send either; both is harmless. |

Tool turns interleave in the same `messages` array ([function-calling.md](function-calling.md)); streamed turns are assembled from deltas before being appended ([streaming.md](streaming.md)).

Replayed reasoning counts as prompt tokens, so long threads consume the daily cap faster than the visible transcript suggests.

## Session routing (`X-Session-ID`)

Hosted requests are load-balanced across compute nodes. An `X-Session-ID` header keeps a conversation on the node holding its KV cache, so the shared prefix isn't recomputed each turn.

```bash
curl https://api.ifm.ai/v1/chat/completions \
  -H "Authorization: Bearer $IFM_API_KEY" \
  -H "X-Session-ID: conv_8f2c14ab" \
  -H "Content-Type: application/json" \
  -d '{"model": "IFM/K2-Horizon-375B-A23B", "messages": [...]}'
```

```python
resp = client.chat.completions.create(
    model="IFM/K2-Horizon-375B-A23B", messages=messages,
    extra_headers={"X-Session-ID": session_id},
)
```

### Choosing a value

| Rule | Why |
| --- | --- |
| Stable per conversation | Turn N+1 must reach the node that served turn N. |
| Opaque and unguessable | A random ID you mint — never an email, user name, or raw account ID. |
| Never shared across users | Shared IDs concentrate load on one node with no benefit. |
| New ID per agent run | A fresh run has a fresh prefix. |

### Semantics

- **Latency/efficiency optimization only, best-effort.** Results are identical with or without it; no state is stored between requests and nothing is retained server-side.
- Affinity can break anytime (nodes added/drained/restarted, cache eviction). Never a substitute for sending full history.
- Pays off most for long multi-turn chats, agent loops resending growing tool transcripts, and large shared system prompts.
- Under the hood: [vLLM Router](https://github.com/vllm-project/router) `consistent_hash` policy.
