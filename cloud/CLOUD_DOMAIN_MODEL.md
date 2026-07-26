# Cloud Domain Model — Haispace Cloud

**Status:** Draft v1.0
**Milestone:** 3 — Cloud Contract
**Penulis:** GPT (Chief Product Architect) + Antigravity (Lead Software Architect)
**Tanggal:** 2026-07-26

---

## Core Principle

> **Cloud stores facts. Runtime executes behavior.**

Cloud tidak memiliki state machine booth.
Cloud tidak pernah menentukan langkah berikutnya dalam sebuah session.
Cloud hanya mengetahui fakta yang dikirim oleh Runtime.

Konsekuensi langsung dari prinsip ini:

- Tidak ada `WorkflowStage`, `AppState`, atau actor di Cloud
- Cloud tidak bisa "membatalkan" atau "mengubah" session yang sedang berjalan
- Recovery selalu berasal dari snapshot lokal — bukan dari Cloud
- Jika Cloud mati selama 24 jam, Runtime tetap berjalan normal

---

## Aggregate Roots

Cloud hanya memiliki delapan Aggregate Root. Semua entity lain berada di bawahnya.

```
Organization
    └── Event
    └── Booth
    └── Operator

Booth
    └── DeviceRegistration

Event
    └── Manifest
    └── Package

Manifest
    └── Asset (referensi)

Asset (berdiri sendiri)

SessionArchive
    └── DomainEventRecord
    └── CaptureSummary
    └── DeliverySummary
    └── PaymentRecord
    └── AuditSummary

AuditEvent (append-only)
```

---

## Ownership Table

> Tidak boleh ada dua Source of Truth untuk satu entitas yang sama.

| Entity | Owner | Writable By | Readable By | Source of Truth |
|---|---|---|---|---|
| `Organization` | Cloud | Admin | Cloud | Cloud |
| `Event` | Cloud | Admin | Runtime + Cloud | Cloud |
| `Booth` | Cloud | Admin | Runtime + Cloud | Cloud |
| `DeviceRegistration` | Cloud | Runtime (self-register) + Admin | Runtime + Cloud | Cloud |
| `Operator` | Cloud | Admin | Runtime + Cloud | Cloud |
| `Manifest` | Cloud | Admin | Runtime | Cloud |
| `Asset` | Cloud | Admin | Runtime | Cloud |
| `Package` | Cloud | Admin | Runtime | Cloud |
| `SessionArchive` | **Runtime** | Runtime | Cloud | **Runtime** |
| `DomainEventRecord` | **Runtime** | Runtime | Cloud | **Runtime** |
| `AuditEvent` | **Runtime** | Runtime | Cloud | **Runtime** |

**Interpretasi:**
- Entity milik Cloud → Cloud dapat membuat, mengubah, dan menghapusnya
- Entity milik Runtime → Cloud hanya menerima dan menyimpan. Cloud tidak boleh mengubahnya.

---

## 1. Organization

Pemilik seluruh hierarki data. Satu organisasi dapat mengelola banyak event.

```
Organization
    organizationId      UUID — permanen
    name                String
    plan                Enum (starter | professional | enterprise)
    createdAt           Timestamp (immutable)
    status              Enum (active | suspended | terminated)
```

**Lifecycle:** Dibuat oleh Admin. Tidak pernah dihapus — hanya di-suspend.

---

## 2. Event

Satu sesi foto booth selalu terjadi dalam konteks sebuah Event (misalnya: "Wisuda BINUS 2026").

```
Event
    eventId             UUID — permanen
    organizationId      FK → Organization
    name                String
    venue               String
    scheduledDate       Date
    status              Enum (draft | active | completed | archived)
    assignedBoothIds    List<boothId>
    activeManifestId    FK → Manifest (versi manifest aktif saat ini)
    createdAt           Timestamp (immutable)
    updatedAt           Timestamp
```

**Lifecycle:** Draft → Active (saat event dimulai) → Completed → Archived.

**Aturan:** Satu booth bisa di-assign ke banyak event (berbeda tanggal). Satu event dapat punya banyak booth.

---

## 3. Booth

Representasi fisik satu unit booth di Cloud. Booth bisa berganti device (iPad) tanpa kehilangan identitas.

```
Booth
    boothId             UUID — permanen (sama dengan yang di Keychain iPad)
    organizationId      FK → Organization
    name                String (contoh: "Booth 01 — Utara")
    status              Enum (active | inactive | suspended)
    currentDeviceId     FK → DeviceRegistration (device yang sedang aktif)
    createdAt           Timestamp (immutable)
    lastSeenAt          Timestamp
```

**Lifecycle:** Dibuat oleh Admin. Dapat berganti device. Tidak pernah dihapus.

**Perbedaan Booth vs DeviceRegistration:**
- `Booth` = identitas logis (tempat, nama, history)
- `DeviceRegistration` = identitas fisik (iPad tertentu, runtime version)

---

## 4. DeviceRegistration

Merepresentasikan satu iPad yang terdaftar. Terpisah dari Booth karena iPad bisa diganti.

```
DeviceRegistration
    deviceId            UUID — di-generate Runtime saat instalasi pertama
    boothId             FK → Booth
    platform            String ("iOS")
    deviceClass         String ("Booth" | "Camera" | "Admin")
    runtimeId           String ("booth-runtime-ios")
    publicKey           String (RSA public key)
    registeredAt        Timestamp (immutable)
    lastSeenAt          Timestamp
    status              Enum (active | revoked)

    lastKnownDescriptor RuntimeDescriptorSnapshot
        architectureVersion String
        runtimeVersion      String
        buildNumber         String
        reportedAt          Timestamp
```

**Lifecycle:** Dibuat oleh Runtime (self-register). Dapat di-revoke oleh Admin. Tidak pernah dihapus.

**`lastKnownDescriptor`** bukan untuk kontrol — hanya untuk observability (Mission Control dapat melihat versi runtime setiap booth).

---

## 5. Operator

Pengguna manusia yang mengelola dan memantau booth.

```
Operator
    operatorId          UUID — permanen
    organizationId      FK → Organization
    name                String
    email               String (unique)
    role                Enum (owner | manager | staff)
    permissions         List<Permission>
    status              Enum (active | suspended)
    createdAt           Timestamp (immutable)
    lastLoginAt         Timestamp
```

**Lifecycle:** Dibuat oleh Admin. Tidak pernah dihapus — hanya di-suspend.

---

## 6. Manifest

Manifest adalah blueprint satu sesi foto: frame yang tersedia, filter, jumlah capture, dll.
Manifest bersifat **immutable** — versi baru berarti object baru. Tidak pernah overwrite.

```
Manifest
    manifestId          UUID — permanen
    eventId             FK → Event
    version             Int (monotonically increasing, 1, 2, 3, ...)
    status              Enum (draft | published | deprecated)
    packageIds          List<FK → Package>
    assetRefs           List<ManifestAssetRef>
    publishedAt         Timestamp (immutable setelah published)
    createdAt           Timestamp (immutable)
    createdBy           FK → Operator

ManifestAssetRef
    assetId             FK → Asset
    role                Enum (frame | filter | overlay | background | sticker)
    displayOrder        Int
```

**Lifecycle:** Draft → Published → Deprecated (ketika versi baru published).

**Aturan:** Runtime selalu mendapatkan manifest versi terbaru yang berstatus `published`. Session yang sedang berjalan tidak pernah mendapat manifest baru (version pinning di Runtime).

---

## 7. Asset

Asset adalah **metadata** dari sebuah file kreatif (frame, filter, overlay). File itu sendiri berada di object storage — bukan di Cloud domain.

```
Asset
    assetId             UUID — permanen
    organizationId      FK → Organization
    type                Enum (frame | filter | overlay | background | sticker | font)
    name                String
    checksum            String (SHA-256 dari file)
    sizeBytes           Int
    storageLocation     String (opaque reference ke object storage — bukan URL langsung)
    mimeType            String
    uploadedBy          FK → Operator
    createdAt           Timestamp (immutable)
    status              Enum (active | deprecated | deleted)
```

**Lifecycle:** Dibuat saat upload. Tidak pernah diubah (immutable metadata). Jika file diganti, buat Asset baru.

**Catatan:** `storageLocation` adalah referensi opaque — Cloud Service yang menghasilkan URL download, bukan Runtime.

---

## 8. Package

Paket yang ditawarkan kepada tamu (misalnya: "3 Foto Rp 35.000").

```
Package
    packageId           UUID — permanen
    organizationId      FK → Organization
    name                String
    captureLimit        Int
    selectionLimit      Int
    priceAmount         Int (dalam satuan terkecil, contoh: sen)
    priceCurrency       String (ISO 4217, contoh: "IDR")
    deliveryMethods     List<Enum (digital | print | both)>
    status              Enum (active | discontinued)
    createdAt           Timestamp (immutable)
    discontinuedAt      Timestamp (null jika masih aktif)
```

**Lifecycle:** Dibuat oleh Admin. Jika discontinued, session lama yang menggunakan package ini tetap valid — hanya session baru yang tidak bisa menggunakannya.

---

## 9. SessionArchive

> **SessionArchive bukan Session Runtime.**

Session Runtime hidup di iPad — memiliki state machine, actor, workflow. Cloud tidak tahu itu semua.
SessionArchive adalah **snapshot akhir** dari sebuah session yang sudah selesai atau yang perlu di-backup untuk recovery.

```
SessionArchive
    sessionId           UUID — berasal dari Runtime (immutable)
    boothId             FK → Booth
    eventId             FK → Event
    manifestVersion     Int (versi manifest yang dipakai saat session)
    packageId           FK → Package
    runtimeVersion      String
    architectureVersion String
    snapshotVersion     Int (schema version dari SessionSnapshot)
    startedAt           Timestamp (immutable)
    completedAt         Timestamp (null jika belum selesai)
    archiveStatus       Enum (in_progress | completed | recovered | abandoned)

    guest               GuestInfo
        name            String
        queueNumber     Int

    payment             PaymentRecord (null jika belum bayar)
        localTransactionId  String
        amount          Int
        currency        String
        method          String
        acceptedAt      Timestamp

    captureSummary      CaptureSummary
        totalCaptured   Int
        selectedCount   Int

    deliverySummary     DeliverySummary (null jika belum delivery)
        method          String
        status          Enum (queued | completed | failed)
        completedAt     Timestamp

    auditSummary        AuditSummary
        eventCount      Int
        lastEventAt     Timestamp
```

**Tidak ada di SessionArchive:**
- `currentStage` — sudah selesai, tidak ada stage
- `WorkflowTimer` — Runtime concern
- `actor state` — Runtime concern
- logic apapun

**Lifecycle:** Dibuat/diperbarui oleh Runtime via upload. Setelah `completed`, tidak boleh diubah kecuali oleh proses recovery.

---

## 10. DomainEventRecord

Cloud menyimpan setiap DomainEvent yang dikirim Runtime sebagai record yang immutable.

```
DomainEventRecord
    eventId             UUID (dari DomainEventEnvelope — immutable)
    sessionId           FK → SessionArchive
    boothId             FK → Booth
    sequenceNumber      Int (urutan dalam session ini)
    eventType           String (nama event, contoh: "PaymentAccepted")
    correlationId       String (dari envelope)
    causationId         String (dari envelope, nullable)
    runtimeVersion      String
    occurredAt          Timestamp (dari envelope — waktu terjadi di Runtime)
    receivedAt          Timestamp (waktu diterima Cloud)
    payload             JSON (konten event — immutable)
```

**Events yang disimpan:**

| Event | Keterangan |
|---|---|
| `PaymentAccepted` | Payment dikonfirmasi Runtime |
| `CaptureAdded` | Satu foto ditambahkan |
| `PhotoSelected` | Tamu memilih foto |
| `DeliveryQueued` | Delivery dimulai |
| `DeliveryCompleted` | Delivery selesai |
| `DeliveryFailed` | Delivery gagal |
| `SessionCompleted` | Session selesai normal |
| `SessionAbandoned` | Session ditinggalkan |
| `RecoveryInitiated` | Session di-recover dari snapshot |

**Lifecycle:** Append-only. Tidak pernah diubah atau dihapus.

**Idempotency:** Cloud mengabaikan event dengan `eventId` yang sudah ada.

---

## 11. AuditEvent

Catatan audit operasional — lebih detail dari DomainEventRecord, mencakup tindakan operator dan sistem.

```
AuditEvent
    auditId             UUID
    organizationId      FK → Organization
    boothId             FK → Booth (nullable)
    operatorId          FK → Operator (nullable)
    sessionId           FK → SessionArchive (nullable)
    category            Enum (session | payment | delivery | manifest | device | operator | system)
    action              String (contoh: "session.created", "payment.accepted")
    outcome             Enum (success | failure | warning)
    metadata            JSON (konteks tambahan)
    occurredAt          Timestamp (waktu terjadi di Runtime)
    receivedAt          Timestamp (waktu diterima Cloud)
```

**Lifecycle:** Append-only. Tidak pernah diubah. Retensi: **7 tahun** (kebutuhan compliance).

---

## 12. Analytics (Projection — Bukan Aggregate)

> **Analytics bukan Aggregate. Analytics adalah projection.**

Semua dashboard — Operator Dashboard, Mission Control, Revenue Report, Printer Health, Live Sessions — dibangun dari **projection** data di atas. Bukan dari Session Runtime secara langsung.

Contoh projection:

```
DailySessionSummary     → dibangun dari SessionArchive + DomainEventRecord
RevenueByEvent          → dibangun dari PaymentRecord dalam SessionArchive
DeliverySuccessRate     → dibangun dari DeliverySummary
ManifestUsageStats      → dibangun dari manifestVersion dalam SessionArchive
BoothHealthSummary      → dibangun dari DeviceRegistration.lastSeenAt + AuditEvent
```

**Aturan:**
- Projection boleh di-regenerate kapan saja dari data domain
- Projection tidak menjadi source of truth
- Perubahan business logic di domain otomatis mempengaruhi projection saat di-regenerate

---

## Relasi Antar Aggregate

```
Organization
    ├── 1..* Event
    ├── 1..* Booth
    ├── 1..* Operator
    └── 1..* Asset

Event
    ├── 1..* Manifest (versi)
    ├── 1..* Package
    └── *..* Booth (assignment)

Booth
    ├── 1..* DeviceRegistration (history perangkat)
    └── 1..* SessionArchive

SessionArchive
    ├── 1  Booth
    ├── 1  Event
    ├── 1  Manifest (version pinned)
    ├── 1  Package
    └── 1..* DomainEventRecord

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
| `Manifest` | Sesuai event | Dihapus bersama event ketika di-archive |
| `Asset` | Sesuai paket event | Dapat di-purge setelah event selesai |
| `DeviceRegistration` | Selama Booth aktif + 1 tahun | Untuk forensik jika ada insiden |
| Analytics Projection | Dapat di-regenerate | Tidak ada retensi — bisa dibuat ulang |
| Application Logs | 90 hari | Debugging operasional |

---

## Out of Scope

Dokumen ini tidak membahas:

- REST API, endpoint, HTTP method
- Authentication, API key, token
- Database, ORM, SQL, PostgreSQL, Redis
- Object storage vendor (S3, GCS, R2)
- Message queue, WebSocket
- Framework backend (Rails, FastAPI, NestJS, dll.)
- Deployment, infra, Kubernetes

Semua hal di atas akan dibahas di `CLOUD_RESOURCES.md`, `CLOUD_CONTRACT.md`, dan ADR implementation.

---

## Acceptance Criteria (dari GPT)

- [x] Setiap entity memiliki satu **Owner** dan satu **Source of Truth** yang jelas
- [x] Tidak ada workflow atau state machine Runtime yang berpindah ke Cloud
- [x] Semua relasi antar aggregate terdokumentasi
- [x] Seluruh data dapat dipetakan ke Runtime tanpa kepemilikan ganda
- [x] Analytics dan dashboard diposisikan sebagai **projection**, bukan domain utama
- [x] Dokumen tetap netral terhadap implementasi backend

---

*CLOUD_DOMAIN_MODEL.md v1.0 — Milestone 3 Cloud Contract*
*Ref: constitution/PLATFORM_RUNTIME_V1.md, cloud/SYNC_STRATEGY.md, cloud/AUTHENTICATION.md, cloud/ERROR_MODEL.md*
