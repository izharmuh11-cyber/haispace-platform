# Cloud Resources — Haispace Cloud

**Status:** Final ✅
**Versi:** 1.1 (frozen after this review)
**Milestone:** 3 — Cloud Contract
**Penulis:** GPT (Chief Product Architect) + Antigravity (Lead Software Architect)
**Tanggal:** 2026-07-26

---

## Core Principle

> **Resources expose domain. They do not redefine it.**

Tidak boleh ada resource yang menciptakan domain baru atau konsep yang tidak ada di `CLOUD_DOMAIN_MODEL.md`.

---

## Global Conventions

### 1. Resource Naming Rules

Seluruh API mengikuti aturan berikut tanpa pengecualian:

- **Plural noun** untuk collection: `/sessions`, `/manifests`, `/assets`
- **Lowercase**: tidak ada huruf kapital di URI
- **Kebab-case**: `/session-events`, `/audit-events`
- **Action sebagai subresource**: `POST /v1/manifests/{id}/publish` ✅ bukan `POST /v1/publishManifest` ❌
- **Nested resource hanya satu level dalam**: `/v1/events/{id}/manifest` ✅ bukan `/v1/orgs/{id}/events/{id}/manifests/{id}/assets` ❌

### 2. Collection vs Singleton

Semua resource mengikuti pola dua tingkat:

| Pola | Contoh | Jenis |
|---|---|---|
| `/v1/{resource}` | `/v1/sessions` | Collection — list + bulk create |
| `/v1/{resource}/{id}` | `/v1/sessions/{sessionId}` | Singleton — satu entity spesifik |
| `/v1/{resource}/{id}/{action}` | `/v1/manifests/{id}/publish` | Lifecycle Action — operasi pada singleton |
| `/v1/{resource-a}/{id}/{resource-b}` | `/v1/events/{id}/manifest` | Sub-collection — scoped by parent |

Collection response selalu menggunakan pagination.
Singleton response mengembalikan satu entity.

### 3. Versioning

Seluruh resource berada di bawah URI versioning:

```
https://api.haispace.id/v1/{resource}
```

**Aturan versioning:**
- Semua endpoint menggunakan prefix `/v1/`
- Breaking change → `/v2/` (jalankan paralel minimum 6 bulan sebelum `/v1/` deprecated)
- Non-breaking addition (field baru, resource baru) → tetap di `/v1/`
- Booth lama yang masih di `/v1/` tidak boleh rusak ketika `/v2/` dirilis

**API Deprecation Lifecycle:**

```
Experimental  → bisa berubah, tidak boleh digunakan di production
    ↓
Supported     → stabil, dapat digunakan
    ↓
Deprecated    → masih berfungsi, jangan digunakan untuk integrasi baru
               (minimum 6 bulan notice sebelum removed)
    ↓
Removed       → hanya boleh terjadi di major version baru (/v2/)
```

Setiap resource dan field memiliki status lifecycle ini. Deprecated field ditandai dengan header respons:
```
Deprecation: true
Sunset: {date}
Link: {url-dokumentasi-pengganti}
```

### 4. Pagination

Seluruh endpoint collection menggunakan **cursor-based pagination**.

**Tidak boleh menggunakan offset-based pagination.** Tidak boleh mencampur keduanya.

```
Query params:
  ?limit=      int (default: 20, max: 100)
  ?cursor=     opaque string (dari response sebelumnya)
  ?direction=  forward | backward (default: forward)

Response shape:
  data:       [...]
  pagination:
    nextCursor:  string | null
    prevCursor:  string | null
    hasMore:     boolean
    totalCount:  int | null  (optional, mahal — hanya jika diminta)
```

Cursor bersifat opaque — client tidak boleh menginterpretasikan isinya.

### 5. Resource Classification

Setiap resource diklasifikasikan untuk membantu backend menentukan strategi penyimpanan dan caching tanpa menyebut implementasi spesifik.

| Resource | Classification | Karakteristik |
|---|---|---|
| `Organization` | Master Data | Jarang berubah, konsistensi tinggi |
| `Event` | Master Data | Jarang berubah, lifecycle terbatas |
| `Booth` | Master Data | Jarang berubah, identity permanen |
| `DeviceRegistration` | Infrastructure | Frekuensi moderate, audit trail penting |
| `Operator` | Identity | Jarang berubah, keamanan tinggi |
| `Manifest` | Configuration | Immutable setelah published |
| `Package` | Configuration | Immutable sebagian besar field |
| `Asset` | Content Metadata | Metadata immutable, file di CDN |
| `SessionArchive` | Runtime Archive | Tulis banyak, baca lambat |
| `DomainEventRecord` | Append-only Event Log | Volume tinggi, append-only, immutable |
| `AuditEvent` | Compliance Log | Volume tinggi, append-only, retensi panjang |

### 6. Typical Resource Frequency

Ekspektasi frekuensi penggunaan per resource — bukan rate limit, hanya panduan kapasitas.

| Resource | Typical Write Frequency | Typical Read Frequency |
|---|---|---|
| `Organization` | Very Low | Low |
| `Event` | Very Low | Low |
| `Booth` | Very Low | Low |
| `DeviceRegistration` | Low | Low |
| `Operator` | Very Low | Low |
| `Manifest` | Low | **High** (setiap booth saat launch) |
| `Package` | Low | **High** (setiap booth sebelum session) |
| `Asset` | Low | **High** (download per session) |
| `SessionArchive` | **High** (setiap session) | Medium |
| `DomainEventRecord` | **Very High** (setiap event domain) | Low |
| `AuditEvent` | **High** (setiap aksi operasional) | Low |

### 7. Concurrency Rules

Cloud **tidak menerima blind overwrite** pada resource mutable.

Semua PATCH pada resource mutable harus menyertakan versi saat ini sebagai guard:

```
If-Match: "{etag}"
```

atau

```
X-Resource-Version: {version}
```

Jika versi tidak cocok, Cloud merespons dengan `409 Conflict` dan `SYNC_CONFLICT`.

**Resource yang memerlukan concurrency guard:**

| Resource | Concurrency Strategy |
|---|---|
| `Event` | Optimistic lock via `updatedAt` |
| `Booth` | Optimistic lock via `updatedAt` |
| `DeviceRegistration` | `lastSeenAt` guard untuk descriptor update |
| `SessionArchive` | `snapshotVersion` (dikirim dari Runtime) |
| `Manifest` | N/A — immutable setelah published |

**Resource yang append-only** (`DomainEventRecord`, `AuditEvent`) tidak memerlukan concurrency guard karena tidak ada update.

### 8. Bulk Operations Policy

| Resource | Bulk POST | Atomik | Urutan Dipertahankan | Duplicate Handling |
|---|---|---|---|---|
| `DomainEventRecord` | ✅ (utama) | ❌ Parsial | ✅ Ya | Silent — diabaikan |
| `AuditEvent` | ✅ | ❌ Parsial | ✅ Ya | Silent — diabaikan |
| `Asset` | ✅ (future) | ❌ Parsial | — | Error per item |
| Semua lainnya | ❌ | — | — | — |

**Aturan umum bulk:**
- Bulk endpoint menggunakan URI yang sama (`POST /v1/session-events`) dengan array sebagai payload
- Respons parsial menggunakan `207 Multi-Status` — setiap item memiliki status sendiri
- Kegagalan satu item tidak membatalkan item lainnya
- Urutan array dipertahankan dalam `sequenceNumber` untuk `DomainEventRecord`

### 9. Long-Running Operations

Beberapa operasi Cloud bersifat asynchronous dan tidak bisa selesai dalam satu HTTP request.

> **Operasi yang membutuhkan waktu lama harus mengembalikan resource status, bukan menahan koneksi HTTP.**

Pola yang digunakan:

```
POST /v1/assets          → 202 Accepted
{
  "jobId": "job_abc",
  "status": "processing",
  "statusUrl": "/v1/jobs/job_abc"
}

GET /v1/jobs/job_abc     → polling
{
  "jobId": "job_abc",
  "status": "completed | failed | processing",
  "result": { ... } | null
}
```

**Operasi yang bersifat long-running:**

| Operasi | Alasan |
|---|---|
| Asset upload + verification | Checksum validation, CDN propagation |
| Manifest publish | Validasi semua asset refs, CDN warm-up |
| Session archive export | Query besar, file generation |
| AI processing (future) | Inferensi model membutuhkan waktu |

### 10. Error Reference

Dokumen ini **tidak mendefinisikan ulang error**. Setiap resource mereferensikan kode error dari [`ERROR_MODEL.md`](./ERROR_MODEL.md).

---

## Resource Index

| Resource | URI | Aggregate Root | Classification |
|---|---|---|---|
| Organizations | `/v1/organizations` | `Organization` | Master Data |
| Events | `/v1/events` | `Event` | Master Data |
| Booths | `/v1/booths` | `Booth` | Master Data |
| Device Registration | `/v1/devices` | `DeviceRegistration` | Infrastructure |
| Operators | `/v1/operators` | `Operator` | Identity |
| Manifests | `/v1/manifests` | `Manifest` | Configuration |
| Assets | `/v1/assets` | `Asset` | Content Metadata |
| Packages | `/v1/packages` | `Package` | Configuration |
| Session Archives | `/v1/sessions` | `SessionArchive` | Runtime Archive |
| Domain Event Records | `/v1/session-events` | `DomainEventRecord` | Event Log |
| Audit Events | `/v1/audit-events` | `AuditEvent` | Compliance Log |

---

## Permission Levels

| Level | Deskripsi |
|---|---|
| `Admin` | Superuser organisasi — akses penuh |
| `Operator` | Pengguna manusia — akses baca dan operasional |
| `Booth` | Runtime iPad — akses terbatas sesuai identity booth |
| `—` | Tidak diizinkan sama sekali |

---

## 1. Organizations

**URI:** `/v1/organizations`, `/v1/organizations/{organizationId}`
**Aggregate:** `Organization` | **Classification:** Master Data

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| List organizations | GET | `/v1/organizations` | — | — | ✅ |
| Get organization | GET | `/v1/organizations/{id}` | — | ✅ | ✅ |
| Create organization | POST | `/v1/organizations` | — | — | ✅ |
| Update organization | PATCH | `/v1/organizations/{id}` | — | — | ✅ |
| Suspend | POST | `/v1/organizations/{id}/suspend` | — | — | ✅ |
| Reinstate | POST | `/v1/organizations/{id}/reinstate` | — | — | ✅ |

### Idempotency
POST create: `organizationId` unik. Tidak dapat membuat dua org dengan ID yang sama.

### Filtering
`?status=active|suspended|terminated` `?plan=starter|professional|enterprise`

### Error References
`PERMISSION_DENIED`, `VALIDATION_ERROR`

---

## 2. Events

**URI:** `/v1/events`, `/v1/events/{eventId}`
**Aggregate:** `Event` | **Classification:** Master Data

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| List events | GET | `/v1/events` | ✅ (assigned only) | ✅ | ✅ |
| Get event | GET | `/v1/events/{id}` | ✅ (assigned) | ✅ | ✅ |
| Create event | POST | `/v1/events` | — | Manager+ | ✅ |
| Update event | PATCH | `/v1/events/{id}` | — | Manager+ | ✅ |
| Activate | POST | `/v1/events/{id}/activate` | — | Manager+ | ✅ |
| Complete | POST | `/v1/events/{id}/complete` | — | Manager+ | ✅ |
| Archive | POST | `/v1/events/{id}/archive` | — | — | ✅ |
| Assign booth | POST | `/v1/events/{id}/booths/{boothId}` | — | Manager+ | ✅ |
| Remove booth | DELETE | `/v1/events/{id}/booths/{boothId}` | — | — | ✅ |

### Idempotency
PATCH: requires `If-Match` header (optimistic lock via `updatedAt`).

### Filtering
`?organizationId=` `?status=draft|active|completed|archived` `?from=` `?to=` `?boothId=`

### Error References
`VALIDATION_ERROR`, `PERMISSION_DENIED`, `EVENT_ACCESS_DENIED`

---

## 3. Booths

**URI:** `/v1/booths`, `/v1/booths/{boothId}`
**Aggregate:** `Booth` | **Classification:** Master Data

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| List booths | GET | `/v1/booths` | — | ✅ | ✅ |
| Get booth | GET | `/v1/booths/{id}` | ✅ (self) | ✅ | ✅ |
| Create booth | POST | `/v1/booths` | — | — | ✅ |
| Update booth | PATCH | `/v1/booths/{id}` | — | Manager+ | ✅ |
| Suspend | POST | `/v1/booths/{id}/suspend` | — | — | ✅ |

### Idempotency
PATCH: requires `If-Match` header. POST: `boothId` unique constraint.

### Filtering
`?organizationId=` `?eventId=` `?status=active|inactive|suspended`

### Error References
`BOOTH_NOT_FOUND`, `BOOTH_SUSPENDED`, `PERMISSION_DENIED`

---

## 4. Device Registration

**URI:** `/v1/devices`, `/v1/devices/{deviceId}`
**Aggregate:** `DeviceRegistration` | **Classification:** Infrastructure

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| Register device | POST | `/v1/devices` | ✅ (self) | — | ✅ |
| Get registration | GET | `/v1/devices/{id}` | ✅ (self) | ✅ | ✅ |
| List registrations | GET | `/v1/devices` | — | ✅ | ✅ |
| Update descriptor | PATCH | `/v1/devices/{id}/descriptor` | ✅ (self) | — | — |
| Revoke | POST | `/v1/devices/{id}/revoke` | — | — | ✅ |

### Idempotency
POST register: idempotent — `deviceId` sudah terdaftar dan `active` → kembalikan data existing.
PATCH descriptor: selalu overwrite (`lastSeenAt` guard).

### Filtering
`?boothId=` `?status=active|revoked`

### Error References
`DEVICE_NOT_REGISTERED`, `AUTHENTICATION_ERROR`, `BOOTH_SUSPENDED`

---

## 5. Operators

**URI:** `/v1/operators`, `/v1/operators/{operatorId}`
**Aggregate:** `Operator` | **Classification:** Identity

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| List operators | GET | `/v1/operators` | — | ✅ (own org) | ✅ |
| Get operator | GET | `/v1/operators/{id}` | — | ✅ (self) | ✅ |
| Create operator | POST | `/v1/operators` | — | Owner | ✅ |
| Update operator | PATCH | `/v1/operators/{id}` | — | Owner | ✅ |
| Suspend | POST | `/v1/operators/{id}/suspend` | — | Owner | ✅ |
| Login | POST | `/v1/auth/operator/login` | — | ✅ | ✅ |
| Logout | POST | `/v1/auth/operator/logout` | — | ✅ | ✅ |

### Filtering
`?organizationId=` `?role=owner|manager|staff` `?status=active|suspended`

### Error References
`TOKEN_EXPIRED`, `AUTHENTICATION_ERROR`, `PERMISSION_DENIED`

---

## 6. Manifests

**URI:** `/v1/manifests`, `/v1/manifests/{manifestId}`
**Aggregate:** `Manifest` | **Classification:** Configuration

> Manifest immutable setelah `published`. PATCH hanya saat `draft`. Long-running: publish bisa membutuhkan waktu (CDN warm-up).

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| Get manifest | GET | `/v1/manifests/{id}` | ✅ (assigned event) | ✅ | ✅ |
| Get active manifest | GET | `/v1/events/{id}/manifest` | ✅ (assigned) | ✅ | ✅ |
| List versions | GET | `/v1/events/{id}/manifests` | — | ✅ | ✅ |
| Create (draft) | POST | `/v1/manifests` | — | Manager+ | ✅ |
| Update draft | PATCH | `/v1/manifests/{id}` | — | Manager+ | ✅ |
| Publish | POST | `/v1/manifests/{id}/publish` | — | Manager+ | ✅ |

### Idempotency
POST publish: idempotent — memanggil publish pada manifest yang sudah `published` tidak mengubah apa pun.

### Long-running
`POST /v1/manifests/{id}/publish` → `202 Accepted` + `jobId` jika validasi asset dan CDN warm-up membutuhkan waktu.

### Error References
`MANIFEST_NOT_FOUND`, `MANIFEST_OUTDATED`, `PERMISSION_DENIED`, `EVENT_ACCESS_DENIED`

---

## 7. Assets

**URI:** `/v1/assets`, `/v1/assets/{assetId}`
**Aggregate:** `Asset` | **Classification:** Content Metadata

> Asset metadata immutable setelah upload. File tidak pernah dimodifikasi. Download via pre-signed URL.

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| List assets | GET | `/v1/assets` | — | ✅ | ✅ |
| Get asset metadata | GET | `/v1/assets/{id}` | ✅ | ✅ | ✅ |
| Get download URL | GET | `/v1/assets/{id}/download-url` | ✅ | ✅ | ✅ |
| Upload asset | POST | `/v1/assets` | — | Manager+ | ✅ |
| Deprecate | POST | `/v1/assets/{id}/deprecate` | — | — | ✅ |

### Idempotency
POST upload: idempotent berdasarkan `checksum + organizationId` — file yang sama tidak di-upload ulang.

### Long-running
`POST /v1/assets` → `202 Accepted` + `jobId` untuk checksum validation dan CDN propagation.

### Bulk
`POST /v1/assets` dengan array: parsial, setiap item memiliki status sendiri (`207 Multi-Status`).

### Filtering
`?organizationId=` `?type=frame|filter|overlay|background|sticker|font` `?status=active|deprecated`

### Error References
`ASSET_NOT_FOUND`, `PERMISSION_DENIED`

---

## 8. Packages

**URI:** `/v1/packages`, `/v1/packages/{packageId}`
**Aggregate:** `Package` | **Classification:** Configuration

> Tidak ada PATCH — `priceAmount` dan `captureLimit` immutable setelah ada SessionArchive.

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| List packages | GET | `/v1/packages` | ✅ (event-scoped) | ✅ | ✅ |
| Get package | GET | `/v1/packages/{id}` | ✅ | ✅ | ✅ |
| Create package | POST | `/v1/packages` | — | Manager+ | ✅ |
| Discontinue | POST | `/v1/packages/{id}/discontinue` | — | — | ✅ |

### Filtering
`?eventId=` `?status=active|discontinued`

### Error References
`VALIDATION_ERROR`, `PERMISSION_DENIED`

---

## 9. Session Archives

**URI:** `/v1/sessions`, `/v1/sessions/{sessionId}`
**Aggregate:** `SessionArchive` | **Classification:** Runtime Archive

> Dikirim oleh Runtime. Cloud menerima dan menyimpan. Booth hanya bisa melihat session milik boothId-nya sendiri.

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| Ingest snapshot | POST | `/v1/sessions` | ✅ (self) | — | — |
| Update archive | PATCH | `/v1/sessions/{id}` | ✅ (self) | — | — |
| Get session | GET | `/v1/sessions/{id}` | ✅ (self) | ✅ | ✅ |
| List sessions | GET | `/v1/sessions` | ✅ (self only) | ✅ | ✅ |

### Idempotency
POST ingest: `sessionId` adalah idempotency key. Jika sessionId sudah ada, Cloud memperbarui field mutable (tidak membuat record baru).
PATCH: `snapshotVersion` digunakan sebagai optimistic lock.

### Filtering
`?eventId=` `?boothId=` `?status=in_progress|completed|recovered|abandoned` `?from=` `?to=`

### Error References
`SESSION_NOT_FOUND`, `AUTHENTICATION_ERROR`, `VALIDATION_ERROR`, `STALE_SNAPSHOT`, `SYNC_CONFLICT`

---

## 10. Domain Event Records

**URI:** `/v1/session-events`
**Aggregate:** `DomainEventRecord` | **Classification:** Append-only Event Log

> Append-only. Tidak ada PATCH. Tidak ada DELETE. Duplikat diabaikan secara silent.

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| Upload events (batch) | POST | `/v1/session-events` | ✅ | — | — |
| List events | GET | `/v1/session-events` | ✅ (self) | ✅ | ✅ |
| Get event | GET | `/v1/session-events/{eventId}` | ✅ (self) | ✅ | ✅ |

### Idempotency
POST: `eventId` (dari `DomainEventEnvelope`) adalah idempotency key. Duplicate → silent ignore, tidak error.
Respons parsial: `207 Multi-Status` jika ada campuran accepted dan duplicate.

### Bulk
- Diizinkan ✅ — ini adalah pola utama (bukan exception)
- Parsial ✅ — kegagalan satu item tidak membatalkan lainnya
- Urutan dipertahankan ✅ — `sequenceNumber` ditentukan dari urutan array
- Duplicate silent ✅

### Filtering
`?sessionId=` `?boothId=` `?eventType=` `?from=` `?to=`

### Error References
`AUTHENTICATION_ERROR`, `INVALID_EVENT_ENVELOPE`

---

## 11. Audit Events

**URI:** `/v1/audit-events`
**Aggregate:** `AuditEvent` | **Classification:** Compliance Log

> Append-only. Tidak ada PATCH. Tidak ada DELETE. Retensi 7 tahun.

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| Upload audit batch | POST | `/v1/audit-events` | ✅ | — | — |
| List audit events | GET | `/v1/audit-events` | — | ✅ | ✅ |
| Get audit event | GET | `/v1/audit-events/{auditId}` | — | ✅ | ✅ |

### Idempotency
POST: `auditId` adalah idempotency key. Duplikat diabaikan secara silent.

### Bulk
- Diizinkan ✅
- Parsial ✅
- Urutan dipertahankan ✅

### Filtering
`?organizationId=` `?boothId=` `?operatorId=` `?sessionId=` `?category=` `?outcome=success|failure|warning` `?from=` `?to=`

### Error References
`AUTHENTICATION_ERROR`, `PERMISSION_DENIED`

---

## Operations Summary

| Resource | GET list | GET one | POST | PATCH | DELETE | Lifecycle |
|---|---|---|---|---|---|---|
| Organizations | Admin | Op+Admin | Admin | Admin | Admin | suspend, reinstate |
| Events | All | All | Manager+ | Manager+ | Admin | activate, complete, archive |
| Booths | Op+Admin | All | Admin | Manager+ | — | suspend |
| Devices | Op+Admin | Booth+Op+Admin | Booth | Booth (descriptor) | — | revoke |
| Operators | Op+Admin | Self+Admin | Owner+ | Owner+ | — | suspend |
| Manifests | Op+Admin | All | Manager+ | Manager+ (draft) | — | publish |
| Assets | Op+Admin | All | Manager+ | — | — | deprecate |
| Packages | All | All | Manager+ | — | — | discontinue |
| Sessions | All (scoped) | All (scoped) | Booth | Booth | — | — |
| Session Events | All (scoped) | All (scoped) | Booth (batch) | — | — | — |
| Audit Events | Op+Admin | Op+Admin | Booth (batch) | — | — | — |

---

## Acceptance Criteria (Final)

- [x] Seluruh Aggregate Root memiliki representasi resource yang jelas
- [x] Setiap resource memiliki operasi yang diizinkan (tabel eksplisit)
- [x] Idempotency strategy terdokumentasi per resource
- [x] Versioning konsisten (`/v1/`) di seluruh API
- [x] Pagination hanya cursor-based (satu pendekatan)
- [x] Permission matrix lengkap (Booth | Operator | Admin)
- [x] Tidak ada payload JSON atau detail implementasi
- [x] Semua resource dipetakan ke Domain Model tanpa konsep baru
- [x] Resource Classification tersedia
- [x] Collection vs Singleton convention terdokumentasi
- [x] Concurrency policy dijelaskan (no blind overwrite)
- [x] Bulk operation policy terdokumentasi per resource
- [x] Typical resource frequency terdokumentasi
- [x] Long-running operation pattern dijelaskan
- [x] API deprecation lifecycle terdokumentasi
- [x] Resource naming convention sebagai aturan global

---

*CLOUD_RESOURCES.md v1.1 — FINAL*
*Ref: CLOUD_DOMAIN_MODEL.md v1.1, ERROR_MODEL.md v1.0, AUTHENTICATION.md v1.0*
