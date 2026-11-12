# PDF Translation

Contract: [production translation OpenAPI](https://openapi.billionsintelligence.com/.well-known/openapi/doc-translation.json). API base: `https://openapi.billionsintelligence.com/api`.

Use this for translated PDF output, not for extracting Markdown (use [document.md](document.md)). Only user-authorized PDFs are uploaded. The API accepts editable and scanned PDFs, at most 209715200 bytes (200 MiB, described as 200 MB in the contract) and 2000 pages.

## Workflow

| Step | Method and path |
| --- | --- |
| Create upload | `POST /v2/doc-translation/uploads` |
| Direct upload | PUT the returned HTTPS `presigned_put_url` |
| Complete upload | `POST /v2/doc-translation/uploads/{upload_id}/complete` |
| Submit translation | `POST /v2/doc-translation/tasks` |
| Poll | `GET /v2/doc-translation/tasks/{task_id}/status` |
| Retrieve result | `GET /v2/doc-translation/tasks/{task_id}/result` |
| Cancel when requested | `POST /v2/doc-translation/tasks/{task_id}/cancel` |

Read [PDF upload and credential isolation](document.md#upload-and-submit) for the shared upload sequence. Use the translation paths above, not the parsing paths. Send `X-API-KEY` only to the platform gateway. PUT the bytes with all returned `presigned_headers`, without API key or Portal cookies. The upload URL expires after 900 seconds per current contract; honor the actual `expires_at`.

Create upload (replace the size with the actual byte count):

```json
{"filename":"report.pdf","content_type":"application/pdf","size_bytes":1048576,"idempotency_key":"translation-upload-unique-001"}
```

Wire filename is 1–128 ASCII letters/digits/`.`/`_`/`-`; a safe wire alias may be used for a Chinese local filename without changing the user's file. Complete must succeed and probe the source page count before task submission.

Submit:

```json
{"upload_id":"<ready upload_id>","source_language":"auto","target_language":"zh-CN","idempotency_key":"translation-task-unique-001"}
```

`source_language` only accepts `auto` (default); `target_language` accepts `zh-CN` (default) or `en`. Only `upload_id`, `idempotency_key`, `source_language` and `target_language` are allowed at submission. Do not copy parsing's `mode`/boolean `download_url`, or internal `workload_class`/`translation_depth`; undeclared fields return 422.

## Completion And Downloads

202 means accepted; save top-level `task_id`. Poll `/status` with a delay (for example 30 seconds) and a bounded wait (for example 10 minutes), honoring `Retry-After`. Nonterminal states: `submitting`, `pending`, `processing`. Success is `completed`; then fetch `/result`. Stop on `failed`, `cancelled`, `unavailable`, or `stalled`, and report returned errors. Unknown states or a local deadline are not reasons to submit a duplicate or cancel automatically.

Use the result's `download_url` for the translated PDF and `comparison_download_url` for the comparison PDF, which may be null if no comparison artifact exists. Preserve the returned response structure and signed query parameters. Current contract gives result links a 30-day lifetime; do not treat them as permanent or expose them in logs. Download without the platform key/cookies and do not overwrite local user files without approval. Report only artifacts actually returned.

## Idempotency And Billing

- Optional `idempotency_key` is at most 256 characters. Persist it before submission. Duplicate upload creation returns 409 without a new presigned URL; duplicate task submission returns 200 and reuses the existing task. Do not generate a new task key to blindly retry a timeout or change languages on an already accepted task.
- Current documented concurrency is 10 globally and one in-progress task per user, with at most five unexpired pending uploads per user. Task concurrency refusal returns 503 with `Retry-After: 30`; it is not acceptance. These are service limits, not client-side permission to generate batches.
- Metering is `document.translate`, in pages: reserve source pages, settle source pages on `completed`; failed/cancelled/unavailable/stalled tasks release reservations under the published contract. This differs from parsing's successfully parsed page count. Do not promise a fixed account price or infer settlement only from HTTP status.
- Keep the same account/key for its uploads and tasks. Check both HTTP errors and response state; consult the current contract for detailed error codes. Do not supply trusted identity, price, or quota fields yourself.
