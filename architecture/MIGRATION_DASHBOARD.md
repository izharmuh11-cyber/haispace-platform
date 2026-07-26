# Migration Dashboard — Haispace Platform

**Purpose:** Dokumen ini menunjukkan status migrasi setiap bounded context secara visual.
Ketika semua baris mencapai `✅ Ready for Cleanup`, PR Cleanup boleh dieksekusi.

**Updated by:** Antigravity (Lead Software Architect)
**Updated at:** 2026-07-26
**Strategy:** ADR-010 — Compatibility Window (Strangler Fig + 4-Step)

> **Aturan Merge:** Legacy Store tidak boleh dihapus sebelum Mismatch Rate = 0% selama 2 minggu runtime aktif.

---

## 📊 Payment Bounded Context

```
Step                    Status          Note
────────────────────────────────────────────────────────────────────────
Shadow Write            ✅ Done         PR-01 — Session.acceptPayment()
                                        + SessionRepository.save()
                                        + Legacy PaymentStore.update()

Read Compare            ✅ Done         PR-02 — PaymentCompatibilityChecker
                                        5 fields: status, amount, ref, acceptedAt, providerRef
                                        Timestamp delta tolerance: 1s
                                        .unknown: acceptedAt (field tidak ada di legacy)
                                        .unknown: providerReference (field baru Aggregate)

Mismatch Rate           🔵 Monitoring   0 session sejauh ini (butuh min. 2 minggu data)
Unknown Fields          2               acceptedAt, providerReference

Read Switch             ⏳ Blocked      Menunggu: Mismatch Rate < 0.1% (PR-03)
                                        Menunggu: RuntimeContainer stable (PR-00)

Legacy Cleanup          ⏳ Blocked      PaymentStore.swift belum boleh dihapus (PR-04)
────────────────────────────────────────────────────────────────────────
Ready for Cleanup?      ❌ NO
Reason:                 RuntimeContainer belum selesai.
                        Mismatch Rate monitoring belum punya data yang cukup.
```

---

## 📊 Capture Bounded Context

```
Step                    Status          Note
────────────────────────────────────────────────────────────────────────
Shadow Write            ⏳ Not started  PR-05 — target setelah RuntimeContainer
Read Compare            ⏳ Not started  PR-06
Mismatch Rate           —               belum ada data
Read Switch             ⏳ Not started  PR-07
Legacy Cleanup          ⏳ Not started  PR-08 — PhotoStore.swift
────────────────────────────────────────────────────────────────────────
Ready for Cleanup?      ❌ NO
```

---

## 📊 Delivery Bounded Context

```
Step                    Status          Note
────────────────────────────────────────────────────────────────────────
Shadow Write            ⏳ Not started  PR-09 — target setelah Capture selesai
Read Compare            ⏳ Not started  PR-10
Mismatch Rate           —               belum ada data
Read Switch             ⏳ Not started  PR-11
Legacy Cleanup          ⏳ Not started  PR-12 — DeliveryStore.swift
────────────────────────────────────────────────────────────────────────
Ready for Cleanup?      ❌ NO
```

---

## 📊 Session Root Bounded Context

```
Step                    Status          Note
────────────────────────────────────────────────────────────────────────
SessionRepository       ✅ Done         LocalSessionRepository live
                                        Atomic write, crash-safe
SessionFactory          ✅ Done         create(), restore() stub, migrate() stub
RuntimeContainer        🔄 In Progress  DomainEventPublisher + Subscribers
                                        CapabilityRegistry (contract only)
AppState Cleanup        ⏳ Blocked      PR-13 — menunggu RuntimeContainer stable
Session Root Cleanup    ⏳ Blocked      PR-14–15 — SessionStore.swift + AppState
────────────────────────────────────────────────────────────────────────
Ready for Cleanup?      ❌ NO
Reason:                 RuntimeContainer sedang dibangun.
```

---

## 🚦 Platform-Wide Status

| Component | Status | Blocking? |
|---|---|---|
| `HaispaceSession` Aggregate Root | ✅ Done | — |
| `SessionRepository` (LocalSessionRepository) | ✅ Done | — |
| `SessionFactory` (create + restore stub) | ✅ Done | — |
| `SessionSnapshot` (crash-safe, snapshotSchemaVersion) | ✅ Done | — |
| `SessionDomainEvent` (22 events) | ✅ Done | — |
| `DomainEventPublisher` + Subscribers | 🔄 In Progress | PR-03 |
| `RuntimeContainer` | 🔄 In Progress | PR-03 |
| `CapabilityRegistry` (contract) | 🔄 In Progress | Milestone 3 |
| Payment Shadow Write (PR-01) | ✅ Done | — |
| Payment Read Compare (PR-02) | ✅ Done | — |
| Payment `.unknown` support | ✅ Done | — |
| Payment Read Switch (PR-03) | ⏳ Blocked | RuntimeContainer |
| Payment Cleanup (PR-04) | ⏳ Blocked | PR-03 + Mismatch = 0 |

---

## 📋 Definition of Done — Phase B Complete

- [ ] Payment: Mismatch Rate = 0% (14 hari runtime aktif)
- [ ] Capture: Shadow Write → Read Compare → Read Switch selesai
- [ ] Delivery: Shadow Write → Read Compare → Read Switch selesai
- [ ] RuntimeContainer: stable, menjadi satu-satunya composition root
- [ ] AppState tidak lagi menjadi Service Locator
- [ ] `SessionStore`, `PaymentStore`, `DeliveryStore`, `PhotoStore` dihapus dari codebase

---

*Dashboard ini diperbarui setiap kali PR Compatibility selesai.*
*Referensi: `architecture/MIGRATION_PLAN.md` dan `adr/ADR-010-compatibility-window-strategy.md`*
