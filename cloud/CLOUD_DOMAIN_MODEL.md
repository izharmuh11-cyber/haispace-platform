# Cloud Domain Model — Haispace Cloud

**Status:** Final ✅
**Versi:** 1.1 (frozen after this review)
**Milestone:** 3 — Cloud Contract
**Penulis:** GPT (Chief Product Architect) + Antigravity (Lead Software Architect)
**Tanggal:** 2026-07-26

---

## Core Principle

> **Cloud stores facts. Runtime executes behavior.**

- Cloud tidak memiliki state machine booth
- Cloud tidak pernah menentukan langkah berikutnya dalam sebuah session
- Cloud hanya mengetahui fakta yang dikirim oleh Runtime
- Jika Cloud mati selama 24 jam, Runtime tetap berjalan normal

---

## SessionSnapshot vs SessionArchive

Dua konsep ini harus dibedakan secara eksplisit.

| Konsep | Lapisan | Pemilik | Tujuan |
|---|---|---|---|
| `SessionSnapshot` | **Transport Contract** | Runtime | Payload teknis yang dikirim Runtime ke Cloud |
| `SessionArchive` | **Domain Entity** | Cloud | Model domain hasil ingest dari Snapshot |

```
Runtime (iPad)
    │
    │  SessionSnapshot
    │  (transport payload — format teknis)
    ▼
Cloud Ingestion Layer
    │
    │  validasi, transform, enrich
    ▼
SessionArchive
    (domain entity — bahasa bisnis Cloud)
```

**Implikasi:**
- Cloud **tidak menyimpan** SessionSnapshot sebagai domain utama
- Jika format SessionSnapshot berubah (schema bump), Cloud Ingestion Layer yang mengabsorb perubahan itu — bukan SessionArchive
- SessionArchive adalah kontrak domain yang stabil; SessionSnapshot adalah kontrak teknis yang boleh berkembang

---

## Identity Rules

> **Identity tidak pernah berubah selama lifecycle entity.**

| Aggregate | Identity Key | Di-generate oleh | Format |
|---|---|---|---|
| `Organization` | `organizationId` | Cloud (Admin) | UUID v4 |
| `Event` | `eventId` | Cloud (Admin) | UUID v4 |
| `Booth` | `boothId` | Cloud (Admin) | UUID v4 |
| `DeviceRegistration` | `deviceId` | Runtime (self-register) | UUID v4 |
| `Operator` | `operatorId` | Cloud (Admin) | UUID v4 |
| `Manifest` | `manifestId` | Cloud (Admin) | UUID v4 |
| `Asset` | `assetId` | Cloud (Admin) | UUID v4 |
| `Package` | `packageId` | Cloud (Admin) | UUID v4 |
| `SessionArchive` | `sessionId` | **Runtime** | UUID v4 (dari Aggregate) |
| `DomainEventRecord` | `eventId` | **Runtime** | UUID v4 (dari Envelope) |
| `AuditEvent` | `auditId` | Cloud | UUID v4 |

**Catatan penting:** `sessionId` dan `eventId` di-generate oleh Runtime dan diterima Cloud apa adanya. Cloud **tidak pernah** mengubah atau mengganti identity yang berasal dari Runtime.

---

## Referential Integrity Rules

| Rule | Keterangan |
|---|---|
| `Booth` ∈ tepat satu `Organization` | Booth tidak bisa berpindah organisasi |
| `Event` ∈ tepat satu `Organization` | Event tidak bisa berpindah organisasi |
| `Package` ∈ tepat satu `Event` | Package tidak shared antar event |
| `Manifest` ∈ tepat satu `Event` | Manifest tidak shared antar event |
| `Asset` ∈ banyak `Manifest` | Asset bisa dipakai di banyak manifest |
| `DeviceRegistration` ∈ tepat satu `Booth` | Device tidak bisa dipakai dua booth bersamaan |
| `SessionArchive.manifestVersion` → harus ada di `Manifest` yang valid | Tidak boleh archive session dengan manifest yang tidak dikenal |
| `SessionArchive.boothId` → harus `Booth` yang terdaftar | Tidak boleh archive dari booth yang tidak dikenal |
| `SessionArchive.eventId` → harus `Event` yang `active` atau `completed` | Tidak boleh archive ke event yang masih `draft` |
| `DomainEventRecord.sessionId` → harus ada di `SessionArchive` | Event tanpa session diabaikan |
| `AuditEvent.boothId` (nullable) → jika ada, harus booth yang terdaftar | |

---

## Ownership Table

> Tidak boleh ada dua Source of Truth untuk satu entitas.

| Entity | Owner | Writable By | Readable By | Source of Truth |
|---|---|---|---|---|
| `Organization` | Cloud | Admin | Cloud | Cloud |
| `Event` | Cloud | Admin | Runtime + Cloud | Cloud |
| `Booth` | Cloud | Admin | Runtime + Cloud | Cloud |
| `DeviceRegistration` | Cloud | Runtime (self) + Admin | Runtime + Cloud | Cloud |
| `Operator` | Cloud | Admin | Cloud | Cloud |
| `Manifest` | Cloud | Admin | Runtime | Cloud |
| `Asset` | Cloud | Admin | Runtime | Cloud |
| `Package` | Cloud | Admin | Runtime | Cloud |
| `SessionArchive` | **Runtime** | Runtime | Cloud | **Runtime** |
| `DomainEventRecord` | **Runtime** | Runtime | Cloud | **Runtime** |
| `AuditEvent` | **Runtime** | Runtime | Cloud | **Runtime** |

---

## Aggregate Detail

---

### 1. Organization

**Fields:**

| Field | Type | Mutability |
|---|---|---|
| `organizationId` | UUID | **Immutable** |
| `createdAt` | Timestamp | **Immutable** |
| `name` | String | Mutable |
| `plan` | Enum (starter\|professional\|enterprise) | Mutable |
| `status` | Enum (active\|suspended\|terminated) | Mutable |

**Lifecycle:**
```
Created → Active → Suspended → Terminated
```
Tidak pernah dihapus secara fisik.

**Invariants:**
- `organizationId` unik di seluruh sistem
- `status = terminated` bersifat final — tidak bisa kembali ke `active`
- Minimal satu Operator harus ada dengan `role = owner`

---

### 2. Event

**Fields:**

| Field | Type | Mutability |
|---|---|---|
| `eventId` | UUID | **Immutable** |
| `organizationId` | UUID | **Immutable** |
| `createdAt` | Timestamp | **Immutable** |
| `name` | String | Mutable (saat `draft`) |
| `venue` | String | Mutable (saat `draft`) |
| `scheduledDate` | Date | Mutable (saat `draft`) |
| `status` | Enum | Mutable |
| `assignedBoothIds` | List\<UUID\> | Mutable (saat `draft` atau `active`) |
| `activeManifestId` | UUID | Mutable (saat `active`) |

**Lifecycle:**
```
draft → active → completed → archived
```
`draft`: belum ada session yang boleh dibuat.
`active`: session dapat dibuat. Booth sudah di-assign.
`completed`: event selesai. Tidak ada session baru.
`archived`: data boleh dipindahkan ke cold storage.

**Invariants:**
- `status` hanya bisa maju, tidak bisa mundur
- `active` event harus memiliki minimal satu `Booth` dan satu `Manifest`
- Booth tidak boleh di-unassign dari event yang sudah `active` jika ada session aktif pada booth tersebut

---

### 3. Booth

**Fields:**

| Field | Type | Mutability |
|---|---|---|
| `boothId` | UUID | **Immutable** |
| `organizationId` | UUID | **Immutable** |
| `createdAt` | Timestamp | **Immutable** |
| `name` | String | Mutable |
| `status` | Enum (active\|inactive\|suspended) | Mutable |
| `currentDeviceId` | UUID | Mutable (saat ganti device) |
| `lastSeenAt` | Timestamp | Mutable (otomatis) |

**Lifecycle:**
```
Created → Active → Inactive ⇄ Active → Suspended
```
Tidak pernah dihapus secara fisik.

**Invariants:**
- Satu Booth hanya bisa memiliki satu `currentDeviceId` yang `active` pada satu waktu
- Booth `suspended` tidak boleh menerima session baru
- `boothId` tidak pernah berubah meski device berganti

---

### 4. DeviceRegistration

**Fields:**

| Field | Type | Mutability |
|---|---|---|
| `deviceId` | UUID | **Immutable** |
| `boothId` | UUID | **Immutable** |
| `registeredAt` | Timestamp | **Immutable** |
| `platform` | String | **Immutable** |
| `deviceClass` | String | **Immutable** |
| `publicKey` | String | **Immutable** |
| `runtimeId` | String | **Immutable** |
| `status` | Enum (active\|revoked) | Mutable |
| `lastSeenAt` | Timestamp | Mutable |
| `lastKnownDescriptor` | RuntimeDescriptorSnapshot | Mutable |

**Lifecycle:**
```
Registered → Active → Revoked
```
`revoked` bersifat final. Device baru harus register ulang dengan `deviceId` baru.

**Invariants:**
- Hanya satu `DeviceRegistration` dengan `status = active` per `Booth` pada satu waktu
- `publicKey` tidak pernah diubah setelah registrasi
- `lastKnownDescriptor` adalah observability — tidak digunakan untuk kontrol akses

---

### 5. Operator

**Fields:**

| Field | Type | Mutability |
|---|---|---|
| `operatorId` | UUID | **Immutable** |
| `organizationId` | UUID | **Immutable** |
| `createdAt` | Timestamp | **Immutable** |
| `email` | String | **Immutable** (identity) |
| `name` | String | Mutable |
| `role` | Enum (owner\|manager\|staff) | Mutable (oleh owner) |
| `permissions` | List\<Permission\> | Mutable |
| `status` | Enum (active\|suspended) | Mutable |
| `lastLoginAt` | Timestamp | Mutable |

**Lifecycle:**
```
Created → Active → Suspended → Active (dapat direaktivasi)
```

**Invariants:**
- Minimal satu operator `role = owner` per Organization harus selalu ada
- Owner terakhir tidak bisa di-suspend tanpa memindahkan ownership terlebih dahulu

---

### 6. Manifest

**Fields:**

| Field | Type | Mutability |
|---|---|---|
| `manifestId` | UUID | **Immutable** |
| `eventId` | UUID | **Immutable** |
| `version` | Int | **Immutable** |
| `publishedAt` | Timestamp | **Immutable** (setelah published) |
| `createdAt` | Timestamp | **Immutable** |
| `createdBy` | UUID | **Immutable** |
| `status` | Enum (draft\|published\|superseded) | Mutable |
| `packageIds` | List\<UUID\> | Mutable (saat `draft`) |
| `assetRefs` | List\<ManifestAssetRef\> | Mutable (saat `draft`) |

**Lifecycle:**
```
draft → published → superseded
```
Manifest tidak pernah dihapus. `superseded` ketika versi baru di-publish.

**Invariants:**
- Manifest `published` tidak boleh diubah (semua fields immutable setelah published)
- Satu Event hanya boleh memiliki satu Manifest `published` pada satu waktu
- `version` selalu monotonically increasing per Event
- Manifest `draft` tidak boleh digunakan oleh booth

---

### 7. Asset

**Fields:**

| Field | Type | Mutability |
|---|---|---|
| `assetId` | UUID | **Immutable** |
| `organizationId` | UUID | **Immutable** |
| `createdAt` | Timestamp | **Immutable** |
| `uploadedBy` | UUID | **Immutable** |
| `checksum` | String | **Immutable** |
| `sizeBytes` | Int | **Immutable** |
| `mimeType` | String | **Immutable** |
| `storageLocation` | String | **Immutable** |
| `type` | Enum | **Immutable** |
| `name` | String | Mutable (display only) |
| `status` | Enum (active\|deprecated\|deleted) | Mutable |

**Lifecycle:**
```
Uploaded → Verified → Referenced → Archived
```
`deleted` adalah logical delete — file di storage tidak langsung dihapus.

**Invariants:**
- Asset dengan `status = referenced` (dipakai oleh Manifest `published`) tidak boleh dihapus
- `checksum` digunakan Runtime untuk memvalidasi integritas file yang di-download
- File yang sudah di-upload tidak pernah dimodifikasi — hanya bisa dibuat Asset baru

---

### 8. Package

**Fields:**

| Field | Type | Mutability |
|---|---|---|
| `packageId` | UUID | **Immutable** |
| `organizationId` | UUID | **Immutable** |
| `createdAt` | Timestamp | **Immutable** |
| `priceAmount` | Int | **Immutable** (historical integrity) |
| `priceCurrency` | String | **Immutable** |
| `captureLimit` | Int | **Immutable** |
| `selectionLimit` | Int | **Immutable** |
| `name` | String | Mutable |
| `deliveryMethods` | List\<Enum\> | Mutable (saat `active`) |
| `status` | Enum (active\|discontinued) | Mutable |
| `discontinuedAt` | Timestamp | Mutable (set sekali) |

**Lifecycle:**
```
Created → Active → Discontinued
```
`discontinued` bersifat final untuk session baru. Session lama yang menggunakan package ini tetap valid.

**Invariants:**
- `priceAmount` tidak boleh diubah setelah ada `SessionArchive` yang menggunakannya
- Package `discontinued` tidak boleh dipakai untuk session baru
- `captureLimit` dan `selectionLimit` tidak boleh diubah (berdampak pada session yang sedang berjalan)

---

### 9. SessionArchive

> **SessionArchive ≠ Session Runtime.**
> SessionArchive adalah hasil ingest dari `SessionSnapshot`. Ia adalah representasi domain Cloud dari fakta yang sudah terjadi.

**Fields:**

| Field | Type | Mutability |
|---|---|---|
| `sessionId` | UUID (dari Runtime) | **Immutable** |
| `boothId` | UUID | **Immutable** |
| `eventId` | UUID | **Immutable** |
| `manifestVersion` | Int | **Immutable** |
| `packageId` | UUID | **Immutable** |
| `runtimeVersion` | String | **Immutable** |
| `architectureVersion` | String | **Immutable** |
| `snapshotVersion` | Int | **Immutable** |
| `startedAt` | Timestamp | **Immutable** |
| `guest.name` | String | **Immutable** |
| `guest.queueNumber` | Int | **Immutable** |
| `payment` | PaymentRecord | **Immutable** (setelah ingest) |
| `completedAt` | Timestamp | Mutable (null → set saat complete) |
| `archiveStatus` | Enum | Mutable |
| `captureSummary` | CaptureSummary | Mutable (update by Runtime) |
| `deliverySummary` | DeliverySummary | Mutable (update by Runtime) |
| `auditSummary` | AuditSummary | Mutable (update by Runtime) |

**`archiveStatus` lifecycle:**
```
in_progress → completed
           → recovered (jika ada recovery dari crash)
           → abandoned (jika session tidak pernah selesai)
```

**Lifecycle:**
```
Ingested (snapshot pertama diterima)
    ↓
Verified (data valid, referential integrity OK)
    ↓
Archived (session selesai, completedAt di-set)
```

**Invariants:**
- `sessionId` unik di seluruh Cloud
- `completedAt >= startedAt` jika `completedAt` tidak null
- `manifestVersion` wajib ada dan harus cocok dengan Manifest yang terdaftar di Event
- `payment` wajib immutable setelah pertama kali di-ingest (tidak boleh di-overwrite)
- `DomainEventRecord` untuk session ini bersifat append-only
- `archiveStatus` hanya bisa maju ke `completed`, `recovered`, atau `abandoned` — tidak bisa kembali ke `in_progress`

---

### 10. DomainEventRecord

**Fields:**

| Field | Type | Mutability |
|---|---|---|
| `eventId` | UUID (dari Runtime) | **Immutable** |
| `sessionId` | UUID | **Immutable** |
| `boothId` | UUID | **Immutable** |
| `sequenceNumber` | Int | **Immutable** |
| `eventType` | String | **Immutable** |
| `correlationId` | String | **Immutable** |
| `causationId` | String (nullable) | **Immutable** |
| `runtimeVersion` | String | **Immutable** |
| `occurredAt` | Timestamp | **Immutable** |
| `receivedAt` | Timestamp | **Immutable** |
| `payload` | JSON | **Immutable** |

Semua fields **immutable**. Append-only. Tidak ada update.

**Lifecycle:**
```
Received → Stored
```
Tidak ada state lain. Record yang sudah tersimpan tidak pernah berubah.

**Invariants:**
- `eventId` unik di seluruh sistem (idempotency: duplicate `eventId` diabaikan)
- `sequenceNumber` monotonically increasing per `sessionId`
- `occurredAt` tidak boleh di masa depan
- `payload` tidak boleh di-modify setelah disimpan

---

### 11. AuditEvent

**Fields:**

| Field | Type | Mutability |
|---|---|---|
| `auditId` | UUID | **Immutable** |
| `organizationId` | UUID | **Immutable** |
| `boothId` | UUID (nullable) | **Immutable** |
| `operatorId` | UUID (nullable) | **Immutable** |
| `sessionId` | UUID (nullable) | **Immutable** |
| `category` | Enum | **Immutable** |
| `action` | String | **Immutable** |
| `outcome` | Enum (success\|failure\|warning) | **Immutable** |
| `metadata` | JSON | **Immutable** |
| `occurredAt` | Timestamp | **Immutable** |
| `receivedAt` | Timestamp | **Immutable** |

Semua fields **immutable**. Append-only. Tidak ada update atau delete.

**Lifecycle:**
```
Received → Stored (permanent)
```

**Invariants:**
- Tidak pernah dihapus
- `occurredAt` tidak boleh di masa depan
- Retensi minimum 7 tahun (compliance keuangan)

---

## Analytics — Projection Only

> **Analytics bukan Aggregate. Analytics adalah projection.**

Semua dashboard dibangun dari projection — bukan dari data domain secara langsung.

| Projection | Dibangun Dari |
|---|---|
| `DailySessionSummary` | `SessionArchive` + `DomainEventRecord` |
| `RevenueByEvent` | `PaymentRecord` dalam `SessionArchive` |
| `DeliverySuccessRate` | `DeliverySummary` dalam `SessionArchive` |
| `ManifestUsageStats` | `manifestVersion` dalam `SessionArchive` |
| `BoothHealthSummary` | `DeviceRegistration.lastSeenAt` + `AuditEvent` |
| `LiveSessionCount` | `SessionArchive` dengan `archiveStatus = in_progress` |
| `OperatorActivityLog` | `AuditEvent` dengan `operatorId != null` |

**Aturan:**
- Projection boleh di-regenerate kapan saja dari data domain
- Projection tidak pernah menjadi source of truth
- Perubahan business logic di domain otomatis terabsorb saat projection di-regenerate

---

## Relasi Antar Aggregate

```
Organization
    ├── 1..* Event
    ├── 1..* Booth
    ├── 1..* Operator
    └── 1..* Asset

Event
    ├── 1..* Manifest (versi immutable)
    ├── 1..* Package
    └── *..* Booth (assignment per event)

Manifest
    └── *..* Asset (via ManifestAssetRef)

Booth
    ├── 1..* DeviceRegistration (history perangkat)
    └── 1..* SessionArchive

SessionArchive
    ├── 1  Booth
    ├── 1  Event
    ├── 1  Manifest (version pinned, immutable)
    ├── 1  Package (immutable)
    └── 1..* DomainEventRecord (append-only)

AuditEvent
    ├── 0..1 Booth
    ├── 0..1 Operator
    └── 0..1 SessionArchive
```

---

## Data Retention Policy

| Data | Retensi | Alasan |
|---|---|---|
| `SessionArchive` | 2 tahun | Kebutuhan operasional dan sengketa |
| `DomainEventRecord` | 2 tahun | Bersamaan dengan SessionArchive |
| `AuditEvent` | **7 tahun** | Compliance keuangan |
| `Manifest` | Sesuai event + 1 tahun | Dibutuhkan untuk verifikasi archive lama |
| `Asset` | Sesuai paket event | Dapat di-purge setelah event diarsipkan |
| `DeviceRegistration` | Selama Booth aktif + 1 tahun | Forensik jika ada insiden keamanan |
| Analytics Projection | Dapat di-regenerate | Tidak ada retensi tetap |
| Application Logs | 90 hari | Debugging operasional |

---

## Out of Scope

Dokumen ini tidak membahas:
REST API, endpoint, HTTP method, authentication, token, database, ORM, SQL, object storage vendor, message queue, WebSocket, framework backend.

---

## Acceptance Criteria (Final)

- [x] Setiap entity memiliki satu Owner dan satu Source of Truth yang jelas
- [x] Tidak ada workflow atau state machine Runtime yang berpindah ke Cloud
- [x] Semua relasi antar aggregate terdokumentasi
- [x] Tidak ada kepemilikan ganda
- [x] Analytics diposisikan sebagai projection, bukan domain utama
- [x] Netral terhadap implementasi backend
- [x] `SessionSnapshot` (transport) dipisahkan dari `SessionArchive` (domain)
- [x] Identity rules terdokumentasi per aggregate
- [x] Referential integrity rules terdokumentasi
- [x] Immutable vs mutable fields jelas per entity
- [x] Lifecycle domain setiap aggregate terdokumentasi
- [x] Invariant utama setiap aggregate tertulis eksplisit

---

*CLOUD_DOMAIN_MODEL.md v1.1 — FINAL*
*Ref: constitution/PLATFORM_RUNTIME_V1.md, cloud/SYNC_STRATEGY.md, cloud/AUTHENTICATION.md, cloud/ERROR_MODEL.md*
