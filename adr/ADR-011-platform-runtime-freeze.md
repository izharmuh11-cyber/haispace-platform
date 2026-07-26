# ADR-011: Platform Runtime v1.0 Freeze

**Status:** Accepted
**Date:** 2026-07-26
**Decision by:** GPT (Chief Product Architect) + Antigravity (Lead Software Architect)
**Refs:** ADR-007, ADR-008, ADR-009, ADR-010
**Constitution:** `constitution/PLATFORM_RUNTIME_V1.md`

---

## Context

Sejak awal Milestone 2, kita telah membangun fondasi Platform Runtime Haispace secara bertahap:
- **ADR-007**: Session Aggregate Root + WorkflowStage sebagai satu-satunya state machine bisnis
- **ADR-008**: Domain Event Publisher + AsyncStream sebagai implementasi pertama
- **ADR-009**: Session Aggregate Repository + SessionFactory sebagai satu-satunya lifecycle manager
- **ADR-010**: Compatibility Window Strategy (Strangler Fig 4-step migration)

Per 2026-07-26, fondasi berikut telah selesai secara arsitektur:
- `HaispaceSession` Aggregate Root (actor, 22 domain events, invariant guards)
- `SessionRepository` + `LocalSessionRepository` (atomic write, crash-safe)
- `SessionFactory` (create/restore/migrate API stabil)
- `DomainEventPublisher` (priority-ordered: critical/normal/low)
- `DomainEventEnvelope` (tracing, sequenceNumber, correlationId)
- `RuntimeClock` (SystemClock, FixedClock, ReplayClock stub)
- `RuntimeModules` (SessionModule, CapabilityModule, InfrastructureModule, ObservabilityModule)
- `RuntimeContainer` (module-based composition root)
- `CapabilityManager` (Registry + Resolver + Policy contract)
- `RuntimeDescriptor` (self-describing runtime object)
- `Compatibility Window` untuk Payment bounded context (PR-01 + PR-02)
- `Architecture Acceptance Tests` (AAT-001..005)

---

## Decision

### 1. Platform Runtime v1.0.0 dibekukan

Seluruh komponen di atas menjadi **Platform Contracts** yang tunduk pada governance ADR.

> **Freeze berarti:** Tidak ada perubahan pada Platform Contract tanpa ADR baru yang disetujui.

### 2. Tiga komponen baru di-introduce sebagai bagian dari freeze

| Komponen | Tanggung Jawab |
|---|---|
| `RuntimeClock` | Abstraksi tunggal untuk semua timestamp — menggantikan `Date()` langsung |
| `DomainEventEnvelope` | Metadata wrapper untuk tracing, replay, dan distributed sync |
| `RuntimeDescriptor` | Self-description runtime — untuk handshake dengan Mission Control dan Cloud |

### 3. RuntimeContainer direstrukturisasi menjadi Module Orchestrator

`RuntimeContainer` sebelumnya (pre-ADR-011) membuat semua dependency secara langsung. Ini adalah anti-pattern yang akan menjadikannya God Object seiring pertumbuhan platform.

**Keputusan**: RuntimeContainer hanya menyusun Module. Setiap Module bertanggung jawab atas dependensinya sendiri.

```
Sebelum ADR-011:                  Setelah ADR-011:
RuntimeContainer {                 RuntimeContainer {
  repository                         session: SessionModule
  publisher                          capabilities: CapabilityModule
  audit                              infrastructure: InfrastructureModule
  compatibility                      observability: ObservabilityModule
  missionControl               }
  analytics
  orchestrator
}
```

### 4. DomainEventPublisher menggunakan Priority Ordering

Subscriber dieksekusi dalam urutan: `.critical` → `.normal` → `.low`.

Jika subscriber `.low` (Analytics, Cloud Metrics) gagal atau lambat, subscriber `.critical` (Audit, Compatibility) tidak terganggu.

### 5. CapabilityManager didefinisikan sebagai Registry + Resolver + Policy

`CapabilityRegistry` (pre-ADR-011) hanya berperan sebagai register/resolve. GPT menilai ini tidak cukup untuk platform yang akan memiliki beberapa printer, beberapa payment gateway, dan AI capabilities.

`CapabilityManager` memiliki tiga tanggung jawab:
- **Registry**: mendaftarkan capability instance
- **Resolver**: memilih instance terbaik (health check, priority)
- **Policy**: menentukan behavior saat capability degraded/unavailable (fallback, failFast, operatorPrompt)

Implementasi konkret (`DefaultCapabilityManager`) akan dibuat pada Milestone 3.

### 6. Architecture Acceptance Tests (AAT) wajib untuk setiap Runtime Guarantee

> Jika tidak ada testnya, Guarantee itu belum benar-benar ada.

Setiap `RuntimeGuarantee` di `RuntimeDescriptor.current` harus memiliki `acceptanceTestId`. Test yang tidak bisa sepenuhnya di-run sekarang (karena Phase C/D belum ada) sudah memiliki struktur dan `TODO` yang jelas.

---

## Runtime Governance

### Yang Boleh Berubah Tanpa ADR

- Implementasi internal module (selama interface tidak berubah)
- Penambahan subscriber baru ke DomainEventPublisher
- Bug fixes yang tidak mengubah interface publik

### Yang Tidak Boleh Berubah Tanpa ADR

- Protocol signatures di Platform Contracts (tabel di Constitution)
- `SessionDomainEvent` enum cases
- `SessionSnapshot` schema fields (bump `snapshotSchemaVersion` via ADR)
- `RuntimeContainer.build(for:)` factory signature
- `DomainEventEnvelope` fields
- `RuntimeDescriptor.architectureVersion`

---

## Architecture Version Numbering

Format: `MAJOR.MINOR.PATCH` (SemVer-style)

| Change Type | Version Bump | Contoh |
|---|---|---|
| Perubahan breaking pada Platform Contract | MAJOR | 1.0.0 → 2.0.0 |
| Penambahan contract baru (non-breaking) | MINOR | 1.0.0 → 1.1.0 |
| Bug fix / internal refactor | PATCH | 1.0.0 → 1.0.1 |

`v1.0.0` adalah baseline. Milestone berikutnya dimulai sebagai `v1.1.0` jika ada contract baru yang ditambahkan.

---

## Consequences

**Positif:**
- Tidak ada milestone berikutnya yang bisa mengubah fondasi secara diam-diam
- `RuntimeDescriptor` memungkinkan Cloud dan Mission Control handshake tanpa membaca kode
- CapabilityManager dengan Policy membuka jalan untuk multi-printer, multi-payment, AI capability
- Architecture Acceptance Tests memberikan confidence sebelum setiap Read Switch

**Negatif / Trade-off:**
- Governance ADR menambah overhead untuk setiap perubahan platform contract
- Beberapa AAT belum sepenuhnya dapat dijalankan (butuh Phase C/D)
- `FixedClock` perlu di-inject ke semua komponen yang saat ini masih menggunakan `Date()` langsung

---

*ADR-011 — Platform Runtime v1.0 Freeze*
*Konstitusi penuh: `constitution/PLATFORM_RUNTIME_V1.md`*
