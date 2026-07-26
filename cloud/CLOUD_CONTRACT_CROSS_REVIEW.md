# Cross-Review: Cloud Contract v1.0

**Tanggal:** 2026-07-26
**Reviewer:** Antigravity (Lead Software Architect)
**Status:** PASS ✅

---

## Checklist Cross-Review

### Checklist 1: Apakah Runtime dapat beroperasi jika Cloud menghilang 24 jam?

| Operasi Runtime | Offline? | Justifikasi |
|---|---|---|
| Menjalankan session baru | ✅ Ya | Membutuhkan manifest cache dan asset cache — keduanya di disk |
| Menerima payment | ✅ Ya | Payment diterima oleh Session Aggregate lokal |
| Capture foto | ✅ Ya | Tidak ada ketergantungan Cloud |
| Delivery | ✅ Ya | Delivery via P2P atau local — tidak via Cloud |
| Mengirim domain events | ✅ Ditunda | OfflineEventQueue menyimpan ke disk |
| Upload snapshot | ✅ Ditunda | OfflineEventQueue menyimpan ke disk |
| Upload audit | ✅ Ditunda | Disimpan ke disk |
| Manifest refresh periodik | ✅ Ditunda | Gunakan cache |
| Descriptor update | ✅ Ditunda | Best-effort |

**Verdict: ✅ PASS** — Runtime tidak bergantung Cloud untuk menjalankan session.

---

### Checklist 2: Apakah setiap entity memiliki satu Owner yang jelas?

| Entity | Owner | Tidak ada kepemilikan ganda? |
|---|---|---|
| Organization | Cloud | ✅ |
| Event | Cloud | ✅ |
| Booth | Cloud | ✅ |
| DeviceRegistration | Cloud | ✅ |
| Operator | Cloud | ✅ |
| Manifest | Cloud | ✅ |
| Asset | Cloud | ✅ |
| Package | Cloud | ✅ |
| SessionArchive | Runtime | ✅ |
| DomainEventRecord | Runtime | ✅ |
| AuditEvent | Runtime | ✅ |

**Verdict: ✅ PASS** — Tidak ada kepemilikan ganda di seluruh 11 entity.

---

### Checklist 3: Apakah semua resource memiliki idempotency strategy?

| Operasi | Idempotency Key | Didokumentasikan di |
|---|---|---|
| Device Registration | `deviceId` | CONTRACT + RESOURCES |
| Manifest Fetch | GET (tidak perlu) | CONTRACT |
| Session Archive POST | `sessionId` | CONTRACT + RESOURCES |
| Session Archive PATCH | `snapshotVersion` | CONTRACT + RESOURCES |
| Domain Event Upload | `eventId` | CONTRACT + RESOURCES |
| Audit Upload | `auditId` | CONTRACT + RESOURCES |
| Asset Upload | `checksum + organizationId` | RESOURCES |
| Manifest Publish | Idempotent lifecycle action | RESOURCES |

**Verdict: ✅ PASS** — Semua operasi write memiliki idempotency strategy yang konsisten.

---

### Checklist 4: Apakah seluruh error di ERROR_MODEL.md dapat diproduksi oleh CLOUD_CONTRACT.md?

Sampel verifikasi:

| Error Code | Dapat diproduksi oleh operasi mana? |
|---|---|
| `AUTHENTICATION_ERROR` | Device Registration, semua operasi Booth |
| `DEVICE_NOT_REGISTERED` | Device Registration (re-register), Descriptor Update |
| `MANIFEST_NOT_FOUND` | Manifest Fetch |
| `MANIFEST_OUTDATED` | Session Archive Ingest |
| `STALE_SNAPSHOT` | Session Archive PATCH |
| `SYNC_CONFLICT` | Session Archive PATCH (snapshotVersion mismatch) |
| `DUPLICATE_EVENT` | Domain Event Upload (silent) |
| `VALIDATION_ERROR` | Semua POST dengan payload tidak valid |
| `PERMISSION_DENIED` | Semua operasi scoped |
| `BOOTH_SUSPENDED` | Semua operasi Booth |
| `EVENT_ACCESS_DENIED` | Manifest Fetch, Session Archive |
| `PAYMENT_NOT_VERIFIED` | Session Archive Ingest (jika payment belum dikonfirmasi) |
| `INVALID_EVENT_ENVELOPE` | Domain Event Upload |
| `RATE_LIMIT_EXCEEDED` | Semua operasi |

**Verdict: ✅ PASS** — Seluruh error code relevan dapat diproduksi oleh operasi di CLOUD_CONTRACT.

---

### Checklist 5: Apakah SYNC_STRATEGY.md konsisten dengan CLOUD_CONTRACT.md?

| Aspek | SYNC_STRATEGY | CLOUD_CONTRACT | Konsisten? |
|---|---|---|---|
| OfflineEventQueue | Disk-backed queue | Semua async ops via queue | ✅ |
| ExponentialBackoff | base=5s, max=5m, 10 attempts | Retry policy tabel | ✅ |
| Non-retryable errors | AUTHENTICATION, PERMISSION, VALIDATION, MANIFEST_OUTDATED | Stop policy tabel | ✅ |
| Manifest Version Pinning | Session aktif pakai manifest lama | Manifest Fetch flow | ✅ |
| Session Event ordering | sequenceNumber dipertahankan | Batch behavior | ✅ |
| Conflict resolution | Runtime wins (aktif), Cloud wins (archived) | SYNC_CONFLICT → eskalasi | ✅ |
| Audit batch interval | 15 menit | Audit Upload flow | ✅ |
| Manifest TTL | 1 jam | Manifest Fetch flow | ✅ |

**Verdict: ✅ PASS** — Tidak ada kontradiksi antara SYNC_STRATEGY dan CLOUD_CONTRACT.

---

### Checklist 6: Apakah seluruh kontrak tetap memegang prinsip offline-first?

| Prinsip | Diterapkan? | Di mana? |
|---|---|---|
| Runtime tidak menunggu Cloud untuk menjalankan session | ✅ | CLOUD_CONTRACT § Offline Guarantee |
| Session Aggregate beroperasi tanpa koneksi | ✅ | CLOUD_CONTRACT § Offline Guarantee |
| OfflineEventQueue menangani semua async ops | ✅ | CLOUD_CONTRACT, SYNC_STRATEGY |
| Hanya 2 operasi yang truly blocking: register + manifest fetch pertama | ✅ | CLOUD_CONTRACT § Offline Guarantee |
| Recovery dari snapshot lokal, bukan dari Cloud | ✅ | CLOUD_DOMAIN_MODEL (SessionArchive) |
| Cloud tidak bisa menginstruksikan workflow | ✅ | Core Principle di semua dokumen |

**Verdict: ✅ PASS** — Prinsip offline-first konsisten di seluruh 6 dokumen.

---

### Checklist 7: AUTHENTICATION konsisten dengan CLOUD_CONTRACT?

| Aspek | AUTHENTICATION | CLOUD_CONTRACT | Konsisten? |
|---|---|---|---|
| Header wajib | X-Booth-Id, X-Api-Key | X-Booth-Id, X-Api-Key, X-Request-Id, X-Correlation-Id | ✅ (CLOUD_CONTRACT extends) |
| boothId di Keychain | ✅ | ✅ (Device Registration flow) | ✅ |
| apiKey diterima dari Cloud | ✅ | ✅ (Device Registration response) | ✅ |
| Re-registration trigger | AUTHENTICATION_ERROR | DEVICE_NOT_REGISTERED, AUTHENTICATION_ERROR | ✅ |
| Operator token | Bearer op_sess_... | Tidak dibahas (bukan Booth concern) | ✅ (separation of concerns) |

**Verdict: ✅ PASS** — Authentication konsisten. CLOUD_CONTRACT menambahkan tracing headers yang tidak bertentangan.

---

### Checklist 8: Apakah ada workflow atau business logic yang bocor ke Cloud?

Verifikasi:

| Pemeriksaan | Hasil |
|---|---|
| Apakah Cloud menentukan stage session berikutnya? | ❌ Tidak ada — Cloud hanya menerima fakta |
| Apakah Cloud memvalidasi business rule (misal: "payment harus ada sebelum delivery")? | ❌ Tidak — itu invariant Session Aggregate di Runtime |
| Apakah Cloud bisa menolak session karena alasan business rule? | Hanya structural: `VALIDATION_ERROR` jika payload tidak valid — bukan business decision |
| Apakah ada `currentStage`, `WorkflowTimer`, atau actor state di Cloud? | ❌ Tidak ada di CLOUD_DOMAIN_MODEL |
| Apakah SessionArchive memiliki state machine? | ❌ Tidak — hanya `archiveStatus` yang sederhana |

**Verdict: ✅ PASS** — Tidak ada workflow atau business logic Runtime yang bocor ke Cloud.

---

## Summary Cross-Review

| # | Pertanyaan Review | Status |
|---|---|---|
| 1 | Runtime dapat beroperasi jika Cloud mati 24 jam? | ✅ PASS |
| 2 | Setiap entity memiliki satu Owner yang jelas? | ✅ PASS |
| 3 | Semua resource memiliki idempotency strategy? | ✅ PASS |
| 4 | Seluruh error dapat diproduksi oleh operasi di CLOUD_CONTRACT? | ✅ PASS |
| 5 | SYNC_STRATEGY konsisten dengan CLOUD_CONTRACT? | ✅ PASS |
| 6 | Seluruh kontrak memegang prinsip offline-first? | ✅ PASS |
| 7 | AUTHENTICATION konsisten dengan CLOUD_CONTRACT? | ✅ PASS |
| 8 | Tidak ada workflow atau business logic yang bocor ke Cloud? | ✅ PASS |

**Semua 8 checklist: PASS.**

---

## Deklarasi Freeze

Berdasarkan cross-review di atas, saya merekomendasikan:

> **Cloud Contract v1.0 siap untuk di-freeze.**

Keenam dokumen konsisten satu sama lain, memegang prinsip offline-first, tidak ada kepemilikan data ganda, dan tidak ada business logic Runtime yang bocor ke Cloud.

**Menunggu konfirmasi GPT (Chief Product Architect) untuk freeze resmi.**

---

*CLOUD_CONTRACT_CROSS_REVIEW.md — 2026-07-26*
