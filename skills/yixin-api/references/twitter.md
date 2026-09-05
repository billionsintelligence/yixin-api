# Twitter / X Search

Contract: [production Twitter OpenAPI](https://openapi.billionsintelligence.com/docs/openapi/twitter.json).
Use the shared key and [error handling](apis.md#status-handling).

```text
POST https://openapi.billionsintelligence.com/api/v2/twitter/search
```

Use this standalone endpoint for tweets. Do not add `source=twitter` to ordinary document search examples.

## Request

```json
{
  "query": "特斯拉 Robotaxi 进展",
  "search_mode": "fast",
  "count": 10
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `query` | yes | Nonempty keywords or question. |
| `search_mode` | no | `fast` (default), `advanced`, or `expert`; tweet retrieval depth. |
| `count` | no | Maximum results, 1–50; default 10. |
| `timeout` | no | Server waiting time, 1–120 seconds; omitted defaults are 15/60/110 for fast/advanced/expert. |

For deeper modes, keep the client timeout above 120 seconds (for example 150 seconds). Account allowances follow the portal; do not assume a permanent free daily quota from an old API description.

Example for a POSIX shell with `YIXIN_API_KEY` already configured:

```bash
curl --silent --show-error --fail-with-body --max-time 150 \
  'https://openapi.billionsintelligence.com/api/v2/twitter/search' \
  -H "X-API-KEY: ${YIXIN_API_KEY}" \
  -H 'Content-Type: application/json' \
  -d '{"query":"特斯拉 Robotaxi 进展","search_mode":"fast","count":10}'
```

## Read The Result

Check top-level `success` and each `result[].status` before using `result[].content[]`. HTTP `200` with `success:false` is a failed search; report `error` and apply bounded retry guidance.

Each hit can contain:

- `title`: author handle and a text prefix.
- `snippet`: tweet text; `link`: public tweet URL; `date`: publication time (documented as Beijing time).
- `extra.username`, `author_name`, `post_id`, `view_count`, `profile_image_url` when provided.

Preserve tweet links and timestamps when summarizing. Treat missing optional fields and an empty results array normally. Do not invent view counts or attribute an author's statements to another author.
