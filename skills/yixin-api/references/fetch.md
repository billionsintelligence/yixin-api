# Webpage, Document And Twitter Full Text

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
{"doc_id":"<exact extra.doc_id returned by announcement search>","source":"announcement"}
```

Treat `doc_id` as an opaque handle. Do not decode/re-encode it, trim parts, substitute a title or document number, or manufacture one. If the hit has no handle, report that full text is unavailable for that hit. `INVALID_DOC_ID` requires correcting the input; repeatedly retrying the same handle will not help.

Every `doc_id` request must include `source`. Missing/invalid source returns `400 INVALID_ARGUMENT`; a mismatch returns `422 SOURCE_MISMATCH`. Use the source matching the handle, never change it to bypass access controls:

| Search source | Fetch source | Additional access requirement |
| --- | --- | --- |
| `announcement` | `announcement` | No business whitelist |
| `report` | `report` | Account report-full-text whitelist |
| `expert` | `meeting` (`expert` alias accepted) | Account meeting-full-text whitelist |
| Standalone Twitter search | `twitter` | No business whitelist |

All still require a valid API key and Fetch API access. Environment source availability also applies. Receiving a handle or upgrading a plan does not grant whitelist access. Do not work around `SOURCE_NOT_LICENSED`; report the refusal and direct the user to support for required access.

For Twitter use the exact `extra.doc_id` from Twitter search: a 1–25 character ASCII digit string. Do not convert it to a number, derive it from optional `post_id`, or send `url`/`keyword`:

```json
{"doc_id":"2071534140995112960","source":"twitter"}
```

The sample ID only illustrates the shape; execute with the selected search hit's ID. Twitter returns `type=document`, `source=source_id=twitter`, and the same value in `doc_id` and legacy `id`. Its `content` is tweet text; preserve author/link/date from the search hit rather than expecting them in the full text.

## Images And Billing

Optional JSON boolean `public_image_url` defaults to `false`. For announcement/report/meeting (including `expert`) set it to `true` to receive accessible, temporarily signed image URLs. Preserve signatures, keep them out of logs, and retrieve again after expiry. Web and Twitter retain original image URLs regardless of this flag; `false` does not mean all images are removed. Never attach the platform key when fetching images.

Webpage, announcement, report and meeting full text share `fetch.requests`; Twitter full text shares `twitter.requests` with Twitter search. Search and full-text retrieval are separate potentially billable requests. Shared metrics do not make the second request free or remove whitelists. Prices, quotas and old resource packs must be checked against actual account entitlements.

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
    base + "/v2/fetch", headers=headers,
    json={"doc_id": doc_id, "source": "announcement"}, timeout=150,
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

Without `page` or `max_chars`, Fetch returns Markdown in `content` with `total_pages=1`, capped at 200,000 characters for URL requests and 60,000 for document/Twitter `doc_id`. Inspect `truncated`; do not describe truncated text as complete or infer completeness from `total_pages=1`.

To paginate, send `page` (1-based) and/or `max_chars` (500–12000). If only `page` is sent, the width defaults to 6000 for URLs and 3000 for doc_id. Keep the same URL/handle, source and page width across requests; paginated content includes a `[Page N/M]` marker:

```json
{"doc_id":"<exact announcement search handle>","source":"announcement","page":2,"max_chars":3000}
```

Use returned `pages` and `total_pages` to track progress. An out-of-range request can return the last page, so stop at the total or on a repeated page instead of looping indefinitely. Each page is another request. `keyword` checks whether non-Twitter document full text contains the keyword; it does not locate a page or replace pagination. A miss returns HTTP 200 with `success:false` and `DOCUMENT_NOT_FOUND`. It is invalid with URL or Twitter requests.

Successful responses include `success:true`, `type` (`web`/`document`), `source`, `title`, `content`, `pages`, `total_pages`, `total_chars`, and `truncated`. HTTP `200` with `success:false`/`UPSTREAM_ERROR` is a failed retrieval, not an empty successful document. Inspect `code` for `400`/`403`/`422`; do not treat every `403` as a broken API key.
