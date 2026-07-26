# ADR-010: Compatibility Window Strategy

**Status:** Accepted
**Date:** 2026-07-26
**Decision by:** GPT (Chief Product Architect) + Antigravity (Lead Software Architect)
**Refs:** ADR-009, ARP-004, architecture/MIGRATION_PLAN.md

---

## Context

Milestone 2 memperkenalkan `HaispaceSession` Aggregate Root sebagai pengganti empat Legacy Stores (`SessionStore`, `PaymentStore`, `DeliveryStore`, `PhotoStore`). Mengganti semua store sekaligus (*Big Bang Migration*) terlalu berisiko: sulit di-review, sulit di-rollback, dan menghasilkan PR yang menyentuh terlalu banyak komponen sekaligus.

Diperlukan sebuah strategi migrasi yang:
1. **Aman**: Tidak memaksa runtime melakukan lompatan besar
2. **Terukur**: Menyediakan sinyal bahwa Aggregate benar-benar akurat sebelum Legacy dihapus
3. **Dapat diulang**: Bisa diterapkan pada setiap bounded context (Payment, Capture, Delivery, Session)

---

## Decision

### Strangler Fig Pattern + 4-Step Compatibility Window

Untuk setiap bounded context, migrasi dibagi menjadi empat PR terkontrol:

---

#### Step 1 — Shadow Write (PR ganjil pertama)

> Tulis ke Aggregate **dan** Repository terlebih dahulu. Legacy Store tetap menerima update sebagai compatibility fallback.

```
Workflow Event
    │
    ├─► Session.mutate()           ← NEW: Aggregate menjadi sumber kebenaran
    ├─► SessionRepository.save()  ← NEW: Snapshot di-persist ke disk
    └─► LegacyStore.update()      ← TETAP: Compatibility adapter
```

**Invariant**: Return value untuk caller tetap dari Legacy Store (belum beralih baca).
**Tujuan**: Membuktikan bahwa Aggregate dapat menerima data dengan benar.

---

#### Step 2 — Read Compare + Divergence Detection (PR ganjil kedua)

> Baca dari Aggregate dan Legacy. Bandingkan setiap field (nilai + timestamp). Publikasikan `CompatibilityEvent`.

```
Read Request
    │
    ├─► Read Aggregate             ← NEW
    ├─► Read Legacy                ← TETAP
    ├─► Compare (value + timing)   ← NEW
    ├─► Emit CompatibilityEvent    ← NEW
    └─► Return Aggregate value     ← NEW: Aggregate mulai dijadikan primary read
```

**Field yang dibandingkan harus mencakup timestamp**, bukan hanya nilai. Bug sinkronisasi sering tersembunyi pada urutan waktu, bukan pada nilai akhir.

**`CompatibilityEvent`** yang dipublikasikan:
- `Compatibility.Matched` — semua field cocok
- `Compatibility.Mismatched` — ditemukan perbedaan (sinyal untuk investigasi)

**Tujuan**: Membangun kepercayaan bahwa Aggregate akurat sebelum menghapus Legacy.

---

#### Step 3 — Read Switch

> Hentikan pembacaan dari Legacy Store. Aggregate menjadi satu-satunya sumber kebenaran untuk read.

```
Read Request
    │
    └─► Read Aggregate             ← ONLY READ SOURCE
         (Legacy tidak dibaca lagi)
```

**Trigger**: Mismatch count = 0 selama periode monitoring yang cukup (minimal 2 minggu runtime aktif).

---

#### Step 4 — Legacy Cleanup

> Hapus Legacy Store dari codebase. Compatibility Checker untuk bounded context ini juga dihapus.

---

### CompatibilityEvent sebagai Metrik Migrasi

Setiap `CompatibilityEvent.mismatched` dicatat oleh `HaispaceLogger` dengan kategori `"compatibility"`. Di masa depan, ini dapat dijumlahkan oleh Dashboard Engineering untuk menjawab:

> "Selama 2 minggu terakhir terdapat 0 mismatch untuk bounded context Payment."

Jika angka nol, PR Cleanup dapat dieksekusi dengan percaya diri.

---

## Coverage Metric

Tiga tingkatan coverage yang digunakan di `MIGRATION_PLAN.md`:

| Level | Coverage | Deskripsi |
|---|---|---|
| Shadow | ~40% | Shadow Write aktif + Divergence Detection aktif |
| Read Switch | ~70% | Aggregate menjadi primary read; Legacy hanya ditulis |
| Clean | 100% | Legacy Store dihapus sepenuhnya |

---

## Consequences

**Positif:**
- Tidak ada *Big Bang* — setiap PR kecil dan aman untuk rollback
- Mismatch detection memberikan kepercayaan objektif sebelum cleanup
- Pola dapat direplikasi untuk setiap bounded context tanpa dokumentasi baru
- Legacy Store tidak pernah dihapus secara prematur

**Negatif / Trade-off:**
- Selama window aktif, ada *temporary dual write* (Aggregate + Legacy)
- Complexity sementara meningkat — ada dua code path untuk operasi yang sama
- Window harus ditutup secara disiplin; jika tidak, Legacy Store menjadi "zombie code"

---

## Penerapan

Pola ini digunakan untuk:

| Bounded Context | Status |
|---|---|
| Payment | PR-01 ✅ PR-02 ✅ PR-03 ⏳ PR-04 ⏳ |
| Capture | PR-05–08 ⏳ |
| Delivery | PR-09–12 ⏳ |
| Session Root | PR-13–15 ⏳ |

---

*ADR-010 — Compatibility Window Strategy*
*Pola ini akan digunakan kembali untuk setiap bounded context, Sync Engine, dan Cloud Integration.*
