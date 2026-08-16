---
name: secton-api-list-models
description: >-
  Discover which models a Secton API key can actually use, and verify a key works, using the single
  read-only operation the API publishes.
api: secton-api:secton-api-models-api
operations:
  - get-external-api-v1-models
base_url: https://api.secton.org
generated: '2026-08-16'
method: generated
source: >-
  openapi/secton-api-models-api-openapi.yml (operationId verified against the spec),
  conventions/secton-api-conventions.yml, errors/secton-api-problem-types.yml,
  data-model/secton-api-data-model.yml
---

# List the models available to a Secton API key

## What this is for

Two jobs, both served by one operation:

1. Find the `model` id to pass to `POST /v1/chat/completions`.
2. **Validate a credential.** This is the only read-only, side-effect-free, non-billable operation
   Secton publishes, which makes it the correct health check and key-verification call. Never probe
   with a chat completion — that costs credits.

## The operation

Operation: `get-external-api-v1-models` (`GET /v1/models`).

```
GET https://api.secton.org/v1/models
Authorization: Bearer <SECTON_API_KEY>
```

No parameters. The operation declares `parameters: []` — there is no filtering, no sorting, no
`limit`, no cursor. You always get the complete set.

Both `Authorization: <key>` and `Authorization: Bearer <key>` were accepted when probed on
2026-08-16, though the OpenAPI documents only the raw form. Prefer bearer.

## The response

```json
{
  "object": "list",
  "data": [
    {"id": "...", "object": "model"}
  ]
}
```

That is the whole schema. Each entry has exactly two fields, `id` and `object`. There is **no**
context window, no pricing, no capability flags, no modality, no deprecation marker, no
`created` timestamp. If you need to know what a model can do, Secton does not tell you through the
API — and it publishes no model documentation outside the console either.

## Interpreting the list

- The list is **scoped to the credential**. Two keys may legitimately return different sets, so
  cache per-key, not globally.
- The only model ids Secton has named in public are `throb-v1`
  (https://secton.org/blog/throb-deprecation) and `copilot-zero` (the npm SDK README). Treat those
  as examples. Do not hard-code them — call this operation and use what comes back.
- Because there is no deprecation marker on a model entry, a model can disappear from the list
  between calls with no in-band warning. Re-list before a long-running job, and fail soft if a
  previously-seen id is gone.

## Errors

Envelope is `{"error": "<string>"}` — not RFC 9457, no error code.

| Status | Body | Meaning |
|---|---|---|
| 401 | `API key is missing from bearer. Get your API key at https://console.secton.org/api` | No `Authorization` header. |
| 401 | `Invalid or expired API key` | Key rejected — revoked, expired, or wrong. |
| 200 | `{"message":"The /v1/models endpoint doesn't exist!"}` | You are not talking to the API you think you are. **This host returns HTTP 200 for unrouted paths.** A 200 whose body has a `message` key rather than `object: "list"` is a soft-404, not a result. |

Check the body shape, not just the status. Assert `object === "list"` before trusting `data`.

The published OpenAPI declares only a 200 for this operation, so every row above except the first
is undocumented behaviour observed live.

## Using it as a health check

Safe to call on a schedule: it is a `GET`, it consumes no credits, and it is the only operation
that can distinguish "key is valid" from "service is up". Combine with
https://status.secton.org, which publishes an `API` component with 90 days of history.
