# ADR-009: Session Aggregate Root & Repository Pattern

**Status:** Accepted
**Date:** 2026-07-26
**Decision by:** GPT (Chief Product Architect) + Antigravity (Lead Software Architect)
**Refs:** ARP-003, ARP-004, ADR-008

---

## Context

Sebelum Milestone 2, state Session tersebar di empat lokasi terpisah:
`SessionStore`, `PaymentStore`, `DeliveryStore`, `PhotoStore` — semuanya `@Observable` class di Application Layer, tanpa koordinasi antar store, dan tidak ada persistence ke disk.

Akibatnya:
- Tidak ada satu objek yang merepresentasikan Session secara utuh
- Business invariants (boleh deliver?) tersebar di berbagai tempat
- Tidak ada session state yang survive crash
- Recovery hanya bisa menbaca audit trail — tidak bisa merekonstruksi state

---

## Decision

### 1. Session sebagai DDD Aggregate Root

`HaispaceSession` adalah satu-satunya objek yang merepresentasikan Session secara lengkap. Semua mutasi state Session terjadi melalui methods Aggregate.

**Prinsip utama:**
> **Workflow knows what happens next. Aggregate decides whether it is allowed.**

`WorkflowOrchestrator` bertanggung jawab mengorkestrasi urutan langkah. `HaispaceSession` bertanggung jawab memutuskan apakah setiap langkah diizinkan berdasarkan business invariants.

Business invariants yang dijaga oleh Session (bukan Workflow):
- `canProceedToPayment` — apakah foto minimum sudah dipilih?
- `canBeginDelivery` — apakah payment sudah Accepted?
- `canComplete` — apakah delivery sudah selesai?
- `canOperatorAbort` — apakah belum ada transaksi finansial?
- `isAtCaptureLimit` — apakah belum mencapai batas maxCount?

### 2. Swift Actor sebagai Detail Concurrency

`HaispaceSession` diimplementasikan sebagai Swift `actor` untuk thread-safety. Namun Actor adalah detail implementasi concurrency Swift — bukan bagian dari domain.

Konsep Session tetap valid di lingkungan non-Actor (server, backend, Android) karena domain logic tidak bergantung pada mekanisme concurrency.

Hierarki yang dijaga:
```
Domain (HaispaceSession concept)
  ↓
Implementation (Swift struct/class)
  ↓
Concurrency (actor)
```

### 3. Domain Event Collection tanpa Publisher Coupling

`HaispaceSession` mengumpulkan `SessionDomainEvent` di `pendingEvents` array. Aggregate tidak mengetahui Publisher.

Unit of Work pattern:
```
1. session.mutate()
2. let events = session.flushEvents()    ← aggregate menghasilkan fakta
3. repository.save(snapshot)             ← persist ke disk
4. publisher.publish(events)             ← dunia luar yang broadcast
```

Testing: cukup panggil `session.flushEvents()` tanpa memerlukan Publisher mock.

### 4. SessionSnapshot sebagai Kontrak Persistence Stabil

`SessionSnapshot` menggunakan tipe primitif saja (String, Int, Date, Bool). Tidak ada Swift enum yang disimpan langsung — `workflowStageId` adalah `String` (`WorkflowStage.rawValue`).

Tiga versi yang berbeda dan independen:
- `snapshotSchemaVersion` — versi struktur SessionSnapshot (increment saat breaking change)
- `manifestVersion` — versi Manifest saat Session dimulai (dari BoothConfig)
- `packageVersion` — versi Package yang dipilih tamu

### 5. PaymentCommitment: Pending → Accepted → Verified | Rejected

Empat state yang masing-masing memiliki semantik domain berbeda:
- `pending` — proses pembayaran sedang berlangsung
- `accepted` — point of no return; Workflow boleh dilanjutkan
- `verified` — Cloud konfirmasi async; tidak memblokir Workflow
- `rejected` — bukan Abort Session; tamu bisa retry (bergantung `PaymentRejectionReason`)

### 6. SessionRepository: 4 Operasi

`SessionRepositoryProtocol` hanya mendefinisikan: `save`, `load`, `delete`, `exists`.

Implementasi pertama: `LocalSessionRepository` — JSON file per session di `Documents/sessions/`. Menggunakan atomic write (temp file + rename) untuk crash safety.

---

## Consequences

**Positif:**
- Satu aggregate root = satu lokasi untuk semua business invariants
- Session state survive crash karena snapshot tersimpan ke disk
- Recovery Engine bisa merekonstruksi Session dari snapshot (membaca fakta)
- Testing aggregate tidak membutuhkan framework — pure Swift
- Repository dapat diganti (Cloud, encrypted) tanpa mengubah Domain

**Negatif / Trade-off:**
- Migrasi dari 4 store lama ke Aggregate membutuhkan waktu (dilakukan bertahap per PR)
- `SessionStore`, `PaymentStore`, dll. masih ada selama migrasi — temporary dual authority

---

## Migration Plan

Setiap PR mengurangi satu store lama:

| PR | Hapus | Ganti dengan |
|----|-------|-------------|
| Phase B.1 | `PaymentStore` | `HaispaceSession.paymentCommitment` |
| Phase B.2 | `PhotoStore` | `HaispaceSession.captures` |
| Phase B.3 | `DeliveryStore` | `HaispaceSession.deliveryState` |
| Phase B.4 | `SessionStore` | `HaispaceSession` (aggregate) |
| Phase C | Old `AppState.currentSession` | `SessionRepository.load()` |

Phase B selesai apabila:
1. Workflow hanya membaca SessionRepository
2. SessionSnapshot tersimpan ke disk
3. Restart aplikasi mengembalikan Session Aggregate
4. Tidak ada state session yang hilang
5. SessionStore lama tidak lagi menjadi source of truth

---

*ADR-009 — Session Aggregate Root & Repository Pattern*
*Dokumen ini menjelaskan MENGAPA, bukan hanya APA yang diimplementasikan.*
