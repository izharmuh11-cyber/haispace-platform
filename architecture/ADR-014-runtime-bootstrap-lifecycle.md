# ADR-014 — Runtime Bootstrap Lifecycle

**Status:** Proposed (M-005)  
**Date:** 2026-08-01  
**Author:** GPT (Chief Product Architect), Antigravity (Lead Software Architect)

---

## 1. Context

Sebelum Kiosk (iPad) siap menerima tamu, *Runtime Engine* harus memastikan bahwa perangkat keras, jaringan, dan status platform berada dalam kondisi yang valid. 

Pada arsitektur lama, proses *booting* dilakukan secara serabutan dan *fire-and-forget*. Jika ada konfigurasi yang kurang, aplikasi akan *crash* di tengah jalan atau beroperasi dengan state yang kotor. 

Di arsitektur baru (Platform Independence), kita membutuhkan *Bootstrap Lifecycle* yang kokoh, terobservasi (Observability-First), dan memiliki toleransi kesalahan (Failure Policy) yang jelas.

---

## 2. State Transition & Sequence

Proses *bootstrap* wajib berjalan berurutan sesuai urutan berikut:

1. **BOOT STARTED** — Aplikasi diluncurkan, container diinisialisasi.
2. **CAPABILITY DISCOVERY** — Mengidentifikasi status dari hardware dan sistem operasi (Kamera, Printer, Network, Storage).
3. **CLOCK SYNC** — Sinkronisasi waktu dengan server untuk mencegah fraud pada timestamp sesi.
4. **DEVICE REGISTRATION / AUTH** — Memverifikasi identitas perangkat (Booth ID) dengan Cloud API.
5. **MANIFEST DOWNLOAD** — Mengunduh konfigurasi terbaru (Tema, Harga, Asset, Logika Promosi) dari Cloud.
6. **STATE READY** — Semua prasyarat terpenuhi. Kiosk siap digunakan oleh tamu.

*(Jika di tengah jalan gagal, *state* akan berpindah ke `ERROR / DEGRADED` dan menunggu campur tangan Operator).*

---

## 3. Timeout & Retry Policy

Setiap tahapan yang melibatkan jaringan (*network-bound*) harus mematuhi aturan berikut:

| Tahapan | Timeout | Retry Policy | Fallback |
|---------|---------|--------------|----------|
| Capability Discovery | 5 detik | Tidak ada | Anggap *Unavailable* |
| Clock Sync | 10 detik | 3x (Backoff 2s) | *Hard Failure* |
| Device Registration | 15 detik | 3x (Backoff 2s) | *Hard Failure* |
| Manifest Download | 30 detik | 3x (Backoff 5s) | *Soft Failure* (gunakan cache) |

---

## 4. Failure & Offline Policy

### Offline Policy (Saat Booting)
- Jika perangkat **TIDAK ADA KONEKSI INTERNET** saat melakukan *boot*, Kiosk **TIDAK DIIZINKAN** untuk beroperasi secara mandiri.
- *Clock Sync* dan *Device Auth* bersifat **mutlak** saat boot awal demi keamanan transaksi.
- *Pengecualian:* Jika aplikasi di-*restart* karena *crash* saat event berlangsung (Recovery Mode), Kiosk diizinkan menggunakan Manifest Cache dan melewati Clock Sync dengan mencatat `TAMPER_RISK`.

### Failure Policy
- **Hard Failure:** Proses bootstrap dihentikan. UI menampilkan layar *Mission Control* dengan mode darurat (Merah). Tamu tidak bisa memulai sesi.
- **Soft Failure:** Proses dilanjutkan, namun kemampuan akan dibatasi (Degraded State). Contoh: Printer rusak $\rightarrow$ Kiosk tetap hidup namun opsi cetak disembunyikan.

---

## 5. Observability Events (Log Standard)

*Observability* adalah warga kelas satu (*first-class citizen*). Setiap *state transition* harus mencetak log dengan format standar berikut ke *console* dan *audit trail*:

```
[BOOTSTRAP] 21:15:01 | BOOT STARTED
[BOOTSTRAP] 21:15:01 | CAPABILITY DISCOVERY STARTED
[BOOTSTRAP] 21:15:02 | CAPABILITIES LOADED: [Camera: Avail, Printer: Unavail, Net: Online]
[BOOTSTRAP] 21:15:02 | CLOCK SYNC SUCCESS
[BOOTSTRAP] 21:15:03 | DEVICE REGISTERED
[BOOTSTRAP] 21:15:03 | MANIFEST READY (v12)
[BOOTSTRAP] 21:15:04 | STATE READY
```

Format ini mempermudah operator dan tim teknis membaca riwayat *startup* perangkat saat terjadi masalah (insiden).

---

## 6. Implementation Notes for Antigravity

- **Bootstrap Engine:** Harus berupa `Swift Actor` yang mengelola transisi ini secara terpusat.
- **Capability Manager:** Menjadi sumber kebenaran tunggal untuk ketersediaan hardware/services.
- **No Business Logic:** Jangan izinkan transaksi atau fitur Kiosk lainnya terpanggil sebelum log `STATE READY` tercetak.
