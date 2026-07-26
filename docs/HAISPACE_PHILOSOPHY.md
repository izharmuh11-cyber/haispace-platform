# Haispace Philosophy

> Dokumen ini adalah nilai dan prinsip yang tidak berubah.
> Setiap keputusan teknis, produk, dan desain harus dapat dipertanggungjawabkan terhadap prinsip-prinsip ini.

---

## 12 Prinsip Haispace

### 1. Operator tidak boleh berpikir tentang teknologi.
Teknologi adalah tanggung jawab sistem, bukan operator. Operator datang untuk menjalankan event, bukan mengelola infrastruktur. Setiap antarmuka yang memerlukan pemahaman teknis adalah kegagalan desain.

### 2. Booth harus tetap menghasilkan uang tanpa Cloud.
Cloud adalah mitra, bukan pengendali. Selama Asset dan Manifest sudah tersedia secara lokal, Booth harus mampu menyelesaikan event dan menghasilkan revenue, terlepas dari kondisi koneksi internet.

### 3. Cloud memperkaya operasional, bukan mengendalikan operasional.
Cloud bertugas untuk sinkronisasi, distribusi, analitik, dan notifikasi. Cloud tidak pernah menjadi syarat agar Workflow dapat berjalan.

### 4. Runtime tidak mengetahui CMS.
HaiBooth tidak tahu cara membuat paket, mengunggah frame, atau mengelola operator. HaiBooth hanya tahu cara menjalankan event. Batas ini tidak boleh dilanggar.

### 5. Setiap Asset memiliki lifecycle.
Asset dibuat di Admin, dipublikasikan ke Cloud, disinkronkan ke Booth, digunakan dalam Session, dan diarsipkan setelah tidak aktif. Lifecycle ini harus eksplisit dan dapat dilacak.

### 6. Semua keputusan engineering harus mengurangi beban operator.
Sebelum menambahkan fitur, tanya: *"Apakah ini membantu operator bekerja lebih mudah?"* Kalau jawabannya tidak, fitur tersebut harus dipertimbangkan ulang.

### 7. Session lebih penting daripada Hardware.
Hardware bisa diganti. Session yang hilang adalah kegagalan bisnis. Sistem harus memprioritaskan keberlangsungan Session di atas segalanya.

### 8. Data boleh terlambat. Session tidak boleh hilang.
Analitik boleh diupload nanti. Audit boleh disinkronkan nanti. Tetapi Session yang sedang berjalan tidak boleh hilang karena alasan apapun — restart, crash, atau koneksi putus.

### 9. Semua proses harus dapat diaudit.
Setiap perubahan state menghasilkan DomainEvent yang tercatat. Tidak ada tindakan yang terjadi tanpa jejak. Ini adalah syarat untuk kepercayaan, debugging, dan kepatuhan bisnis.

### 10. Produk harus dapat dipahami tanpa membaca kode.
Dokumentasi ini adalah produk. Kalau seseorang tidak dapat memahami cara kerja Haispace hanya dari membaca repository ini, maka dokumentasinya belum selesai.

### 11. Complexity is a liability.
Setiap abstraksi harus dapat dijelaskan dalam satu kalimat. Kalau tidak bisa, terlalu kompleks. Kesederhanaan adalah fitur, bukan keterbatasan.

### 12. The contract is the product.
API boleh berubah. Implementasi boleh berubah. Tetapi DomainEvent dan Runtime Guarantees tidak boleh berubah tanpa versi major. Kontrak yang stabil adalah fondasi ekosistem yang dapat dipercaya.

---

## Konstitusi Platform

> **Booth adalah produk yang harus tetap menghasilkan uang walaupun Cloud sedang tidak bisa membantu.**

Kalau suatu hari Vultr mati, Cloudflare bermasalah, atau internet venue putus, operator tetap harus bisa melayani tamu, mencetak foto, dan menyelesaikan event.

Cloud ada untuk memperkaya pengalaman dan menyinkronkan data, bukan menjadi syarat agar Booth dapat bekerja.

---

*Haispace Philosophy v1.0 — ditetapkan sebagai fondasi platform.*
