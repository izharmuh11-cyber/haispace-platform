# Haispace Glossary

> Bahasa bersama seluruh ekosistem Haispace.
> Semua tim — produk, engineering, desain, operasional — menggunakan definisi ini.
> Kalau definisi berubah, dokumen ini harus diperbarui terlebih dahulu sebelum implementasi berubah.

---

## Domain Utama

### Session
**Unit bisnis terkecil yang menghasilkan nilai bagi pelanggan.**

Satu Session dimulai ketika tamu memilih paket dan berakhir ketika seluruh proses delivery selesai. Session adalah unit yang tidak boleh hilang. Setiap produk Haispace — Video Booth, AI Booth, 360 Booth — tetap menggunakan Session sebagai unit dasar.

*Lifecycle:* `created → packageSelected → capturing → processing → delivering → completed`

---

### Capture
**Proses memperoleh media.**

Capture bukan hanya foto. Capture adalah tindakan sistem untuk mengambil media dari sumber manapun. Implementasinya bisa berupa:
- Photo (still image)
- Video (motion capture)
- Burst (serial photos)
- Live (streaming frame)

Satu Session bisa menghasilkan banyak Capture.

---

### Package
**Pengalaman yang dibeli pelanggan, termasuk seluruh policy yang menyertainya.**

Package bukan hanya kumpulan aset. Package mendefinisikan:
- Harga dan metode pembayaran
- Jumlah Capture yang diizinkan
- Delivery Policy (QR, Print, WhatsApp, dll)
- Printing Policy (berapa kali boleh print)
- Workflow Policy (apakah boleh pilih filter, dll)
- Asset Set yang digunakan

*Package adalah policy. Policy menentukan behavior Workflow.*

---

### Layout
**Template komposisi visual yang mendefinisikan struktur tampilan akhir.**

Layout menentukan:
- Jumlah foto dalam satu strip
- Posisi dan ukuran setiap foto
- Orientasi (portrait/landscape)
- Rasio aspek

Layout tidak menentukan warna atau identitas visual. Itu urusan Theme.

---

### Theme
**Identitas visual sebuah event atau paket.**

Theme bukan hanya warna. Theme adalah keseluruhan identitas yang bisa berisi:
- Color palette
- Typography (Font)
- Sticker set
- Decoration
- Animation style
- Audio ambient

Theme yang sama bisa dipakai di banyak Package.

---

### Frame
**Visual overlay yang ditempel di atas foto hasil Capture.**

Frame adalah elemen grafis transparan (PNG/vector) yang dikomposisikan bersama foto. Frame bukan Layout dan bukan Theme. Frame hanya mengubah tampilan foto secara visual.

---

### Asset
**Unit konten yang dapat diverifikasi, diversion, dan didistribusikan.**

Semua konten dalam Haispace adalah Asset. Setiap Asset memiliki:
- `assetId` (UUID)
- `version` (integer, increment)
- `checksum` (SHA-256)
- `url` (lokasi di Cloud Storage)
- `size` (bytes)
- `updatedAt` (timestamp)
- `type` (Frame | Font | Music | Video | Sticker | Animation | Countdown | Overlay)

Booth tidak menyimpan file Asset secara langsung. Booth hanya mengacu ke Manifest.

---

### Manifest
**Deklarasi seluruh resource yang dibutuhkan Runtime untuk menjalankan sebuah Event.**

Manifest bukan daftar file. Manifest adalah kontrak antara Cloud dan Booth yang mendefinisikan:
- Daftar Asset beserta versi dan checksumnya
- Capabilities yang dibutuhkan (printer, camera type, dll)
- Policy minimum (minimum app version, dll)
- Dependencies antar asset

Booth membandingkan Manifest versi lokal dengan versi cloud. Kalau ada perbedaan checksum, hanya file yang berubah yang didownload ulang.

---

### Operator
**Pengguna manusia yang bertanggung jawab menjalankan Booth selama Event berlangsung.**

Operator bukan admin. Operator tidak membuat konten, tidak mengubah harga, tidak mengelola paket. Operator hanya:
- Login ke Booth
- Memilih Event yang akan dijalankan
- Memonitor jalannya Event
- Menangani situasi darurat

Operator berganti tidak mempengaruhi BoothIdentity.

---

### Organization
**Entitas bisnis pemilik platform Haispace.**

Hierarki kepemilikan:
```
Organization
  └── Venue
        └── Event
              └── Booth
                    └── Operator (pengguna, bukan pemilik)
```

---

### Venue
**Lokasi fisik tempat Booth beroperasi.**

Venue bisa memiliki banyak Booth. Venue dikelola oleh Organization.

---

### Event
**Satu periode operasional Booth dengan konfigurasi tertentu.**

Misalnya: "Wedding Rina & Budi — 26 Juli 2026". Event memiliki:
- Waktu mulai dan selesai
- Package yang tersedia
- Manifest yang berlaku
- Booth yang ditugaskan

---

### Delivery
**Proses mengirimkan hasil Session kepada pelanggan.**

Delivery adalah langkah akhir setelah foto diproses. Channel yang tersedia:
- QR Code download
- WhatsApp
- Print (fisik)
- Email

Delivery masuk ke Queue dan diproses dengan retry otomatis jika gagal.

---

### BoothIdentity
**Identitas permanen sebuah perangkat Booth.**

BoothIdentity tidak berubah ketika Operator berganti. Berisi:
- `boothId` (UUID)
- `organizationId`
- `hardwareFingerprint`
- `displayName`
- `capabilityProfile` (kemampuan hardware)

---

### PaymentCommitment
**Status komitmen pembayaran dalam sebuah Session.**

Tiga status:
- `Pending` — proses pembayaran sedang berlangsung
- `Accepted` — Booth memiliki alasan cukup untuk melanjutkan Workflow
- `Verified` — Cloud sudah mengkonfirmasi pembayaran

`Accepted ≠ Verified`. Workflow hanya membutuhkan `Accepted`. `Verified` diurus oleh SyncEngine secara asinkron.

---

## Status Sistem

| Term | Definisi |
|------|----------|
| **Healthy** | Sistem berjalan dalam parameter normal, semua dependency available |
| **Degraded** | Sistem berjalan tapi ada dependency terbatas, masih bisa operasi |
| **Unavailable** | Dependency kritis tidak tersedia, operasi terhenti |
| **Ready** | Booth sudah melewati self-check dan siap menerima tamu |
| **Syncing** | Booth sedang menyinkronkan data dengan Cloud |
| **Offline** | Booth beroperasi tanpa koneksi Cloud (kondisi normal yang didukung) |

---

*Haispace Glossary v1.0*
