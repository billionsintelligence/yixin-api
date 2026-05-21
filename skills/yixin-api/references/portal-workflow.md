# Portal Workflow

Use this reference for onboarding, login, API discovery, API key creation, and key mapping.

## Register And Log In

1. Open `https://openapi.billionsintelligence.com`.
2. Click `统一身份登陆`.
3. On the SSO login page, click the registration button if the user has no account.
4. Register with phone number and SMS verification code.
5. After registration, log in through the same SSO page.

If the user cannot receive the SMS code, ask them to verify the phone number and contact platform support. Do not invent alternate registration channels.

## Find Public APIs

After login, guide the user to the public API list in the portal. The currently supported public APIs for this skill are:

| API key name | Portal/API display intent | Gateway path |
| --- | --- | --- |
| `search` | SearchAggregator 搜索聚合 API | `/api/v2/search` |
| `fin_db` | FinData 数据库问数 API | `/api/v1/fin_db` |

Use the portal's current API list as the source of truth for the exact visible display names.

## Create API Keys

1. Choose the target public API.
2. Create an API key for that API.
3. Record which API the key belongs to.

Current behavior: one API key can call only the API it was created for. Create separate keys for `search` and `fin_db`.

Future behavior may allow one key to call multiple APIs; do not assume this until the portal explicitly supports it.

## Save API-To-Key Mapping

Prefer a local private file outside the repository:

```text
~/.config/yixin-api/api-keys.json
```

Recommended shape:

```json
{
  "search": "<search-api-key>",
  "fin_db": "<fin-db-api-key>"
}
```

Set strict local permissions when creating this file:

```bash
mkdir -p ~/.config/yixin-api
chmod 700 ~/.config/yixin-api
chmod 600 ~/.config/yixin-api/api-keys.json
```

If the user uses a secret manager, store the same API-to-key mapping there instead.

## Pick The Correct Key

Before making a request:

1. Identify the requested API: `search` or `fin_db`.
2. Load the matching key from the mapping.
3. Send it as `X-API-KEY`.

If no key exists for the requested API, guide the user back to the portal to create one.

## User-Friendly Limit Handling

If the gateway returns `429`, use this exact user-facing message:

```text
额度已用完，请联系销售升级：https://www.billionsintelligence.com
```
