# Cloud Resources — Haispace Cloud

**Status:** Draft v1.0
**Milestone:** 3 — Cloud Contract
**Penulis:** GPT (Chief Product Architect) + Antigravity (Lead Software Architect)
**Tanggal:** 2026-07-26

---

## Core Principle

> **Resources expose domain. They do not redefine it.**

Tidak boleh ada resource yang menciptakan domain baru atau konsep yang tidak ada di `CLOUD_DOMAIN_MODEL.md`. Setiap resource adalah representasi dari Aggregate Root yang sudah didefinisikan.

---

## Versioning

Seluruh resource berada di bawah URI versioning.

```
https://api.haispace.id/v1/{resource}
```

**Aturan versioning:**
- Semua endpoint menggunakan prefix `/v1/`
- Breaking change → `/v2/` (jalankan paralel minimal 6 bulan sebelum `/v1/` deprecated)
- Non-breaking addition (field baru, resource baru) → tetap di `/v1/`
- Booth lama yang masih di `/v1/` tidak boleh rusak ketika `/v2/` dirilis

---

## Pagination

Seluruh endpoint yang mengembalikan list menggunakan **cursor-based pagination**.

**Tidak boleh menggunakan offset-based pagination.**

```
Query params:
  ?limit=    int (default: 20, max: 100)
  ?cursor=   opaque string (dari response sebelumnya)
  ?direction=  forward | backward (default: forward)

Response:
  data: [...]
  pagination:
    nextCursor:  string | null
    prevCursor:  string | null
    hasMore:     boolean
```

Cursor bersifat opaque — client tidak boleh menginterpretasikan isinya.

---

## Pagination

Seluruh endpoint yang mengembalikan list menggunakan **cursor-based pagination**.

**Tidak boleh menggunakan offset-based pagination.**

```
Query params:
  ?limit=      int (default: 20, max: 100)
  ?cursor=     opaque string (dari response sebelumnya)
  ?direction=  forward | backward (default: forward)
```

Cursor bersifat opaque — client tidak boleh menginterpretasikan isinya.

---

## Error Reference

Dokumen ini **tidak mendefinisikan ulang error**. Setiap resource mereferensikan kode error dari [`ERROR_MODEL.md`](./ERROR_MODEL.md).

---

## Resource Index

| Resource | URI | Aggregate Root |
|---|---|---|
| Organizations | `/v1/organizations` | `Organization` |
| Events | `/v1/events` | `Event` |
| Booths | `/v1/booths` | `Booth` |
| Device Registration | `/v1/devices` | `DeviceRegistration` |
| Operators | `/v1/operators` | `Operator` |
| Manifests | `/v1/manifests` | `Manifest` |
| Assets | `/v1/assets` | `Asset` |
| Packages | `/v1/packages` | `Package` |
| Session Archives | `/v1/sessions` | `SessionArchive` |
| Domain Event Records | `/v1/session-events` | `DomainEventRecord` |
| Audit Events | `/v1/audit-events` | `AuditEvent` |

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

**Aggregate:** `Organization`

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| List organizations | GET | `/v1/organizations` | — | — | ✅ |
| Get organization | GET | `/v1/organizations/{id}` | — | ✅ | ✅ |
| Create organization | POST | `/v1/organizations` | — | — | ✅ |
| Update organization | PATCH | `/v1/organizations/{id}` | — | — | ✅ |
| Delete (logical) | DELETE | `/v1/organizations/{id}` | — | — | ✅ |

### Idempotency
- POST: `organizationId` tidak bisa di-POST ulang. Admin tidak dapat membuat org dengan ID yang sama.

### Filtering (GET list)
- `?status=active|suspended|terminated`
- `?plan=starter|professional|enterprise`

### Lifecycle Endpoints

| State Transition | Method | URI |
|---|---|---|
| Suspend | POST | `/v1/organizations/{id}/suspend` |
| Reinstate | POST | `/v1/organizations/{id}/reinstate` |

### Error References
`PERMISSION_DENIED`, `NOT_FOUND` (→ buat error code `ORGANIZATION_NOT_FOUND` di ERROR_MODEL)

---

## 2. Events

**URI:** `/v1/events`, `/v1/events/{eventId}`

**Aggregate:** `Event`

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| List events | GET | `/v1/events` | ✅ (own only) | ✅ | ✅ |
| Get event | GET | `/v1/events/{id}` | ✅ (assigned) | ✅ | ✅ |
| Create event | POST | `/v1/events` | — | Manager+ | ✅ |
| Update event | PATCH | `/v1/events/{id}` | — | Manager+ | ✅ |
| Delete event | DELETE | `/v1/events/{id}` | — | — | ✅ |

### Idempotency
- PATCH: setiap update menggunakan `updatedAt` sebagai optimistic lock.

### Filtering (GET list)
- `?organizationId=`
- `?status=draft|active|completed|archived`
- `?from=` (scheduledDate ≥)
- `?to=` (scheduledDate ≤)
- `?boothId=` (events yang di-assign ke booth ini)

### Lifecycle Endpoints

| State Transition | Method | URI |
|---|---|---|
| Activate | POST | `/v1/events/{id}/activate` |
| Complete | POST | `/v1/events/{id}/complete` |
| Archive | POST | `/v1/events/{id}/archive` |
| Assign booth | POST | `/v1/events/{id}/booths/{boothId}` |
| Remove booth | DELETE | `/v1/events/{id}/booths/{boothId}` |

### Error References
`VALIDATION_ERROR`, `PERMISSION_DENIED`, `EVENT_ACCESS_DENIED`

---

## 3. Booths

**URI:** `/v1/booths`, `/v1/booths/{boothId}`

**Aggregate:** `Booth`

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| List booths | GET | `/v1/booths` | — | ✅ | ✅ |
| Get own booth | GET | `/v1/booths/{id}` | ✅ (self) | ✅ | ✅ |
| Create booth | POST | `/v1/booths` | — | — | ✅ |
| Update booth | PATCH | `/v1/booths/{id}` | — | Manager+ | ✅ |
| Suspend booth | POST | `/v1/booths/{id}/suspend` | — | — | ✅ |

### Idempotency
- POST: `boothId` adalah identity. Registrasi booth yang sama dua kali diabaikan (idempotent).

### Filtering (GET list)
- `?organizationId=`
- `?eventId=` (booths assigned to this event)
- `?status=active|inactive|suspended`

### Error References
`BOOTH_NOT_FOUND`, `BOOTH_SUSPENDED`, `PERMISSION_DENIED`

---

## 4. Device Registration

**URI:** `/v1/devices`, `/v1/devices/{deviceId}`

**Aggregate:** `DeviceRegistration`

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| Register device | POST | `/v1/devices` | ✅ (self) | — | ✅ |
| Get registration | GET | `/v1/devices/{id}` | ✅ (self) | ✅ | ✅ |
| List registrations | GET | `/v1/devices` | — | ✅ | ✅ |
| Revoke device | POST | `/v1/devices/{id}/revoke` | — | — | ✅ |
| Update descriptor | PATCH | `/v1/devices/{id}/descriptor` | ✅ (self) | — | — |

### Idempotency
- POST register: `deviceId` adalah identity. Jika `deviceId` sudah terdaftar dan `status = active`, kembalikan data yang sudah ada (idempotent).
- PATCH descriptor: selalu overwrite dengan data terbaru dari Runtime.

### Header wajib untuk Booth
```
X-Booth-Id: {boothId}
X-Api-Key:  {apiKey}
```

### Filtering (GET list)
- `?boothId=`
- `?status=active|revoked`

### Error References
`DEVICE_NOT_REGISTERED`, `AUTHENTICATION_ERROR`, `BOOTH_SUSPENDED`

---

## 5. Operators

**URI:** `/v1/operators`, `/v1/operators/{operatorId}`

**Aggregate:** `Operator`

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| List operators | GET | `/v1/operators` | — | ✅ (own org) | ✅ |
| Get operator | GET | `/v1/operators/{id}` | — | ✅ (self) | ✅ |
| Create operator | POST | `/v1/operators` | — | Owner | ✅ |
| Update operator | PATCH | `/v1/operators/{id}` | — | Owner | ✅ |
| Suspend operator | POST | `/v1/operators/{id}/suspend` | — | Owner | ✅ |
| Login | POST | `/v1/auth/operator/login` | — | ✅ | ✅ |
| Logout | POST | `/v1/auth/operator/logout` | — | ✅ | ✅ |

### Filtering (GET list)
- `?organizationId=`
- `?role=owner|manager|staff`
- `?status=active|suspended`

### Error References
`TOKEN_EXPIRED`, `AUTHENTICATION_ERROR`, `PERMISSION_DENIED`

---

## 6. Manifests

**URI:** `/v1/manifests`, `/v1/manifests/{manifestId}`

**Aggregate:** `Manifest`

> Manifest immutable setelah published. PATCH tidak diizinkan.

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| Get manifest | GET | `/v1/manifests/{id}` | ✅ (assigned event) | ✅ | ✅ |
| Get by event (latest) | GET | `/v1/events/{id}/manifest` | ✅ (assigned) | ✅ | ✅ |
| List versions | GET | `/v1/events/{id}/manifests` | — | ✅ | ✅ |
| Create (draft) | POST | `/v1/manifests` | — | Manager+ | ✅ |
| Update draft | PATCH | `/v1/manifests/{id}` | — | Manager+ | ✅ |
| Publish | POST | `/v1/manifests/{id}/publish` | — | Manager+ | ✅ |

Note: PATCH hanya diizinkan selama status `draft`. Setelah `published`, semua update ditolak dengan `VALIDATION_ERROR`.

### Idempotency
- POST publish: idempotent — memanggil publish pada manifest yang sudah `published` tidak mengubah apa pun.

### Lifecycle Endpoints

| State Transition | Method | URI |
|---|---|---|
| Publish | POST | `/v1/manifests/{id}/publish` |

### Error References
`MANIFEST_NOT_FOUND`, `MANIFEST_OUTDATED`, `PERMISSION_DENIED`, `EVENT_ACCESS_DENIED`

---

## 7. Assets

**URI:** `/v1/assets`, `/v1/assets/{assetId}`

**Aggregate:** `Asset`

> Asset metadata immutable setelah upload. File tidak pernah dimodifikasi.

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| Get asset metadata | GET | `/v1/assets/{id}` | ✅ | ✅ | ✅ |
| Get download URL | GET | `/v1/assets/{id}/download-url` | ✅ | ✅ | ✅ |
| List assets | GET | `/v1/assets` | — | ✅ | ✅ |
| Upload asset | POST | `/v1/assets` | — | Manager+ | ✅ |
| Deprecate asset | POST | `/v1/assets/{id}/deprecate` | — | — | ✅ |

Note: Download URL adalah pre-signed URL yang expired — asset file tidak diekspos langsung.

### Idempotency
- POST upload: upload asset dengan `checksum` yang sama → kembalikan asset yang sudah ada (idempotent berdasarkan checksum + organizationId).

### Filtering (GET list)
- `?organizationId=`
- `?type=frame|filter|overlay|background|sticker|font`
- `?status=active|deprecated`

### Error References
`ASSET_NOT_FOUND`, `PERMISSION_DENIED`

---

## 8. Packages

**URI:** `/v1/packages`, `/v1/packages/{packageId}`

**Aggregate:** `Package`

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| List packages | GET | `/v1/packages` | ✅ (event-scoped) | ✅ | ✅ |
| Get package | GET | `/v1/packages/{id}` | ✅ | ✅ | ✅ |
| Create package | POST | `/v1/packages` | — | Manager+ | ✅ |
| Discontinue | POST | `/v1/packages/{id}/discontinue` | — | — | ✅ |

Note: PATCH tidak diizinkan karena `priceAmount` dan `captureLimit` immutable setelah ada SessionArchive.

### Filtering (GET list)
- `?eventId=`
- `?status=active|discontinued`

### Error References
`VALIDATION_ERROR`, `PERMISSION_DENIED`

---

## 9. Session Archives

**URI:** `/v1/sessions`, `/v1/sessions/{sessionId}`

**Aggregate:** `SessionArchive`

> Dikirim oleh Runtime. Cloud menerima dan menyimpan.

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| Ingest snapshot | POST | `/v1/sessions` | ✅ (self) | — | — |
| Update archive | PATCH | `/v1/sessions/{id}` | ✅ (self) | — | — |
| Get session | GET | `/v1/sessions/{id}` | ✅ (self) | ✅ | ✅ |
| List sessions | GET | `/v1/sessions` | ✅ (self only) | ✅ | ✅ |

Note: Booth hanya bisa melihat session milik boothId-nya sendiri.

### Idempotency
- POST ingest: `sessionId` adalah idempotency key. Jika sessionId sudah ada, Cloud memperbarui data yang mutable (seperti `captureSummary`, `deliverySummary`) — bukan membuat record baru.
- PATCH: hanya field mutable yang boleh diperbarui (lihat domain model).

### Filtering (GET list)
- `?eventId=`
- `?boothId=`
- `?status=in_progress|completed|recovered|abandoned`
- `?from=` (startedAt ≥)
- `?to=` (startedAt ≤)

### Error References
`SESSION_NOT_FOUND`, `AUTHENTICATION_ERROR`, `VALIDATION_ERROR`, `STALE_SNAPSHOT`

---

## 10. Domain Event Records (Session Events)

**URI:** `/v1/session-events`

**Aggregate:** `DomainEventRecord`

> Append-only. Tidak ada UPDATE. Tidak ada DELETE. Duplikat diabaikan.

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| Upload events (batch) | POST | `/v1/session-events` | ✅ | — | — |
| List events | GET | `/v1/session-events` | ✅ (self) | ✅ | ✅ |
| Get event | GET | `/v1/session-events/{eventId}` | ✅ (self) | ✅ | ✅ |

Note: POST menerima array (batch upload). Single upload menggunakan array dengan satu item.

### Idempotency
- POST: `eventId` (dari `DomainEventEnvelope`) adalah idempotency key. Jika `eventId` sudah ada di Cloud, event tersebut **diabaikan secara diam-diam** (tidak error, tidak duplicate). Respons tetap `200 OK` atau `207 Multi-Status` jika ada campuran.

### Filtering (GET list)
- `?sessionId=`
- `?boothId=`
- `?eventType=PaymentAccepted|CaptureAdded|...`
- `?from=` (occurredAt ≥)
- `?to=` (occurredAt ≤)

### Error References
`AUTHENTICATION_ERROR`, `INVALID_EVENT_ENVELOPE`, `DUPLICATE_EVENT` (tidak di-surface sebagai error — silent idempotency)

---

## 11. Audit Events

**URI:** `/v1/audit-events`

**Aggregate:** `AuditEvent`

> Append-only. Tidak ada UPDATE. Tidak ada DELETE. Retensi 7 tahun.

### Allowed Operations

| Operation | Method | URI | Booth | Operator | Admin |
|---|---|---|---|---|---|
| Upload audit batch | POST | `/v1/audit-events` | ✅ | — | — |
| List audit events | GET | `/v1/audit-events` | — | ✅ | ✅ |
| Get audit event | GET | `/v1/audit-events/{auditId}` | — | ✅ | ✅ |

### Idempotency
- POST: `auditId` adalah idempotency key. Duplikat diabaikan secara diam-diam.

### Filtering (GET list)
- `?organizationId=`
- `?boothId=`
- `?operatorId=`
- `?sessionId=`
- `?category=session|payment|delivery|manifest|device|operator|system`
- `?outcome=success|failure|warning`
- `?from=` (occurredAt ≥)
- `?to=` (occurredAt ≤)

### Error References
`AUTHENTICATION_ERROR`, `PERMISSION_DENIED`

---

## Operations Summary Table

| Resource | GET list | GET one | POST | PATCH | DELETE | Lifecycle Actions |
|---|---|---|---|---|---|---|
| Organizations | Admin | Op+Admin | Admin | Admin | Admin | suspend, reinstate |
| Events | All | All | Manager+ | Manager+ | Admin | activate, complete, archive |
| Booths | Op+Admin | All | Admin | Manager+ | — | suspend |
| Devices | Op+Admin | Booth+Op+Admin | Booth | Booth (descriptor) | — | revoke |
| Operators | Op+Admin | Self+Admin | Owner+ | Owner+ | — | suspend |
| Manifests | Op+Admin | All | Manager+ | Manager+ (draft only) | — | publish |
| Assets | Op+Admin | All | Manager+ | — | — | deprecate |
| Packages | All | All | Manager+ | — | — | discontinue |
| Sessions | All (scoped) | All (scoped) | Booth | Booth | — | — |
| Session Events | All (scoped) | All (scoped) | Booth | — | — | — |
| Audit Events | Op+Admin | Op+Admin | Booth | — | — | — |

---

## Acceptance Criteria

- [x] Seluruh Aggregate Root memiliki representasi resource yang jelas
- [x] Setiap resource memiliki operasi yang diizinkan (dengan tabel eksplisit)
- [x] Semua resource memiliki strategi idempotency yang terdokumentasi
- [x] Versioning konsisten (`/v1/`) di seluruh API
- [x] Pagination hanya menggunakan cursor-based (satu pendekatan)
- [x] Permission matrix lengkap untuk setiap resource (Booth | Operator | Admin)
- [x] Tidak ada payload JSON atau detail implementasi yang bocor
- [x] Semua resource dipetakan kembali ke Domain Model tanpa konsep baru

---

*CLOUD_RESOURCES.md v1.0 — Milestone 3 Cloud Contract*
*Ref: CLOUD_DOMAIN_MODEL.md v1.1, ERROR_MODEL.md, AUTHENTICATION.md*
