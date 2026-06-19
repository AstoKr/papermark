# Finding: Non-RFC-5987 Content-Disposition on S3 Presigned URL

## Severity
LOW

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:N = 0.0 (UX defect, not a security vulnerability).

## Affected Code
- File: `pages/api/file/s3/get-presigned-get-url.ts`
- Line: 78 (`ResponseContentDisposition: \`attachment; filename="${encodeURIComponent(s3Key.key.split("/").pop() || "download.zip")}"\``)
- Function: `generateFreshPresignedUrl`
- Endpoint: `GET /api/file/s3/get-presigned-get-url` (and the sibling `/api/file/s3/get-presigned-get-url-proxy` at `lib/files/bulk-download-presign.ts:78`)

## Description
The recommended pattern, used by `lib/utils.ts:387-392` (`buildContentDisposition`), is RFC 5987: ASCII fallback in `filename="..."`, percent-encoded UTF-8 in `filename*=UTF-8''...`. Here the same helper is NOT reused — instead `encodeURIComponent` percent-encodes the input and shoves it into the ASCII `filename` slot. If the slugified S3 key segment contains a `-` (which `safeSlugify` produces freely), it gets through unchanged; if it contains a space (which `safeSlugify` also produces, see `lib/utils.ts`), it becomes `%20`. In practice `safeSlugify` replaces spaces with `-`, so the visible defect is usually minor — but the function does accept non-ASCII bytes if the S3 key is malformed.

The sibling helper at `lib/files/bulk-download-presign.ts:78` has the same pattern, so the bug is duplicated.

## Proof of Concept

### Pre-conditions
- An S3 key whose last path segment contains non-ASCII characters or a space.

### Step-by-step demonstration
1. A user downloads a file with a name like `Résumé.pdf` (which `safeSlugify` may or may not handle correctly).
2. The S3 key is stored as `teamId/docId/Resume.pdf` (or similar).
3. The presigned URL is generated with `ResponseContentDisposition: attachment; filename="Resume.pdf"`.
4. The browser displays the filename as `Resume.pdf` — works correctly because `safeSlugify` produced an ASCII-safe name.
5. **If `safeSlugify` is bypassed** (e.g. by direct S3 upload, or by a future code path that allows non-ASCII names), the S3 key may contain `Rés` (with the é character).
6. The presigned URL is generated with `ResponseContentDisposition: attachment; filename="R%C3%A9s.pdf"`.
7. The browser displays the filename as `R%C3%A9s.pdf` literally — `encodeURIComponent` percent-encoded the input, but the percent-encoding is in the wrong context (RFC 6266 says `filename="..."` should be ASCII, and non-ASCII should go into `filename*=UTF-8''...`).

## Impact
- **UX defect** — the downloaded file has a `R%C3%A9s.pdf` filename instead of `Résumé.pdf`.
- **Not a security vulnerability** — no bypass, no privilege escalation, no data disclosure.

## Evidence
```ts
// pages/api/file/s3/get-presigned-get-url.ts:75-80
const command = new GetObjectCommand({
  Bucket: s3Key.bucket,
  Key: s3Key.key,
  ResponseContentDisposition: `attachment; filename="${encodeURIComponent(s3Key.key.split("/").pop() || "download.zip")}"`,
  ResponseCacheControl: "no-cache, no-store, must-revalidate",
});
```

## Methodology
- **M2 — Encoding analysis** (`methodology-raw/01-encoding-analysis.md` Finding 4)

## Chain Potential
- None directly. The finding is a UX defect.

## Remediation
Replace this line with `buildAttachmentDispositionForName(s3Key.key.split("/").pop() || "download.zip")` from `lib/utils.ts:403-410`. That helper produces the RFC 5987 `filename*=UTF-8''...` form, which S3 honors correctly:
```ts
import { buildAttachmentDispositionForName } from "@/lib/utils";

const filename = s3Key.key.split("/").pop() || "download.zip";
const command = new GetObjectCommand({
  Bucket: s3Key.bucket,
  Key: s3Key.key,
  ResponseContentDisposition: buildAttachmentDispositionForName(filename),
  ResponseCacheControl: "no-cache, no-store, must-revalidate",
});
```

Apply the same fix to `lib/files/bulk-download-presign.ts:78`.
