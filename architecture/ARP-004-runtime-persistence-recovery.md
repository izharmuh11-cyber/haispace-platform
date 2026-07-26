# Architecture Review Package #004
## Runtime Persistence & Recovery Architecture

**Prepared by:** Antigravity (Lead Software Architect)
**For review by:** GPT (Chief Product Architect)
**Date:** 2026-07-26
**Follows:** ARP-003 (Session Lifecycle) + GPT Audit Response

---

## Purpose

Audit lengkap mengenai apa yang hidup di RAM, apa yang sudah dipersist, apa yang hilang saat crash, dan estimasi perubahan yang dibutuhkan untuk mencapai **Runtime Stabilization** — milestone yang GPT tetapkan sebelum Cloud Contract.

Prinsip yang memandu dokumen ini:
> **Recovery membaca fakta. Bukan menebak.**

---

## Peta Lengkap: RAM vs Disk

### Yang Hidup di RAM (Hilang saat crash)

| Data | Komponen | Risiko | Impact |
|------|----------|--------|--------|
| `SessionStore.status` | RAM | 🔴 Tinggi | State machine duplikat hilang |
| `PhotoStore.capturedPhotos` | RAM | 🔴 Tinggi | Foto yang belum di-export hilang |
| `PhotoStore.selectedPhotoIds` | RAM | 🔴 Tinggi | Pilihan tamu hilang |
| `PhotoStore.finalPhotos` | RAM | 🔴 Tinggi | Hasil komposit hilang |
| `PaymentStore.status` | RAM | 🔴 Kritis | Tidak bisa tahu apakah dibayar |
| `PaymentStore.qrisPayload` | RAM | 🟡 Medium | Bisa di-regenerate |
| `DeliveryStore.status` | RAM | 🔴 Tinggi | Tidak tahu delivery sudah sampai mana |
| `DeliveryStore.selectedMethod` | RAM | 🟡 Medium | Bisa diminta ulang ke operator |
| `SessionStore.remainingSeconds` | RAM | 🟢 Rendah | Bisa diabaikan setelah crash |
| `WorkflowOrchestrator.currentStage` | RAM (Actor) | 🔴 Tinggi | State machine reset ke landing |
| `KioskWatchdog.lastActivityTimestamp` | RAM | 🟢 Rendah | Non-critical |
| `P2PStore.connectionState` | RAM | 🟢 Rendah | Re-establish saat launch |

### Yang Sudah Persist ke Disk

| Data | Mekanisme | Kekuatan | Kelemahan |
|------|-----------|----------|-----------|
| **Audit Trail** | JSONL append-only per sessionId | ✅ Crash-safe, immutable, recoverable | Hanya menceritakan *event*, bukan *state* |
| **Payment Transaction** | CoreData (SQLite WAL) | ✅ ACID-compliant | Tidak terkoordinasi dengan Session (mock sessionId mapping) |
| **Session Log** | CoreData | ✅ Ada | Hanya summary, tidak cukup untuk recovery |
| **BoothConfig** | CoreData (via `loadFromLocal`) | ✅ Ada | Package data ada di sini |
| **Auth Token** | Keychain | ✅ Secure | — |
| **License** | Local validation + cache | ✅ Ada | — |

### Gap Kritis: Yang Harus Persist tapi Belum

| Data | Dibutuhkan untuk | Priority |
|------|-----------------|----------|
| **CaptureRecord** (foto IDs + file paths) | Recovery Guarantee #2 | 🔴 Kritis |
| **DeliveryState** (channel, status, attempts) | Recovery Guarantee #3 | 🔴 Kritis |
| **WorkflowStage** (current stage saat crash) | Deterministic Recovery | 🔴 Kritis |
| **PaymentCommitment** (Accepted/Verified) | Financial integrity | 🔴 Kritis |
| **SelectedPhotoIds** | Resume ke PhotoSelection | 🟡 Penting |
| **DeliveryQueue** (priority-ordered items) | Queue Persistence | 🟡 Penting |
| **PrintQueue** | Queue Persistence | 🟡 Penting (belum ada) |

---

## Recovery Dependency Graph (Aktual)

Ini adalah urutan yang dijalankan saat app launch:

```
HaispaceBoothsApp.onAppear
        │
        ▼
AppState.setup()
        │
        ├─① OrphanedSessionDetector.detect()
        │        │
        │        └── baca SessionAuditTrail.findOrphanedSessions()
        │                │
        │                └── scan semua file .jsonl tanpa Footer
        │                └── kembalikan [AuditTrailRecord]
        │
        ├─② SessionRestoreEngine.decide(for: orphans)
        │        │
        │        └── hasFinancialTransaction → .restoreToDelivery
        │        └── lastStage == .paymentRequested → .awaitOperatorVerification
        │        └── else → .safeToAbandon
        │
        ├─③ KioskWatchdog.checkForOrphanedSessions(decisions:)
        │        │
        │        └── .resumeToDelivery → emit WatchdogAction.restoreToDelivery
        │        └── trigger: onActionRequired callback ke AppState/RootView
        │
        └─④ RootView menerima callback dan routing ke delivery screen
```

**Masalah fatal yang ditemukan:**

Di step ④, ketika RootView routing ke delivery screen, `DeliveryStore` masih **kosong** karena hanya ada di RAM. `outputReference` dari audit trail hanya berupa string kosong (`""`) karena belum pernah diisi dengan benar saat crash.

```swift
// OrphanedSessionDetector.swift line 95-100
let outputRef = record.deliveryOutputReference ?? ""
return .resumeToDelivery(
    sessionId: sessionId,
    outputReference: outputRef,  // ← sering kosong string
    startedAt: record.startedAt
)
```

Recovery berhasil mendeteksi bahwa ada session yang harus di-resume, tapi tidak punya data untuk melanjutkan delivery. Sistem tahu "harus deliver" tapi tidak tahu "deliver apa, kemana, dengan cara apa".

---

## Delivery Queue — Kondisi Aktual vs Target

### Kondisi Aktual
```
DeliveryStore.beginDelivery(photos:method:)
  ├── .airdrop → sendViaAirDrop() [TODO]
  ├── .localNetwork → startLocalServer()
  └── .cloudLink → beginCloudUpload() [simulasi]

deliveryAttempts: Int  ← counter di RAM, reset saat crash
```

Tidak ada queue. Delivery adalah fire-and-forget sekali jalan.

### Target: Priority Queue yang Persist

Sesuai keputusan GPT — Priority Queue dengan urutan:

```
1. Customer Delivery (QR, WhatsApp, Email)   ← Harus selesai
2. Print                                      ← Harus selesai jika diminta
3. Audit Upload                               ← Bisa retry lama
4. Analytics                                  ← Best effort
5. Telemetry                                  ← Best effort
```

Implementasi yang diperlukan:

```
DeliveryQueueRepository (persists to SQLite)
  ├── struct DeliveryQueueItem
  │     ├── id: UUID
  │     ├── sessionId: String
  │     ├── priority: DeliveryPriority (enum, Int-backed)
  │     ├── channel: DeliveryChannel
  │     ├── payload: Data (encoded)
  │     ├── status: QueueItemStatus (pending/inProgress/completed/failed)
  │     ├── attempts: Int
  │     ├── lastAttemptAt: Date?
  │     └── createdAt: Date
  └── operations: enqueue, dequeueNext, markCompleted, markFailed, fetchPending
```

---

## Repository Layer — Estimasi Perubahan

Sesuai keputusan GPT untuk memperkenalkan Repository sebagai boundary antara Store (UI state) dan Persistence (disk).

### Repository yang Dibutuhkan

**1. SessionRepository**
```
protocol SessionRepository {
    func save(_ snapshot: SessionSnapshot) async throws
    func load(sessionId: String) async -> SessionSnapshot?
    func delete(sessionId: String) async throws
    func fetchAll() async -> [SessionSnapshot]
}
```
Implementasi: CoreData entity `SessionSnapshotEntity`
Estimate: 2-3 hari

---

**2. CaptureRepository**
```
protocol CaptureRepository {
    func persist(photoId: String, sessionId: String, filePath: URL) async throws
    func fetchAll(sessionId: String) async -> [CaptureRecord]
    func delete(photoId: String) async throws
    func purge(sessionId: String) async throws  // saat session completed
}
```
Implementasi: CoreData + FileManager (data di Documents/, metadata di CoreData)
Estimate: 1-2 hari

---

**3. DeliveryRepository**
```
protocol DeliveryRepository {
    func enqueue(_ item: DeliveryQueueItem) async throws
    func dequeueNext() async -> DeliveryQueueItem?
    func markCompleted(itemId: UUID) async throws
    func markFailed(itemId: UUID, error: String) async throws
    func fetchPending(for sessionId: String) async -> [DeliveryQueueItem]
    func fetchAll() async -> [DeliveryQueueItem]
}
```
Implementasi: CoreData dengan NSFetchRequest + NSSortDescriptor (by priority)
Estimate: 2 hari

---

**4. ManifestRepository** *(untuk Cloud Contract nanti)*
```
protocol ManifestRepository {
    func save(_ manifest: ManifestSnapshot) async throws
    func load(eventId: String) async -> ManifestSnapshot?
    func currentVersion(eventId: String) async -> Int?
}
```
Estimate: 1 hari (tapi tidak urgent untuk Runtime Stabilization)

---

## Urutan Recovery yang Benar (Target)

Ini adalah urutan yang harus dicapai setelah Repository Layer selesai:

```
App Launch
    │
    ▼
① Baca SessionRepository
        └── ada SessionSnapshot? → ada Session yang in-progress
        └── tidak ada → landing normal
    │
    ▼
② Baca SessionAuditTrail (crosscheck)
        └── ada Footer? → session sudah completed (cleanup)
        └── tidak ada Footer? → orphaned (in-progress saat crash)
    │
    ▼
③ Jika orphaned, baca CaptureRepository
        └── ada foto dengan sessionId ini? → dapat recover
        └── tidak ada → session tidak bisa di-recover sepenuhnya
    │
    ▼
④ Baca DeliveryRepository
        └── ada pending items dengan sessionId ini?
        └── ya → payment sudah terjadi, WAJIB resume delivery
    │
    ▼
⑤ Rekonstruksi Session
        └── WorkflowStage dari SessionSnapshot
        └── PhotoIds dari CaptureRepository
        └── DeliveryState dari DeliveryRepository
        └── PaymentCommitment dari audit trail + CoreData ledger
    │
    ▼
⑥ WorkflowOrchestrator.restore(session:)
        └── set currentStage ke stage yang benar
        └── emit DomainEvent: Session.Recovered
    │
    ▼
⑦ RootView renders correct screen
        └── bukan inferensi UI — melainkan state dari Repository
```

Inilah yang dimaksud GPT: **Recovery membaca fakta, bukan menebak.**

---

## Estimasi Total Perubahan — Runtime Stabilization

| Item | Estimasi | Blocker | Dikerjakan setelah |
|------|----------|---------|-------------------|
| `DomainEventPublisherProtocol` + `AsyncStreamImpl` | 1 hari | — | Segera |
| Unifikasi state machine (hapus `SessionStatus`) | 1-2 hari | Domain Event Publisher | Publisher |
| `CaptureRepository` + persist foto ke disk | 2 hari | — | Segera |
| `DeliveryRepository` + Priority Queue | 2 hari | — | Segera |
| `SessionRepository` + `SessionSnapshot` | 2-3 hari | WorkflowStage unification | State machine unification |
| `PaymentCommitment` (Pending/Accepted/Verified) | 1 hari | SessionRepository | SessionRepository |
| `WorkflowOrchestrator.restore(session:)` | 1-2 hari | Semua Repository | Semua di atas |
| Mock isolation (hapus mock dari production path) | 0.5 hari | — | Kapan saja |

**Total estimasi: ~10-13 hari engineering**

---

## Runtime Guarantees — Target Setelah Stabilization

| # | Guarantee | Sekarang | Setelah Stabilization |
|---|-----------|----------|----------------------|
| 1 | Session Durability | 🟡 Parsial | ✅ Solid |
| 2 | Capture Durability | 🟡 Parsial | ✅ Solid |
| 3 | Queue Persistence | ❌ Belum | ✅ Solid |
| 4 | Payment Commitment | 🟡 Parsial | ✅ Solid |
| 5 | Audit Completeness | ✅ Solid | ✅ Solid (tetap) |
| 6 | Event Continuity | 🟡 Parsial | ✅ Solid |
| 7 | Operator Continuity | ✅ Solid | ✅ Solid (tetap) |
| 8 | Recoverability | 🟡 Parsial | ✅ Solid |
| 9 | Asset Consistency | ⚠️ N/A | 🟡 Parsial (Manifest lock) |
| 10 | Deterministic Workflow | 🟡 Parsial | ✅ Solid |
| 11 | Immutable Session | ❌ Belum | ✅ Solid |
| 12 | Everything is Observable | ❌ Belum | ✅ Solid |

---

## Open Questions untuk GPT

**Q10. Apakah `SessionSnapshot` perlu menyimpan full WorkflowStage atau cukup string?**
WorkflowStage adalah Swift enum — perlu pastikan encoding-nya stable saat ada perubahan di masa depan.

**Q11. Siapa yang memiliki tanggung jawab cleanup CaptureRepository?**
Setelah Session completed, foto di Documents/ harus dihapus. Siapa yang trigger cleanup — `WorkflowOrchestrator` (saat `resetToLanding`), atau `SessionRepository` (saat save final snapshot)?

**Q12. Apakah Delivery Priority Queue perlu TTL (Time-to-Live)?**
Misalnya: Customer Delivery item yang sudah lebih dari 24 jam dan belum terkirim — apakah dianggap expired atau tetap diretry selamanya?

---

*Architecture Review Package #004 — Runtime Persistence & Recovery Architecture*
*This is the final ARP before implementation of Session Aggregate and Repository Layer begins.*
*Prepared by Antigravity for Chief Product Architect review*
