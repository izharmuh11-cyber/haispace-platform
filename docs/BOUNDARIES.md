# Haispace Boundaries

> **Batas tanggung jawab setiap komponen dalam ekosistem Haispace.**
>
> Dokumen ini menyelesaikan pertanyaan: *"Fitur ini masuk ke mana?"*
> Jika sebuah fitur tidak sesuai dengan boundary komponennya, fitur tersebut harus dipindahkan atau didesain ulang.

---

## HaiBooth (Runtime)

### Boleh:
- Login Operator dan validasi sesi
- Membaca Event yang sudah di-assign
- Sinkronisasi dan validasi Manifest dari Cloud
- Men-cache Asset secara lokal
- Menjalankan Workflow (Capture → Processing → Payment → Delivery)
- Membuat dan menyimpan Session
- Mengelola Print Queue dan Delivery Queue
- Menampilkan status operasional (Readiness, Health)
- Mengirim Domain Events ke local audit trail
- Retry delivery secara otomatis
- Beroperasi secara penuh tanpa koneksi internet

### Tidak Boleh:
- Membuat atau mengedit Package
- Mengunggah atau mengubah Asset
- Mengelola Operator (tambah, edit, hapus)
- Mengubah harga atau pricing policy
- Membuat atau menutup Event
- Mengakses data Session milik Booth lain
- Mengekspos endpoint API eksternal
- Mengambil keputusan bisnis yang tidak didefinisikan dalam Manifest

---

## HaiCamera (Capture Device)

### Boleh:
- Menerima perintah Capture dari HaiBooth via P2P
- Mengambil foto/video sesuai konfigurasi
- Mengirim hasil Capture ke HaiBooth
- Melaporkan status kamera (ready, capturing, error)
- Menyimpan hasil sementara jika transfer gagal

### Tidak Boleh:
- Membuat Session
- Memproses pembayaran
- Berkomunikasi langsung ke Cloud (bypass HaiBooth)
- Menyimpan data tamu
- Mengakses Manifest atau Package

---

## HaiAdmin (CMS)

### Boleh:
- Mengelola Organization, Venue, Event
- Membuat dan mengedit Package
- Mengunggah dan mempublikasikan Asset
- Mengelola Operator dan permission
- Mengatur pricing dan promo
- Melihat analytics dan laporan
- Mengelola Booth (registrasi, assignment, konfigurasi)
- Membuat dan mempublikasikan Manifest

### Tidak Boleh:
- Mengeksekusi Workflow
- Mengakses atau mengubah Session yang sedang berjalan
- Mengirim instruksi langsung ke Booth saat Event berlangsung
- Membuat keputusan operasional real-time
- Melihat data pembayaran individual tamu (hanya summary)

---

## Cloud (Backend Services)

### Boleh:
- Autentikasi Operator dan BoothIdentity
- Menyimpan dan mendistribusikan Asset via Storage
- Melayani Manifest request dari Booth
- Menyimpan analytics dan audit trail yang dikirim Booth
- Mengirim notifikasi ke Operator
- Menyimpan definisi Event, Package, Organization
- Menyediakan API untuk HaiAdmin

### Tidak Boleh:
- Mengendalikan Workflow yang sedang berjalan di Booth
- Membatalkan Session secara paksa (tanpa consent Operator)
- Menjadi single point of failure bagi operasional Booth
- Menyimpan media foto/video tamu secara permanen tanpa consent
- Mengubah konfigurasi Booth saat Event sedang berlangsung

---

## Operator (Human Actor)

### Boleh:
- Login ke Booth yang di-assign
- Memilih Event yang akan dijalankan
- Memulai, pause, dan resume Event
- Memonitor status Session hari ini
- Melakukan emergency stop jika diperlukan
- Retry delivery yang gagal
- Menjalankan self-check sebelum Event dimulai

### Tidak Boleh:
- Membuat atau mengedit Package
- Mengunggah Asset
- Mengubah harga
- Menambah atau menghapus Operator lain
- Mengakses data tamu dari event lain
- Mengakses Mission Control tanpa otoritas

---

## AI Assistant (Future)

### Boleh:
- Membaca Domain Events untuk analisis
- Memberikan rekomendasi kepada Operator berdasarkan kondisi runtime
- Mendeteksi anomali dan memberikan peringatan dini
- Membantu self-check sebelum Event dimulai
- Memberikan laporan ringkas kepada Owner

### Tidak Boleh:
- Mengeksekusi tindakan tanpa konfirmasi manusia (kecuali yang didefinisikan eksplisit)
- Mengakses data pribadi tamu
- Mengubah konfigurasi Event secara langsung
- Menggantikan keputusan Operator dalam situasi darurat

---

## Prinsip Boundary

> Jika ada fitur yang terasa "perlu ada di mana-mana", kemungkinan besar ia adalah Domain yang belum terdefinisi — bukan fitur yang harus duplikasi.

Sebelum mendebat di mana sebuah fitur harus berada, periksa dulu:
1. Siapa *aktor* yang membutuhkan fitur ini?
2. Data apa yang disentuh fitur ini, dan siapa *owner*-nya?
3. Apakah fitur ini berkaitan dengan operasional real-time atau pengelolaan konten?

Jawaban dari ketiga pertanyaan ini biasanya langsung menunjukkan di mana fitur tersebut harus berada.

---

*Haispace Boundaries v1.0*
