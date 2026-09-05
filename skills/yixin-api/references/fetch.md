# Webpage And Announcement Full Text

Contract: [production Fetch OpenAPI](https://openapi.billionsintelligence.com/docs/openapi/fetch.json).
Use the shared key and [error handling](apis.md#status-handling).

```text
POST https://openapi.billionsintelligence.com/api/v2/fetch
```

## Choose One Input

Send exactly one of `url` or `doc_id`:

```json
{"url":"https://example.com/article"}
```

The URL must be a public HTTP(S) webpage, at most 2048 characters. Internal addresses and platform URLs are not permitted targets. An article may still fail due to authentication, removal, or anti-crawling restrictions.

For announcement full text, first use `POST /v2/search` with `source=announcement`. Select a relevant hit from successful `result[].content[]`, take its nonempty `extra.doc_id`, and pass the exact string:

```json
{"doc_id":"<exact extra.doc_id returned by announcement search>"}
```

Treat `doc_id` as an opaque handle. Do not decode/re-encode it, trim parts, substitute a title or document number, or manufacture one. If the hit has no handle, report that full text is unavailable for that hit. `INVALID_DOC_ID` requires correcting the input; repeatedly retrying the same handle will not help.

The published production contract currently exposes announcement full text; report/expert search snippets do not imply access to those sources' full text. Do not work around `SOURCE_NOT_LICENSED`. Future source availability must be verified against the current contract and account permissions.

## Announcement Workflow In Python

Requires `requests` and the user's `YIXIN_API_KEY` environment variable. Execute only for a requested search/full-text retrieval; each API call can consume usage.

```python
import os
import requests

base = "https://openapi.billionsintelligence.com/api"
headers = {"X-API-KEY": os.environ["YIXIN_API_KEY"]}
response = requests.post(
    base + "/v2/search", headers=headers,
    json={"query": "宁德时代 回购进展", "source": "announcement", "count": 5},
    timeout=150,
)
response.raise_for_status()
found = response.json()
if found.get("success") is not True:
    raise RuntimeError(found.get("error") or "Search failed")

candidates = [
    hit
    for result in found.get("result", [])
    if result.get("status") == "success"
    for hit in result.get("content", [])
    if (hit.get("extra") or {}).get("doc_id")
]
if not candidates:
    raise RuntimeError("No accessible announcement doc_id in these results")
# For this example, take the first candidate; select for relevance in real work.
doc_id = candidates[0]["extra"]["doc_id"]
response = requests.post(
    base + "/v2/fetch", headers=headers, json={"doc_id": doc_id}, timeout=150,
)
response.raise_for_status()
document = response.json()
if document.get("success") is not True:
    raise RuntimeError(f"{document.get('code')}: {document.get('error')}")
print(document.get("title", ""))
print(document.get("content", ""))
if document.get("truncated"):
    print("Full text was truncated; use pagination to retrieve the remainder.")
```

## Full Text And Pagination

Without `page` or `max_chars`, Fetch returns full Markdown in `content`, subject to the documented 200,000-character cap. Inspect `truncated`; do not describe truncated text as complete.

To paginate, send `page` (1-based) and/or `max_chars` (500–12000, default 6000), keeping the same URL/handle and page width across requests:

```json
{"doc_id":"<exact search handle>","page":2,"max_chars":6000}
```

Use returned `pages` and `total_pages` to track progress. An out-of-range request can return the last page, so stop at the total or on a repeated page instead of looping indefinitely. Each page is another request. `keyword` is optional for locating document pages and is valid only with `doc_id`, not `url`.

Successful responses include `success:true`, `type` (`web`/`document`), `source`, `title`, `content`, `pages`, `total_pages`, `total_chars`, and `truncated`. HTTP `200` with `success:false`/`UPSTREAM_ERROR` is a failed retrieval, not an empty successful document. Inspect `code` for `400`/`403`/`422`; do not treat every `403` as a broken API key.
