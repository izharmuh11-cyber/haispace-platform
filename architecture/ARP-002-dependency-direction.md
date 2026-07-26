# Architecture Review Package #002
## Dependency Direction & Layer Analysis

**Prepared by:** Antigravity (Lead Software Architect)
**For review by:** GPT (Chief Product Architect)
**Date:** 2026-07-26
**Follows:** ARP-001 (Workflow Runtime)

---

## Purpose

Menjawab permintaan GPT untuk melihat **dependency direction** aktual dan mengidentifikasi dependency yang salah arah — sebelum memutuskan refactor mana yang paling mendesak.

---

## Actual Dependency Map (Current State)

```
┌─────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                   │
│                                                         │
│  SwiftUI Views  ─────────────────────────────────────── │
│  (Guest, Operator, DesignSystem, Components)            │
└─────────────┬───────────────────────────────────────────┘
              │ reads @Environment
              ▼
┌─────────────────────────────────────────────────────────┐
│                  APPLICATION LAYER                      │
│                                                         │
│  AppState (MainActor, @Observable)                      │
│    ├── AuthStore                                        │
│    ├── LicenseStore                                     │
│    ├── P2PStore           ← ⚠️  lihat catatan #1        │
│    ├── BoothConfigStore                                 │
│    ├── OperatorStore                                    │
│    └── WorkflowOrchestrator                            │
│                                                         │
│  WorkflowRouteMapper (stateless bridge)                 │
└─────────────┬───────────────────────────────────────────┘
              │ calls handleIntent / reads currentStage
              ▼
┌─────────────────────────────────────────────────────────┐
│                    DOMAIN LAYER                         │
│                                                         │
│  WorkflowOrchestrator (Swift Actor)                     │
│    ├── WorkflowStage (state machine)                    │
│    ├── WorkflowIntent (command)                         │
│    └── WorkflowEvent (sparse, unused)                   │
│                                                         │
│  SessionAuditTrail (static) ← ⚠️  lihat catatan #2     │
│  OrphanedSessionDetector (static)                       │
└─────────────┬───────────────────────────────────────────┘
              │ calls protocol methods
              ▼
┌─────────────────────────────────────────────────────────┐
│                 CAPABILITY LAYER                        │
│                                                         │
│  CameraCapabilityProtocol ──── CameraCapability         │
│  EditingCapabilityProtocol ─── EditingCapability        │
│  PaymentCapabilityProtocol ─── PaymentCapability        │
│  DeliveryCapabilityProtocol ── DeliveryCapability       │
│  P2PCapabilityProtocol ─────── P2PCapability            │
│                         └───── NoOp* (safe defaults)   │
└─────────────┬───────────────────────────────────────────┘
              │ calls runtime implementations
              ▼
┌─────────────────────────────────────────────────────────┐
│               INFRASTRUCTURE LAYER                      │
│                                                         │
│  AVFoundation (Camera)                                  │
│  CoreImage (Editing)                                    │
│  MultipeerConnectivity (P2P)                            │
│  Local QRIS (Payment)                                   │
│  Bonjour/WiFi (Delivery)                                │
│  FileSystem / UserDefaults (Persistence)                │
│  CoreData (Session, Config)                             │
└─────────────────────────────────────────────────────────┘
              │
              ▼ (future)
┌─────────────────────────────────────────────────────────┐
│                    CLOUD LAYER                          │
│                                                         │
│  Vultr API                                              │
│  Cloudflare R2 (Asset Storage)                          │
│  Analytics Service                                      │
│  Manifest Service                                       │
└─────────────────────────────────────────────────────────┘
```

**Dependency rule yang seharusnya:**
> Setiap layer hanya boleh bergantung ke layer di bawahnya. Tidak pernah ke atas.

---

## Dependency Violations (Yang Salah Arah)

### ⚠️ Violation #1 — P2PStore ada di Application Layer, P2PCapability ada di Capability Layer

```
AppState
  ├── P2PStore (Application Layer)    ← mengontrol UI state P2P
  └── WorkflowOrchestrator
        └── P2PCapabilityProtocol     ← mengontrol transfer data P2P
```

**Masalah:** Dua komponen yang mengurus P2P dengan concern berbeda, tidak ada koordinasi jelas.
`P2PStore` mengelola connection state untuk UI (latency, peer name, battery).
`P2PCapability` mengelola business operation (transfer foto).

**Idealnya:** Satu source of truth — P2PCapability adalah authoritative, P2PStore hanya membaca snapshot-nya.

---

### ⚠️ Violation #2 — SessionAuditTrail di Domain Layer menggunakan static methods

```
WorkflowOrchestrator (Domain Layer)
  └── SessionAuditTrail.create()       ← static call ke file I/O
  └── SessionAuditTrail.append()       ← static call ke file I/O
```

**Masalah:** Domain Layer seharusnya tidak memanggil Infrastructure langsung. `SessionAuditTrail` adalah Infrastructure (file system), bukan Domain. Dependency arahnya terbalik.

**Idealnya:**
```
WorkflowOrchestrator
  └── AuditTrailProtocol (abstraction)
        └── impl: SessionAuditTrail (Infrastructure)
        └── impl: CloudAuditTrail (future)
        └── impl: MockAuditTrail (testing)
```

---

### ⚠️ Violation #3 — BoothPackage ada di Models Layer, bukan Domain Layer

```
Models/BoothPackage.swift        ← Model layer (generic)
WorkflowOrchestrator             ← Domain layer
AppState.startNewSession(        ← Application layer
  package: BoothPackage.mockStandard   ← menggunakan mock di production path
)
```

**Masalah:** `BoothPackage` seharusnya adalah Domain Entity dengan lifecycle (Draft → Published → Active → Archived). Saat ini ia adalah plain struct tanpa lifecycle dan tanpa hubungan eksplisit ke `WorkflowOrchestrator`.

**Lebih parah:** `PaymentAmount(amountValue: 35000)` hardcoded di `WorkflowOrchestrator` mengabaikan `BoothPackage.price` yang sudah ada di codebase. Package sudah punya `price: Int` — tapi tidak dipakai.

---

### ⚠️ Violation #4 — Observability Layer bergantung ke Capability Layer secara tidak langsung

```
HealthAggregator (Observability)
  ├── reads: camera.healthSnapshot
  ├── reads: p2pHealth (dari P2PStore)
  ├── reads: payment.healthSnapshot
  └── reads: delivery.healthSnapshot
```

`HealthAggregator` harus memanggil capability secara langsung untuk mendapatkan health snapshot. Ini coupling yang sulit ditest dan tidak ada abstraksi di antaranya.

---

### ⚠️ Violation #5 — WorkflowEvent tidak terhubung ke Domain Events

```
// WorkflowSharedTypes.swift
enum WorkflowEvent: Sendable {
    case sessionCreated       ← ada tapi tidak diterbitkan
    case photoCaptured        ← ada tapi tidak diterbitkan
    case paymentCompleted     ← ada tapi tidak diterbitkan
    ...
}

// Apa yang benar-benar dipakai:
enum AuditEventType: String, Codable {
    case sessionStarted       ← ditulis ke file JSONL
    case paymentConfirmed     ← ditulis ke file JSONL
    ...
}
```

Ada dua event system yang paralel dan tidak terhubung. `WorkflowEvent` adalah domain event yang ideal tapi orphan. `AuditEventType` adalah persistence event yang berfungsi tapi terisolasi.

---

## Capture Policy vs Layout vs Theme — Analisis

Merespons rekomendasi GPT untuk memisahkan ketiga konsep ini.

**Kondisi saat ini di BoothPackage:**
```swift
struct BoothPackage {
    let maxPhotoCount: Int      // ← Capture Policy
    let minPhotoCount: Int      // ← Capture Policy
    let intervalSeconds: Int    // ← Capture Policy
    let durationSeconds: Int    // ← Capture Policy
    let price: Int              // ← Commerce
    let includedAddonIds: [String]  // ← Policy (tidak terstruktur)
    // Layout: TIDAK ADA
    // Theme: TIDAK ADA
    // Frame: hanya di BoothConfigStore.availableFrames
    // Filter: hanya di AddonType.filter
}
```

**Yang diusulkan GPT (dan saya setuju):**
```
Package
  ├── CapturePolicy
  │     ├── maxPhotoCount
  │     ├── minPhotoCount
  │     ├── countdownSeconds
  │     ├── intervalSeconds
  │     ├── retakePolicy (allowed/forbidden/limited)
  │     └── durationSeconds
  │
  ├── Layout (reference to Layout entity)
  │
  ├── Theme (reference to Theme entity)
  │
  └── CommercePolicy
        ├── price
        ├── paymentMethods
        ├── printingPolicy
        └── deliveryPolicy
```

Refactor ini **tidak mengganggu Runtime** karena `WorkflowOrchestrator` sudah menerima Package via `selectPackage(packageId: String)` — bukan struct langsung. Yang perlu diubah adalah bagaimana Orchestrator membaca property Package.

---

## Target Architecture (Yang Harus Dicapai)

```
┌──────────────────────────────────┐
│          Presentation            │
│    Views hanya render state,     │
│    kirim intent                  │
└──────────┬───────────────────────┘
           │ Intent
           ▼
┌──────────────────────────────────┐
│           Application            │
│  AppState — koordinasi saja,     │
│  tidak ada business logic        │
└──────────┬───────────────────────┘
           │ handleIntent
           ▼
┌──────────────────────────────────┐
│            Domain                │
│  WorkflowOrchestrator            │
│  + Domain Event Publisher        │  ← perlu dibangun
│  + Package Runtime (harga etc)   │  ← perlu dipisahkan dari mock
└──────┬───────────────────────────┘
       │ protocol calls      │ publish events
       ▼                     ▼
┌──────────────┐    ┌────────────────────────────────┐
│  Capability  │    │      Event Subscribers          │
│   Layer      │    │  AuditTrail  (file)             │
│  (Protocol)  │    │  Analytics   (batch upload)     │
│              │    │  MissionCtrl (real-time UI)     │
└──────┬───────┘    │  CloudSync   (future)           │
       │            │  AI Assistant (future)           │
       ▼            └────────────────────────────────┘
┌──────────────┐
│Infrastructure│
│ AVFoundation │
│ CoreImage    │
│ FileSystem   │
│ CoreData     │
└──────────────┘
```

---

## Recommended Refactor Order (dengan justifikasi)

| Prioritas | Item | Alasan |
|-----------|------|--------|
| 1 | **Domain Event Publisher** | Semua subscriber menunggu ini. Tidak ada yang bisa dibangun dengan benar tanpa ini |
| 2 | **Payment dari Package** | Bug bisnis aktif — harga bisa salah. Risk tinggi |
| 3 | **AuditTrailProtocol** | Hapus static, buka jalan untuk CloudAudit & testing |
| 4 | **CapturePolicy extraction** | Enabler untuk PhotoSelection flow yang benar |
| 5 | **Mock isolation** | Hygiene — cegah data palsu masuk production |

---

## Open Question (Tambahan dari ARP-001)

**Q6. Domain Event Publisher — Sync atau Async?**

Dua pilihan teknis untuk event delivery:

| Pilihan | Mekanisme | Trade-off |
|---------|-----------|-----------|
| A | `AsyncStream` per subscriber | Native Swift Concurrency, per-consumer backpressure, no external dependency |
| B | `NotificationCenter` | Familiar, tapi tidak type-safe dan tidak async-native |
| C | Custom `EventBus` actor | Paling fleksibel, tapi lebih banyak code |

**Rekomendasi saya: A (AsyncStream)** — sesuai model Swift Actor yang sudah ada, type-safe, tidak perlu library external.

GPT perlu konfirmasi sebelum saya mulai implementasi.

---

*Architecture Review Package #002 — Dependency Direction & Layer Analysis*
*Prepared by Antigravity for Chief Product Architect review*
