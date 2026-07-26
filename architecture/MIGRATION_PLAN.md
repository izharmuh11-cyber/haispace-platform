# Migration Roadmap: Legacy Stores to Session Aggregate

**Author:** Antigravity (Lead Software Architect)
**Approved by:** GPT (Chief Product Architect)
**Strategy:** Strangler Fig Pattern & 4-Step Compatibility Window

---

## 🎯 High-Level Goal

Migrasikan seluruh state management `HaispaceBooths` dari empat `@Observable` store lama (`SessionStore`, `PaymentStore`, `DeliveryStore`, `PhotoStore`) ke `HaispaceSession` (Aggregate Root) dan `SessionRepository` secara **bertahap, tanpa Big Bang, dan aman tanpa breaking changes**.

---

## 📊 Compatibility Coverage — Current Status

> Metrik ini diperbarui setiap PR selesai.
> Target akhir: **0 mismatch selama 2 minggu berturut-turut** sebelum Legacy Store aman dihapus.

```
Bounded Context   Coverage            Status
──────────────────────────────────────────────────────────────
Payment           ████░░░░░░  40%     PR-01 ✅  PR-02 ✅  PR-03 ⏳  PR-04 ⏳
Capture           ░░░░░░░░░░   0%     PR-05 ⏳  PR-06 ⏳  PR-07 ⏳  PR-08 ⏳
Delivery          ░░░░░░░░░░   0%     PR-09 ⏳  PR-10 ⏳  PR-11 ⏳  PR-12 ⏳
Session Root      ░░░░░░░░░░   0%     PR-13 ⏳  PR-14 ⏳  PR-15 ⏳
```

**Legenda Coverage:**
- `40%` = Shadow Write + Divergence Detection aktif, belum Read Switch
- `70%` = Read Switch aktif (Aggregate jadi primary read)
- `100%` = Legacy Store dihapus (sepenuhnya clean)

---

## 🔄 4-Step Compatibility Window Pattern

Untuk setiap bounded context (Payment, Capture, Delivery, Session), migrasi dilakukan dengan 4 tahap terkontrol:

```
[PR Step 1: Shadow Write]  → Tulis ke Aggregate TERLEBIH DAHULU, lalu update Store Lama (Adapter)
[PR Step 2: Read Compare] → Baca dari Aggregate, cocokkan dengan Store Lama, log jika ada beda
[PR Step 3: Read Switch]  → Hentikan pembacaan dari Store Lama, pembacaan 100% dari Aggregate
[PR Step 4: Cleanup]      → Hapus Store Lama sepenuhnya dari codebase
```

---

## 🛣️ Phased Migration Roadmap

```
Phase B.1 (Payment)    ──►  Phase B.2 (Capture)    ──►  Phase B.3 (Delivery)    ──►  Phase B.4 (Session Root)
[PR-01 to PR-04]            [PR-05 to PR-08]            [PR-09 to PR-12]            [PR-13 to PR-15]
```

---

### Phase B.1 — Payment Bounded Context

| PR | Judul | Target Impact | Status |
|---|---|---|---|
| **PR-01** | **Payment Shadow Write & SessionFactory** | • Buat `SessionFactory` (create/restore/migrate stubs)<br>• Enriched `PaymentCommitment` + `PaymentMetadata`<br>• `Workflow.confirmPayment` → `Session.acceptPayment()` → `SessionRepository.save()` + Legacy update | ✅ Done |
| **PR-02** | **Payment Read Compare + Divergence Detection** | • `PaymentCompatibilityChecker` (5 fields + timestamp delta)<br>• `CompatibilityEvent.matched/mismatched` dipublikasikan<br>• Log per-field delta ke HaispaceLogger[compatibility] | ✅ Done |
| **PR-03** | **Payment Read Switch** | • SwiftUI Payment View 100% mengonsumsi `HaispaceSession`<br>• Hentikan pembacaan dari `PaymentStore` | ⏳ Pending |
| **PR-04** | **PaymentStore Cleanup** | • Hapus `PaymentStore.swift`<br>• Refactor tests yang masih mengacu pada `PaymentStore` | ⏳ Pending |

---

### Phase B.2 — Capture Bounded Context

| PR | Judul | Target Impact | Status |
|---|---|---|---|
| **PR-05** | **Capture Shadow Write** | • `PhotoStore` menjadi wrapper di sekitar `HaispaceSession.captures`<br>• P2P receipt langsung mendaftarkan `CaptureRecord` ke Session Aggregate | ⏳ Pending |
| **PR-06** | **Capture Read Compare** | • Selection View membaca dari `HaispaceSession.captures`<br>• Cross-check dengan `PhotoStore.selectedPhotoIds` | ⏳ Pending |
| **PR-07** | **Capture Read Switch** | • Grid photo selection & preview 100% membaca dari `HaispaceSession` | ⏳ Pending |
| **PR-08** | **PhotoStore Cleanup** | • Hapus `PhotoStore.swift` | ⏳ Pending |

---

### Phase B.3 — Delivery Bounded Context

| PR | Judul | Target Impact | Status |
|---|---|---|---|
| **PR-09** | **Delivery Shadow Write** | • `DeliveryStore` enqueue ke `HaispaceSession.deliveryState`<br>• Auto-persist snapshot saat item di-queue | ⏳ Pending |
| **PR-10** | **Delivery Read Compare** | • Status UI delivery membaca dari `HaispaceSession.deliveryState` | ⏳ Pending |
| **PR-11** | **Delivery Read Switch** | • AirDrop & Bonjour Server membaca dari Session Aggregate | ⏳ Pending |
| **PR-12** | **DeliveryStore Cleanup** | • Hapus `DeliveryStore.swift` | ⏳ Pending |

---

### Phase B.4 — Session & AppState Root Cleanup

| PR | Judul | Target Impact | Status |
|---|---|---|---|
| **PR-13** | **Workflow & Session Integration** | • `WorkflowOrchestrator` memegang `HaispaceSession` via `SessionRepository`<br>• `SessionStore` dikurangi menjadi facade | ⏳ Pending |
| **PR-14** | **AppState CurrentSession Switch** | • `AppState.currentSession` tipe berubah dari `SessionStore?` ke `HaispaceSession?` | ⏳ Pending |
| **PR-15** | **SessionStore Cleanup & Phase B Completion** | • Hapus `SessionStore.swift`<br>• **Milestone 2 Phase B Definition of Done Tercapai** 🎉 | ⏳ Pending |

---

## 📊 Definition of Done Matrix (Phase B Completion)

- [ ] **Criteria 1**: Workflow hanya membaca `SessionRepository`
- [ ] **Criteria 2**: `SessionSnapshot` ter-persist otomatis ke disk pada setiap mutasi
- [ ] **Criteria 3**: Restart aplikasi 100% mengembalikan `HaispaceSession` Aggregate
- [ ] **Criteria 4**: Tidak ada state session yang hilang saat crash
- [ ] **Criteria 5**: Keempat store lama (`SessionStore`, `PaymentStore`, `DeliveryStore`, `PhotoStore`) terhapus dari codebase
