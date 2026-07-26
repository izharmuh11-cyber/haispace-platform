# System Context — Haispace Platform

**Versi:** 1.0
**Tanggal:** 2026-07-26
**Audience:** Engineer baru, Cloud team, Operator team, AI team

> Dokumen ini adalah pintu pertama yang dibuka engineer baru sebelum menyentuh satu baris kode.

---

## Gambaran Besar

Haispace adalah **Platform Runtime** yang mengelola sesi foto booth dari ujung ke ujung:
dari tamu datang, memilih paket, berfoto, membayar, hingga menerima hasil foto.

Semua produk Haispace (Booth, Camera, Admin, AI Assistant, Cloud) adalah **konsumen** dari Platform Runtime ini.

---

## System Context Diagram

```
                        ┌─────────────┐
                        │   Operator  │
                        │  (Manusia)  │
                        └──────┬──────┘
                               │ Interaksi via
                               │ Mission Control / iPad
                               ▼
┌──────────────────────────────────────────────────────────┐
│                      SwiftUI Layer                       │
│                 (render state, no logic)                 │
└──────────────────────┬───────────────────────────────────┘
                       │ intent / state
                       ▼
┌──────────────────────────────────────────────────────────┐
│                       AppState                           │
│      (thin-wrapper: meneruskan intent, pantulkan state)  │
└──────────────────────┬───────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│                  RuntimeContainer                        │
│             (Composition Root — Module Orchestrator)     │
│                                                          │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐ │
│  │SessionModule │ │CapabilityMod │ │InfrastructureMod │ │
│  │              │ │              │ │                  │ │
│  │ Repository   │ │ Camera       │ │ Clock            │ │
│  │ Factory      │ │ Printer      │ │ Persistence      │ │
│  │              │ │ Payment      │ │ Compatibility    │ │
│  └──────────────┘ │ Editing      │ └──────────────────┘ │
│                   │ Delivery     │ ┌──────────────────┐ │
│                   │ P2P          │ │ ObservabilityMod │ │
│                   └──────────────┘ │                  │ │
│                                    │ EventPublisher   │ │
│  ┌─────────────────────────────┐   │ AuditSubscriber  │ │
│  │    WorkflowOrchestrator     │   │ CompatMonitor    │ │
│  │  (business process engine)  │   │ MissionControl   │ │
│  └─────────────────────────────┘   │ Analytics        │ │
│                                    └──────────────────┘ │
└──────────────────────┬───────────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
  ┌───────────┐ ┌───────────┐ ┌───────────────┐
  │  Domain   │ │Capability │ │Infrastructure │
  │  Layer    │ │  Layer    │ │    Layer      │
  │           │ │           │ │               │
  │ Session   │ │ Camera    │ │ LocalSession  │
  │ Aggregate │ │ Impl      │ │ Repository    │
  │           │ │           │ │               │
  │ Domain    │ │ Printer   │ │ SessionSnap   │
  │ Events    │ │ Impl      │ │ shot          │
  │           │ │           │ │               │
  │ Session   │ │ Payment   │ │ Compat        │
  │ Factory   │ │ Gateway   │ │ Checker       │
  └───────────┘ └─────┬─────┘ └───────────────┘
                      │
                      │ Device / Hardware
                      ▼
         ┌────────────────────────────┐
         │     Physical Devices       │
         │                            │
         │  HaiCamera (P2P / WiFi)    │
         │  Epson Printer             │
         │  QRIS Terminal / NFC       │
         │  iPad Display              │
         └────────────────────────────┘
```

---

## External Systems Context

```
                  ┌─────────────────────────────────────────┐
                  │           Platform Runtime               │
                  │          (Booth / iPad)                  │
                  └──────────────┬──────────────────────────┘
                                 │
           ┌─────────────────────┼───────────────────────┐
           │                     │                       │
           ▼                     ▼                       ▼
  ┌────────────────┐   ┌─────────────────┐   ┌──────────────────┐
  │  Mission       │   │   Cloud API     │   │  Asset CDN       │
  │  Control       │   │                 │   │                  │
  │  (Operator     │   │  Session Sync   │   │  Frame assets    │
  │   Dashboard)   │   │  Recovery data  │   │  Filter packages │
  │                │   │  Analytics      │   │  Manifest files  │
  └────────────────┘   └────────┬────────┘   └──────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
  ┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐
  │  Payment         │ │  Analytics   │ │  AI Service      │
  │  Provider        │ │  Pipeline    │ │  (future)        │
  │                  │ │              │ │                  │
  │  QRIS (EMVCo)    │ │  Session     │ │  Auto-framing    │
  │  Midtrans        │ │  funnel      │ │  Enhancement     │
  │  Xendit          │ │  conversion  │ │  Recommendation  │
  └──────────────────┘ └──────────────┘ └──────────────────┘
```

---

## Produk dan Konsumen Runtime

| Produk | Platform | Runtime Role |
|---|---|---|
| **HaispaceBooths** (iPad) | iOS | Booth Runtime — implementasi pertama Platform Runtime |
| **HaiCamera** (iPhone/iPod) | iOS | Camera Runtime — capture-only capability profile |
| **Mission Control** (iPad / Mac) | iOS / macOS | Admin Runtime — management capability profile |
| **Cloud API** | Server | Cloud Contract Consumer |
| **AI Assistant** | Server / iOS | AI Capability Subscriber |
| **Operator Dashboard** | Web | REST consumer dari Cloud API |
| **Admin CMS** | Web | REST consumer dari Cloud API |

---

## Alur Sesi (Happy Path)

```
Operator menyalakan Booth
          │
          ▼
RuntimeContainer.build(for: .production)
          │
          ▼
          ┌── performLaunchRecovery()
          │   └── ada session in-progress? → RestoreSession (Phase C)
          │
          ▼
Tamu datang → Landing Screen
          │
          ▼
PackageSelection → Tamu pilih paket
          │
          ▼
SessionFactory.createSession(guest, package)
          │
          ▼
WorkflowOrchestrator.handle(.startCapture)
          │
          ├── Camera.capture() via CapabilityModule
          ├── HaispaceSession.recordCapture()
          └── SessionRepository.save(snapshot) ← atomic disk write
          │
          ▼
WorkflowOrchestrator.handle(.requestPayment)
          │
          ├── Payment.request() via CapabilityModule
          ├── HaispaceSession.acceptPayment()
          ├── SessionRepository.save(snapshot) ← sebelum delivery
          └── CompatibilityChecker.check() → publish CompatibilityEvent
          │
          ▼
WorkflowOrchestrator.handle(.startDelivery)
          │
          ├── Editing.composite() via CapabilityModule
          ├── Delivery.deliver() via CapabilityModule
          └── HaispaceSession.complete()
          │
          ▼
DomainEventPublisher.flush(from: session)
          │
          ├── AuditSubscriber → SessionAuditTrail
          ├── MissionControlSubscriber → operator sees session complete
          └── AnalyticsSubscriber → funnel event recorded
          │
          ▼
Session selesai → Landing Screen (next guest)
```

---

## Batas Tanggung Jawab yang Keras

| Aturan | Detail |
|---|---|
| SwiftUI tidak boleh mengakses Repository | Semua akses melalui WorkflowOrchestrator |
| AppState tidak boleh mengandung business logic | Hanya meneruskan intent dan memantulkan state |
| WorkflowOrchestrator tidak boleh tahu device mana | Hanya meminta capability melalui CapabilityModule/Manager |
| Repository hanya tahu: save/load/exists/delete | Tidak boleh tahu Workflow, Payment, Cloud |
| Session Aggregate tidak boleh memanggil external service | Aggregate adalah pure domain object |
| Semua timestamp dari RuntimeClock | Tidak ada `Date()` langsung di Domain atau Runtime |

---

## File Penting untuk Dibaca Pertama

1. [`PLATFORM_RUNTIME_V1.md`](./PLATFORM_RUNTIME_V1.md) — Konstitusi platform
2. [`../adr/ADR-011-platform-runtime-freeze.md`](../adr/ADR-011-platform-runtime-freeze.md) — Keputusan freeze
3. [`../adr/ADR-009-session-aggregate-repository.md`](../adr/ADR-009-session-aggregate-repository.md) — Session Aggregate
4. [`../architecture/MIGRATION_DASHBOARD.md`](../architecture/MIGRATION_DASHBOARD.md) — Status migrasi saat ini
5. [`../architecture/MIGRATION_PLAN.md`](../architecture/MIGRATION_PLAN.md) — Roadmap migrasi

---

*SYSTEM_CONTEXT.md — Versi 1.0*
*Diperbarui setiap kali ada perubahan signifikan pada System Context.*
