# Cloud Contract — Haispace Cloud

**Status:** Draft v1.0
**Milestone:** 3 — Cloud Contract
**Penulis:** GPT (Chief Product Architect) + Antigravity (Lead Software Architect)
**Tanggal:** 2026-07-26

---

## Core Principle

> **Cloud is eventually consistent with Runtime. Runtime is never eventually consistent with Cloud.**

Konsekuensi langsung yang tidak boleh dilanggar:

- Runtime tetap berjalan meskipun Cloud mati selama 24 jam
- Cloud hanya menerima fakta yang sudah terjadi di Runtime — tidak pernah menginstruksikan workflow
- Cloud tidak boleh mengoreksi session yang sedang berjalan
- Recovery selalu berasal dari snapshot lokal, bukan dari Cloud
- Sync Engine hanya mengirim fakta, bukan menjalankan workflow

---

## Dokumen Referensi

Dokumen ini adalah jembatan antara:

| Dokumen | Peran dalam Kontrak |
|---|---|
| [`CLOUD_DOMAIN_MODEL.md`](./CLOUD_DOMAIN_MODEL.md) | Bahasa domain yang digunakan dalam payload |
| [`CLOUD_RESOURCES.md`](./CLOUD_RESOURCES.md) | URI dan operasi yang tersedia |
| [`SYNC_STRATEGY.md`](./SYNC_STRATEGY.md) | Kapan dan bagaimana Runtime melakukan sync |
| [`AUTHENTICATION.md`](./AUTHENTICATION.md) | Identitas dan header wajib |
| [`ERROR_MODEL.md`](./ERROR_MODEL.md) | Kode error yang dikembalikan per operasi |

---

## Version Negotiation

Setiap komunikasi Runtime ↔ Cloud melibatkan empat versi yang harus selalu konsisten.

### Empat Versi

| Versi | Sumber | Dikirim Oleh | Keterangan |
|---|---|---|---|
| `cloudContractVersion` | Cloud | Cloud (response header) | Versi kontrak Cloud yang aktif |
| `architectureVersion` | RuntimeDescriptor | Booth (request header) | Versi Platform Runtime yang berjalan |
| `manifestVersion` | SessionSnapshot | Booth (payload) | Versi manifest yang digunakan session |
| `snapshotSchemaVersion` | SessionSnapshot | Booth (payload) | Versi schema SessionSnapshot |

### Request Headers (Booth → Cloud)

```
X-Booth-Id:               {boothId}
X-Api-Key:                {apiKey}
X-Request-Id:             {UUID}            ← untuk idempotency dan tracing
X-Correlation-Id:         {UUID}            ← untuk end-to-end tracing
X-Runtime-Version:        {runtimeVersion}
X-Architecture-Version:   {architectureVersion}
```

### Response Headers (Cloud → Booth)

```
X-Cloud-Contract-Version: {cloudContractVersion}
X-Request-Id:             {UUID}            ← echo dari request
X-Correlation-Id:         {UUID}            ← echo dari request
```

### Compatibility Rules

| Kondisi | Behavior |
|---|---|
| `architectureVersion` dikenal Cloud | Request diproses normal |
| `architectureVersion` tidak dikenal Cloud | `400 UNSUPPORTED_RUNTIME_VERSION` — Runtime harus update |
| `cloudContractVersion` berubah dari sebelumnya | Runtime log warning, tetap berfungsi jika backward compatible |
| `manifestVersion` dalam payload tidak cocok dengan Event | `422 MANIFEST_OUTDATED` — Runtime trigger manifest refresh |
| `snapshotSchemaVersion` lebih baru dari yang Cloud kenal | Cloud Ingestion Layer absorb perbedaan — tidak error |

**Aturan:** Cloud bertanggung jawab untuk backward compatibility terhadap `architectureVersion` yang masih dalam window support. Runtime **tidak** bertanggung jawab memastikan Cloud sudah update.

---

## Request Tracing

Setiap request memiliki dua identifier untuk tracing.

### requestId

- Di-generate oleh Runtime sebelum setiap request
- Unik per request (UUID v4)
- Dikirim via header `X-Request-Id`
- Cloud echo kembali di response
- Digunakan untuk: idempotency, log korelasi per request

### correlationId

- Di-generate oleh Runtime saat operasi dimulai (misalnya: saat session dibuat)
- Satu `correlationId` bisa mencakup banyak request (seluruh lifecycle session)
- Dikirim via header `X-Correlation-Id`
- Digunakan untuk: end-to-end tracing dari satu session melintasi banyak request
- Sama dengan `correlationId` dalam `DomainEventEnvelope`

### Tracing Flow Contoh (Session Lifecycle)

```
Session dibuat di Runtime
correlationId = "corr-sess-abc"

  POST /v1/sessions
  X-Request-Id: req-001
  X-Correlation-Id: corr-sess-abc

  POST /v1/session-events (PaymentAccepted)
  X-Request-Id: req-002
  X-Correlation-Id: corr-sess-abc

  PATCH /v1/sessions/{id}
  X-Request-Id: req-003
  X-Correlation-Id: corr-sess-abc

  POST /v1/audit-events (batch)
  X-Request-Id: req-004
  X-Correlation-Id: corr-sess-abc
```

Satu `correlationId` dapat ditelusuri di seluruh Cloud log untuk merekonstruksi perjalanan satu session.

---

## Operasi Utama

### 1. Device Registration

**Tujuan:** Booth mendaftarkan diri ke Cloud sebelum bisa beroperasi.
**Kapan terjadi:** Pertama kali app dijalankan, atau saat apiKey di-revoke.
**Sifat:** Synchronous — Booth tidak bisa beroperasi sebelum selesai.

**Flow:**
```
Runtime: periksa Keychain — apakah boothId dan apiKey sudah ada?
    ↓ (tidak ada)
Runtime: generate boothId dan RSA key pair
    ↓
POST /v1/devices
    payload: boothId, runtimeId, architectureVersion, platform, deviceClass, publicKey
    ↓
Cloud: validasi, simpan DeviceRegistration
Cloud: kembalikan apiKey
    ↓
Runtime: simpan apiKey ke Keychain
    ↓
Booth siap beroperasi
```

**Payload (konseptual):**

*Request:*
- Identity: `boothId`, `runtimeId`, `architectureVersion`, `platform`, `deviceClass`
- Security: `publicKey`
- Observability: `buildNumber`

*Response:*
- Credentials: `boothId`, `apiKey`
- Metadata: `registeredAt`, `cloudContractVersion`

**Idempotency:** `boothId` adalah key. Request kedua dengan `boothId` yang sudah terdaftar mengembalikan data existing.

**Errors:** `VALIDATION_ERROR`, `AUTHENTICATION_ERROR`

**Retry:** Tidak perlu — jika gagal, Runtime retry saat launch berikutnya. Tidak blocking kecuali apiKey hilang.

---

### 2. Manifest Fetch

**Tujuan:** Booth mendapatkan manifest terbaru untuk event yang di-assign.
**Kapan terjadi:** Saat launch, setiap 1 jam, atau saat `MANIFEST_OUTDATED` diterima.
**Sifat:** Synchronous untuk launch pertama — async untuk refresh periodik.

**Flow:**
```
Runtime: cek manifest cache (TTL: 1 jam)
    ↓ (cache expired atau tidak ada)
GET /v1/events/{eventId}/manifest
    header: X-Booth-Id, X-Api-Key
    ↓
Cloud: validasi akses booth ke event
Cloud: kembalikan manifest versi published terbaru
    ↓
Runtime: simpan manifest ke disk (cache)
Runtime: set manifestVersion untuk session berikutnya
    ↓
Session BARU menggunakan manifest ini
Session YANG SEDANG BERJALAN tidak terganggu (version pinning)
```

**Payload (konseptual):**

*Response:*
- Identity: `manifestId`, `eventId`, `version`
- Content: `packageIds`, `assetRefs` (list of assetId + role + displayOrder)
- Metadata: `publishedAt`, `checksum`

**Idempotency:** GET — tidak perlu idempotency key.

**Errors:** `MANIFEST_NOT_FOUND`, `EVENT_ACCESS_DENIED`, `BOOTH_SUSPENDED`

**Retry:** Ya — ExponentialBackoff jika `5xx`. Tidak retry jika `BOOTH_SUSPENDED` atau `EVENT_ACCESS_DENIED`.

**Ref:** [`SYNC_STRATEGY.md § Manifest Refresh Strategy`](./SYNC_STRATEGY.md)

---

### 3. Asset Download

**Tujuan:** Booth mengunduh file kreatif (frame, filter) untuk digunakan dalam session.
**Kapan terjadi:** Setelah manifest fetch, untuk setiap asset yang belum ada di cache lokal.
**Sifat:** Async — session dapat dimulai sementara asset background-download.

**Flow:**
```
Runtime: setelah manifest fetch, compare assetIds dengan local cache
    ↓ (ada asset yang belum diunduh)
GET /v1/assets/{assetId}/download-url
    ↓
Cloud: kembalikan pre-signed URL (expired setelah 15 menit)
    ↓
Runtime: download file dari pre-signed URL (ke object storage langsung)
Runtime: validasi checksum (SHA-256 dari metadata vs file yang diunduh)
    ↓
Runtime: simpan ke local asset cache
    ↓
Jika checksum tidak cocok: hapus file, retry download
```

**Payload (konseptual):**

*Response (download-url):*
- `downloadUrl`: pre-signed URL (opaque)
- `expiresAt`: kapan URL expired
- `expectedChecksum`: SHA-256 untuk validasi
- `sizeBytes`: ukuran file yang diharapkan

**Idempotency:** GET — tidak perlu idempotency key. Jika URL expired, request ulang.

**Errors:** `ASSET_NOT_FOUND`, `EVENT_ACCESS_DENIED`

**Retry:** Ya — jika download gagal (network), retry download dari URL yang sama (selama belum expired). Jika URL expired, fetch URL baru.

---

### 4. Session Archive Ingest (Snapshot Upload)

**Tujuan:** Runtime mengirim SessionSnapshot ke Cloud sebagai recovery anchor dan data archive.
**Kapan terjadi:** Dua checkpoint kritis — saat payment diterima, dan saat session selesai.
**Sifat:** Async — Runtime tidak menunggu konfirmasi untuk melanjutkan.

**Flow:**
```
Runtime: payment accepted → HaispaceSession.acceptPayment()
    ↓
SessionRepository.save(snapshot) ke disk (ATOMIC — terjadi SEBELUM upload)
    ↓
OfflineEventQueue.enqueue(SyncPayload.snapshot)
    ↓
SyncEngine: POST /v1/sessions
    payload: SessionSnapshot (konseptual)
    ↓
Cloud Ingestion Layer: transform Snapshot → SessionArchive
Cloud: kembalikan 201 Created atau 200 OK (idempotent)
    ↓
SyncEngine: mark payload sebagai .uploaded
```

**Payload (konseptual):**

*Request (pertama kali — create):*
- Identity: `sessionId`, `boothId`, `eventId`
- Version: `manifestVersion`, `snapshotSchemaVersion`, `architectureVersion`
- Guest: `name`, `queueNumber`
- Package: `packageId`
- Payment: `localTransactionId`, `amount`, `currency`, `method`, `acceptedAt`
- Summary: `captureSummary`, `deliverySummary` (null jika belum)
- Status: `archiveStatus`

*Request (update — session selesai):*
- Identity: `sessionId`
- Update: `completedAt`, `archiveStatus`, `captureSummary`, `deliverySummary`, `auditSummary`
- Concurrency: `snapshotVersion` (untuk optimistic lock)

**Idempotency:** `sessionId` adalah key. POST dengan `sessionId` yang sudah ada → update field mutable (bukan error, bukan duplikasi).

**Errors:** `VALIDATION_ERROR`, `STALE_SNAPSHOT`, `SYNC_CONFLICT`, `BOOTH_SUSPENDED`

**Retry:** Ya — ExponentialBackoff. Tidak retry jika `SYNC_CONFLICT` (eskalasi ke operator). Tidak retry jika `VALIDATION_ERROR` (payload salah, perlu diperbaiki).

**Ref:** [`SYNC_STRATEGY.md § Session Snapshot`](./SYNC_STRATEGY.md), [`SYNC_STRATEGY.md § Conflict Resolution`](./SYNC_STRATEGY.md)

---

### 5. Domain Event Upload

**Tujuan:** Runtime mengirim DomainEvent yang dihasilkan Session Aggregate ke Cloud.
**Kapan terjadi:** Setelah setiap intent yang menghasilkan domain event. Upload secara batch.
**Sifat:** Async, fire-and-sync. Urutan dipertahankan.

**Flow:**
```
Session Aggregate menghasilkan DomainEvent
    ↓
DomainEventEnvelope dibuat:
    eventId, sessionId, boothId, sequenceNumber,
    eventType, correlationId, causationId,
    runtimeVersion, occurredAt, payload
    ↓
CloudSyncSubscriber menerima (priority: .low)
    ↓
OfflineEventQueue.enqueue(SyncPayload.event)
    ↓
SyncEngine: POST /v1/session-events  (batch)
    payload: array of DomainEventEnvelope
    ↓
Cloud: untuk setiap event:
    - eventId sudah ada → silent ignore
    - eventId baru → simpan sebagai DomainEventRecord
    ↓
Cloud: kembalikan 200 OK atau 207 Multi-Status
SyncEngine: mark uploaded events sebagai .uploaded
```

**Payload per event (konseptual):**
- Identity: `eventId`, `sessionId`, `boothId`
- Sequencing: `sequenceNumber`, `correlationId`, `causationId`
- Content: `eventType`, `occurredAt`, `payload` (domain-specific data)
- Version: `runtimeVersion`

**Domain Events yang dikirim:**

| Event Type | Trigger |
|---|---|
| `PaymentAccepted` | Payment dikonfirmasi Runtime |
| `CaptureAdded` | Satu foto selesai diambil |
| `PhotoSelected` | Tamu memilih foto |
| `DeliveryQueued` | Delivery dimulai |
| `DeliveryCompleted` | Delivery berhasil |
| `DeliveryFailed` | Delivery gagal |
| `SessionCompleted` | Session selesai normal |
| `SessionAbandoned` | Session ditinggalkan tanpa selesai |
| `RecoveryInitiated` | Session di-recover dari snapshot |

**Idempotency:** `eventId` adalah key. Duplicate → silent ignore. Tidak ada `409 Conflict`.

**Batch Behavior:**
- Urutan array dipertahankan → `sequenceNumber` ditentukan dari urutan kirim
- Parsial: jika satu event gagal validasi, event lain tetap diproses
- Respons `207 Multi-Status` jika ada campuran sukses dan gagal

**Errors:** `INVALID_EVENT_ENVELOPE`, `AUTHENTICATION_ERROR`

**Retry:** Ya — ExponentialBackoff untuk `5xx`. Tidak retry jika `INVALID_EVENT_ENVELOPE`.

**Ref:** [`SYNC_STRATEGY.md § Session Events`](./SYNC_STRATEGY.md)

---

### 6. Audit Upload

**Tujuan:** Runtime mengirim audit trail operasional ke Cloud untuk compliance.
**Kapan terjadi:** Batch setiap 15 menit, atau saat app kembali aktif.
**Sifat:** Async, low-priority, batched.

**Flow:**
```
Runtime: setiap 15 menit atau saat app active
    ↓
AuditBatchCollector: kumpulkan pending AuditEvents dari disk
    ↓
POST /v1/audit-events (batch, max 100 records)
    ↓
Cloud: simpan setiap record (silent ignore untuk duplicate auditId)
    ↓
Runtime: mark batch sebagai uploaded
```

**Payload per audit record (konseptual):**
- Identity: `auditId`, `organizationId`, `boothId`, `sessionId` (nullable), `operatorId` (nullable)
- Content: `category`, `action`, `outcome`, `metadata`
- Timing: `occurredAt`

**Idempotency:** `auditId` adalah key. Duplicate → silent ignore.

**Errors:** `AUTHENTICATION_ERROR`, `PERMISSION_DENIED`

**Retry:** Ya — ExponentialBackoff. Audit tidak time-critical.

---

### 7. Descriptor Update

**Tujuan:** Cloud mengetahui versi Runtime terbaru yang aktif di setiap booth (observability).
**Kapan terjadi:** Setiap launch, atau setiap 24 jam.
**Sifat:** Async, best-effort. Kegagalan tidak menghalangi operasi.

**Flow:**
```
Runtime launch → atau 24 jam berlalu
    ↓
PATCH /v1/devices/{deviceId}/descriptor
    payload: architectureVersion, runtimeVersion, buildNumber, reportedAt
    ↓
Cloud: update lastKnownDescriptor di DeviceRegistration
Cloud: tidak memblokir Booth jika gagal
```

**Payload (konseptual):**
- `architectureVersion`, `runtimeVersion`, `buildNumber`, `reportedAt`

**Idempotency:** PATCH — `lastSeenAt` guard. Update selalu overwrite dengan data terbaru.

**Errors:** `DEVICE_NOT_REGISTERED` → trigger re-registration

**Retry:** Tidak perlu — low priority. Runtime mencoba lagi saat launch berikutnya.

---

## Retry & Stop Policy

Tabel keputusan kapan Runtime retry dan kapan berhenti.

| Kondisi | Retry? | Behavior |
|---|---|---|
| Network timeout / 5xx | ✅ | ExponentialBackoff, max 10 attempts |
| `RATE_LIMIT_EXCEEDED` | ✅ | Tunggu `retryAfterSeconds` |
| `AUTHENTICATION_ERROR` | ❌ | Trigger re-registration |
| `DEVICE_NOT_REGISTERED` | ❌ | Trigger re-registration |
| `PERMISSION_DENIED` | ❌ | Log, eskalasi ke operator |
| `BOOTH_SUSPENDED` | ❌ | Block semua operasi, notify operator |
| `VALIDATION_ERROR` | ❌ | Payload salah — tidak akan berhasil diretry |
| `MANIFEST_OUTDATED` | ❌ | Trigger manifest refresh, lalu retry operasi |
| `SYNC_CONFLICT` | ❌ | Log `SyncConflictDetected` (Critical), eskalasi ke operator |
| `DUPLICATE_EVENT` | ❌ (silent) | Tidak error — idempotency enforced |
| Max attempts tercapai | ❌ | Mark `.failed`, simpan 7 hari untuk manual recovery |

**Ref:** [`SYNC_STRATEGY.md § Retry & Backoff`](./SYNC_STRATEGY.md), [`ERROR_MODEL.md § Retry Decision Tree`](./ERROR_MODEL.md)

---

## Offline Guarantee

Tabel operasi yang boleh ditunda saat offline dan yang harus diblokir.

| Operasi | Offline Behavior |
|---|---|
| Device Registration | **Blocked** — booth tidak bisa beroperasi tanpa registrasi |
| Manifest Fetch (pertama) | **Blocked** — booth tidak bisa mulai session tanpa manifest |
| Manifest Refresh (periodik) | ✅ Ditunda — gunakan cache yang ada |
| Session Archive Ingest | ✅ Ditunda — simpan ke OfflineEventQueue |
| Domain Event Upload | ✅ Ditunda — simpan ke OfflineEventQueue |
| Audit Upload | ✅ Ditunda — simpan ke disk |
| Descriptor Update | ✅ Ditunda — best-effort |
| Asset Download (cache miss) | ⚠️ Degraded — gunakan asset yang sudah ada di cache; tampilkan warning |

**Prinsip:** Session tetap dapat dimulai dan diselesaikan tanpa internet, selama:
1. Booth sudah terdaftar (`boothId` + `apiKey` di Keychain)
2. Manifest sudah di-cache di disk
3. Asset yang diperlukan sudah di-cache di disk

---

## Contract Versioning

### cloudContractVersion

Cloud selalu menyertakan `cloudContractVersion` di response header. Format: SemVer.

| Change Type | Version Bump |
|---|---|
| Breaking change (field dihapus, semantik berubah) | MAJOR (v1 → v2) |
| Non-breaking addition (field baru, resource baru) | MINOR (v1.0 → v1.1) |
| Bug fix tanpa semantic change | PATCH (v1.0.0 → v1.0.1) |

### Compatibility Window

Cloud mendukung dua MAJOR version secara paralel selama minimum 6 bulan sebelum yang lama di-sunset.

Runtime yang berjalan di `/v1/` tetap berfungsi ketika Cloud merilis `/v2/`. Booth tidak perlu update untuk tetap beroperasi.

### snapshotSchemaVersion

Setiap SessionSnapshot membawa `snapshotSchemaVersion`. Cloud Ingestion Layer bertanggung jawab mengabsorb perbedaan versi schema. Runtime tidak perlu tahu versi Cloud.

| Runtime sends | Cloud knows | Behavior |
|---|---|---|
| v1 | v1 | Normal |
| v2 | v1, v2 | Absorb via Ingestion Layer |
| v1 | v2 only | `UNSUPPORTED_RUNTIME_VERSION` — Runtime perlu update |

---

## Security Contract

Setiap request dari Booth harus menyertakan:

```
X-Booth-Id: {boothId}           ← permanent identity
X-Api-Key:  {apiKey}            ← dari Keychain
X-Request-Id: {UUID}            ← per-request
X-Correlation-Id: {UUID}        ← per-operation (satu session = satu correlation)
```

Cloud memvalidasi:
1. `boothId` terdaftar dan `status = active`
2. `apiKey` valid untuk `boothId` ini
3. Booth memiliki akses ke resource yang diminta (event assignment, dll.)

**Ref:** [`AUTHENTICATION.md`](./AUTHENTICATION.md)

---

## End-to-End Communication Map

Peta lengkap semua operasi Runtime ↔ Cloud:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Runtime (iPad)                               │
│                                                                     │
│  App Launch                                                         │
│    ├── DeviceRegistration    → POST /v1/devices          [SYNC]    │
│    ├── ManifestFetch         → GET /v1/events/{id}/manifest [SYNC] │
│    └── AssetDownload         → GET /v1/assets/{id}/download-url    │
│                                                                     │
│  Session Active                                                     │
│    ├── SnapshotUpload (payment checkpoint)                          │
│    │   └── POST /v1/sessions                             [ASYNC]   │
│    ├── DomainEventUpload (batch)                                    │
│    │   └── POST /v1/session-events                       [ASYNC]   │
│    └── DescriptorUpdate                                             │
│        └── PATCH /v1/devices/{id}/descriptor             [ASYNC]   │
│                                                                     │
│  Session Complete                                                   │
│    ├── SnapshotUpload (final)                                       │
│    │   └── PATCH /v1/sessions/{id}                       [ASYNC]   │
│    └── DomainEventUpload (SessionCompleted)              [ASYNC]   │
│                                                                     │
│  Background (every 15 min)                                          │
│    └── AuditUpload           → POST /v1/audit-events     [ASYNC]   │
│                                                                     │
│  OfflineEventQueue drains automatically when online                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Acceptance Criteria

- [x] Alur komunikasi Runtime ↔ Cloud end-to-end terdokumentasi
- [x] Payload konseptual untuk setiap operasi utama (tanpa JSON detail)
- [x] Idempotency terdefinisi untuk seluruh operasi write
- [x] Setiap operasi mengacu ke `ERROR_MODEL.md`
- [x] Retry policy dan stop condition terdokumentasi (mengacu `SYNC_STRATEGY.md`)
- [x] Version negotiation (`architectureVersion`, `manifestVersion`, `snapshotSchemaVersion`, `cloudContractVersion`)
- [x] Request tracing via `correlationId` dan `requestId`
- [x] Core principle ditegaskan kembali
- [x] Offline guarantee terdokumentasi per operasi
- [x] Security contract terdokumentasi

---

*CLOUD_CONTRACT.md v1.0 — Milestone 3 Cloud Contract*
*Ref: CLOUD_DOMAIN_MODEL.md v1.1, CLOUD_RESOURCES.md v1.1, SYNC_STRATEGY.md v1.0, AUTHENTICATION.md v1.0, ERROR_MODEL.md v1.0*
