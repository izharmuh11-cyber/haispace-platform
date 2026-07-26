# Architecture Review Package #001
## Workflow Runtime — HaispaceBooths

**Prepared by:** Antigravity (Lead Software Architect)
**For review by:** GPT (Chief Product Architect)
**Date:** 2026-07-26
**Codebase:** `hsp-internal/HaispaceBooths`

---

## Purpose

Workflow Runtime adalah jantung HaiBooth. Ia bertanggung jawab untuk mengorkestrasi seluruh alur bisnis dari tamu masuk hingga foto terkirim — mengkoordinasikan lima Business Capabilities: Camera, Editing, Payment, Delivery, dan P2P.

---

## Current Architecture

```
SwiftUI View Layer
      │
      │ send(WorkflowIntent)
      ▼
   AppState  ──────────────────────────────────────┐
   (MainActor, @Observable)                        │
      │                                            │
      │ handleIntent(_:)                    currentRoute
      ▼                                   (WorkflowRouteMapper)
WorkflowOrchestrator  ──── SessionAuditTrail (JSONL append-only)
   (Swift Actor)
      │
      ├── CameraCapabilityProtocol
      ├── EditingCapabilityProtocol
      ├── PaymentCapabilityProtocol
      ├── DeliveryCapabilityProtocol
      └── P2PCapabilityProtocol
```

**Prinsip yang sudah diterapkan:**
- View hanya memanggil `send(intent)` — tidak pernah memanipulasi route langsung
- `WorkflowOrchestrator` adalah `actor` (thread-safe, sendable)
- Semua capability diakses via protocol (testable, injectable)
- `WorkflowRouteMapper` adalah satu-satunya bridge antara domain dan UI route

---

## Related Files

| File | Lokasi | Peran |
|------|--------|-------|
| `WorkflowOrchestrator.swift` | `Core/Workflow/` | State machine utama, koordinasi capability |
| `WorkflowSharedTypes.swift` | `Core/Workflow/` | WorkflowStage, WorkflowIntent, value objects |
| `WorkflowOrchestratorProtocol.swift` | `Core/Workflow/` | Kontrak publik orchestrator |
| `WorkflowRouteMapper.swift` | `Core/Workflow/` | Bridge WorkflowStage → KioskRoute (UI) |
| `AppState.swift` | `Core/State/` | Root singleton, intent dispatcher, session factory |
| `SessionAuditTrail.swift` | `Core/Audit/` | JSONL append-only event log per-session |
| `OrphanedSessionDetector.swift` | `Core/Audit/` | Deteksi crash session saat launch |
| `NoOpCapabilities.swift` | `Core/Capabilities/` | Safe default sebelum setup() selesai |
| `MissionControlSnapshot.swift` | `Core/Observability/` | Snapshot untuk operator dashboard |
| `HealthAggregator.swift` | `Core/Observability/` | Agregasi health semua capability |

---

## Runtime Flow (Actual Implementation)

```
Landing
  │
  │ .startGuestRegistration
  ▼
Guest Registration
  │
  │ .guestSubmittedInfo(name, email)
  ▼
Package Selection
  │
  │ .selectPackage(packageId)
  │   → camera.prepare() + camera.startSession()   ← capability disiapkan di sini
  ▼
Capturing
  │
  │ .triggerShutter
  │   → camera.requestCapture()
  ▼
Editing Preview        ← tamu bisa pilih filter (.selectFilter)
  │
  │ .acceptPreview
  │   → editing.requestExport()
  │   → payment.prepare() + payment.requestPayment()
  ▼
Payment Requested
  │
  │ .confirmPaymentSuccess
  │   → delivery.prepare() + delivery.requestDelivery()
  ▼
Delivery Dispatch
  │
  │ .finishSession
  ▼
Landing (reset)
```

**Catatan kritis:** `selectTemplate` (frame selection) terjadi **setelah** Package Selected tapi **sebelum** Capture. Ini sudah sesuai dengan workflow baru yang diminta ("foto dulu, frame belakangan" **belum** sepenuhnya diimplementasikan — lihat Technical Debt).

---

## Current Responsibilities

### Yang diurus Workflow:
- Mengelola `currentStage` (state machine)
- Memanggil `prepare()` dan `start/stop` pada setiap Capability
- Menulis ke `SessionAuditTrail` di setiap transisi penting
- Memutuskan recovery strategy saat `cancelSessionByOperator`
- Mem-`processEvent` dari external event bus (Payment.Confirmed, Delivery.Completed)
- Reset semua capability saat `resetToLanding()`

### Yang TIDAK boleh diurus Workflow (dan sudah dijaga):
- UI routing — diurus oleh `WorkflowRouteMapper` + `AppState`
- Session factory — diurus oleh `AppState.startNewSession()`
- Health monitoring — diurus oleh `HealthAggregator`
- Incident tracking — diurus oleh `IncidentEngine`

---

## Dependencies

```
WorkflowOrchestrator
│
├── CameraCapabilityProtocol
│     └── impl: CameraCapability (AVFoundation) / NoOpCameraCapability
│
├── EditingCapabilityProtocol
│     └── impl: EditingCapability (CoreImage) / NoOpEditingCapability
│
├── PaymentCapabilityProtocol
│     └── impl: PaymentCapability (QRIS/local) / NoOpPaymentCapability
│
├── DeliveryCapabilityProtocol
│     └── impl: DeliveryCapability (Bonjour/WiFi) / NoOpDeliveryCapability
│
└── P2PCapabilityProtocol
      └── impl: P2PCapability (MultipeerConnectivity) / NoOpP2PCapability

WorkflowOrchestrator reads:
└── SessionAuditTrail (static methods — file I/O)

AppState depends on:
├── WorkflowOrchestrator
├── WorkflowRouteMapper (static)
├── SessionAuditTrail (static)
└── OrphanedSessionDetector (static)
```

---

## Known Technical Debt

### 🔴 Critical

**1. Workflow tidak sepenuhnya mencerminkan new flow "foto dulu, frame belakangan"**

`WorkflowStage` masih memiliki `templateSelection` yang muncul *sebelum* `capturing`. Dalam diskusi produk, flow yang diminta adalah:
```
Package → Capture → PhotoSelection → FrameSelection → Payment
```
Tapi `WorkflowOrchestrator.handleIntent(.selectTemplate)` sudah memanggil `editing.prepare()` dan `requestExport()` langsung — artinya frame dipilih dan foto di-export sekaligus, bukan frame dipilih setelah foto dipilih.

**PhotoSelection stage belum ada.** Tamu belum bisa memilih foto mana yang ingin digunakan dari beberapa foto yang diambil.

---

**2. AppState menyimpan Session Factory logic dengan mock data**

```swift
// AppState.swift line 116
let pkg = BoothPackage.mockStandard   // ← mock data di production code path
```

Ini berbahaya. Package yang dipakai dalam Session seharusnya datang dari Manifest/Package yang dipilih operator, bukan dari `mockStandard`.

---

**3. Payment masih hardcoded amount**

```swift
// WorkflowOrchestrator.swift
amount: PaymentAmount(amountValue: 35000, method: .localQRIS)
```

Amount tidak datang dari Package. Ini melanggar Principle 4 (Configuration via Manifest).

---

### 🟡 Medium

**4. `WorkflowOrchestrator` melakukan terlalu banyak dalam satu `handleIntent`**

Contoh `.acceptPreview`:
- Export foto
- Prepare payment
- Request payment
- Update audit trail
- Update stage

Ini adalah Sequential Coupling yang rawan gagal di tengah jalan tanpa proper recovery. Seharusnya ini dipecah menjadi stage eksplisit.

---

**5. `SessionAuditTrail` menggunakan static methods (global state)**

```swift
SessionAuditTrail.create(sessionId:)
SessionAuditTrail.append(sessionId:stage:eventType:)
```

Tidak bisa di-inject, tidak bisa di-test dengan isolasi. Ini harus diubah menjadi injectable dependency.

---

**6. `WorkflowEvent` enum belum terhubung ke Domain Events**

`WorkflowEvent` di `WorkflowSharedTypes.swift` hanya punya 7 case dan tidak dipakai secara aktif. `AuditEventType` di `SessionAuditTrail` adalah event bus yang sesungguhnya — tapi hanya untuk audit, tidak untuk observability platform.

Platform Domain Events yang sudah kita definisikan di `haispace-platform` belum diimplementasikan sama sekali di codebase.

---

**7. `navigateTo()` deprecated bridge masih ada**

```swift
@available(*, deprecated) func navigateTo(_ route: KioskRoute)
```

Artinya masih ada View yang belum migrasi ke `send(intent:)`. View mana saja belum diketahui tanpa audit lebih lanjut.

---

### 🟢 Good — Pertahankan

- `WorkflowOrchestratorProtocol` sudah ada → testable dan injectable
- `WorkflowRouteMapper` stateless dan exhaustive → compiler guard untuk regression
- `SessionAuditTrail` append-only JSONL → sesuai Runtime Guarantee #5
- `NoOpCapabilities` pattern → safe default, tidak crash saat setup belum selesai
- Semua Capability diakses via Protocol → platform-independent
- `OrphanedSessionDetector` sudah ada → Runtime Guarantee #8 (Recoverability) sebagian terpenuhi

---

## Open Questions

**Q1. Siapa yang memiliki Package data?**
Saat ini Package datang dari `BoothConfigStore.activePackages` yang di-load dari lokal. Belum ada mekanisme sync dari Cloud. Apakah Package termasuk dalam Manifest, atau service terpisah?

**Q2. Bagaimana flow PhotoSelection yang benar?**
Tamu mengambil berapa foto? Semua langsung diproses, atau tamu memilih dulu mana yang dipakai? Ini menentukan apakah `editingPreview` perlu dipecah menjadi `photoSelection` + `frameSelection`.

**Q3. Apakah `SessionAuditTrail` adalah implementasi dari Domain Events?**
Atau keduanya harus hidup berdampingan? Saat ini AuditTrail hanya untuk persistence lokal, bukan untuk konsumsi oleh Analytics/AI/Mission Control secara real-time.

**Q4. Kapan P2P diinisialisasi?**
P2P disetup di `AppState` level (via `P2PStore`) tapi juga ada `P2PCapabilityProtocol` di `WorkflowOrchestrator`. Dua jalur ini perlu diklarifikasi — mana yang authoritative?

**Q5. Bagaimana Manifest akan masuk ke Workflow?**
Saat ini Workflow tidak tahu tentang Manifest. Package dikonfigurasi secara statis. Ini perlu didesain sebelum Asset Platform dibangun.

---

## Summary untuk Architecture Audit

| Area | Status | Catatan |
|------|--------|---------|
| State machine structure | ✅ Solid | Actor-based, protocol-driven |
| UI separation | ✅ Good | RouteMapper memisahkan domain dari UI |
| Audit trail | ✅ Good | Append-only JSONL, crash-safe |
| Recovery detection | ✅ Partial | Orphan detection ada, resume belum |
| Domain Events | ❌ Missing | Belum ada publisher/consumer |
| Manifest integration | ❌ Missing | Config masih hardcoded/mock |
| Photo selection flow | ❌ Incomplete | Stage belum ada |
| Payment amount | ❌ Hardcoded | Melanggar Principle 4 |
| Session factory | 🟡 Risk | Mock data di production path |
| Dependency injection (Audit) | 🟡 Risk | SessionAuditTrail masih static |

---

*Architecture Review Package #001 — Workflow Runtime*
*Prepared by Antigravity for Chief Product Architect review*
