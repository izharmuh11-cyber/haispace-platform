# Platform Principles

> **Konstitusi Haispace Platform.**
>
> Ini adalah aturan yang tidak boleh dilanggar oleh seluruh platform, oleh siapapun, kapanpun.
> Philosophy menjelaskan *mengapa*. Platform Principles menjelaskan *apa yang tidak boleh dilanggar*.
>
> Jika implementasi bertentangan dengan dokumen ini, yang salah adalah implementasi — bukan dokumen.

---

## Principle 1 — Cloud Assists, Never Controls

Cloud membantu Booth. Cloud tidak pernah mengendalikan Booth.

Booth beroperasi secara mandiri. Cloud hanya menyediakan sinkronisasi, distribusi, dan analitik. Tidak ada keputusan operasional di dalam Booth yang bergantung pada respons real-time dari Cloud.

---

## Principle 2 — Single Source of Truth

Semua data memiliki satu sumber kebenaran. Tidak boleh ada dua sistem yang merasa paling benar untuk data yang sama.

| Data | Owner | Source of Truth | Cached By | Conflict Resolution |
|------|-------|----------------|-----------|---------------------|
| Session state | Booth Runtime | Booth (lokal) | Cloud (upload) | Booth wins — session sedang berjalan |
| Asset files | Asset Service | Cloud Storage (R2) | Booth (lokal) | Cloud wins — berdasarkan checksum |
| Asset metadata | Asset Service | Cloud (manifest) | Booth (lokal) | Cloud wins — berdasarkan version |
| Package definition | Cloud CMS | Cloud | Booth (synced) | Cloud wins — kecuali Session sedang aktif |
| Operator identity | Auth Service | Cloud | Booth (session cache) | Cloud wins |
| Audit trail | Booth Runtime | Booth → Cloud | — | Booth wins — immutable setelah tercatat |
| Event configuration | Cloud CMS | Cloud | Booth (synced) | Cloud wins — hanya berlaku untuk Session berikutnya |

---

## Principle 3 — Workflow Belongs to Runtime

Workflow selalu dimiliki Runtime. Bukan Admin. Bukan API. Bukan Dashboard.

Cloud dapat mengkonfigurasi Workflow melalui Package dan Manifest, tetapi eksekusi Workflow selalu dilakukan oleh Booth secara lokal. Tidak ada instruksi step-by-step yang datang dari Cloud saat Workflow sedang berjalan.

---

## Principle 4 — Configuration via Manifest, Not Behavior

Semua konfigurasi datang dari Manifest. Tidak ada *magic value* yang dikodekan langsung di dalam aplikasi.

Ini termasuk: layout, pricing policy, workflow policy, delivery channel, asset list, capability requirements, dan minimum version.

**Manifest mendeskripsikan konfigurasi, bukan perilaku runtime.**

Manifest boleh mengatakan:
- Gunakan layout A
- Gunakan theme B
- Harga Rp50.000
- Printer wajib tersedia

Manifest **tidak boleh** mengatakan:
- Jika payment sukses, lanjut ke Capture
- Jika gagal, ulangi dari step X

Workflow logic tetap sepenuhnya milik Runtime.

---

## Principle 5 — Revenue Must Not Depend on Connectivity

Booth harus bisa menghasilkan revenue walaupun internet mati.

Selama Asset dan Manifest sudah tersedia secara lokal dan Operator sudah login, tidak ada alasan teknis yang boleh menghentikan Booth dari melayani tamu dan mencetak foto.

**Cloud outage tidak boleh menjadi alasan event berhenti.**

Ini adalah pegangan seluruh tim engineering. Jika sebuah fitur baru membuat Booth bergantung pada koneksi real-time, fitur tersebut melanggar prinsip ini dan harus didesain ulang.

---

## Principle 6 — Assets are Data, Not Code

Asset adalah data yang memiliki lifecycle, versi, dan checksum — bukan bagian dari source code aplikasi.

Asset tidak di-bundle ke dalam binary aplikasi. Asset dikelola secara independen melalui Asset Service dan didistribusikan melalui Cloud Storage.

Asset mencakup semua jenis media dan konfigurasi konten:
- Gambar: Frame, Overlay, Sticker, Border
- Tipografi: Font
- Video: Intro, Outro, Countdown, Animation
- Audio: Background Music, Sound Effect
- Warna & Visual: LUT Filter, Color Grading Profile
- Masa depan: AI Prompt, 3D Asset, AR Marker

Setiap jenis Asset mengikuti lifecycle yang sama: Created → Uploaded → Validated → Published → Cached → Expired.

---

## Principle 7 — Remote Configurability

Semua perubahan konfigurasi yang besar harus bisa dilakukan dari Cloud tanpa mempublikasikan versi aplikasi baru.

Pengecualian hanya berlaku untuk perubahan yang memerlukan kemampuan platform baru (fitur OS, hardware baru). Perubahan bisnis — harga, paket, frame, tema — tidak pernah memerlukan update aplikasi.

---

## Principle 8 — UI Has No Business Logic

Tidak ada UI yang memiliki business logic.

Semua logic bisnis berada pada Domain. UI hanya merender state dan mengirim intent. Keputusan tentang apa yang boleh dan tidak boleh dilakukan selalu dibuat oleh Domain, bukan oleh tampilan.

---

## Principle 9 — Resilient Runtime

Semua service boleh gagal. Runtime tetap hidup.

Jika kamera gagal, Booth memberikan pesan yang jelas dan memungkinkan retry. Jika printer gagal, Delivery masuk ke queue. Jika Cloud gagal, Booth tetap beroperasi. Tidak ada single point of failure yang dapat menghentikan Runtime sepenuhnya.

---

## Principle 10 — Operator is Not a Debugger

Operator tidak pernah menjadi debugger.

Operator hanya menjalankan event. Jika terjadi masalah teknis, sistem harus menanganinya secara otomatis atau memberikan instruksi yang sangat jelas dalam bahasa manusia — bukan error code, bukan stack trace, bukan dialog teknis.

---

## Principle 11 — Immutable Session

Setelah Session dimulai, Package, Asset Version, Layout, Theme, dan Workflow Policy untuk Session tersebut tidak boleh berubah.

Update Manifest, Package, atau Asset dari Cloud hanya berlaku untuk Session berikutnya — tidak pernah menginterupsi Session yang sedang berjalan.

Ini adalah garansi konsistensi bagi tamu: pengalaman yang mereka mulai adalah pengalaman yang mereka selesaikan.

---

## Principle 12 — Everything is Observable

Setiap keputusan penting Runtime menghasilkan Domain Event.

Jika sesuatu terjadi tetapi tidak menghasilkan event, maka bagi platform kejadian itu dianggap tidak pernah terjadi. Tidak ada tindakan yang tersembunyi dari sistem observability.

Ini adalah fondasi untuk: Audit Trail, Analytics, Mission Control, AI Assistant, dan Debugging.

---

## Violation Protocol

Jika ada keputusan teknis atau produk yang berpotensi melanggar salah satu principle di atas:

1. **Dokumentasikan trade-off** dalam ADR baru di `/adr/`
2. **Dapatkan persetujuan eksplisit** dari Lead Software Architect dan Chief Product Architect
3. **Catat sebagai exception** dengan batas waktu yang jelas untuk diperbaiki

Tidak ada principle yang boleh dilanggar secara diam-diam.

---

*Haispace Platform Principles v1.1 — revised per Architecture Review*
