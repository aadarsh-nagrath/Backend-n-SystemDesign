# Object Storage, File Uploads, and Media Handling

> Nearly every backend eventually handles files: avatars, documents, invoices, exports, videos, and backups. This note covers object storage (S3 and similar), upload and download patterns, security, processing pipelines, and serving files through CDNs.

## Table of Contents
1. [Storage Types: Block vs File vs Object](#types)
2. [Object Storage Model (S3 Semantics)](#s3)
3. [Consistency, Durability, Availability](#consistency)
4. [Storage Classes and Lifecycle Policies](#classes)
5. [Upload Patterns: Through the Server vs Direct-to-Storage](#uploads)
6. [Presigned URLs and POST Policies](#presigned)
7. [Multipart and Resumable Uploads](#multipart)
8. [Download Patterns: Presigned GET, CDN, Signed Cookies](#downloads)
9. [Security: Validation, Malware, Access Control, Encryption](#security)
10. [Processing Pipelines: Thumbnails, Transcoding, Virus Scanning](#processing)
11. [Metadata in the Database](#metadata)
12. [Performance and Cost](#perf)
13. [Video Streaming Basics (HLS/DASH)](#video)
14. [S3-Compatible Alternatives](#alternatives)
15. [Interview Questions](#qa)

---

## 1. Storage Types {#types}

| | Block storage | File storage | Object storage |
|---|---|---|---|
| Examples | EBS, Persistent Disk, local NVMe | EFS, Filestore, NFS, SMB | **S3**, GCS, Azure Blob, R2, MinIO |
| Interface | Raw blocks; formatted with a filesystem, attached to one VM (mostly) | POSIX filesystem shared by many clients | HTTP API: PUT/GET/DELETE whole objects by key |
| Mutability | In-place byte updates | In-place | **Immutable objects** (replace whole object; no append/partial update) |
| Latency | Sub-ms | ms | ~10–100 ms first byte |
| Scale | TBs per volume | TBs–PBs | Virtually unlimited (exabytes) |
| Cost | $$ | $$$ | $ |
| Use | Databases, OS disks | Shared app data, legacy apps, ML training data | Media, backups, data lakes, static assets, logs, exports |

**Rule for stateless services: never store user files on the app server's local disk.** Instances are ephemeral and horizontally scaled, so use object storage.

---

## 2. Object Storage Model {#s3}

- **Bucket** (globally unique name in S3) → **objects** identified by **key** (`users/42/avatar/2025-06-01.webp`). Keys are flat strings, and "folders" are just a prefix plus delimiter convention (`ListObjectsV2` with `Prefix` and `Delimiter=/`).
- An object = data (up to **5 TB**, single PUT up to 5 GB) + **metadata** (system: Content-Type, Content-Length, ETag, Last-Modified; user-defined `x-amz-meta-*`) + optional **tags** (for lifecycle and IAM conditions).
- **Versioning** (per bucket): keeps all versions, and deletes create delete markers. It protects against accidental overwrite or delete (pair it with lifecycle rules to expire old versions).
- **Object Lock** (WORM: write once, read many): governance or compliance retention modes. Use it for immutable backups (ransomware protection) and regulatory retention.
- **Conditional writes** (`If-None-Match: *` create-only, and `If-Match` ETag compare-and-swap, added to S3 in 2024) enable safe coordination patterns directly on S3.
- Events: S3 Event Notifications / EventBridge on object create or delete trigger processing pipelines (Lambda, SQS).
- Replication: Cross-Region Replication (CRR) and Same-Region Replication for DR or compliance.

---

## 3. Consistency, Durability, Availability {#consistency}

- **S3 provides strong read-after-write consistency** for PUTs and DELETEs of objects, plus list operations (since December 2020). Before that, overwrite and delete were eventually consistent, which is why old blog posts warn about it.
- **Durability**: designed for **99.999999999% (11 nines)** by storing data redundantly across ≥3 AZs (Standard class). Losing data due to S3 itself is extraordinarily unlikely. **Losing data due to your own bugs or deletes is likely**, so use versioning, Object Lock, and backups to a separate account.
- **Availability**: 99.99% design for Standard (SLA 99.9%). One Zone-IA lives in a single AZ.
- GCS and Azure Blob have similar strong consistency guarantees.

---

## 4. Storage Classes and Lifecycle {#classes}

| S3 class | Access pattern | Notes |
|---|---|---|
| Standard | Frequent | Default |
| Intelligent-Tiering | Unknown/changing | Auto-moves between tiers; small monitoring fee per object |
| Standard-IA / One Zone-IA | Infrequent (≥ 30 days) | Cheaper storage, retrieval fee, 128 KB minimum billable size |
| Glacier Instant Retrieval | Rare, ms access | Archive with instant access |
| Glacier Flexible Retrieval | Rare; minutes–hours restore | Backups |
| Glacier Deep Archive | Very rare; 12–48 h restore | Cheapest; compliance archives |
| Express One Zone | Ultra-low latency, single AZ, directory buckets | ML training, high-performance analytics |

**Lifecycle rules**: transition objects by age (Standard → IA after 30 days → Glacier after 180), expire temporary objects (exports after 7 days, incomplete multipart uploads after 1 day: **always add this rule**, because abandoned multipart parts are billed invisibly), and expire noncurrent versions.

---

## 5. Upload Patterns {#uploads}

### a) Through the application server (proxy upload)
```
Client ──(multipart/form-data 50 MB)──► API server ──► S3
```
- Pros: simple, the server sees everything (validate, process, authorize in one place).
- Cons: the server's bandwidth, memory, and connection time are tied up. Large uploads hit request timeouts and body size limits (Nginx `client_max_body_size`, API Gateway 10 MB, Lambda 6 MB payload). It doesn't scale for big files.
- If you must, **stream** to S3 (don't buffer whole files in memory or on disk).

### b) Direct-to-storage with presigned URLs (**recommended**)
```
1. Client → API: "I want to upload avatar.png, image/png, 2.3 MB"
2. API: authorize; validate declared type/size; create a DB record (status=pending, key);
        generate presigned PUT URL (or POST policy) for that exact key, short expiry (5–15 min)
3. Client → S3: PUT file directly using the presigned URL
4. S3 → event (ObjectCreated) → processing worker (validate actual content, scan, thumbnail) → DB status=ready
   (or client calls API "upload complete", and the API verifies via HeadObject)
5. Client/API uses the file once status=ready
```
- Pros: the app server never touches bytes, so uploads scale independently, there's no proxy timeout, and costs are lower.
- Cons: more moving parts (pending states, orphaned uploads → cleanup via lifecycle or a reconciliation job). Validation must happen **after** upload, because the client could lie about type and size.

---

## 6. Presigned URLs and POST Policies {#presigned}

A **presigned URL** embeds a signature (SigV4) granting temporary permission for a specific operation on a specific key, using the signer's credentials:
```python
url = s3.generate_presigned_url(
    "put_object",
    Params={"Bucket": "uploads", "Key": f"users/{user_id}/avatars/{uuid7()}.png", "ContentType": "image/png"},
    ExpiresIn=600,
)
```
- **Generate the key server-side** (never let the client choose arbitrary keys: overwrites, path tricks).
- **Short expiry**.
- A presigned PUT can't enforce maximum size on its own. Use a **presigned POST** with a **policy** that enforces `content-length-range`, an exact key or prefix, and Content-Type:
  ```python
  post = s3.generate_presigned_post(
      Bucket="uploads", Key=key,
      Fields={"Content-Type": "image/png"},
      Conditions=[["content-length-range", 1, 5_000_000], {"Content-Type": "image/png"}],
      ExpiresIn=600)
  # client does multipart/form-data POST to post["url"] with post["fields"] + file
  ```
- CORS on the bucket must allow the browser origin and the needed methods and headers.
- GCS: signed URLs/policies. Azure: SAS tokens (scope and expiry). R2 and MinIO support S3 presigning.

---

## 7. Multipart and Resumable Uploads {#multipart}

For large files (S3 recommends multipart above ~100 MB, and it's required above 5 GB):
1. `CreateMultipartUpload` → UploadId.
2. Upload parts (5 MB–5 GB each, up to 10,000 parts) **in parallel**, each with its own presigned URL. Retry individual failed parts.
3. `CompleteMultipartUpload` with the list of part ETags. S3 assembles the object atomically.
4. `AbortMultipartUpload` on failure. A lifecycle rule cleans up abandoned ones.

Benefits: parallel throughput, resume after network failure (re-upload only the missing parts), and uploading before the total size is known (streaming).

Client libraries: AWS SDK managed uploaders (`Upload` in JS v3, `TransferManager`, `upload_fileobj`), **Uppy** (browser, with an S3 multipart plugin), **tus** (open resumable upload protocol with tusd servers), and the GCS resumable upload API.

**S3 Transfer Acceleration** routes uploads through CloudFront edge locations for faster long-distance uploads.

---

## 8. Download Patterns {#downloads}

| Pattern | Use |
|---|---|
| **Public bucket/objects** | Rarely appropriate. Keep **Block Public Access** on, and use a CDN with origin access control instead |
| **CDN in front of a private bucket** (CloudFront + Origin Access Control, Cloudflare + R2) | Public-ish static assets, product images: cached at the edge, the bucket isn't directly reachable |
| **Presigned GET URLs** (short-lived) | Private user files (invoices, documents). The API authorizes and then redirects (302) or returns the URL |
| **CDN signed URLs / signed cookies** | Private content at scale through the CDN (paid videos, course materials). Signed cookies cover many files (HLS segments) |
| **Proxy through the app** | Only when you must transform or authorize per byte (watermarking). Otherwise it's wasteful |

Headers to set on objects: `Content-Type` (correct MIME), `Content-Disposition` (`attachment; filename="invoice-42.pdf"` to force download, which also matters for security, below), `Cache-Control` (long `max-age` + **immutable** for content-addressed keys like `app.3f9a2c.js`; short for mutable keys), `Content-Encoding` for pre-compressed assets.

**Cache busting**: use new keys for new content (`avatar-<hash>.webp`) rather than overwriting the same key and fighting CDN caches (invalidations are slow and cost money).

---

## 9. Security {#security}

1. **Authorization**: check that the user may upload to or read this resource **before** issuing presigned URLs. Keys should be unguessable (UUIDs), but obscurity isn't authorization.
2. **Validate actual content** after upload: check magic bytes / file signature (`libmagic`, `file-type`), not the client-provided extension or Content-Type. Enforce size limits. Re-encode images (decode + re-encode strips malicious payloads, polyglots, and EXIF metadata, including **GPS location**, a privacy leak).
3. **Malware scanning** for user-uploaded documents shared with others: ClamAV (cdr), cloud services (Amazon GuardDuty Malware Protection for S3, Microsoft Defender for Storage), or third-party APIs. Quarantine until clean (a separate bucket or prefix).
4. **Serve user content from a separate domain** (`usercontent.example-cdn.com`, like `githubusercontent.com` or `googleusercontent.com`). If an attacker uploads HTML or SVG with scripts and it's served from your main domain, that's **stored XSS** with access to your cookies. Also set `Content-Disposition: attachment` for untrusted types, `X-Content-Type-Options: nosniff`, and a restrictive CSP on the user-content domain. Don't serve SVG inline unless it's sanitized.
5. **Filename handling**: never use user-provided filenames as storage keys or local paths (path traversal `../../etc/passwd`, overwriting). Store the original filename as metadata only, sanitized for `Content-Disposition` (RFC 6266/5987 encoding).
6. **Decompression bombs / image bombs**: a tiny file that expands to gigabytes (zip bombs, a 100,000×100,000 pixel PNG). Limit decompressed size and pixel dimensions before processing (Pillow's `MAX_IMAGE_PIXELS`, libvips limits).
7. **Processing vulnerabilities**: ImageMagick (ImageTragick, 2016), Ghostscript, and FFmpeg have had RCE bugs. Run processing in **sandboxed, isolated workers** (separate service or containers with no network, least privilege, seccomp, or Lambda), keep libraries patched, and restrict ImageMagick policies to allowed formats.
8. **Bucket hardening**: Block Public Access, bucket policies with least privilege, deny non-TLS (`aws:SecureTransport`), versioning + MFA delete or Object Lock for critical data, access logging / CloudTrail data events, and no wildcard principals. Misconfigured public buckets are a top cause of data breaches.
9. **Encryption**: SSE-S3 (default since 2023), SSE-KMS (customer-managed keys, auditable key usage, per-tenant keys possible), or client-side encryption for end-to-end needs. TLS in transit.
10. **Data retention and deletion**: implement user deletion (GDPR/DPDP) across all buckets, versions, replicas, and CDN caches.

---

## 10. Processing Pipelines {#processing}

```
S3 upload (raw/ prefix) ──ObjectCreated──► SQS ──► worker(s)
   worker: validate type/size → malware scan → strip metadata
         → generate variants (thumb 128px, medium 512px, WebP/AVIF) → write to processed/ prefix
         → update DB (status=ready, variant keys, width/height, checksum) → emit FileReady event
   failures → retries → DLQ
```
- Decouple with a queue (not direct Lambda-on-every-event for heavy work, if you need throttling and retries).
- Idempotent processing (the same event may arrive twice; deterministic output keys).
- **On-the-fly image transformation** services as an alternative to pre-generating variants: imgproxy, Thumbor, Cloudinary, Imgix, Cloudflare Images, AWS Serverless Image Handler. They resize on request, cache at the CDN, and need signed URLs to prevent abuse (an attacker requesting infinite sizes).
- Modern formats: **WebP/AVIF** for images (30–50% smaller than JPEG), responsive `srcset`.
- Documents: PDF generation (headless Chrome/Playwright, wkhtmltopdf legacy, WeasyPrint, Gotenberg), OCR (Textract, Tesseract), previews.
- Integrity: store checksums (SHA-256, or S3 additional checksums CRC32C/SHA256) to verify and deduplicate (content-addressed storage: key = hash means automatic dedup).

---

## 11. Metadata in the Database {#metadata}

Keep a `files` (or `attachments`) table as the source of truth for **ownership, state, and references**, with object storage holding only bytes:
```sql
CREATE TABLE files (
  id            UUID PRIMARY KEY,
  owner_id      BIGINT NOT NULL REFERENCES users(id),
  tenant_id     BIGINT NOT NULL,
  bucket        TEXT NOT NULL,
  storage_key   TEXT NOT NULL UNIQUE,
  original_name TEXT,
  content_type  TEXT,
  size_bytes    BIGINT,
  sha256        BYTEA,
  status        TEXT NOT NULL CHECK (status IN ('pending','scanning','ready','rejected','deleted')),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```
- Orphans: uploads never completed (pending older than 1 day → delete object + row) and objects without rows (periodic reconciliation using S3 Inventory reports).
- Deleting: mark deleted in the DB → async delete of the object and variants (and CDN purge if needed).
- Don't store binary blobs in the database (see [`databases/fundamentals/04-storage-engine-internals.md`](../databases/fundamentals/04-storage-engine-internals.md#toast)).

---

## 12. Performance and Cost {#perf}

- S3 request rate scales per **prefix partition**: at least 3,500 writes and 5,500 reads per second per prefix, auto-scaling with load. The old advice to randomize key prefixes is mostly obsolete, but extremely hot workloads still benefit from spreading keys.
- **Parallelism** for throughput: multipart uploads and **byte-range GETs** in parallel.
- Use a **CDN** for read-heavy public content: lower latency and much lower egress cost.
- **Costs**: storage (GB-month by class), **requests** (PUT/LIST are pricier than GET; millions of tiny objects are expensive, so batch small records into larger files for analytics), **data transfer out** (egress to the internet is often the biggest cost; Cloudflare R2 has zero egress fees, which is its main selling point), retrieval fees for IA/Glacier, and minimum storage durations.
- Small-object problem: millions of 1 KB objects cost more in requests and per-object overhead than the storage itself. Aggregate them (Parquet files, tar bundles).
- S3 Select / Athena to query data in place, and S3 Inventory for large bucket listings (listing billions of keys via API is slow).

---

## 13. Video Streaming Basics {#video}

- **Don't serve raw MP4 uploads directly** for playback at scale. **Transcode** into multiple bitrates and resolutions (an encoding ladder: 240p…4K) with codecs H.264/AVC (universal), H.265/HEVC, VP9, AV1 (efficient, growing hardware support).
- **Adaptive bitrate streaming (ABR)**: **HLS** (Apple; `.m3u8` playlists + segments, the most widely supported) and **MPEG-DASH** (`.mpd`). The video is split into 2–6 s segments per rendition, and the player switches rendition based on measured bandwidth. CMAF unifies segment formats for both.
- Pipeline: upload to S3 → transcoding (FFmpeg on workers/GPU, AWS Elemental MediaConvert, Mux, Cloudflare Stream, Bitmovin) → segments to S3 → CDN delivery → player (hls.js, Video.js, Shaka, ExoPlayer, AVPlayer).
- DRM for premium content (Widevine, FairPlay, PlayReady) plus signed URLs and cookies.
- Live streaming: RTMP/SRT/WHIP ingest → transcoder → HLS/DASH (latency 6–30 s; LL-HLS 2–5 s) or WebRTC for sub-second (see [`networking/05-real-time-communication.md`](../networking/05-real-time-communication.md)).
- This is a common system design interview: "Design YouTube/Netflix" (see [`interview-prep/system-design/04-case-studies-walkthroughs.md`](../interview-prep/system-design/04-case-studies-walkthroughs.md)).

---

## 14. S3-Compatible Alternatives {#alternatives}

| Service | Notes |
|---|---|
| Google Cloud Storage | Strong consistency, dual/multi-region buckets, signed URLs |
| Azure Blob Storage | Hot/cool/cold/archive tiers, SAS tokens, Data Lake Storage Gen2 (hierarchical namespace) |
| **Cloudflare R2** | S3 API, **no egress fees**, integrated with Workers/CDN |
| Backblaze B2, Wasabi | Cheap storage; S3-compatible |
| **MinIO** | Self-hosted S3-compatible (Kubernetes, on-prem, local dev) |
| Ceph RGW, SeaweedFS, Garage | Self-hosted options |
| LocalStack | Local AWS emulation for tests |

Use the **S3 API as the abstraction** (most providers support it) to stay portable. Test against MinIO or LocalStack in CI.

---

## 15. Interview Questions {#qa}

1. Why shouldn't uploads go through your API servers? Design direct-to-S3 uploads with presigned URLs.
2. How do you enforce max file size and type with presigned uploads?
3. How do you handle a 10 GB upload over an unreliable mobile network?
4. How do you securely serve private user documents? Public product images?
5. Why serve user-generated content from a separate domain?
6. Design an image upload pipeline with thumbnails and malware scanning.
7. What's the S3 consistency model today?
8. How would you reduce object storage costs for a system with petabytes of logs?
9. Explain adaptive bitrate streaming (HLS).
10. What metadata do you keep in the database vs object storage, and how do you clean up orphans?
