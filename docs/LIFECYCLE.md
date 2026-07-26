# Haispace Entity Lifecycle

> **Lifecycle setiap entitas utama dalam platform Haispace.**
>
> Lifecycle mendefinisikan status yang valid dan transisi yang diizinkan.
> Workflow mengikuti lifecycle — bukan sebaliknya.

---

## Organization

```
Created → Active → Suspended → Terminated
                ↑      ↓
                └──────┘ (Reactivated)
```

| Status | Deskripsi |
|--------|-----------|
| `Created` | Organization terdaftar, belum diverifikasi |
| `Active` | Beroperasi normal |
| `Suspended` | Dinonaktifkan sementara (billing, pelanggaran) |
| `Terminated` | Tidak aktif permanen |

---

## Venue

```
Created → Active → Archived
```

| Status | Deskripsi |
|--------|-----------|
| `Created` | Venue ditambahkan ke Organization |
| `Active` | Venue dapat digunakan untuk Event |
| `Archived` | Venue tidak aktif, tidak dapat menerima Event baru |

---

## Event

```
Draft → Published → Assigned → Active → Ended → Archived
                                  ↑        ↓
                                  └─Paused─┘
```

| Status | Deskripsi |
|--------|-----------|
| `Draft` | Event sedang dikonfigurasi di Admin |
| `Published` | Event siap di-assign ke Booth |
| `Assigned` | Booth telah menerima Event |
| `Active` | Event sedang berlangsung, Booth menerima tamu |
| `Paused` | Event dihentikan sementara oleh Operator |
| `Ended` | Event selesai, tidak ada Session baru |
| `Archived` | Data Event diarsipkan |

---

## Booth

```
Registered → Provisioned → Online → Ready → Active → Offline
                                       ↑               ↓
                                       └───────────────┘
```

| Status | Deskripsi |
|--------|-----------|
| `Registered` | Booth terdaftar di sistem, belum dikonfigurasi |
| `Provisioned` | BoothIdentity dan capability profile sudah ditetapkan |
| `Online` | Booth menyala dan terhubung |
| `Ready` | Self-check passed, Manifest tersedia, siap menerima tamu |
| `Active` | Booth sedang melayani tamu (ada Session berjalan) |
| `Offline` | Booth tidak terhubung (beroperasi lokal jika Event aktif) |

---

## Session

```
Created → PackageSelected → Capturing → Processing → PaymentPending
                                                           ↓
Archived ← Completed ← Delivering ← PaymentAccepted ←────┘
                                          ↑
                                    PaymentVerified (async)
```

| Status | Deskripsi |
|--------|-----------|
| `Created` | Session dimulai, tamu memilih paket |
| `PackageSelected` | Paket dipilih, Workflow dikonfigurasi |
| `Capturing` | Proses pengambilan media berlangsung |
| `Processing` | Editing, compositing, export |
| `PaymentPending` | Menunggu konfirmasi pembayaran |
| `PaymentAccepted` | Booth boleh melanjutkan (lokal) |
| `PaymentVerified` | Cloud mengkonfirmasi (async, tidak memblokir workflow) |
| `Delivering` | Foto sedang dikirim ke tamu |
| `Completed` | Semua delivery berhasil |
| `Archived` | Session diarsipkan setelah periode retensi |

---

## Package

```
Draft → Published → Active → Archived
            ↑          ↓
            └──────────┘ (Updated)
```

| Status | Deskripsi |
|--------|-----------|
| `Draft` | Package sedang dibuat di Admin |
| `Published` | Package tersedia untuk di-assign ke Event |
| `Active` | Package sedang digunakan dalam Event yang berlangsung |
| `Archived` | Package tidak aktif, tidak dapat digunakan di Event baru |

---

## Asset

```
Created → Uploaded → Validated → Published → Cached → Expired
                         ↓                      ↓
                      Rejected              Invalidated
```

| Status | Deskripsi |
|--------|-----------|
| `Created` | Asset entry dibuat di Admin |
| `Uploaded` | File berhasil diunggah ke Cloud Storage |
| `Validated` | Checksum, format, dan ukuran terverifikasi |
| `Rejected` | Validasi gagal, perlu upload ulang |
| `Published` | Asset siap didistribusikan ke Booth |
| `Cached` | Asset sudah ada di local storage Booth |
| `Expired` | Asset tidak lagi valid (digantikan versi baru) |
| `Invalidated` | Asset ditarik karena masalah konten |

---

## Manifest

```
Draft → Published → Distributed → Active → Superseded
```

| Status | Deskripsi |
|--------|-----------|
| `Draft` | Manifest sedang disusun di Admin |
| `Published` | Manifest tersedia di Cloud untuk didownload |
| `Distributed` | Setidaknya satu Booth sudah menerima Manifest ini |
| `Active` | Manifest sedang digunakan oleh Booth dalam Event |
| `Superseded` | Manifest digantikan versi yang lebih baru |

---

## Delivery

```
Queued → Started → Completed
   ↓         ↓
Failed → Retrying → [Completed | PermanentlyFailed]
```

| Status | Deskripsi |
|--------|-----------|
| `Queued` | Delivery masuk antrian |
| `Started` | Proses pengiriman dimulai |
| `Completed` | Berhasil diterima oleh tamu |
| `Failed` | Percobaan gagal, akan diretry |
| `Retrying` | Sedang menunggu retry berikutnya |
| `PermanentlyFailed` | Sudah melebihi batas retry, perlu intervensi manual |

---

## Operator

```
Invited → Active → Suspended → Deactivated
              ↑         ↓
              └─────────┘ (Reactivated)
```

| Status | Deskripsi |
|--------|-----------|
| `Invited` | Operator diundang, belum set password |
| `Active` | Dapat login dan menjalankan Event |
| `Suspended` | Akses sementara dicabut |
| `Deactivated` | Tidak dapat login, data tetap tersimpan |

---

## Transisi Lifecycle

Aturan transisi lifecycle:
1. **Transisi hanya boleh maju** kecuali ada keterangan eksplisit
2. **Setiap transisi menghasilkan Domain Event**
3. **Tidak ada transisi yang terjadi diam-diam** — semua tercatat di audit trail
4. **Transisi yang invalid harus ditolak** dengan pesan error yang jelas

---

*Haispace Lifecycle v1.0*
