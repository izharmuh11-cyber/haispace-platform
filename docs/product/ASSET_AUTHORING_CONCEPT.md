# Asset Authoring Concept (A-001)
**Terakhir diperbarui: 2026-08-02 | Oleh: Chief Product Architect + Antigravity**

---

## 🎯 Visi & Posisi Produk

**Asset Authoring Platform (A-001)** bukanlah sebuah aplikasi yang berdiri sendiri, melainkan **Modul / Workspace di dalam Mission Control**.

Ini mengembalikan fitur-fitur terbaik dari *legacy photobooth web*, tetapi dengan output dan arsitektur yang 100% berbeda mengikuti prinsip Haispace Platform:
> *"Cloud stores facts. Runtime executes behavior."*

---

## 🔄 Evolusi Workflow

### Legacy Project (Ditinggalkan)
```
Admin Upload PNG
       │
Auto Detect Slot (Server-side)
       │
Hapus Green Screen (Server-side)
       │
Geser / Rotate Manual
       │
Save (frame.png + koordinat masuk database SQL)
```
*Masalah: Membebani server, lambat, dan tidak portable.*

### Haispace Platform (A-001)
```
Mission Control (Browser Web)
       │
Upload File (PNG, LUT, dll)
       │
Auto Detect Slot (Client-side WebAssembly / Canvas)
       │
Hapus Green Screen (Client-side Canvas)
       │
Geser / Rotate Manual (Drag & Drop Editor)
       │
Publish
```
*Kelebihan: Server tidak memproses gambar sama sekali.*

---

## 📦 Output Akhir: Asset Package

Perubahan paling radikal ada pada output akhir. Setelah menekan tombol **Publish**, Mission Control akan menghasilkan **Asset Package** lengkap dan mengirimkannya ke Cloud CDN.

Contoh untuk Frame Asset:
```
wedding-classic/
    ├── frame.png           ← Gambar final bersih
    ├── template.json       ← Koordinat slot (otomatis/manual)
    ├── thumbnail.webp      ← Gambar kecil untuk UI
    ├── preview.jpg         ← Gambar contoh hasil
    └── asset-manifest.json ← Metadata, versi, checksum
```

**Alur Distribusi:**
1. **Mission Control** → Upload Asset Package ke Cloud CDN.
2. **Cloud** → Menerima, memvalidasi checksum, dan mendaftarkannya sebagai entitas `Asset` di database.
3. **Mission Control** → Operator memasukkan `Asset` tersebut ke dalam `Manifest` milik sebuah `Event`.
4. **iPad (Runtime)** → Fetch Manifest → Download Asset Package dari Cloud CDN ke cache lokal.
5. **iPad (Runtime)** → Merender foto tanpa perlu tahu bagaimana frame tersebut dibuat.

---

## ✅ Kesimpulan Arsitektural

1. **UX Dipertahankan:** Kemudahan desainer (auto-detect, drag & drop) tidak hilang, justru menjadi alat utama (*Authoring Tool*).
2. **Beban Server Hilang:** Seluruh *image processing* pindah ke laptop desainer (browser client-side).
3. **Runtime Sederhana:** iPad tidak memiliki editor frame. iPad fokus 100% melayani tamu dan mengeksekusi asset yang sudah matang.
