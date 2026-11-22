# Kelpgate Client for Node.js

[![npm](https://img.shields.io/npm/v/@kelpgate/client)](https://www.npmjs.com/package/@kelpgate/client)
[![CI](https://github.com/KelpGate/kelpgate-node/actions/workflows/ci.yml/badge.svg)](https://github.com/KelpGate/kelpgate-node/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

Official Node.js client for the [Kelpgate](https://kelpgate.cloud) malware-scanning API. It scans files, buffers, streams and URLs, manages OAuth 2.0 access tokens, retries transient failures idempotently and maps API responses to typed results and errors. The package includes Express middleware and a command-line interface, has no runtime dependencies, supports CommonJS and ES modules, and ships TypeScript definitions.

The package is open source under the MIT licence. The Kelpgate scanning service is a commercial hosted product; using it requires a Kelpgate account.

## Contents

- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Scan results](#scan-results)
- [Errors](#errors)
- [Retries and idempotency](#retries-and-idempotency)
- [Express middleware](#express-middleware)
- [Command-line interface](#command-line-interface)
- [Failure policy](#failure-policy)
- [Security](#security)
- [Troubleshooting](#troubleshooting)
- [Compatibility and versioning](#compatibility-and-versioning)
- [Support](#support)
- [Licence](#licence)

## Requirements

- Node.js 18 or later.
- A Kelpgate API client (client ID and secret) created in the [Kelpgate Console](https://console.kelpgate.cloud).

## Installation

```bash
npm install @kelpgate/client
```

## Configuration

```js
const { KelpgateClient } = require('@kelpgate/client');

const kelpgate = new KelpgateClient({
  apiUrl: 'https://api.kelpgate.cloud',
  tokenUrl: 'https://auth.kelpgate.cloud/oauth2/token',
  clientId: process.env.KELPGATE_CLIENT_ID,
  clientSecret: process.env.KELPGATE_CLIENT_SECRET,
});
```

| Option | Default | Description |
|---|---|---|
| `apiUrl` | required | Base URL of the Kelpgate API |
| `tokenUrl` | required with client credentials | OAuth 2.0 token endpoint |
| `clientId`, `clientSecret` | | Client credentials |
| `bearerToken` | | Static access token, used instead of client credentials |
| `scope` | `scan` | OAuth 2.0 scope requested |
| `timeoutMs` | `60000` | Per-request timeout |
| `maxRetries` | `2` | Retries for network errors and HTTP 429, 502, 503 and 504 |
| `retryDelayMs` | `500` | Base delay for exponential backoff |
| `maxBytes` | 128 MiB | Largest file the client sends; larger files fail before upload |
| `fetch` | global `fetch` | Alternative `fetch` implementation, for proxies or tests |

`apiUrl` and `tokenUrl` must use HTTPS; plain HTTP is accepted only for `localhost`.

`fromEnv()` creates a client from environment variables:

| Variable | Default |
|---|---|
| `KELPGATE_CLIENT_ID`, `KELPGATE_CLIENT_SECRET` | |
| `KELPGATE_TOKEN` | Static token, instead of client credentials |
| `KELPGATE_API_URL` | `https://api.kelpgate.cloud` |
| `KELPGATE_TOKEN_URL` | `https://auth.kelpgate.cloud/oauth2/token` |

```js
const { fromEnv } = require('@kelpgate/client');
const kelpgate = fromEnv();
```

## Usage

```js
const { assertClean } = require('@kelpgate/client');

const result = await kelpgate.scanFile('./upload.pdf');
if (result.infected) {
  // reject the upload
}

// Alternatively, throw InfectedError for an infected file:
assertClean(await kelpgate.scanBytes(buffer, { filename: 'report.docx' }));
```

| Method | Description |
|---|---|
| `scanFile(source, options?)` | Scan a path, `Buffer`, `Uint8Array`, `Blob` or readable stream |
| `scanBytes(bytes, options?)` | Scan in-memory content. `scanBuffer` is an alias. |
| `scanUrl(url, options?)` | Have the service fetch and scan a URL |
| `scanFileAsync(source, options?)`, `scanUrlAsync(url, options?)` | Submit a background scan; returns `{ scanId }`. Accepts `webhookUrl`. |
| `getAsyncResult(scanId, options?)` | Returns `{ pending: true }` until the result is available |
| `waitForResult(scanId, { intervalMs, timeoutMs })` | Poll until the result is available |
| `health()` | Service readiness; requires no credentials |

Upload methods accept `filename` and `idempotencyKey`. Streams are buffered in memory before upload; pass a path for large files.

## Scan results

An infected file is a result, not an exception.

| Field | Type | Description |
|---|---|---|
| `infected` | `boolean` | `true` for malware or a policy rejection |
| `status` | `string` | `CLEAN` or `INFECTED` |
| `signature` | `string` | Detection names, joined with ` \| ` |
| `policyRejected` | `boolean` | Rejected by policy (for example an encrypted archive) rather than detected |
| `degraded` | `boolean` | An engine did not complete; coverage was reduced |
| `sha256` | `string` | SHA-256 of the scanned content |
| `ruleAuthors` | `string[]` | Authors of matching detection rules. Preserve them when displaying or forwarding a detection, as required by the Detection Rule License. |
| `scanId`, `requestId` | `string` | Identifiers; quote `requestId` in support requests |
| `filename`, `timestamp`, `s3Path`, `downloadUrl` | `string` | As returned by the service |
| `raw` | `object` | Response body as received |

Integrations that require full coverage should treat `degraded: true` as not verified.

## Errors

All errors extend `KelpgateError` and expose `status`, `code`, `requestId` and `body`.

| Error | Condition |
|---|---|
| `ConfigError` | Invalid options or idempotency key; thrown before any request |
| `AuthError` | Credentials rejected (401), or organization or client not permitted (403) |
| `QuotaExceededError` | Monthly quota exhausted (402). Exposes `quota`, `usage` and `upgradeUrl`. |
| `PayloadTooLargeError` | File over the plan or scanner limit (413), or over `maxBytes` |
| `UnsupportedFileError` | File type not permitted, or archive rejected by policy (415) |
| `RateLimitedError` | Concurrency limit reached (429). Exposes `retryAfterSeconds`. |
| `ScanError` | Invalid request (400), or content that could not be parsed (412) |
| `ServerError` | HTTP 5xx after retries |
| `NetworkError`, `TimeoutError` | The service could not be reached or did not respond in time |
| `InfectedError` | Thrown by `assertClean`; exposes `result` and `signature` |
| `KelpgateError` | Any other status, including 404 and 422 |

## Retries and idempotency

Every scan request carries an `Idempotency-Key`. The client generates one per call and reuses it for its own retries, so a retried scan is not counted twice against the quota. A rejected access token is discarded and the request retried once with a new token.

Supply a key to make retries across process restarts or job re-runs idempotent:

```js
await kelpgate.scanFile(path, { idempotencyKey: `upload-${job.id}` });
```

Keys are 1 to 128 characters from `A-Z a-z 0-9 . _ : -`. The service binds a key to the request content; reusing a key for a different file is counted as a new scan.

## Express middleware

```js
const multer = require('multer');
const { kelpgateScan } = require('@kelpgate/client/express');

const upload = multer({ dest: 'uploads/' });

app.post('/upload', upload.single('document'), kelpgateScan({ client: kelpgate }), (req, res) => {
  res.json({ scan: req.kelpgate[0] });
});
```

| Option | Default | Behaviour |
|---|---|---|
| `client` | required | A `KelpgateClient` |
| `onInfected` | `reject` | `reject` responds `422` and deletes the uploaded files; `pass` continues with results in `req.kelpgate` |
| `onError` | `block` | `block` responds `402`, `413`, `415` or `503` according to the error and deletes the uploaded files; `allow` continues with the error in `req.kelpgateError` |
| `deleteInfected` | `true` | Delete infected files from disk, including when `onInfected` is `pass` |

## Command-line interface

```bash
export KELPGATE_CLIENT_ID=... KELPGATE_CLIENT_SECRET=...
npx kelpgate scan ./archive.zip ./report.pdf
npx kelpgate scan-url https://example.com/file.exe
npx kelpgate health
```

| Exit code | Meaning |
|---|---|
| `0` | All files clean |
| `1` | At least one file infected |
| `2` | Error |

`--json` prints machine-readable output.

## Failure policy

When the service cannot return a verdict, the client throws. The middleware converts the error into a response according to `onError`:

| Condition | Client | `onError: 'block'` | `onError: 'allow'` |
|---|---|---|---|
| Service unreachable, timed out, or 5xx after retries | `NetworkError`, `TimeoutError`, `ServerError` | `503` | Continue |
| Authentication failed | `AuthError` | `503` | Continue |
| Quota exhausted | `QuotaExceededError` | `402` | Continue |
| File too large | `PayloadTooLargeError` | `413` | Continue |
| File type not permitted | `UnsupportedFileError` | `415` | Continue |

Use `block` where preventing malware takes priority over availability, and `allow` where uploads must continue during a service outage.

## Security

| Aspect | Behaviour |
|---|---|
| Data sent | The file content and its name, over HTTPS, to the configured `apiUrl` only |
| Transport | HTTPS required except for `localhost`; redirects are never followed |
| Credentials | Never included in error messages or serialised output (`toJSON` omits them) |
| Tokens | Held in memory only and renewed before expiry |
| Local storage | The package writes nothing to disk; the middleware deletes rejected uploads |
| Service retention | Clean files are retained briefly and infected files quarantined, according to the Kelpgate plan |

Report vulnerabilities according to the [security policy](https://github.com/KelpGate/.github/blob/main/SECURITY.md).

## Troubleshooting

Verify an installation with the [EICAR test file](https://www.eicar.org/download-anti-malware-testfile/), which every scanner reports as infected:

```bash
npx kelpgate scan ./eicar.com    # INFECTED, exit code 1
```

| Symptom | Cause |
|---|---|
| `ConfigError: ... must use https` | `apiUrl` or `tokenUrl` is not HTTPS |
| `AuthError` with code `TOKEN_REQUEST_FAILED` | Invalid client ID or secret, or the client lacks the `scan` scope |
| `AuthError` with status 403 | Organization suspended, or client not linked to an organization |
| `RateLimitedError` | Too many concurrent scans; the client honours `Retry-After`. Reduce concurrency. |
| `TimeoutError` on large files | Increase `timeoutMs`, or use `scanFileAsync` with `waitForResult` |
| `degraded: true` | An engine did not complete for that scan |

## Compatibility and versioning

| Component | Supported |
|---|---|
| Node.js | 18, 20, 22, 24 (tested in CI) |
| Module systems | CommonJS, ES modules |
| TypeScript | Definitions included and type-checked in CI |
| Kelpgate API | Version 1 (`/api/v1`) |

The package follows [Semantic Versioning](https://semver.org). Fixes, including security fixes, are released for the latest version only. Changes are listed in the [changelog](CHANGELOG.md). The organisation-wide policy is in [versioning and releases](https://github.com/KelpGate/.github/blob/main/docs/versioning-and-releases.md).

## Support

See [getting help](https://github.com/KelpGate/.github/blob/main/SUPPORT.md) or contact support@kelpgate.cloud. Contributions follow the [contributing guide](https://github.com/KelpGate/.github/blob/main/CONTRIBUTING.md).

## Licence

MIT. See [LICENSE](LICENSE). The Kelpgate scanning service is subject to its own terms.
