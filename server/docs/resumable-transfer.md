# Resumable upload and download

Opt-in byte transfer for large report exports and file imports. A prepare call returns a **transfer ticket**. The file then moves in chunks, and the same ticket can be continued after a dropped connection.

Both directions share one in-memory registry: `TransferTicketService` in the `export` module. Tickets are not written to the database and are not visible to other JVMs.

Paths below are servlet paths. The WAR is deployed at `/hi-ee`, so a browser calls `/hi-ee/fileDownload`.

## When it runs

| Setting (`hi-repository/System/Admin/setting.xml`) | Default | Effect |
|------------------------------------------------------|---------|--------|
| `resumableDownload` | `false` | Client may send `resumable=true` on report and resource export |
| `resumableUpload` | `false` | Client may send `resumable=true` on file import |
| `streamResponse` | `false` | `GET /downloadEvents` is served. When `false`, that route returns **404** and clients poll instead |
| `downloadChunkSize` | `1048576` | Client `Range` size for downloads (bytes) |
| `uploadChunkSize` | `1048576` | Server chunk size for uploads (bytes). The prepare response is authoritative |
| `eventBatchSize` | `100` | How many buffered progress events are replayed per SSE `batch` |

`EFWController` copies these into `applicationSettingsData` (`resumableDownload`, `resumableUpload`, `downloadChunkSize`, `uploadChunkSize`, `streamResponse`). Omitted tags use the defaults in `JsonUtils`.

The React client turns a direction on only when the matching flag is set:

- Download: `client/src/base/service.js` (`postDownloadRequest`, `postPrintDownloadRequest`, `postExportDownloadRequest`)
- Upload: `uploadFilePostRequest` in the same file, and only when the `FormData` contains a `File` or `Blob`

Scheduler and other legacy callers that omit `resumable=true` stay on the synchronous `/downloadReport` path. Do not add `resumable=true` to those jobs.

## Ticket

| Field | Meaning |
|-------|---------|
| `status` | `PREPARING`, `READY`, `ABORTED`, `EXPIRED` |
| `ownerUserId` | Logged-in user who created the ticket. Other users get **403** |
| `requestId` | Required. Also the key for `/cancelRequest` and for upload resume |
| `expiresAt` | 60 minutes after create (`ResumableDownloadService` and `ResumableUploadService` both pass `60`) |

`System/Admin/SystemSchedule/systemschedule.json` job `transferTicketPurge` (`TransferTicketPurgeJob`, cron `0 0/5 * * * ?`) deletes expired tickets and their temp files every five minutes. A JVM restart drops the registry immediately; in-flight clients then see **410**.

`POST` or `GET /cancelRequest?requestId=` cancels the executor task and calls `TransferTicketService.abortByRequestId`. Abort deletes tracked temp files and emits SSE `aborted`.

## Download

```text
POST /downloadReport or /exportResource  (resumable=true, requestId)
        │  200 JSON  status=PREPARING
        ▼
wait until READY
        │  streamResponse=true  →  GET /downloadEvents
        │  streamResponse=false →  HEAD /fileDownload every 2s (up to 450 tries)
        ▼
GET /fileDownload?ticket=…   Range: bytes=start-end
        │  repeat until the file is assembled
        ▼
compare SHA-256 to X-Checksum-SHA256
```

Prepare entry points:

| Route | Producer | Ticket id material |
|-------|----------|--------------------|
| `POST /downloadReport` | `ReportDownloadProducer` when `data` is absent; `DownloadCacheProducer` when `data` is present | user, report location/format/parameters (or cache report id), `requestId` |
| `POST /exportResource` | `ResourceExportDownloadProducer` | user, `exportResource`, resource path, `requestId` |

Missing `requestId` is **400**. Prepare JSON:

```json
{
  "ticket": "<id>",
  "requestId": "<id>",
  "downloadUrl": "/fileDownload?ticket=<id>",
  "eventsUrl": "/downloadEvents?ticket=<id>&requestId=<id>",
  "status": "PREPARING"
}
```

`/fileDownload` (`DownloadController`):

| Method | Status | Meaning |
|--------|--------|---------|
| `HEAD` or `GET` | **202** | Still `PREPARING`. `Cache-Control: no-store` |
| `HEAD` | **200** | `READY`. `Accept-Ranges: bytes`, `Content-Length`, `Content-Disposition`, `X-Checksum-SHA256` |
| `GET` with `Range` | **206** | Partial body. `Content-Range: bytes start-end/total` |
| `GET` without `Range` | **200** | Entire file |
| `GET` bad `Range` | **416** | `Content-Range: bytes */<length>` |
| any | **403** | Ticket belongs to another user |
| any | **410** | Unknown, expired, or aborted ticket |

`HttpRangeFileStreamer` accepts only `bytes=start-end` and `bytes=start-`. The client (`resumable-download.js`) retries the same offset after network errors (**502**, **503**, **504**, or no response) with backoff capped at 30 seconds. Pause and resume are local; they do not call the server.

When `streamResponse` is true, the export thread waits up to **60 seconds** for a `/downloadEvents` subscriber before it builds the file. If nobody connects, the ticket is aborted with `Timed out waiting for event subscriber`. If the ticket is already aborted, prepare stops and deletes tracked temp files. Open the event stream as soon as prepare returns. When `streamResponse` is false, prepare does not wait for a subscriber.

## Upload

```text
POST /importFile or /importResource   (form fields, no file bytes, resumable=true)
        │  JSON session
        ▼
POST /uploadChunk?ticket=&index=      raw chunk body, repeated
        │  last new chunk starts assemble + import
        ▼
wait until READY
        │  streamResponse=true  →  SSE ready.importResult
        │  streamResponse=false →  re-POST the same prepare URL until importResult
        ▼
existing import handler result
```

Prepare fields (application/x-www-form-urlencoded):

| Field | Required | Notes |
|-------|----------|-------|
| `resumable` | yes | `true` |
| `requestId` | yes | Same id resumes a `PREPARING` or `READY` upload for this user |
| `fileName` | yes | |
| `fileSize` | yes | Byte length. `0` is one empty chunk |
| `checksumSha256` | no | Lower/upper case hex. Blank skips verify; a mismatch aborts after assembly |
| `contentType` | no | |
| `formData` | for `/importResource` | Same JSON the synchronous import expects |
| `destination`, `type`, `config`, `fileExtension` | for `/importFile` | Copied onto the ticket and applied after assembly |

Session JSON from `uploadSessionPayload`:

```json
{
  "ticket": "<id>",
  "requestId": "<id>",
  "status": "PREPARING",
  "chunkSize": 1048576,
  "totalChunks": 3,
  "receivedChunks": [0],
  "fileName": "pack.zip",
  "fileSize": 2500000,
  "eventsUrl": "/downloadEvents?ticket=<id>&requestId=<id>"
}
```

`eventsUrl` is included only when `streamResponse` is true. `importResult` appears once status is `READY`.

`POST /uploadChunk` body is the raw chunk (`Content-Type: application/octet-stream`). `Content-Length` must equal the expected slice: full `chunkSize`, or the remainder on the last chunk. A short or long body is **400** (`Incomplete chunk` / `Unexpected chunk length` / `Chunk larger than expected`). An index outside `0 .. totalChunks-1` is **400**. Sending an index that is already in `receivedChunks` is success and does not rewrite the file.

Response:

```json
{
  "ticket": "<id>",
  "index": 1,
  "received": true,
  "receivedChunks": [0, 1],
  "complete": false,
  "eventsUrl": "/downloadEvents?ticket=<id>&requestId=<id>"
}
```

When `complete` is true, `ResumableUploadService` assembles `upload-<ticketId>/` under the temp directory, checks SHA-256, then calls the normal import for `/importFile` or `/importResource`. Unsupported `targetUrl` values abort the ticket. With `streamResponse` true, finalize also waits up to 60 seconds for an event subscriber; the client opens `/downloadEvents` before the first chunk so that wait is already satisfied.

Chunks live only for the ticket TTL. Re-POST prepare with the **same** `requestId` to read `receivedChunks` and skip those indexes. A new `requestId` starts a new ticket.

## Events (`GET /downloadEvents`)

Requires `streamResponse=true`. Query: `ticket`, optional `requestId`. Response headers include `Cache-Control: no-cache` and `X-Accel-Buffering: no`.

| SSE event | When | `data` |
|-----------|------|--------|
| `ready` | Download file is on disk, or upload import finished | `status=READY`, `fileName`, `contentType`, `contentLength`, `checksum` (SHA-256). Downloads also include `downloadUrl`. Uploads include `importResult` |
| `aborted` | Failure, cancel, expiry, or unknown ticket | `status=ABORTED`, `message` |
| `progress` | Each accepted upload chunk | `status=PREPARING`, `receivedChunks`, `totalChunks` |
| `batch` | Buffered `ProgressEvent`s (`log`, `info`, `warning`, `error`) | array of `{channel, path, message, ts}` |

An unknown ticket still returns an SSE body with a single `aborted` event (it does not use HTTP 404). A `batch` item with `channel=error` is a progress message; the terminal failure is the later `aborted` event. `wait-ticket-events.js` keeps that progress message and surfaces it when `aborted` arrives.

`/importResource` with `upload=true`, `stream=true`, and `streamResponse` enabled is a different path: one multipart body, no `/uploadChunk`. It opens an import-stream ticket and emits the same `/downloadEvents` progress. Chunked resumable upload is selected only by `resumable=true`.

## Client map

| Module | Role |
|--------|------|
| `client/src/base/resumable-download.js` | Prepare, wait, `Range` loop, checksum, save |
| `client/src/base/resumable-upload.js` | Hash, prepare, chunk loop, wait for `importResult` |
| `client/src/base/resumable-transfer.js` | Pause, resume, online wait, retry classification |
| `client/src/base/wait-ticket-events.js` | Shared SSE wait used by upload, download, and the import console |
| `client/src/base/service.js` | Gates the flows on application settings |

Byte downloads omit `X-Requested-With`. `ResponseMetadataFilter` already skips `/fileDownload`, `/downloadEvents`, and resumable prepare on `/downloadReport` and `/exportResource`, so those responses are not wrapped or given a replaced `Content-Length`.

## Pitfalls

- **Flags default off.** A client that always sends `resumable=true` against a server with `resumableDownload=false` still enters the ticket path, because the server keys off the request parameter. Keep the setting and the client in agreement.
- **`streamResponse` and the 60s subscriber wait.** Enable streaming only if the client opens `/downloadEvents` immediately. Otherwise prepare (download) or finalize (upload) aborts.
- **One JVM.** A load balancer that sends prepare and `/fileDownload` to different nodes returns **410**. Sticky sessions do not survive a restart.
- **Upload chunk length is exact.** A proxy that changes `Content-Length` produces **400**. The browser retries HTTP 400 for upload chunks; it does not retry HTTP 400 for downloads.
- **Checksum is the whole file.** A mismatch after download throws `Download checksum mismatch` and does not save. Upload mismatch deletes the assembled file and aborts.
- **Pause is not a server hold.** The ticket TTL keeps running while the UI is paused.
- **Empty download.** `contentLength` 0 saves an empty file and does not issue `Range` requests.
- **Filter and prepare URLs.** Resumable prepare is excluded from response wrapping only for `/downloadReport` and `/exportResource`. `/importFile` and `/importResource` prepare JSON can still pass through `ResponseMetadataFilter` when the request is marked AJAX.
