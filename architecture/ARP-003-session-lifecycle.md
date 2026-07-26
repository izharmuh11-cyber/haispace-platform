# Architecture Review Package #003
## Session Lifecycle — Aggregate Root Analysis

**Prepared by:** Antigravity (Lead Software Architect)
**For review by:** GPT (Chief Product Architect)
**Date:** 2026-07-26
**Follows:** ARP-002 (Dependency Direction)

---

## Purpose

Session adalah aggregate root paling penting di seluruh platform. Sebelum Cloud dibangun, aggregate ini harus sehat. ARP ini memetakan kondisi aktual Session Lifecycle dan mengidentifikasi gap terhadap Runtime Guarantees.

---

## Session Aggregate — Komponen Aktual

Session di HaiBooth saat ini bukan satu entitas — ia tersebar di tiga lapisan:

```
AppState.currentSession (Application Layer)
        │
        └── SessionStore (@Observable class)
                │
                ├── Identity:   sessionId (UUID string), guest, package
                ├── State:      status (SessionStatus enum), remainingSeconds
                ├── Sub-Stores: PhotoStore, PaymentStore, DeliveryStore
                └── Timers:     sessionTimerTask, captureTimerTask (Tasks)

WorkflowOrchestrator (Domain Layer)
        │
        └── activeSessionId (SessionID)
        └── currentCorrelationId (CorrelationID)
        └── activePhotoId (PhotoID)
        └── activeOutputReference (String)

SessionAuditTrail (Infrastructure Layer)
        │
        └── JSONL file per sessionId
        └── AuditTrailHeader + AuditEvent[] + AuditTrailFooter
```

**Masalah pertama:** Session aggregate tersebar di tiga lokasi berbeda. Tidak ada satu "Session" yang menjadi aggregate root tunggal.

---

## Session Status — Dua State Machine yang Tidak Sinkron

### State Machine #1 — WorkflowStage (di WorkflowOrchestrator)

```
landing
  → guestRegistration
  → packageSelection
  → templateSelection    ← ⚠️  frame sebelum foto (workflow lama)
  → capturing
  → editingPreview
  → exporting
  → paymentRequested
  → paymentConfirmed
  → deliveryDispatch
  → sessionCompleted
  → recoveryMode
```

### State Machine #2 — SessionStatus (di SessionStore)

```
briefing
  → active
  → paused              ← operator pause (tidak ada di WorkflowStage)
  → photoSelection
  → frameSelection
  → filterSelection     ← opsional (tidak ada di WorkflowStage)
  → payment
  → processing
  → delivery
  → completed
```

**Masalah kedua:** Dua state machine yang paralel untuk hal yang sama — satu di Domain layer, satu di Application layer. Keduanya tidak saling sinkronkan. Ini adalah dual authority yang paling berbahaya karena menyangkut Session state.

---

## Capture Lifecycle (Aktual)

```
1. WorkflowOrchestrator.handleIntent(.triggerShutter)
      └── camera.requestCapture(correlationId:)

2. HaiCamera (P2P) mengirim foto ke HaiBooth via P2PMessageRouter

3. SessionStore.startSessionTimer()
      └── previewListenerTask: for await P2PMessageRouter.messageStream(.photoPreview)
            └── photos.receiveThumbnail(thumbnail)
      └── fullPhotoListenerTask: for await P2PMessageRouter.messageStream(.photoFull)
            └── photos.upgradeToFullQuality(photoId:fullData:)
            └── kirim ACK kembali ke HaiCamera

4. Tamu memilih foto → session.proceedToPhotoSelection()

5. Tamu memilih frame → session.proceedToFrameSelection()

6. Tamu lanjut ke payment → session.proceedToPayment()
      └── calculateTotalAmount()  ← BENAR — menggunakan package_.price ✅
      └── payment.preparePayment(amount:method:)
```

**Temuan positif:** `SessionStore.calculateTotalAmount()` sudah menggunakan `package_.price` dengan benar. Hardcoded `35000` di `WorkflowOrchestrator` adalah dead code path yang berbeda — keduanya tidak berkoordinasi.

---

## Payment Commitment Lifecycle (Aktual)

```
PaymentStore.status:
  idle → generatingQRIS → waitingForPayment → paid

WorkflowOrchestrator (via handleIntent):
  paymentRequested → paymentConfirmed

PaymentStatus yang ada di GLOSSARY:
  Pending → Accepted → Verified
```

**Masalah ketiga:** Tiga model payment status yang berbeda — tidak ada yang mengimplementasikan `PaymentCommitment` yang sudah kita definisikan di GLOSSARY. `PaymentStore.paid` ≈ `Accepted`, tapi `Verified` (konfirmasi cloud) sama sekali tidak ada.

---

## Delivery Lifecycle (Aktual)

```
DeliveryStore.status:
  idle → preparingFiles → delivering → delivered
                       → uploading → cloudReady

DeliveryMethod:
  .airdrop      → UIActivityViewController (TODO: Fase 2)
  .localNetwork → BonjourDownloadServer (diimplementasikan)
  .cloudLink    → simulasi progress (TODO: Fase 3)
```

**Masalah keempat:** Delivery Queue tidak persisten. Jika app crash setelah `PaymentAccepted` tapi sebelum delivery selesai, tidak ada mekanisme untuk melanjutkan delivery. `SessionRestoreEngine` sudah ada dan bisa mendeteksi ini (action: `.restoreToDelivery`), tapi data delivery state tidak disimpan ke disk.

---

## Recovery After Crash — Analisis

### Yang sudah ada:

```
Launch → OrphanedSessionDetector.detect()
              └── baca semua file JSONL tanpa Footer
              └── tandai sebagai orphaned

AppState.orphanedSessionDecisions = [OrphanedSessionDecision]
              └── RootView menangani routing

SessionRestoreEngine.decide(for:)
              └── hasFinancialTransaction → restoreToDelivery
              └── tidak ada transaksi → resetToLanding
```

### Gap yang ditemukan:

1. **Delivery state tidak disimpan ke disk.** `DeliveryStore` hanya ada di memory. Setelah crash, `SessionRestoreEngine` bisa tahu "harus restore ke delivery" tapi tidak tahu foto mana yang harus dikirim, via channel apa, dan berapa kali sudah dicoba.

2. **Photo data tidak dijamin ada setelah crash.** `PhotoStore` menyimpan data foto di memory (`[CapturedPhoto]`). Apakah foto di-cache ke disk sebelum crash tidak jelas dari codebase yang ada.

3. **`WorkflowOrchestrator` tidak bisa di-restore ke state sebelum crash.** Setelah app restart, `WorkflowOrchestrator` selalu mulai dari `currentStage = .landing`. Recovery hanya dilakukan via `RootView` dengan routing ke delivery screen — tapi Orchestrator state tidak di-restore.

---

## Queue Persistence — Kondisi Aktual

| Queue | Persisten? | Mekanisme | Gap |
|-------|-----------|-----------|-----|
| Delivery Queue | ❌ Tidak | Memory only | Data hilang saat crash |
| Print Queue | ❌ Tidak ada | Belum diimplementasikan | Seluruh fitur belum ada |
| Retry Queue | ❌ Tidak | `deliveryAttempts: Int` hanya di memory | Reset setelah crash |
| Audit Trail | ✅ Ya | JSONL append-only ke disk | Solid |
| Payment Ledger | ✅ Sebagian | CoreData via `CoreDataService` | Ada tapi tidak terintegrasi ke Session |

---

## Runtime Guarantees — Pemenuhan Aktual

| # | Guarantee | Status | Catatan |
|---|-----------|--------|---------|
| 1 | Session Durability | 🟡 Parsial | Orphan detection ada, tapi restore tidak lengkap |
| 2 | Capture Durability | 🟡 Parsial | Foto diterima via P2P, tapi tidak di-persist ke disk |
| 3 | Queue Persistence | ❌ Belum | Delivery Queue hanya di memory |
| 4 | Payment Commitment | 🟡 Parsial | `paid` state ada, tapi `Verified` belum ada |
| 5 | Audit Completeness | ✅ Solid | JSONL append-only, crash-safe |
| 6 | Event Continuity | 🟡 Parsial | Bergantung pada foto di memory yang bisa hilang |
| 7 | Operator Continuity | ✅ Solid | Operator login terpisah dari Session |
| 8 | Recoverability | 🟡 Parsial | `SessionRestoreEngine` ada, tapi delivery state tidak di-restore |
| 9 | Asset Consistency | ⚠️ Tidak terukur | Belum ada Manifest lock per Session |
| 10 | Deterministic Workflow | 🟡 Parsial | Dua state machine tidak sinkron = non-deterministic |
| 11 | Immutable Session | ❌ Belum | Tidak ada locking mekanisme saat Session aktif |
| 12 | Everything is Observable | ❌ Belum | Domain Event Publisher belum ada |

---

## Temuan Kritis — Dual State Machine

Ini adalah masalah paling mendasar yang harus diselesaikan sebelum apapun.

**Kondisi saat ini:**
- `WorkflowOrchestrator.currentStage` = sumber kebenaran di Domain
- `SessionStore.status` = sumber kebenaran di Application
- Keduanya digunakan secara bergantian oleh View yang berbeda
- Tidak ada mekanisme sinkronisasi

**Contoh inkonsistensi:**
```
WorkflowStage.capturing    ↔    SessionStatus.active
WorkflowStage.editingPreview  ↔  SessionStatus.photoSelection + frameSelection + filterSelection
WorkflowStage.paymentRequested ↔ SessionStatus.payment
```

`editingPreview` di WorkflowOrchestrator mencakup tiga status di SessionStore. Ini berarti Workflow tidak cukup granular untuk menggambarkan pengalaman tamu yang sebenarnya.

---

## Rekomendasi Sebelum Cloud Dibangun

Urutan yang saya usulkan (menunggu konfirmasi GPT):

1. **Tetapkan satu state machine untuk Session** — WorkflowStage atau SessionStatus, bukan keduanya. Rekomendasi: perluas WorkflowStage untuk mencakup semua status yang ada di SessionStatus, lalu hapus SessionStatus.

2. **Persist Delivery state ke disk** — minimal foto ID yang harus dikirim dan channel yang dipilih. Ini mengaktifkan Runtime Guarantee #3.

3. **Persist Capture ke disk segera setelah diterima** — foto ID + path ke file. Ini mengaktifkan Runtime Guarantee #2.

4. **Implementasikan `PaymentCommitment` sesuai GLOSSARY** — `Pending`, `Accepted`, `Verified` sebagai tiga status yang berbeda.

---

## Open Questions untuk GPT

**Q7. Mana yang menjadi satu-satunya state machine untuk Session?**
`WorkflowStage` (Domain) atau `SessionStatus` (Application)? Atau keduanya tetap ada dengan tanggung jawab yang dipisahkan secara eksplisit?

**Q8. Apakah foto harus di-persist ke disk segera setelah diterima dari HaiCamera?**
Ini adalah trade-off antara Runtime Guarantee #2 (Capture Durability) vs performa dan storage.

**Q9. Apakah Delivery Queue harus FIFO, LIFO, atau priority-based?**
Ini menentukan struktur data yang dipakai untuk persistence.

---

*Architecture Review Package #003 — Session Lifecycle*
*Prepared by Antigravity for Chief Product Architect review*
