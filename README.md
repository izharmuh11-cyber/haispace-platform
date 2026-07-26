# Haispace Platform

> **Haispace Platform** adalah sumber kebenaran tunggal (*single source of truth*) untuk seluruh ekosistem Haispace.

Repository ini bukan kode. Repository ini adalah **fondasi** — tempat di mana semua keputusan produk, arsitektur, dan kontrak sistem didokumentasikan sebelum diimplementasikan.

---

## Prinsip Utama

> **Booth adalah produk yang harus tetap menghasilkan uang walaupun Cloud sedang tidak bisa membantu.**

---

## Ekosistem Haispace

| Aplikasi | Peran | Repository |
|----------|-------|-----------|
| **HaiBooth** | Runtime — menjalankan event | `haispace-booth` |
| **HaiCamera** | Capture Device — mengambil media | `haispace-camera` |
| **HaiAdmin** | CMS — mengelola konten & bisnis | `haispace-admin` |
| **HaiBackend** | Cloud — sinkronisasi & distribusi | `haispace-backend` |

Semua aplikasi mengacu ke repository ini sebagai kontrak bersama.

---

## Struktur Dokumentasi

```
docs/
├── philosophy/       ← Nilai dan prinsip yang tidak berubah
├── product/          ← Deskripsi produk dan aktor
├── runtime/          ← Runtime Guarantees dan Booth behavior
├── cloud/            ← Cloud responsibilities dan strategi
├── glossary/         ← Bahasa bersama seluruh tim
├── architecture/     ← Keputusan arsitektur tingkat tinggi
├── specifications/   ← Domain model dan spesifikasi teknis
├── adr/              ← Architecture Decision Records
└── reference/        ← Contoh JSON, OpenAPI, Manifest samples
```

---

## Dokumen Inti

- [Haispace Philosophy](./HAISPACE_PHILOSOPHY.md) — 12 prinsip yang tidak berubah
- [Glossary](./GLOSSARY.md) — Bahasa bersama seluruh platform
- [Runtime Guarantees](./RUNTIME_GUARANTEES.md) — Janji sistem kepada operator
- [Domain Events](./DOMAIN_EVENTS.md) — Kamus event seluruh platform

---

## Cara Menggunakan Repository Ini

### Sebelum membangun fitur baru:
1. Pastikan domain yang disentuh sudah ada di `GLOSSARY.md`
2. Pastikan tidak ada pelanggaran `RUNTIME_GUARANTEES.md`
3. Tambahkan Domain Event baru ke `DOMAIN_EVENTS.md` jika diperlukan
4. Tulis ADR di `docs/adr/` jika membuat keputusan arsitektur

### Aturan:
- Tidak ada implementasi baru tanpa dokumen yang disepakati
- Setiap ADR harus merujuk ke Philosophy atau Runtime Guarantees yang relevan
- Setiap fitur baru harus menyebutkan domain yang disentuh dan event yang ditambahkan

---

## Tim

| Peran | Tanggung Jawab |
|-------|---------------|
| **Product Owner** (Izhar) | Visi produk, QA akhir, keputusan bisnis |
| **Chief Product Architect** (GPT) | Blueprint produk, domain model, product boundary |
| **Lead Software Architect** (Antigravity) | Implementasi iOS, kontrak teknis, Runtime Guarantees |

---

*Haispace Platform — dibangun untuk bertahan bertahun-tahun.*
