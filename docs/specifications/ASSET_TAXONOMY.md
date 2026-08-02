# Asset Taxonomy & Common Metadata
**Status:** Draft v1.0
**Terakhir diperbarui:** 2026-08-02
**Target Pembaca:** Cloud Backend, Runtime iOS, Mission Control, Authoring Platform

---

## 1. Konsep Dasar

Dalam ekosistem Haispace, **Frame bukanlah entitas yang berdiri sendiri**. Frame hanyalah salah satu tipe dari konsep yang lebih besar yaitu **Asset**. 

Semua *capability* produk yang didistribusikan secara dinamis ke Runtime (iPad) dibungkus sebagai Asset. Hal ini memungkinkan Cloud dan Runtime memiliki satu mekanisme *sync* (`AssetDownloader`) dan satu struktur penyimpanan cache (`AssetCache`) yang seragam untuk semua fitur masa depan.

---

## 2. Asset Type Taxonomy

Berikut adalah taksonomi resmi jenis-jenis Asset yang didukung oleh Haispace Platform:

| Type (`type`) | Fungsi | Konsumen di Runtime | Keterangan |
|---|---|---|---|
| `frame` | Template foto dengan lubang (slot) transparan | `CoreImageEditingRuntime` | Mengandung koordinat slot (X, Y) |
| `filter` | Efek pewarnaan / color grading | `FilterCapability` | Berupa file LUT (`.cube`) atau Metal Shader |
| `overlay` | Grafis yang menimpa foto tanpa slot (misal: *light leak*) | `CoreImageEditingRuntime` | Menimpa seluruh canvas |
| `sticker` | Elemen grafis interaktif (bisa digeser tamu) | `StickerCapability` | Asset *draggable* |
| `watermark` | Logo permanen di pojok foto | `CoreImageEditingRuntime` | Dikunci oleh operator |
| `printer_profile` | Konfigurasi ICC Profile atau setelan thermal printer | `PrinterCapability` | Untuk kalibrasi warna cetak (DNP/AirPrint) |
| `audio` | Musik latar saat *countdown* atau *video booth* | `AudioCapability` | File WAV/MP3 |

---

## 3. Common Metadata Structure

Agar *parser* di Runtime dan Cloud tetap generik, **setiap Asset Package WAJIB memiliki satu file bernama `asset-manifest.json` di root foldernya**. 

File ini berisi metadata umum (Common Metadata) yang strukturnya sama persis terlepas dari tipe assetnya.

### Schema `asset-manifest.json`

```json
{
  "assetId": "string (UUID v4 atau slug unik)",
  "type": "enum (frame | filter | sticker | overlay | watermark | printer_profile | audio)",
  "version": "integer (monotonik naik)",
  "checksum": "string (SHA-256 hash dari isi package)",
  "name": "string (Nama human-readable)",
  "author": "string (Nama kreator/organisasi)",
  "createdAt": "ISO-8601 UTC timestamp",
  "compatibility": {
    "minArchitectureVersion": "integer (versi platform terendah yang didukung)"
  },
  "files": [
    "string (Daftar file yang ada di dalam package ini)"
  ]
}
```

---

## 4. Keuntungan Arsitektur Ini

1. **Zero Parser Duplication:** Runtime `AssetCache` dan `AssetDownloader` hanya perlu membaca `asset-manifest.json` untuk memvalidasi *checksum* dan ketersediaan paket.
2. **Scalable Backend:** Cloud API (`POST /v1/assets`) tidak perlu membedakan *endpoint* untuk *frame* atau *filter*. Semua masuk melalui satu gerbang aset.
3. **Mission Control Agnostic:** UI untuk *assign* aset ke sebuah Event cukup mem-filter berdasarkan field `type`.
