# API Calls

Use this reference to call the production Yixin OpenAPI gateway. All requests use:

```http
Content-Type: application/json
Accept: application/json
X-API-KEY: <api-specific-key>
```

Do not include real API keys in examples. Use environment variables or load keys from the user's private mapping.

## Load Keys From Mapping

Example shell pattern:

```bash
SEARCH_API_KEY="$(jq -r '.search' ~/.config/yixin-api/api-keys.json)"
FIN_DB_API_KEY="$(jq -r '.fin_db' ~/.config/yixin-api/api-keys.json)"
```

## SearchAggregator Search API

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
| `time_range` | no | Examples: `past 3 days`, `past 1 month`, `from 2025-01-01 to 2025-06-30`. |

curl:

```bash
SEARCH_API_KEY="$(jq -r '.search' ~/.config/yixin-api/api-keys.json)"

curl -sS --fail-with-body \
  -X POST "https://openapi.billionsintelligence.com/api/v2/search" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-API-KEY: ${SEARCH_API_KEY}" \
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
api_key = os.environ["SEARCH_API_KEY"]
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
    with urllib.request.urlopen(request, timeout=120) as response:
        print(response.status)
        print(response.read().decode("utf-8", errors="replace"))
except urllib.error.HTTPError as exc:
    body = exc.read().decode("utf-8", errors="replace")
    if exc.code == 429:
        raise SystemExit("额度已用完，请联系销售升级：https://www.billionsintelligence.com") from exc
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

curl:

```bash
FIN_DB_API_KEY="$(jq -r '.fin_db' ~/.config/yixin-api/api-keys.json)"

curl -sS --fail-with-body \
  -X POST "https://openapi.billionsintelligence.com/api/v1/fin_db" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-API-KEY: ${FIN_DB_API_KEY}" \
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
api_key = os.environ["FIN_DB_API_KEY"]
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
    with urllib.request.urlopen(request, timeout=60) as response:
        print(response.status)
        print(response.read().decode("utf-8", errors="replace"))
except urllib.error.HTTPError as exc:
    body = exc.read().decode("utf-8", errors="replace")
    if exc.code == 429:
        raise SystemExit("额度已用完，请联系销售升级：https://www.billionsintelligence.com") from exc
    print(exc.code)
    print(body)
    raise
```

## Status Handling

| Status | Meaning | User guidance |
| --- | --- | --- |
| `200` | Request reached the API. | Check the JSON body's `success`, `result`, and `error` fields. |
| `400` | Invalid request body or empty required field. | Fix JSON and required parameters. |
| `401` | Missing or invalid API key. | Confirm `X-API-KEY` and mapping. |
| `403` | Key is not authorized for this API. | Create or use the key bound to the requested API. |
| `429` | Rate limit or quota exceeded. | `额度已用完，请联系销售升级：https://www.billionsintelligence.com` |
| `5xx` | Gateway or upstream service failure. | Retry later and preserve request/response context for support. |
