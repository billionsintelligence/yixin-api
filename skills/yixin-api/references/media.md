# Asynchronous Media Parsing

Contract: [production Media OpenAPI](https://openapi.billionsintelligence.com/docs/openapi/media.json).
Use the shared key and [error handling](apis.md#status-handling). This covers the public asynchronous download/upload workflow; media search, detail, and synchronous task endpoints are not published capabilities.

## Operations

All paths below are relative to `https://openapi.billionsintelligence.com/api` and require `X-API-KEY`, including polling and cancellation.

| Operation | Method and path |
| --- | --- |
| Submit YouTube/Bilibili download and parsing | `POST /v2/media/task` |
| Upload one local audio/video file and parse | `POST /v2/media/task/upload` |
| Query task | `GET /v2/media/task/{task_id}` |
| Cancel an unfinished task | `POST /v2/media/task/{task_id}/cancel` |

Billing is based on actual media duration (`media.video.duration_seconds`). Submission returns before processing and billing checks finish; do not infer the final charge from the HTTP status. Clients provide the API key; they do not supply internal billing identities or service tokens.

## Download A Video

Send `payload.source` as `youtube` or `bilibili`, `action` as `download`, and exactly one of `video_url` or `video_id`:

```json
{
  "payload": {
    "source": "youtube",
    "action": "download",
    "video_url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
  }
}
```

The sample illustrates the request format; use the user's selected video when executing. No preliminary detail request is needed. Send only the documented payload fields.

POSIX shell example with `YIXIN_API_KEY` configured:

```bash
curl --silent --show-error --fail-with-body --max-time 150 \
  'https://openapi.billionsintelligence.com/api/v2/media/task' \
  -H "X-API-KEY: ${YIXIN_API_KEY}" \
  -H 'Content-Type: application/json' \
  -d '{"payload":{"source":"youtube","action":"download","video_url":"https://www.youtube.com/watch?v=dQw4w9WgXcQ"}}'
```

## Upload A Local File

Upload exactly one multipart `file` using the user's selected file. Supported extensions:

- Audio: `aac`, `flac`, `m4a`, `mp3`, `ogg`, `opus`, `wma`, `wav`.
- Video: `avi`, `flv`, `mkv`, `mov`, `mp4`, `wmv`.

Default maximum is 2 GiB per file, subject to deployment limits. Chunked client uploads and resumable uploads are not supported. An extension alone does not establish that the file content is valid. Do not add MIME/content-type overrides intended to bypass format checks.

```bash
curl --silent --show-error --fail-with-body --max-time 1800 \
  'https://openapi.billionsintelligence.com/api/v2/media/task/upload' \
  -H "X-API-KEY: ${YIXIN_API_KEY}" \
  -F 'file=@/absolute/path/to/recording.mp3'
```

Let the client set the multipart `Content-Type` and boundary. On Windows, use the real path (including Chinese filenames) and `curl.exe` or a multipart-capable Python client; adapt shell quoting instead of copying POSIX continuations into PowerShell. Allow sufficient upload time for the file size and bandwidth.

## Track Completion

Submission/upload returns HTTP `202` with:

```json
{"code":0,"msg":"success","data":{"task_id":"<task-id>"}}
```

Validate the envelope and save `data.task_id` immediately. Query that same task with the same account/key:

```bash
curl --silent --show-error --fail-with-body --max-time 60 \
  "https://openapi.billionsintelligence.com/api/v2/media/task/${TASK_ID}" \
  -H "X-API-KEY: ${YIXIN_API_KEY}"
```

The response's `data` is a task, not the submission object:

| `data.status` | What to do |
| --- | --- |
| `pending` / `processing` | Keep the task ID and poll with a delay. |
| `done` | Read `data.result` and report the fields actually returned. |
| `failed` | Stop polling and report `data.error_code`, `data.error`, or `data.message`. |
| `canceled` | Stop polling and report cancellation. |

Task querying can return `code:0` / `msg:success` even when the task failed. `INSUFFICIENT_BALANCE` can appear asynchronously; waiting or creating duplicate tasks will not solve it. Result fields vary; do not promise transcription, a downloadable URL, or other outputs unless returned by this task.

Use a bounded polling loop: for example, start with a 3-second delay, back off to at most 15 seconds, and stop after a configurable 10-minute waiting window. Respect `Retry-After`. If still pending/processing at the deadline, report the task ID and last status so it can be resumed; a local waiting timeout does not mean the task failed and is not a reason to cancel it automatically. Report unknown states explicitly rather than polling forever.

Do not automatically retry creation/upload after a transport timeout: the server may already have accepted it, and another submission can create a second billable task. Resume a known task ID. If acceptance is unknown, explain that before resubmitting. Never send the platform API key to an external download URL returned in the result.

## Cancel

When cancellation is requested, call:

```bash
curl --silent --show-error --fail-with-body --max-time 60 -X POST \
  "https://openapi.billionsintelligence.com/api/v2/media/task/${TASK_ID}/cancel" \
  -H "X-API-KEY: ${YIXIN_API_KEY}"
```

Check `code`, `data.canceled`, and `data.task.status`. HTTP `200` alone does not prove cancellation; an already terminal task may not be canceled. Do not promise a refund or reservation release from this response alone.
