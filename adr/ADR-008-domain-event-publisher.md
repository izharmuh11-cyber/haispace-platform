# ADR-008: Domain Event Publisher Architecture

**Status:** Accepted
**Date:** 2026-07-26
**Decision by:** GPT (Chief Product Architect) + Antigravity (Lead Software Architect)
**Refs:** Platform Principles #12 (Everything is Observable), ARP-001, ARP-002

---

## Context

Saat ini HaiBooth memiliki dua sistem event yang paralel dan tidak terhubung:

1. `WorkflowEvent` enum di `WorkflowSharedTypes.swift` — domain event yang ideal tapi tidak diterbitkan atau dikonsumsi oleh siapapun.
2. `AuditEventType` di `SessionAuditTrail.swift` — event yang aktif digunakan tapi hanya menulis ke file JSONL lokal.

Akibatnya, komponen seperti Analytics, Mission Control, dan AI Assistant tidak bisa mengonsumsi event Runtime secara real-time. Setiap subscriber baru harus dikopel langsung ke `WorkflowOrchestrator` atau `SessionAuditTrail`.

---

## Decision

### Arsitektur

```
WorkflowOrchestrator
       │
       │ publish(event: DomainEvent)
       ▼
DomainEventPublisherProtocol   ← kontrak, tidak pernah berubah
       │
       ▼
AsyncStreamDomainEventPublisher  ← implementasi pertama (bisa diganti)
       │
       ├── Subscriber: SessionAuditTrail
       ├── Subscriber: MissionControlViewModel
       ├── Subscriber: AnalyticsService
       ├── Subscriber: CloudSyncEngine (future)
       └── Subscriber: AIAssistant (future)
```

### Prinsip yang ditetapkan

1. **`WorkflowOrchestrator` hanya mengenal `DomainEventPublisherProtocol`** — tidak pernah tahu implementasinya.

2. **`AsyncStream` adalah implementasi pertama, bukan arsitektur.** Bisa diganti dengan Kafka, Cloud Event, atau mekanisme lain tanpa mengubah Workflow.

3. **`SessionAuditTrail` menjadi subscriber** — bukan lagi dipanggil langsung dari `WorkflowOrchestrator`.

4. **Domain Event adalah bahasa tunggal platform.** `AuditEventType` akan dimigrasikan ke `DomainEvent`. Tidak boleh ada dua bahasa event.

### Karakteristik yang harus dipenuhi

- **Typed** — setiap event memiliki tipe yang strong-typed
- **Ordered** — event diterbitkan dan dikonsumsi secara berurutan per session
- **Observable** — subscriber bisa consume via `for await`
- **Replayable** — untuk Debug Timeline, minimal N event terakhir bisa di-replay
- **Testable** — bisa di-inject dengan `MockDomainEventPublisher` di tests

---

## Consequences

**Positif:**
- Satu bahasa event untuk seluruh platform
- Workflow tidak lagi coupled ke Audit, Analytics, atau Mission Control
- Mudah menambah subscriber baru tanpa mengubah Domain
- Implementasi dapat diganti tanpa breaking change

**Negatif / Risk:**
- Migrasi dari `SessionAuditTrail` static calls membutuhkan refactor bertahap
- Subscriber yang gagal tidak boleh mempengaruhi event flow — perlu error boundary per subscriber

---

## Implementation Note

`DomainEventPublisherProtocol` akan menjadi bagian dari `HaispacePlatformCore` Swift Package — dikonsumsi oleh HaiBooth, HaiCamera, dan masa depan HaiAdmin.

---

*ADR-008 — Domain Event Publisher Architecture*
