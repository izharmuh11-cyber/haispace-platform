# Error Model — Haispace Cloud

**Status:** Draft
**Versi:** 1.0
**Milestone:** 3 — Cloud Contract
**Penulis:** Antigravity (Lead Software Architect)
**Review:** GPT (Chief Product Architect)

---

> **Prinsip Utama:** Satu bahasa error untuk seluruh platform. Tidak ada string acak dari backend. Setiap error memiliki kode, kategori, retry hint, dan pesan yang dapat dibaca manusia.

---

## 1. Error Envelope

Semua error dari Cloud dikembalikan dalam format yang seragam:

```json
{
  "error": {
    "code": "PAYMENT_NOT_VERIFIED",
    "category": "DOMAIN",
    "message": "Payment for session {sessionId} has not been verified by the provider.",
    "retryable": false,
    "retryAfterSeconds": null,
    "correlationId": "req_abc123",
    "timestamp": "2026-07-26T02:55:00Z",
    "detail": {
      "sessionId": "sess_xyz",
      "paymentProvider": "qris"
    }
  }
}
```

### Field Definitions

| Field | Type | Wajib | Keterangan |
|---|---|---|---|
| `code` | String (SCREAMING_SNAKE_CASE) | ✅ | Kode error unik — tidak boleh berubah antar versi |
| `category` | String | ✅ | Kategori besar error (lihat bagian 2) |
| `message` | String | ✅ | Pesan yang bisa dibaca engineer (bukan untuk tamu) |
| `retryable` | Boolean | ✅ | Apakah Runtime boleh retry request ini? |
| `retryAfterSeconds` | Int atau null | ✅ | Jika retryable, berapa detik minimum sebelum retry |
| `correlationId` | String | ✅ | ID request dari header `X-Request-Id` |
| `timestamp` | ISO 8601 | ✅ | Waktu error terjadi di Cloud |
| `detail` | Object atau null | ❌ | Data tambahan spesifik per error code |

---

## 2. Error Categories

| Category | HTTP Range | Deskripsi |
|---|---|---|
| `VALIDATION` | 400 | Input tidak valid — tidak perlu retry |
| `AUTHENTICATION` | 401 | Identitas tidak dikenal atau token kadaluwarsa |
| `PERMISSION` | 403 | Identitas dikenal tetapi tidak punya akses |
| `NOT_FOUND` | 404 | Resource tidak ditemukan |
| `CONFLICT` | 409 | State konflik — bisa karena duplicate atau versi |
| `DOMAIN` | 422 | Business rule violation — valid secara teknis, tapi tidak valid secara domain |
| `RATE_LIMIT` | 429 | Terlalu banyak request — retry setelah `retryAfterSeconds` |
| `SERVER` | 5xx | Masalah internal Cloud — selalu retryable |

---

## 3. Daftar Error Code

### 3.1 Authentication & Authorization

| Code | HTTP | Retryable | Keterangan |
|---|---|---|---|
| `AUTHENTICATION_ERROR` | 401 | ❌ | API Key tidak valid atau tidak dikenali |
| `DEVICE_NOT_REGISTERED` | 401 | ❌ | `boothId` tidak ada di database Cloud |
| `TOKEN_EXPIRED` | 401 | ❌ | Operator session token kadaluwarsa |
| `PERMISSION_DENIED` | 403 | ❌ | Booth tidak punya akses ke resource ini |
| `BOOTH_SUSPENDED` | 403 | ❌ | Booth di-suspend oleh Admin |
| `EVENT_ACCESS_DENIED` | 403 | ❌ | Booth tidak di-assign ke event ini |

### 3.2 Validation

| Code | HTTP | Retryable | Keterangan |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | ❌ | Satu atau lebih field tidak valid (detail berisi field errors) |
| `MISSING_REQUIRED_FIELD` | 400 | ❌ | Field wajib tidak ada dalam payload |
| `INVALID_BOOTH_ID` | 400 | ❌ | Format boothId tidak valid |
| `INVALID_SESSION_ID` | 400 | ❌ | Format sessionId tidak valid |
| `INVALID_EVENT_ENVELOPE` | 400 | ❌ | Format DomainEventEnvelope tidak valid |
| `UNSUPPORTED_RUNTIME_VERSION` | 400 | ❌ | `X-Architecture-Version` tidak didukung Cloud |

### 3.3 Resource

| Code | HTTP | Retryable | Keterangan |
|---|---|---|---|
| `SESSION_NOT_FOUND` | 404 | ❌ | Session dengan ID ini tidak ada di Cloud |
| `MANIFEST_NOT_FOUND` | 404 | ❌ | Manifest untuk event ini tidak ditemukan |
| `ASSET_NOT_FOUND` | 404 | ❌ | Asset file tidak ditemukan di CDN |
| `BOOTH_NOT_FOUND` | 404 | ❌ | Booth dengan ID ini tidak terdaftar |
| `OPERATOR_NOT_FOUND` | 404 | ❌ | Operator tidak ditemukan |

### 3.4 Conflict & Sync

| Code | HTTP | Retryable | Keterangan |
|---|---|---|---|
| `SYNC_CONFLICT` | 409 | ❌* | State konflik terdeteksi — Runtime harus resolve manual |
| `DUPLICATE_EVENT` | 409 | ❌ | `eventId` sudah pernah diterima (idempotency enforced) |
| `STALE_SNAPSHOT` | 409 | ❌ | Snapshot yang dikirim lebih lama dari yang sudah ada di Cloud |
| `SEQUENCE_MISMATCH` | 409 | ❌ | `sequenceNumber` tidak sesuai urutan yang diharapkan |

*`SYNC_CONFLICT` tidak di-retry otomatis. Runtime harus log sebagai `SyncConflictDetected` dan eskalasi ke operator.

### 3.5 Domain (Business Rule)

| Code | HTTP | Retryable | Keterangan |
|---|---|---|---|
| `PAYMENT_NOT_VERIFIED` | 422 | ❌ | Payment untuk session ini belum terverifikasi oleh provider |
| `SESSION_ALREADY_COMPLETED` | 422 | ❌ | Operasi dikirim untuk session yang sudah selesai |
| `DELIVERY_ALREADY_PROCESSED` | 422 | ❌ | Delivery sudah diproses sebelumnya |
| `MANIFEST_OUTDATED` | 422 | ❌* | Manifest yang dipakai booth lebih lama dari versi aktif |
| `PACKAGE_DISCONTINUED` | 422 | ❌ | Package yang dipakai dalam session sudah tidak aktif |
| `CAPTURE_LIMIT_EXCEEDED` | 422 | ❌ | Jumlah capture melebihi batas package |

*`MANIFEST_OUTDATED` memicu Manifest Refresh otomatis di Runtime.

### 3.6 Rate Limit

| Code | HTTP | Retryable | Keterangan |
|---|---|---|---|
| `RATE_LIMIT_EXCEEDED` | 429 | ✅ | Terlalu banyak request — gunakan `retryAfterSeconds` |
| `SYNC_QUEUE_FULL` | 429 | ✅ | Cloud sedang overloaded, coba lagi nanti |

### 3.7 Server

| Code | HTTP | Retryable | Keterangan |
|---|---|---|---|
| `INTERNAL_SERVER_ERROR` | 500 | ✅ | Error tidak terduga di Cloud |
| `SERVICE_UNAVAILABLE` | 503 | ✅ | Cloud sedang maintenance atau overloaded |
| `GATEWAY_TIMEOUT` | 504 | ✅ | Request timeout di Cloud infrastructure |

---

## 4. Error Handling di Runtime

### Retry Decision Tree

```
Terima Error dari Cloud
    ↓
Cek error.retryable
    ↓ (false)
Log error → DomainEventPublisher.publish(SyncUploadFailed)
Tidak retry
    ↓ (true)
Cek error.code
    ↓
RATE_LIMIT_EXCEEDED → tunggu retryAfterSeconds, lalu retry
SERVER error        → ExponentialBackoff (lihat SYNC_STRATEGY.md)
    ↓
Cek attemptCount >= maxAttempts?
    ↓ (ya)
Mark payload .failed → simpan 7 hari untuk manual recovery
Publish SyncUploadFailed(reason: .maxRetriesExceeded)
```

### Error yang Memicu Side Effect di Runtime

| Error Code | Side Effect |
|---|---|
| `DEVICE_NOT_REGISTERED` | Trigger re-registration flow |
| `AUTHENTICATION_ERROR` | Invalidate apiKey, trigger re-registration |
| `MANIFEST_OUTDATED` | Trigger immediate Manifest Refresh |
| `UNSUPPORTED_RUNTIME_VERSION` | Log critical warning, block upload, notify operator |
| `BOOTH_SUSPENDED` | Block semua operasi, tampilkan pesan ke operator |
| `SYNC_CONFLICT` | Log sebagai `SyncConflictDetected` (Critical priority), eskalasi ke operator |

---

## 5. Error yang Tidak Pernah Diekspos ke Tamu

**Semua error di atas adalah internal** — antara Runtime dan Cloud.

Tamu tidak melihat kode error ini. Yang tamu lihat hanya:
- UI yang tetap berjalan (karena Runtime beroperasi offline-first)
- Pesan sederhana dari Operator jika ada masalah hardware

Operator melihat error ini via **Mission Control** (Milestone 6).

---

## 6. Versioning Error Codes

### Aturan

- **Kode error tidak boleh dihapus** — hanya boleh di-deprecate
- **Kode error tidak boleh berganti arti** — perubahan arti = kode baru
- **Penambahan kode baru** tidak dianggap breaking change
- **Penghapusan `detail` field** adalah breaking change

### Deprecation Format

```json
{
  "error": {
    "code": "OLD_ERROR_CODE",
    "deprecated": true,
    "replacedBy": "NEW_ERROR_CODE",
    ...
  }
}
```

---

*ERROR_MODEL.md v1.0 — Milestone 3 Cloud Contract*
*Ref: constitution/PLATFORM_RUNTIME_V1.md, cloud/SYNC_STRATEGY.md, cloud/AUTHENTICATION.md*
