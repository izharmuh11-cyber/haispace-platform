# Platform Runtime v1.0 — Constitution

**Status:** FROZEN ❄️
**Architecture Version:** 1.0.0
**Frozen Date:** 2026-07-26
**Approved by:** GPT (Chief Product Architect) + Antigravity (Lead Software Architect)
**Compatible Cloud Contract:** v1 *(to be declared at Cloud milestone)*

---

> **Platform Runtime adalah produk pertama Haispace.**
> Semua aplikasi lain — Booth, Camera, Admin, AI Assistant, Cloud — hanyalah konsumen dari Platform Runtime ini.

---

## Arti Kata "Frozen"

> Tidak ada perubahan yang boleh dilakukan terhadap Platform Contract tanpa ADR baru yang disetujui oleh Architecture Review.

Freeze **bukan berarti tidak boleh berubah.** Freeze berarti:
- Perubahan terhadap Platform Contract harus melewati **formal Architecture Review**
- Setiap keputusan perubahan didokumentasikan sebagai **ADR baru** (ADR-012 dst.)
- Implementasi baru di milestone berikutnya **harus tunduk** pada konstitusi ini, bukan mengubahnya

---

## Platform Contracts (Frozen at v1.0.0)

Berikut adalah komponen-komponen yang termasuk dalam Platform Contract. Perubahan pada salah satu dari ini **wajib** melalui Architecture Review:

| Contract | File | ADR |
|---|---|---|
| `HaispaceSession` Aggregate Root | `Core/Domain/Session/HaispaceSession.swift` | ADR-007 |
| `SessionRepositoryProtocol` | `Core/Domain/Session/SessionRepository.swift` | ADR-009 |
| `SessionDomainEvent` | `Core/Domain/Session/SessionDomainEvent.swift` | ADR-008 |
| `DomainEventEnvelope` | `Core/Runtime/DomainEventEnvelope.swift` | ADR-008 |
| `DomainEventPublisher` | `Core/Runtime/DomainEventPublisher.swift` | ADR-008 |
| `RuntimeClock` / `RuntimeClockProtocol` | `Core/Runtime/RuntimeClock.swift` | ADR-011 |
| `RuntimeModule` Protocol | `Core/Runtime/RuntimeModules.swift` | ADR-011 |
| `RuntimeContainer` | `Core/Runtime/RuntimeContainer.swift` | ADR-011 |
| `CapabilityManagerProtocol` | `Core/Runtime/CapabilityManager.swift` | ADR-011 |
| `CapturePolicy` | `Core/Domain/Session/CapturePolicy.swift` | ADR-001 |
| `SessionFactory` | `Core/Domain/Session/SessionFactory.swift` | ADR-009 |
| `SessionSnapshot` | `Core/Infrastructure/Persistence/SessionSnapshot.swift` | ADR-009 |
| `WorkflowOrchestrator` (interface) | `Core/Workflow/WorkflowOrchestrator.swift` | ADR-003 |

---

## Runtime Readiness Matrix

> Status per area pada saat freeze v1.0.0. Diperbarui setiap Architecture Review.

| Area | Status | Keterangan |
|---|---|---|
| Session Aggregate | ✅ Ready | HaispaceSession actor, 22 domain events, invariant guards |
| Session Repository | ✅ Ready | LocalSessionRepository, atomic write, crash-safe |
| Session Factory | ✅ Ready | create(), restore() stub, migrate() stub |
| Session Snapshot | ✅ Ready | snapshotSchemaVersion, Codable, atomic file write |
| Domain Event Publisher | ✅ Ready | Priority-ordered delivery (critical/normal/low) |
| Runtime Clock | ✅ Ready | SystemClock, FixedClock, OffsetClock, ReplayClock stub |
| Domain Event Envelope | ✅ Ready | eventId, sequenceNumber, correlationId, causationId |
| Runtime Modules | ✅ Ready | SessionModule, CapabilityModule, InfrastructureModule, ObservabilityModule |
| RuntimeContainer | ✅ Ready | Module-based composition root, build(for:) factory |
| Runtime Descriptor | ✅ Ready | Self-describing, guarantees list, compatibility matrix |
| Compatibility Window | 🟡 Partial | Payment: Shadow Write + Read Compare done; Read Switch pending |
| Recovery Engine | 🟡 Partial | Snapshot persisted; restore() stub; Phase C implementation pending |
| Capability Manager | 🟡 Partial | Protocol + SimpleCapabilityManager; real policy Milestone 3 |
| AppState Integration | ⬜ Planned | Milestone setelah freeze — AppState bukan God Object |
| Asset Manifest | ⬜ Planned | Milestone D |
| Sync Engine | ⬜ Planned | Milestone E |
| Cloud Contract | ⬜ Planned | Milestone F |
| Operator Platform | ⬜ Planned | Milestone G |
| Admin Platform | ⬜ Planned | Milestone H |

---

## Runtime Guarantees

Setiap guarantee harus memiliki minimal satu Architecture Acceptance Test.

| ID | Guarantee | Status | Test |
|---|---|---|---|
| **RG-001** | Session tidak hilang saat crash (atomic disk write) | ✅ Guaranteed | AAT-002 |
| **RG-002** | Payment terkonfirmasi tidak hilang meski delivery gagal | ✅ Guaranteed | AAT-001 |
| **RG-003** | Manifest baru tidak mengubah asset session yang sedang berjalan | 🟡 Partial | AAT-003 |
| **RG-004** | Capability failure di-handle oleh Policy tanpa mematikan Workflow | 🟡 Partial | AAT-004 |
| **RG-005** | WorkflowOrchestrator tidak mengetahui detail implementasi dependency | ✅ Guaranteed | Structural |
| **RG-006** | AppState tidak mengandung business logic | ⬜ Planned | Pending AppState milestone |
| **RG-007** | Subscriber Critical selalu dieksekusi sebelum subscriber Low | ✅ Guaranteed | Structural |
| **RG-008** | Semua timestamp berasal dari RuntimeClock, bukan `Date()` langsung | 🟡 Partial | Bertahap seiring refactor |

---

## Architecture Layers (Enforced)

```
┌─────────────────────────────────────────────────────┐
│                   SwiftUI Layer                     │
│         (hanya render state, tidak ada logic)       │
└────────────────────────┬────────────────────────────┘
                         │ intent
                         ▼
┌─────────────────────────────────────────────────────┐
│                    AppState                         │
│    (meneruskan intent, memantulkan state — NO LOGIC)│
└────────────────────────┬────────────────────────────┘
                         │ intent
                         ▼
┌─────────────────────────────────────────────────────┐
│               RuntimeContainer                      │
│         (composition root — module orchestrator)    │
│  SessionModule  CapabilityModule  InfraModule  Obs  │
└────────┬────────────────┬────────────────┬──────────┘
         │                │                │
         ▼                ▼                ▼
┌────────────┐  ┌──────────────────┐  ┌──────────────┐
│ Workflow   │  │  Domain          │  │  Infra       │
│Orchestrator│  │  Session         │  │  Persistence │
│            │  │  Aggregate       │  │  Compat      │
└────────────┘  └──────────────────┘  └──────────────┘
```

**Anti-patterns yang dilarang mulai v1.0.0:**
- SwiftUI mengakses Repository langsung
- AppState membuat dependency (hanya menerima dari RuntimeContainer)
- WorkflowOrchestrator memanggil `Date()` langsung (gunakan `runtime.infrastructure.clock.now`)
- Session Aggregate memanggil external service langsung
- Repository tahu tentang Workflow, Capability, atau Cloud

---

## Milestone Roadmap (Post-Freeze)

```
Platform Runtime v1.0.0 ✅ FROZEN
         │
         ▼
Architecture Acceptance Tests (AAT-001..005)
         │
         ▼
AppState Integration
(AppState menjadi thin-wrapper, tidak ada God Object)
         │
         ▼
Read Switch (PR-03) — Payment primary read dari Aggregate
         │
         ▼
Legacy Cleanup (PR-04) — Hapus PaymentStore
         │
         ▼
Capture + Delivery + Session Migration (PR-05..15)
         │
         ▼
CapabilityManager Implementation (Milestone 3)
(Registry + Resolver + Policy, real health checks)
         │
         ▼
Asset Manifest (Milestone 4)
         │
         ▼
Cloud Contract (Milestone 5)
         │
         ▼
Operator Platform (Milestone 6)
         │
         ▼
Admin Platform (Milestone 7)
```

---

## Governance

**Siapa yang bisa mengusulkan perubahan Platform Contract?**
- Lead Software Architect (Antigravity) → mengusulkan ADR baru
- Chief Product Architect (GPT) → menyetujui atau menolak ADR

**Apa yang boleh berubah tanpa ADR?**
- Implementasi internal module (selama interface contract tidak berubah)
- Penambahan subscriber baru ke DomainEventPublisher
- Penambahan Capability baru ke CapabilityModule
- Bug fixes yang tidak mengubah interface

**Apa yang TIDAK boleh berubah tanpa ADR?**
- Protocol signatures (`SessionRepositoryProtocol`, `CapabilityManagerProtocol`, dll.)
- `SessionDomainEvent` enum cases
- `SessionSnapshot` schema fields
- `RuntimeContainer.build(for:)` factory signature
- `DomainEventEnvelope` fields

---

*Dokumen ini adalah Platform Runtime Constitution v1.0.0.*
*Referensi: ADR-007, ADR-008, ADR-009, ADR-010, ADR-011*
