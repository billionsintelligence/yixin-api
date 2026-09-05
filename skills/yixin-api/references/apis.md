# API Calls

Use this reference for shared conventions, financial queries, and document search. API base: `https://openapi.billionsintelligence.com/api`. JSON requests use:

```http
Content-Type: application/json
Accept: application/json
X-API-KEY: <product-subscription-key>
```

Reuse the same key for APIs enabled for its application/subscription. Do not include real API keys in examples. Use `YIXIN_API_KEY` or the user's private key file. Multipart media upload must let the HTTP client set its content type and boundary; it is not a JSON request.

| Need | Method and path (relative to API base) | Reference |
| --- | --- | --- |
| Financial database query | `POST /v1/fin_db` | Below |
| Multi-source document search | `POST /v2/search` | Below |
| Twitter/X search | `POST /v2/twitter/search` | [twitter.md](twitter.md) |
| Webpage / announcement full text | `POST /v2/fetch` | [fetch.md](fetch.md) |
| Media download / upload / query / cancel | `/v2/media/task` family | [media.md](media.md) |

Public contracts: `https://openapi.billionsintelligence.com/docs/openapi/<slug>.json`, where slug is `fin-db`, `search`, `twitter`, `fetch`, or `media`. These references describe observed production contracts; consult the current contract for changes. Account-specific quota/pricing follows the portal, not historical free-period descriptions.

## Load The Key

Example shell pattern:

```bash
YIXIN_API_KEY="$(jq -r '.api_key' ~/.config/yixin-api/api-key.json)"
```

## search API

Endpoint:

```text
POST https://openapi.billionsintelligence.com/api/v2/search
```

Use the path without a trailing slash.

Request body:

```json
{
  "query": "宁德时代最新业绩",
  "source": "report",
  "search_mode": "advanced",
  "count": 10,
  "time_range": "past 1 month"
}
```

Fields:

| Field | Required | Notes |
| --- | --- | --- |
| `query` | yes | Search keywords or natural-language question. |
| `source` | no | `web`, `academic`, `image`, `video`, `announcement`, `report`, or `expert`. Defaults to `web`. |
| `search_mode` | no | `fast`, `advanced`, or `expert`. Defaults to `fast`. |
| `count` | no | Maximum results, `1` to `50`. |
| `timeout` | no | Per-engine timeout in seconds, `1` to `120`. |
| `time_range` | no | Examples: `past 3 days`, `past 2 weeks`, `past 1 month`. Omit for no time filter. |

For advanced/expert mode, allow at least 120 seconds in the HTTP client; if overriding the server's `timeout` (up to 120 seconds), keep the client timeout larger. `source=expert` is an expert-document source; `search_mode=expert` is a retrieval depth. Use the standalone Twitter endpoint for tweets.

Inspect `success` and each `result[].status`; successful items are in `result[].content[]`. Preserve `title`, `link`, `snippet`, `date`, and `extra`. Results may be empty. A snippet is not full text. For announcement full text, pass the returned `extra.doc_id` unchanged to Fetch. Report/expert full text is not currently licensed for public access, even though their search snippets are available. See [fetch.md](fetch.md).

curl:

```bash
YIXIN_API_KEY="$(jq -r '.api_key' ~/.config/yixin-api/api-key.json)"

curl -sS --fail-with-body \
  -X POST "https://openapi.billionsintelligence.com/api/v2/search" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-API-KEY: ${YIXIN_API_KEY}" \
  --data-binary @- <<'JSON'
{
  "query": "宁德时代最新业绩",
  "source": "report",
  "search_mode": "advanced",
  "count": 10,
  "time_range": "past 1 month"
}
JSON
```

Python:

```python
import json
import os
import urllib.error
import urllib.request

url = "https://openapi.billionsintelligence.com/api/v2/search"
api_key = os.environ["YIXIN_API_KEY"]
payload = {
    "query": "宁德时代最新业绩",
    "source": "report",
    "search_mode": "advanced",
    "count": 10,
    "time_range": "past 1 month",
}

request = urllib.request.Request(
    url,
    data=json.dumps(payload, ensure_ascii=False).encode("utf-8"),
    method="POST",
    headers={
        "Content-Type": "application/json",
        "Accept": "application/json",
        "X-API-KEY": api_key,
    },
)

try:
    with urllib.request.urlopen(request, timeout=150) as response:
        print(response.status)
        print(response.read().decode("utf-8", errors="replace"))
except urllib.error.HTTPError as exc:
    body = exc.read().decode("utf-8", errors="replace")
    if exc.code == 402:
        raise SystemExit("计费或额度拒绝，请检查控制台额度与套餐；不要原样重试。") from exc
    if exc.code == 429:
        raise SystemExit("请求受限，请降低频率，并结合 Retry-After 和响应体检查限制类型。") from exc
    print(exc.code)
    print(body)
    raise
```

## FinData fin_db API

Endpoint:

```text
POST https://openapi.billionsintelligence.com/api/v1/fin_db
```

Request body:

```json
{
  "query": "美国2025年平均每日成交金额是多少",
  "data_sources": ["auto"]
}
```

Fields:

| Field | Required | Notes |
| --- | --- | --- |
| `query` | yes | Natural-language financial data question. |
| `data_sources` | no | `auto`, `A股财务行情数据库`, `海外财务行情数据库`, or `宏观行业数据库`. Use string or array form. |

Specify the company/market, indicator, and date range in the question. Allow at least 120 seconds for complex financial queries. HTTP `200` alone is not success: inspect `success` and every `result[].status`. `result[].content` is normally Markdown table/text, unlike the array of search hits returned by search/twitter; preserve source, units, and time period when reporting values.

curl:

```bash
YIXIN_API_KEY="$(jq -r '.api_key' ~/.config/yixin-api/api-key.json)"

curl -sS --fail-with-body \
  -X POST "https://openapi.billionsintelligence.com/api/v1/fin_db" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-API-KEY: ${YIXIN_API_KEY}" \
  --data-binary @- <<'JSON'
{
  "query": "美国2025年平均每日成交金额是多少",
  "data_sources": ["auto"]
}
JSON
```

Python:

```python
import json
import os
import urllib.error
import urllib.request

url = "https://openapi.billionsintelligence.com/api/v1/fin_db"
api_key = os.environ["YIXIN_API_KEY"]
payload = {
    "query": "美国2025年平均每日成交金额是多少",
    "data_sources": ["auto"],
}

request = urllib.request.Request(
    url,
    data=json.dumps(payload, ensure_ascii=False).encode("utf-8"),
    method="POST",
    headers={
        "Content-Type": "application/json",
        "Accept": "application/json",
        "X-API-KEY": api_key,
    },
)

try:
    with urllib.request.urlopen(request, timeout=150) as response:
        print(response.status)
        print(response.read().decode("utf-8", errors="replace"))
except urllib.error.HTTPError as exc:
    body = exc.read().decode("utf-8", errors="replace")
    if exc.code == 402:
        raise SystemExit("计费或额度拒绝，请检查控制台额度与套餐；不要原样重试。") from exc
    if exc.code == 429:
        raise SystemExit("请求受限，请降低频率，并结合 Retry-After 和响应体检查限制类型。") from exc
    print(exc.code)
    print(body)
    raise
```

## Status Handling

| Status | Meaning | User guidance |
| --- | --- | --- |
| `200` | Request processed, not necessarily business success. | Search/twitter/fin_db: check `success` and result statuses. Fetch: check `success`/`code`. Media: check envelope `code` and task status. |
| `202` | Media task accepted. | Save `data.task_id` and query it; neither completion nor billing success is implied. |
| `400` / `422` | Invalid request or parameter. | Fix the indicated field. `INVALID_DOC_ID`: use the original search result handle; do not repeatedly resubmit the invalid value. |
| `401` | Missing or invalid API key. | Confirm `X-API-KEY` and that the key was copied correctly. |
| `402` | Billing/entitlement refusal, such as insufficient available quota/balance. | Inspect the error code and current portal quota. Stop unchanged retries; an upgrade may be needed. |
| `403` | API subscription, source license, or URL policy prevents access. | Inspect the body: `SOURCE_NOT_LICENSED` and `URL_NOT_ALLOWED` are not fixed by key rotation. |
| `404` | Resource/task not found. | Check the task ID and environment. Do not create replacement media tasks automatically. |
| `413` | Upload exceeds deployment size limit. | Check the file and published limit before another upload. |
| `429` | Rate limiting or a quota limit indicated by the body. | Honor `Retry-After`, reduce concurrency, and inspect the error. Do not automatically claim the account is out of credits. |
| `5xx` / transport timeout | Gateway/upstream failure or unknown request outcome. | Preserve request ID/status and redacted error context. See retry guidance below. |

For an actual quota/plan upgrade, direct users to `https://openapi.billionsintelligence.com/usage/quota` and sales at `https://www.billionsintelligence.com`.

For a transient read/query failure or rate limit, use bounded backoff (for example, at most two retries, respecting `Retry-After`; without it, wait 2 then 5 seconds). Stop and report persistent failures. Avoid high-concurrency or unbounded retries. Queries can consume usage; do not promise that retries or failed calls are free.

Do not automatically retry media creation/upload when a timeout leaves acceptance unknown: a duplicate task can incur additional usage. Once a task ID is known, query that same task. See [media.md](media.md).
