# Backend Implementation Plan — Haispace Cloud

**Status:** Active
**Versi:** 1.0
**Tanggal:** 2026-07-26
**Kontrak Acuan:** Cloud Contract v1.0 (ADR-012)

---

## Prinsip Eksekusi

Dokumen ini adalah rencana **eksekusi**, bukan perubahan desain. Semua keputusan arsitektur sudah terkunci di Cloud Contract v1.0.

Tiga aturan selama implementasi:

1. **Ikuti kontrak** — setiap endpoint harus konsisten dengan `CLOUD_RESOURCES.md` dan `CLOUD_CONTRACT.md`
2. **Jangan tambah business logic di Cloud** — Cloud stores facts, Runtime executes behavior
3. **Jika butuh mengubah kontrak** — buat ADR baru, review dulu, baru implementasi

---

## Urutan Phase

```
Phase 1 — Infrastructure Foundation
    ↓
Phase 2 — Content Delivery
    ↓
Phase 3 — Runtime Data Ingest
    ↓
Phase 4 — Operations & Analytics
```

---

## Phase 1 — Infrastructure Foundation

**Target:** Backend siap menerima Booth pertama kali.
**Prioritas:** Tertinggi — tanpa Phase 1, tidak ada yang bisa berjalan.

### 1.1 Device Registration

**Endpoints:**
- `POST /v1/devices` — register booth
- `GET /v1/devices/{deviceId}` — get registration
- `GET /v1/devices` — list (Admin/Operator)
- `PATCH /v1/devices/{deviceId}/descriptor` — update RuntimeDescriptor
- `POST /v1/devices/{deviceId}/revoke` — revoke device (Admin)

**Acceptance Tests:**
- [ ] Booth baru dapat register dan menerima `apiKey`
- [ ] Register dengan `deviceId` yang sama → kembalikan data existing (idempotent)
- [ ] Booth yang di-revoke tidak bisa melakukan request lain
- [ ] `lastKnownDescriptor` terupdate setelah PATCH descriptor

### 1.2 Authentication

**Endpoints:**
- `POST /v1/auth/operator/login`
- `POST /v1/auth/operator/logout`

**Acceptance Tests:**
- [ ] Operator dapat login dan mendapat session token (TTL 8 jam)
- [ ] Token expired → `401 TOKEN_EXPIRED`
- [ ] Semua Booth request tanpa `X-Api-Key` yang valid → `401 AUTHENTICATION_ERROR`
- [ ] Booth suspended → `403 BOOTH_SUSPENDED` untuk semua request

### 1.3 Organization & Operator Management

**Endpoints:**
- `POST /v1/organizations`
- `GET /v1/organizations/{id}`
- `POST /v1/operators`
- `GET /v1/operators`, `GET /v1/operators/{id}`
- `PATCH /v1/operators/{id}`
- `POST /v1/operators/{id}/suspend`

**Acceptance Tests:**
- [ ] Org baru dapat dibuat dengan minimal satu operator `role=owner`
- [ ] Owner tidak bisa di-suspend jika tidak ada owner lain
- [ ] Operator hanya bisa melihat data org sendiri

### 1.4 Booth Management

**Endpoints:**
- `POST /v1/booths`
- `GET /v1/booths`, `GET /v1/booths/{id}`
- `PATCH /v1/booths/{id}`
- `POST /v1/booths/{id}/suspend`

**Acceptance Tests:**
- [ ] Booth dibuat oleh Admin
- [ ] Booth suspended tidak bisa register device baru
- [ ] `boothId` permanen — tidak bisa diubah via PATCH

---

## Phase 2 — Content Delivery

**Target:** Booth dapat mengambil manifest dan asset untuk memulai session.
**Prioritas:** Tinggi — tanpa manifest dan asset, session tidak bisa dimulai.

### 2.1 Event Management

**Endpoints:**
- `POST /v1/events`
- `GET /v1/events`, `GET /v1/events/{id}`
- `PATCH /v1/events/{id}`
- `POST /v1/events/{id}/activate`
- `POST /v1/events/{id}/complete`
- `POST /v1/events/{id}/archive`
- `POST /v1/events/{id}/booths/{boothId}` — assign booth
- `DELETE /v1/events/{id}/booths/{boothId}` — remove booth

**Acceptance Tests:**
- [ ] Event `draft` → Booth tidak bisa akses manifest
- [ ] Event `active` → Booth yang di-assign bisa fetch manifest
- [ ] Booth yang tidak di-assign → `403 EVENT_ACCESS_DENIED`
- [ ] Event `status` hanya bisa maju (draft→active→completed→archived)

### 2.2 Manifest API

**Endpoints:**
- `POST /v1/manifests` — create draft
- `GET /v1/manifests/{id}`
- `GET /v1/events/{id}/manifest` — get published manifest for event
- `GET /v1/events/{id}/manifests` — list versions
- `PATCH /v1/manifests/{id}` — update draft
- `POST /v1/manifests/{id}/publish` — publish (long-running, 202 Accepted)

**Acceptance Tests:**
- [ ] Manifest `draft` dapat diubah; setelah `published` semua PATCH ditolak
- [ ] Satu event hanya boleh punya satu manifest `published` pada satu waktu
- [ ] `version` selalu increment per event
- [ ] Publish manifest lama yang sudah `published` → idempotent (tidak error)
- [ ] Booth fetch manifest event yang di-assign → dapat manifest `published` terbaru
- [ ] `manifestVersion` di response harus cocok dengan yang tersimpan

### 2.3 Asset API

**Endpoints:**
- `POST /v1/assets` — upload (202 Accepted, long-running)
- `GET /v1/assets`, `GET /v1/assets/{id}`
- `GET /v1/assets/{id}/download-url` — pre-signed URL
- `POST /v1/assets/{id}/deprecate`

**Acceptance Tests:**
- [ ] Upload asset → `202 Accepted` + `jobId`
- [ ] Polling `GET /v1/jobs/{jobId}` → status `processing → completed`
- [ ] Download URL expired setelah 15 menit
- [ ] Asset dengan `checksum` yang sama tidak di-upload ulang (idempotent)
- [ ] Asset `status=referenced` tidak bisa di-deprecate

### 2.4 Package API

**Endpoints:**
- `POST /v1/packages`
- `GET /v1/packages`, `GET /v1/packages/{id}`
- `POST /v1/packages/{id}/discontinue`

**Acceptance Tests:**
- [ ] Package `discontinued` tidak bisa dipakai di session baru
- [ ] `priceAmount` dan `captureLimit` immutable — PATCH tidak ada
- [ ] Booth dapat list packages untuk event yang di-assign

---

## Phase 3 — Runtime Data Ingest

**Target:** Runtime dapat mengirim data ke Cloud. Cloud menjadi mirror fakta Runtime.
**Prioritas:** Tinggi — ini adalah inti Cloud Contract.

### 3.1 Session Archive API

**Endpoints:**
- `POST /v1/sessions` — ingest snapshot (create atau update idempotent)
- `PATCH /v1/sessions/{id}` — update mutable fields
- `GET /v1/sessions`, `GET /v1/sessions/{id}`

**Acceptance Tests:**
- [ ] POST dengan `sessionId` baru → create `SessionArchive` baru
- [ ] POST dengan `sessionId` yang sama → update field mutable (idempotent)
- [ ] PATCH dengan `snapshotVersion` lama → `409 STALE_SNAPSHOT`
- [ ] `payment` immutable setelah pertama kali di-ingest
- [ ] Booth hanya bisa melihat session milik `boothId`-nya sendiri
- [ ] `completedAt >= startedAt` divalidasi

### 3.2 Domain Event Upload

**Endpoints:**
- `POST /v1/session-events` — batch upload
- `GET /v1/session-events`, `GET /v1/session-events/{eventId}`

**Acceptance Tests:**
- [ ] Batch upload diterima → `200 OK` atau `207 Multi-Status`
- [ ] `eventId` yang sama dikirim dua kali → silent ignore, tidak `409`
- [ ] `sequenceNumber` urutan dipertahankan sesuai urutan array
- [ ] `DomainEventRecord` tidak bisa di-update atau di-delete
- [ ] Event tanpa `sessionId` yang valid di Cloud → `VALIDATION_ERROR`

### 3.3 Audit Upload

**Endpoints:**
- `POST /v1/audit-events` — batch upload
- `GET /v1/audit-events`, `GET /v1/audit-events/{auditId}`

**Acceptance Tests:**
- [ ] Batch audit diterima
- [ ] `auditId` yang sama dikirim dua kali → silent ignore
- [ ] Operator dan Admin dapat membaca; Booth tidak dapat membaca
- [ ] Audit record tidak pernah bisa di-update atau di-delete

---

## Phase 4 — Operations & Analytics

**Target:** Operator dapat memantau dan mengelola platform via Mission Control.
**Prioritas:** Medium — Runtime sudah bisa berjalan tanpa Phase 4.

### 4.1 Operator Dashboard Data

Read-only endpoints untuk Mission Control:
- `GET /v1/sessions` dengan filtering lengkap
- `GET /v1/session-events` dengan filtering
- `GET /v1/audit-events` dengan filtering
- `GET /v1/devices` — booth health status

**Acceptance Tests:**
- [ ] Operator dapat melihat semua session untuk event yang mereka kelola
- [ ] Filter by `status`, `from`, `to`, `boothId` berfungsi
- [ ] Pagination cursor-based berfungsi di semua endpoint

### 4.2 Analytics Projections

Projection (bukan source of truth) — dibangun dari data Phase 3:

| Projection | Dibangun Dari |
|---|---|
| `DailySessionSummary` | `SessionArchive` + `DomainEventRecord` |
| `RevenueByEvent` | `PaymentRecord` dalam `SessionArchive` |
| `DeliverySuccessRate` | `DeliverySummary` dalam `SessionArchive` |
| `BoothHealthSummary` | `DeviceRegistration.lastSeenAt` + `AuditEvent` |

**Acceptance Tests:**
- [ ] Projection dapat di-regenerate dari data domain tanpa kehilangan informasi
- [ ] Projection tidak menjadi source of truth (tidak ada write ke projection)

---

## Milestone Summary

| Phase | Deliverable | Acceptance |
|---|---|---|
| **Phase 1** | Device Registration + Auth + Org + Booth | Booth pertama bisa register dan berautentikasi |
| **Phase 2** | Event + Manifest + Asset + Package | Booth bisa fetch manifest dan download asset |
| **Phase 3** | Session Archive + Domain Events + Audit | Runtime bisa mengirim fakta ke Cloud |
| **Phase 4** | Dashboard data + Analytics | Operator bisa memantau platform |

---

## Definition of Done (Keseluruhan)

Backend dianggap siap untuk integrasi Runtime ketika:

1. Booth nyata (iPad dev) dapat register ke backend
2. Booth dapat fetch manifest untuk event test
3. Booth dapat download asset dari backend
4. Runtime dapat upload SessionSnapshot setelah payment
5. Runtime dapat upload DomainEvent batch
6. Audit trail dapat di-upload dan dibaca oleh Operator
7. Semua error code di `ERROR_MODEL.md` dapat diproduksi oleh backend (bukan stub)
8. Seluruh idempotency behavior terverifikasi dengan automated test

---

## Catatan Penting

**Yang tidak boleh ada di backend:**
- Business logic tentang "apakah session boleh selesai" — itu urusan Runtime
- State machine workflow — Cloud tidak tahu WorkflowStage
- Decision making berdasarkan isi payment — Cloud hanya menyimpan fakta

**Yang wajib ada di setiap PR backend:**
- Test untuk idempotency behavior
- Test untuk error code yang relevan
- Tidak ada perubahan domain tanpa ADR baru

---

*BACKEND_IMPLEMENTATION_PLAN.md v1.0*
*Ref: ADR-012, CLOUD_CONTRACT.md v1.0, CLOUD_RESOURCES.md v1.1*
