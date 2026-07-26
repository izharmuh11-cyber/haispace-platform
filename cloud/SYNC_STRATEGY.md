# Sync Strategy — Haispace Cloud

**Status:** Draft
**Versi:** 1.0
**Milestone:** 3 — Cloud Contract
**Penulis:** Antigravity (Lead Software Architect)
**Review:** GPT (Chief Product Architect)

---

> **Prinsip Utama:** Runtime tidak bergantung pada Cloud untuk berjalan. Cloud adalah layanan pendukung. Runtime dapat beroperasi penuh dalam mode offline dan melakukan sync ketika koneksi tersedia.

---

## 1. Model Konsistensi

Haispace menggunakan model **Eventually Consistent** untuk seluruh data yang dihasilkan oleh Runtime.

### Alasan

- Booth beroperasi di venue fisik yang koneksinya tidak bisa dijamin (WiFi venue, tethering, dll.)
- Tamu tidak boleh menunggu hanya karena Cloud tidak merespons
- Payload session kecil — eventual consistency aman diterapkan

### Pengecualian (Realtime)

Dua operasi yang **membutuhkan konfirmasi Cloud sebelum lanjut**:

| Operasi | Alasan |
|---|---|
| Device Registration | Booth harus dikenali Cloud sebelum bisa beroperasi |
| Manifest Fetch | Booth harus punya manifest versi terbaru sebelum session dimulai |

Semua operasi lainnya bersifat **Fire-and-Sync**: Runtime melakukan operasi lokal dulu, Cloud menerima hasilnya secara asinkronus.

---

## 2. Sync Model per Resource

### 2.1 Session Events (Domain Events)

| Arah | Strategi |
|---|---|
| Runtime → Cloud | Async Upload, retry on failure |
| Cloud → Runtime | Tidak ada push. Cloud hanya menerima. |

**Flow:**
```
Session Aggregate menghasilkan DomainEvent
    ↓
DomainEventEnvelope dibuat (eventId, sequenceNumber, correlationId)
    ↓
CloudSyncSubscriber menerima event (priority: .low)
    ↓
Masuk ke OfflineEventQueue (disk-backed)
    ↓
SyncEngine mencoba upload ke Cloud saat online
    ↓
Jika berhasil: event di-mark .uploaded
    ↓
Jika gagal: retry dengan ExponentialBackoff
```

**Idempotency:** Setiap event memiliki `eventId` (UUID). Cloud **wajib** mengabaikan event dengan `eventId` yang sudah diterima sebelumnya.

### 2.2 Audit Records

| Arah | Strategi |
|---|---|
| Runtime → Cloud | Batched upload, maksimal 100 records per batch |
| Frekuensi | Setiap 15 menit jika online, atau saat app kembali aktif |

Audit records bersifat **append-only** dan tidak pernah di-update. Tidak ada conflict resolution yang diperlukan.

### 2.3 Manifest

| Arah | Strategi |
|---|---|
| Cloud → Runtime | Pull-on-demand, bukan push |
| Trigger | Saat session baru dimulai, atau secara periodik (1 jam sekali) |

**Manifest Version Pinning:**
```
Runtime menyimpan manifestVersion di SessionSnapshot
    ↓
Session yang sedang berjalan tidak boleh mendapat manifest baru
    ↓
Manifest baru hanya berlaku untuk session berikutnya
```

### 2.4 Session Snapshot (Recovery Data)

| Arah | Strategi |
|---|---|
| Runtime → Cloud | Upload saat payment confirmed (critical checkpoint) |
| Runtime → Cloud | Upload saat session selesai |

Session Snapshot dikirim ke Cloud sebagai **Recovery Anchor** — jika iPad hilang atau reset, operator bisa memulihkan session dari Cloud.

### 2.5 Delivery Status

| Arah | Strategi |
|---|---|
| Runtime → Cloud | Upload saat delivery completed atau failed |
| Cloud → Runtime | Tidak ada. Runtime tidak menunggu konfirmasi Cloud untuk delivery. |

---

## 3. Offline Queue

### Prinsip

Semua operasi sync melewati **OfflineEventQueue** — sebuah antrian yang di-persist ke disk.

```
Runtime Operation
    ↓
OfflineEventQueue.enqueue(SyncPayload)
    ↓
Disimpan ke disk (crash-safe)
    ↓
SyncEngine.drain() mencoba upload saat online
    ↓
Berhasil → .uploaded, hapus dari queue
Gagal    → retry dengan backoff
```

### Struktur SyncPayload

```
SyncPayload
    payloadId       UUID — untuk idempotency
    resourceType    session_event | audit | snapshot | delivery
    payload         JSON blob (DomainEventEnvelope atau resource)
    createdAt       timestamp
    attemptCount    berapa kali sudah dicoba
    lastAttemptAt   kapan terakhir dicoba
    status          .pending | .uploading | .uploaded | .failed
```

### Queue Retention

| Status | Retention |
|---|---|
| `.pending` | Sampai berhasil upload atau di-expire |
| `.uploaded` | Hapus setelah 24 jam (confirmed receipt) |
| `.failed` (max retry) | Simpan 7 hari untuk manual recovery |

---

## 4. Retry & Backoff

Semua operasi sync menggunakan **Exponential Backoff dengan Jitter**.

### Formula

```
delay = min(baseDelay * 2^attempt, maxDelay) + jitter
```

### Parameter Default

| Parameter | Value |
|---|---|
| `baseDelay` | 5 detik |
| `maxDelay` | 5 menit |
| `maxAttempts` | 10 kali |
| `jitter` | Random 0–500ms (mencegah thundering herd) |

### Non-Retryable Errors

Error berikut **tidak di-retry** karena retry tidak akan mengubah hasilnya:

| Error | Alasan |
|---|---|
| `AUTHENTICATION_ERROR` | Token tidak valid — butuh re-registration |
| `PERMISSION_DENIED` | Booth tidak memiliki akses resource ini |
| `VALIDATION_ERROR` | Payload tidak valid — butuh perbaikan payload |
| `MANIFEST_OUTDATED` | Booth perlu fetch manifest baru |

---

## 5. Conflict Resolution

### Prinsip

**Runtime adalah source of truth untuk session yang sedang berjalan.**
**Cloud adalah source of truth untuk resource yang sudah selesai (archived).**

Tidak ada dua session yang aktif di dua device untuk tamu yang sama — konflik tidak bisa terjadi secara by-design.

### Kasus Konflik yang Mungkin Terjadi

| Kasus | Resolusi |
|---|---|
| Event dengan `sequenceNumber` yang sama dikirim dua kali | Cloud abaikan yang kedua (idempotency via `eventId`) |
| Snapshot dikirim dua kali | Cloud gunakan `timestamp` terbaru |
| Manifest di-update sementara session berjalan | Runtime tetap gunakan manifest lama — manifest baru untuk session berikutnya |

### Conflict Header

Jika Cloud mendeteksi konflik versi, ia merespons dengan:
```
409 Conflict
X-Conflict-Reason: SEQUENCE_MISMATCH | DUPLICATE_EVENT | STALE_SNAPSHOT
```

Runtime akan log konflik sebagai `SYNC_CONFLICT` event dan tidak retry untuk kasus `DUPLICATE_EVENT`.

---

## 6. Manifest Refresh Strategy

```
App Launch
    ↓
Cek apakah manifest cache masih valid (TTL: 1 jam)
    ↓
Tidak valid → GET /manifest?booth={boothId}
    ↓
Terima manifest versi terbaru
    ↓
Simpan ke disk (manifest cache)
    ↓
Session baru menggunakan manifest ini
```

### Forced Refresh

Manifest **wajib** di-refresh dalam kondisi:
- Booth baru di-register
- Error `MANIFEST_OUTDATED` diterima dari Cloud
- Operator menekan "Refresh Manual" di Mission Control

---

## 7. Monitoring Sync Health

SyncEngine akan mempublikasikan event berikut ke DomainEventPublisher:

| Event | Priority |
|---|---|
| `SyncQueueDrained` | `.low` |
| `SyncUploadFailed(reason:)` | `.normal` |
| `SyncConflictDetected(type:)` | `.critical` |
| `ManifestRefreshed(version:)` | `.normal` |
| `OfflineQueueDepthWarning(count:)` | `.normal` |

MissionControlSubscriber akan meneruskan event ini ke Operator Dashboard.

---

*SYNC_STRATEGY.md v1.0 — Milestone 3 Cloud Contract*
*Ref: constitution/PLATFORM_RUNTIME_V1.md, adr/ADR-011-platform-runtime-freeze.md*
