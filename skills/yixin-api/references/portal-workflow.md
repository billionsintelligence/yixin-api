# Portal Workflow

Use this reference for onboarding, login, subscription, API key retrieval, and key storage. Reuse an existing working key without repeating onboarding.

## Register And Log In

1. Open `https://openapi.billionsintelligence.com`.
2. Click `统一身份登陆`.
3. On the SSO login page, click the registration button if the user has no account.
4. Register with phone number and SMS verification code.
5. After registration, log in through the same SSO page.

If the user cannot receive the SMS code, ask them to verify the phone number and contact platform support. Do not invent alternate registration channels.

## Subscribe To The Public API Product

After login, open [API services](https://openapi.billionsintelligence.com/api-service) to view the available APIs and subscription/plan status. Follow the current portal's subscription prompts. The production API catalog includes:

| API | Portal/API display intent | Gateway path |
| --- | --- | --- |
| `search` | search | `/api/v2/search` |
| `fin_db` | FinData 数据库问数 API | `/api/v1/fin_db` |
| `twitter` | 推特（X）检索 | `/api/v2/twitter/search` |
| `fetch` | 网页 / 公告全文获取 | `/api/v2/fetch` |
| `media` | 异步媒体解析 | `/api/v2/media/task` and its upload/query/cancel operations |

Use the portal's current product and API list as the source of truth for the exact visible display names.

Reuse one key across the APIs enabled for its application/subscription; do not require a separate key for each API. Access to newly introduced APIs and licensed sources still depends on permissions. If a valid key receives `403`, check the target API subscription and error body instead of assuming the key is invalid.

## Get The API Key

1. Confirm an active subscription in the portal.
2. Open [API Key management](https://openapi.billionsintelligence.com/api-keys) to create or retrieve an active key, following the UI's subscription prompts.
3. View current allowances at [Usage/quota](https://openapi.billionsintelligence.com/usage/quota). Do not hard-code trial allowances or prices from old docs.

Rotate or revoke a key only when requested; this can invalidate existing clients. A missing local copy alone is not a reason to revoke it.

## Save The Key

Prefer the user's existing secret manager or `YIXIN_API_KEY` environment variable. For persistent file storage, use a private file outside the repository:

```text
~/.config/yixin-api/api-key.json
```

Recommended shape:

```json
{
  "api_key": "<product-subscription-key>"
}
```

Set strict local permissions when creating this file. On Unix:

```bash
mkdir -p ~/.config/yixin-api
chmod 700 ~/.config/yixin-api
chmod 600 ~/.config/yixin-api/api-key.json
```

On Windows, use the user's profile directory and a user-restricted file ACL. Saving a key is optional for a one-off request. Never print or commit the real value.

Legacy note: an earlier client may use `api-keys.json` or per-API variables. Check the active credential and subscription before migrating the selected client configuration to `YIXIN_API_KEY`. Preserve working credentials until the replacement is verified; do not automatically delete legacy files.

## Use The Key

Before making a request:

1. Load the key from the file or the `YIXIN_API_KEY` environment variable.
2. Send it as `X-API-KEY` for enabled APIs, including media upload and task queries.

Portal cookies, OAuth user tokens, billing service tokens, and internal identity headers are not required in public API examples. The gateway supplies its trusted billing context.

If the user has no key, guide them back to the portal to subscribe to the public API product.

## User-Friendly Limit Handling

For `402`, check quota/balance and entitlements. For `429`, reduce frequency and follow `Retry-After` when present; the status alone does not prove quota exhaustion. Sales contact for an actual plan upgrade: `https://www.billionsintelligence.com`. See [Status Handling](apis.md#status-handling) for bounded retries and other errors.
