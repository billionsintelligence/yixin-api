# PDF Parsing

Contract: [production Document OpenAPI](https://openapi.billionsintelligence.com/docs/openapi/document.json). API base: `https://openapi.billionsintelligence.com/api`. Use `X-API-KEY` on gateway calls; never on object-storage PUTs or result/image downloads.

This parses an uploaded PDF into Markdown and paragraph-level details; it is not Search or Media. Upload only a user-selected file with authorization. Do not execute the examples just to validate connectivity: task submission can consume quota.

## Upload And Submit

1. `POST /v2/document/uploads` with filename, MIME type and actual file size:

   ```json
   {"filename":"report.pdf","content_type":"application/pdf","size_bytes":1048576,"idempotency_key":"upload-report-unique-001"}
   ```

   Use the actual byte count, not the sample. The wire filename allows only ASCII letters, digits, `.`, `_`, `-`, length 1–128. For a Chinese local filename, send a safe ASCII wire filename without renaming or overwriting the user's file. A successful create returns 201 with `upload_id`, `presigned_put_url`, `presigned_headers`, `expires_at`, and `max_size_bytes`. Honor the returned limit and expiry; do not copy Media's 2 GiB limit.

2. PUT the PDF bytes to `presigned_put_url`, carrying all `presigned_headers` unchanged. Use a separate HTTP request/client without the platform API key, Portal cookies or default gateway authentication. Do not log the signed URL, follow an unexpected cross-host redirect with credentials, or invent a storage URL. Upload success is not task submission.
3. `POST /v2/document/uploads/{upload_id}/complete` with the gateway API key. It validates size, Content-Type and PDF magic and probes total pages. Require `status=ready` and preserve `total_pages` before submitting.
4. `POST /v2/document/tasks`:

   ```json
   {"upload_id":"<ready upload_id>","mode":"standard","download_url":false,"idempotency_key":"parse-report-unique-001"}
   ```

   `mode` is `flash`, `standard` (default), or `pro`. `download_url` is an optional JSON boolean, default false. When true, successful results may contain publicly accessible image URLs; this is not an input file URL. Strings or numbers are invalid (`422 INVALID_DOWNLOAD_URL`). Do not enable it unless the user's requested output needs those image links. Preserve returned signed links and never add the API key when retrieving images.

Submission returns 202 with top-level `task_id` and `status`, not Media's `data.task_id`. Save the ID before doing anything else. Acceptance does not prove processing or billing success.

## Poll, Read And Cancel

| Operation | Method and path |
| --- | --- |
| Status | `GET /v2/document/tasks/{task_id}/status` |
| Result | `GET /v2/document/tasks/{task_id}/result` |
| Cancel when requested | `POST /v2/document/tasks/{task_id}/cancel` |

Poll about every 30 seconds, honor `Retry-After`, and bound the wait (for example 10 minutes). Keep `task_id` and last status for later continuation; reaching the local deadline is not task failure and does not authorize cancellation.

- Nonterminal: `submitting`, `pending`, `processing`.
- Success: `completed`; only then fetch `/result`.
- Other terminal states: `failed`, `cancelled`, `unavailable`, `stalled`; stop and report `error_code` / `error_message`. Unknown states require reporting, not an unbounded polling loop.

Do not reuse Media's `done`/`canceled` spelling. Result-before-completion can return 404; this alone is not proof the task was lost. The result envelope contains `success`, `task_id`, `status`, and `result`. Markdown is `result.data.markdown`; paragraph details are `result.data.detail` (page ID, type, text, position, score). Preserve `total_pages`, `parsed_pages` and `failed_pages` where supplied; do not call a partial parse complete coverage.

Cancellation of an already terminal task returns its current state. Inspect the response, and do not promise a refund solely from HTTP 200.

## Idempotency, Errors And Billing

- Upload creation with the same `idempotency_key` returns 409 and does not issue another presigned URL. Reuse the saved upload response if still valid. If it was lost, explain the ambiguity before creating another upload; a 409 is not upload success.
- Task submission with the same key returns 200 and reuses the existing task. Persist the key before sending and use it to reconcile an uncertain submission; do not blindly switch keys after a timeout. Reuse ignores a changed `download_url`; changing that option requires a new key and a new submission, with the ready-upload constraints still enforced. Do not assume an already consumed upload can be submitted again.
- Complete may return 410 for expiry, 413 for excessive size, 422 for invalid/encrypted PDF, or 409 for upload state/object issues. Fix the cause rather than retrying unchanged.
- Task submission can return 402 for no bound plan, 403 for insufficient quota, 409 for upload state/concurrency conflicts, or 503 for unavailable dependencies/concurrency limits. Honor `Retry-After`; do not treat every 403 as an invalid key.
- Parsing is metered by pages: reserve using probed total pages, settle using successfully parsed pages. Current published contract says failure/cancellation/abnormal terminal states release reservations; report actual task/account evidence rather than claiming a settlement was checked. Prices and entitlements follow current account configuration.

Cross-account upload/task IDs can return 404; do not switch identities or manufacture IDs to bypass ownership checks. Platform identities and billing context are supplied server-side, not in client payloads.
