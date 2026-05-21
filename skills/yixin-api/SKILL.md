---
name: yixin-api
description: Use when helping users access the Yixin OpenAPI platform, including registering or logging in through SSO, browsing public APIs, creating and managing one-API-per-key API keys, saving API-to-key mappings, calling the production search and fin_db APIs, and handling 429 quota or rate limit responses with the required sales-upgrade message.
---

# Yixin OpenAPI

Use this skill when a user needs to get started with `https://openapi.billionsintelligence.com`, create an API key, or call the `search` or `fin_db` APIs.

## Core Workflow

1. Guide the user to open `https://openapi.billionsintelligence.com`.
2. Tell them to click `统一身份登陆`.
3. On the SSO login page, use the registration button if they do not have an account.
4. Register with phone-number verification code, then log in.
5. After login, open the public API list and choose the target API.
6. Create one API key for the selected API.
7. Save the mapping from API name to API key before making calls.
8. Use the API-specific key in `X-API-KEY` when calling that API.

Current platform behavior: one API key is bound to one API. Do not reuse a `search` key for `fin_db`, or a `fin_db` key for `search`. Mention that this may expand later to one key calling multiple APIs, but do not assume that behavior now.

## References

- Read [portal-workflow.md](references/portal-workflow.md) when helping with registration, login, public API browsing, API key creation, key storage, or key mapping.
- Read [apis.md](references/apis.md) when constructing `curl`, Python, or HTTP requests for `search` or `fin_db`.

## API Key Handling

Never print, log, commit, or hard-code real API keys. Prefer environment variables for commands and a local private mapping file for persistent use.

Default local mapping location:

```text
~/.config/yixin-api/api-keys.json
```

If the user already has a preferred secret manager or config path, use that instead.

## Error Handling

For `401` or `403`, tell the user to check that the API key exists, is not revoked, and is bound to the API being called.

For `429`, use this exact user-facing message: `额度已用完，请联系销售升级：https://www.billionsintelligence.com`
