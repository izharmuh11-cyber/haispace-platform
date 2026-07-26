# Haispace Runtime Guarantees

> **Janji sistem kepada operator dan pelanggan.**
>
> Setiap fitur baru harus menjawab: *"Apakah fitur ini melanggar salah satu Runtime Guarantee?"*
> Kalau ya, fitur tersebut harus didesain ulang sebelum diimplementasikan.

---

## Konstitusi

> **Booth adalah produk yang harus tetap menghasilkan uang walaupun Cloud sedang tidak bisa membantu.**

---

## 10 Runtime Guarantees

### 1. Session Durability
Tidak ada Session yang hilang karena aplikasi restart atau device reboot.

Session yang sudah dimulai (`SessionCreated`) akan selalu dapat dilanjutkan, bahkan setelah crash. State Session terakhir yang valid akan dipulihkan secara otomatis.

*Implementasi: SessionAuditTrail + CoreData persistence*

---

### 2. Capture Durability
Tidak ada media yang hilang setelah proses Capture selesai.

Segera setelah shutter ditekan dan media tersimpan, file tersebut tidak dapat hilang karena alasan apapun — crash, restart, atau storage penuh dideteksi sebelum operasi dimulai.

*Implementasi: Atomic write + pre-flight storage check*

---

### 3. Queue Persistence
Print Queue dan Delivery Queue bertahan melalui restart aplikasi dan diproses kembali saat kondisi memungkinkan.

Queue tidak pernah disimpan hanya di memory. Setiap item Queue adalah entitas persisten dengan status retry dan backoff.

*Implementasi: Persistent queue dengan exponential backoff*

---

### 4. Payment Commitment
Setelah status `PaymentAccepted` tercatat, Workflow tidak dapat dibatalkan oleh network failure, Cloud timeout, atau restart aplikasi.

`PaymentAccepted` adalah titik tidak dapat kembali (*point of no return*) untuk operator maupun sistem.

*Catatan: Hanya operator manusia dengan otoritas yang dapat membatalkan Session setelah PaymentAccepted, dan tindakan ini selalu meninggalkan audit trail.*

---

### 5. Audit Completeness
Setiap perubahan state menghasilkan DomainEvent yang tercatat secara lokal dan akan diupload ke Cloud saat koneksi tersedia.

Tidak ada tindakan yang terjadi tanpa jejak. Audit Trail adalah sumber kebenaran untuk dispute, debugging, dan analitik.

*Implementasi: DomainEvent → local AuditTrail → async Cloud upload*

---

### 6. Event Continuity
Booth dapat menyelesaikan seluruh Event selama Asset dan Manifest sudah tersedia secara lokal.

Putusnya koneksi internet tidak menghentikan atau mendegradasi pengalaman tamu. Semua data yang dibutuhkan untuk menjalankan Event sudah harus ada di device sebelum Event dimulai.

*Implementasi: Pre-flight manifest validation + offline-capable workflow*

---

### 7. Operator Continuity
Operator dapat berganti tanpa mengganggu Session yang sedang berjalan, sesuai kebijakan operasional.

BoothIdentity adalah identitas device, bukan identitas operator. Session yang sedang berjalan tidak terikat pada sesi login operator tertentu.

*Implementasi: BoothIdentity terpisah dari OperatorSession*

---

### 8. Recoverability
Setelah aplikasi crash atau perangkat restart, Booth mampu melanjutkan dari state terakhir yang valid secara otomatis.

Booth tidak pernah meminta operator untuk "mulai dari awal" setelah crash. Jika Session sedang berjalan, Session tersebut dipulihkan. Jika tidak ada Session aktif, Booth kembali ke state Ready.

*Implementasi: State machine recovery + SessionAuditTrail replay*

---

### 9. Asset Consistency
Session selalu menggunakan versi Asset yang sama dari awal hingga selesai.

Update Manifest dari Cloud tidak boleh mengubah Asset yang sedang digunakan dalam Session yang aktif. Asset locking dimulai saat `PackageSelected` dan berakhir saat `SessionCompleted`.

*Implementasi: Asset version lock per Session*

---

### 10. Deterministic Workflow
Input yang sama harus menghasilkan transisi Workflow yang sama.

Workflow adalah state machine yang deterministik. Tidak ada transisi yang bergantung pada waktu, random, atau kondisi eksternal yang tidak terdefinisi. Ini adalah syarat mutlak untuk debugging dan audit.

*Implementasi: Pure state machine dengan explicit intent handling*

---

## Violation Policy

Jika sebuah fitur atau keputusan teknis berpotensi melanggar salah satu guarantee di atas:

1. **Hentikan implementasi.**
2. **Dokumentasikan trade-off** dalam ADR baru.
3. **Dapatkan persetujuan** dari Lead Software Architect dan Chief Product Architect.
4. **Update guarantee** dengan versi baru yang lebih spesifik jika memang ada exception yang valid.

Tidak ada exception yang boleh terjadi tanpa dokumentasi eksplisit.

---

*Haispace Runtime Guarantees v1.0*
