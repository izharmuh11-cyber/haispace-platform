# Asset Package Specification v1
**Status:** Draft v1.0
**Terakhir diperbarui:** 2026-08-02
**Target Pembaca:** Cloud Backend, Runtime iOS, Mission Control, Authoring Platform

---

## 1. Definisi Asset Package

**Asset Package** adalah format standar distribusi konten di Haispace Platform. Sebuah aset tidak pernah dikirimkan sebagai file tunggal (misal: hanya `frame.png`), melainkan selalu dibungkus dalam sebuah paket (folder) yang utuh (*self-contained*).

Paket ini di-generate oleh **Asset Authoring Platform (A-001)**, didistribusikan oleh **Cloud (C-002)**, dan dikonsumsi oleh **Runtime (I-001)**.

---

## 2. Struktur Dasar (Berlaku Semua Tipe)

Setiap Asset Package, terlepas dari tipenya, wajib mengikuti struktur folder ini:

```
{assetId}/
    ├── asset-manifest.json    ← WAJIB: Common Metadata (Lihat ASSET_TAXONOMY.md)
    ├── thumbnail.webp         ← WAJIB: Low-res (max 256x256), < 50KB
    └── [file_spesifik_tipe...]
```

### Fungsi `thumbnail.webp`
- **Di Mission Control:** Digunakan untuk daftar/grid aset saat operator merancang *Manifest* untuk suatu Event.
- **Di iPad (Runtime):** Digunakan pada UI *picker* jika tamu diperbolehkan memilih aset (misalnya memilih filter atau frame sebelum foto).
- **Format:** Selalu WebP untuk ukuran sangat kecil, rasio 1:1 atau disesuaikan desain UI picker.

---

## 3. Spesifikasi Tipe: Frame (`type: "frame"`)

Paket dengan `type: "frame"` merupakan template foto.

### Struktur Paket
```
wedding-classic/
    ├── asset-manifest.json
    ├── thumbnail.webp
    ├── frame.png           ← File visual utama
    ├── template.json       ← Konfigurasi koordinat
    └── preview.jpg         ← High-res preview dengan wajah *dummy*
```

### Penjelasan File Spesifik

#### A. `frame.png`
- **Format:** PNG-24 dengan Alpha Channel (Transparansi).
- **Fungsi:** Menjadi *overlay* paling atas. Bagian yang ber-alpha=0 akan menjadi "lubang" tempat foto tamu muncul.
- **Resolusi:** Harus sesuai dengan dimensi cetak (contoh: 1200x1800 untuk cetak 4x6 inci pada 300dpi).

#### B. `template.json`
Dihasilkan secara otomatis oleh *Asset Authoring Platform*. Digunakan oleh `CoreImageEditingRuntime` di iPad untuk menempatkan foto jepretan kamera tepat di bawah lubang PNG.

```json
{
  "canvas": {
    "width": 1200,
    "height": 1800,
    "dpi": 300,
    "orientation": "portrait"
  },
  "bleed": {
    "top": 30, "bottom": 30, "left": 30, "right": 30
  },
  "safeArea": {
    "top": 60, "bottom": 60, "left": 60, "right": 60
  },
  "slots": [
    {
      "index": 0,
      "x": 100,
      "y": 150,
      "width": 1000,
      "height": 600,
      "rotation": 0,
      "zIndex": -1
    },
    {
      "index": 1,
      "x": 100,
      "y": 800,
      "width": 1000,
      "height": 600,
      "rotation": 0,
      "zIndex": -1
    }
  ]
}
```

#### C. `preview.jpg`
- **Format:** JPEG kualitas medium.
- **Fungsi:** Ini adalah gambar *dummy* (contoh) yang menunjukkan bagaimana frame terlihat jika sudah diisi foto wajah orang. 
- **Penggunaan:** **HANYA** digunakan oleh admin/operator di Mission Control untuk melihat pratinjau (*preview*) nyata tanpa perlu merender ulang koordinat di web. 
- **Aturan:** Runtime (iPad) sama sekali tidak mengunduh atau membaca file ini (dapat diabaikan oleh `AssetDownloader` untuk menghemat *bandwidth*).

---

## 4. Siklus Hidup File

1. **Pembuatan:** *Asset Authoring Platform* menghasilkan PNG dan JSON, lalu menggabungkannya dengan foto wajah *dummy* (*stock photo*) untuk merender `preview.jpg`. WebP `thumbnail.webp` di-*generate* untuk grid UI.
2. **Pengiriman:** Paket di-*zip* (atau di-upload per file) ke Cloud CDN. Cloud Backend mencatat *checksum* total paket.
3. **Penyimpanan Cache:** Runtime iPad mengunduh `asset-manifest.json`, `frame.png`, `template.json`, dan `thumbnail.webp` ke folder `~/Library/Caches/HaispaceAssets/wedding-classic/`.
4. **Rendering:** `CoreImageEditingRuntime` mem-parsing `template.json`, memotong (*crop/scale*) foto jepretan tamu sesuai `slots`, dan menumpuknya di bawah `frame.png`.
