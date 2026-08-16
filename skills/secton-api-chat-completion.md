---
name: secton-api-chat-completion
description: >-
  Generate a chat completion from Secton's OpenAI-compatible inference API, blocking or streamed,
  with the correct auth header, the real error envelope, and the retry rules this API actually
  supports.
api: secton-api:secton-api-chat-api
operations:
  - post-external-api-v1-chat-completions
base_url: https://api.secton.org
generated: '2026-08-16'
method: generated
source: >-
  openapi/secton-api-chat-api-openapi.yml (operationIds verified against the spec),
  conventions/secton-api-conventions.yml, errors/secton-api-problem-types.yml,
  rate-limits/secton-api-rate-limits.yml, lifecycle/secton-api-lifecycle.yml
---

# Create a chat completion with the Secton API

## Before you start

Check `lifecycle/secton-api-lifecycle.yml`. Secton publicly announced on 2025-11-18 that Secton API
was being **permanently shut down**, operational only until 2025-12-19. The host still answers as of
2026-08-16 and the Console Terms governing API keys were re-issued effective 2026-08-06, but Secton
has published no retraction. Confirm the service is live before you depend on it.

You need an API key from https://console.secton.org/api. There is no self-serve public key and no
sandbox — the same key hits production.

## Step 1 — pick a model

Operation: `get-external-api-v1-models` (`GET /v1/models`). Covered fully by the
`secton-api-list-models` skill. The `model` value in your completion request must be an `id` from
that list.

The only model identifiers Secton has ever named in public are `throb-v1` and `copilot-zero`. Model
visibility is scoped to your key, so do not hard-code either — list first.

## Step 2 — call the operation

Operation: `post-external-api-v1-chat-completions` (`POST /v1/chat/completions`).

```
POST https://api.secton.org/v1/chat/completions
Authorization: Bearer <SECTON_API_KEY>
Content-Type: application/json

{
  "model": "<id from GET /v1/models>",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "..."}
  ],
  "temperature": 0.7,
  "max_tokens": 1024,
  "stream": false
}
```

Auth detail that trips people up: the OpenAPI declares the credential as a raw `apiKey` in the
`Authorization` header, but the server's own error message says "missing from bearer". Probing on
2026-08-16 showed **both** `Authorization: <key>` and `Authorization: Bearer <key>` are accepted.
Prefer the bearer form.

Request constraints from the schema — enforce them client-side, because the spec declares no 400:

- `model` — required, string.
- `messages` — required, at least one item; each item needs both `role`
  (`system` | `user` | `assistant`) and `content`.
- `temperature` — optional, defaults to `0.7`.
- `max_tokens` — optional integer, greater than 0 and at most **4096**.
- `stream` — optional boolean, defaults to `false`.

## Step 3 — read the response

Blocking (`stream: false`):

```json
{
  "id": "...",
  "object": "chat.completion",
  "created": 1750000000,
  "model": "...",
  "choices": [{"index": 0, "message": {"role": "assistant", "content": "..."}, "finish_reason": "stop"}],
  "usage": {"prompt_tokens": 0, "completion_tokens": 0, "total_tokens": 0}
}
```

Take the text from `choices[0].message.content`. Record `usage.total_tokens` — it is the only
metering signal you get, and Secton publishes no pricing, so your own accounting is the only
accounting.

## Step 4 — streaming

Set `stream: true` and read `chat.completion.chunk` frames. Append `choices[].delta.content` as it
arrives; the stream is finished when `choices[].finish_reason` is non-null.

Be aware the **published spec cannot tell you this**. The streaming response is declared under the
key `"200 ChatCompletionChunkSchema"`, which `$ref`s a `components.responses` entry that does not
exist. A generated client will not have a streaming type. Use
`overlays/secton-api-chat-api-overlay.yaml` in this repo, which restores
`ChatCompletionChunkSchema` and re-expresses the contract as a resolvable `x-streaming` block.

## Step 5 — handle errors

The envelope is `{"error": "<string>"}`. It is **not** RFC 9457 and carries no machine-readable
code, so branch on the HTTP status.

| Status | Body | What to do |
|---|---|---|
| 401 | `API key is missing from bearer. Get your API key at https://console.secton.org/api` | No `Authorization` header was sent. Add it. Do not retry as-is. |
| 401 | `Invalid or expired API key` | Credential rejected. Re-issue in the console. Never retry the same key. |
| 429 | not published | Rate limited. Back off. See below. |
| 200 | `{"message":"The <path> endpoint doesn't exist!"}` | **Soft-404.** You hit an unrouted path and got HTTP 200 anyway. Check the body key: `message` means "no such route", `error` means a real failure. |

Neither operation declares a single 4xx or 5xx response in the published OpenAPI, so treat any
non-200 as unmodelled and log the raw body.

## Step 6 — retries, and why they are dangerous here

**This operation is not idempotent and Secton offers no idempotency key.** A retried completion is a
second generation and a second charge against your credits. The first-party npm SDK retries three
times with exponential backoff *by default* — if you need at-most-once semantics, turn that off and
handle retries yourself at a layer that knows whether the first attempt succeeded.

Rate limits are not published. Console Terms §13 lists limit categories (request, rate, token,
throughput, concurrency, model-specific) but no values, and no `RateLimit-*` or `Retry-After`
header was observed on any response reachable without a key. The SDK's `RateLimitError` exposes a
`retryAfter`, so a retry hint exists on throttle — read it if present, otherwise back off
exponentially with jitter and cap your concurrency conservatively.

## Browser calls

The API returns `Access-Control-Allow-Origin: *` and allows the `Authorization` header, so it is
callable directly from a browser. Do not do it: that ships your key to every visitor, and Secton's
own Console Terms §6 forbid embedding credentials in publicly accessible source. Proxy through your
own backend.
