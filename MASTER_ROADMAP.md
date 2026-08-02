# Haispace Platform — Master Roadmap
**Terakhir diperbarui: 2026-08-02 | Oleh: Chief Product Architect + Antigravity**

---

## Platform DNA

> **"Cloud stores facts. Runtime executes behavior."**

Haispace adalah **satu platform yang terdiri dari beberapa produk** — masing-masing berkembang dengan kecepatan dan tim yang berbeda, tetapi semua bertemu di konstitusi yang sama.

---

## Struktur 4-Track

```
Runtime Track  (M-xxx)     Cloud Track  (C-xxx)
──────────────             ─────────────────────
M-010  ✅                  C-001  📋
M-011  ✅                  C-002  📋
M-012  🔄                  C-003  📋
M-013  📋                  C-004  📋
M-014  📋
    │                           │
    └───────────────────────────┘
                │
        Integration Track (I-xxx)
        ──────────────────────────
        I-001  Asset Sync  📋
                │
        Authoring Track (A-xxx)
        ──────────────────────────
        A-001  Asset Authoring  📋
```

### Prinsip Paralel

> Runtime dan Cloud **boleh berkembang secara paralel**.
> Tidak ada yang perlu menunggu yang lain.
>
> Integration (I-xxx) baru dimulai ketika:
> - Cloud sudah menyediakan API yang dibutuhkan, **DAN**
> - Runtime sudah siap menggunakannya.
>
> Siapa yang lebih dulu selesai, menunggu yang lain. Tidak ada urutan yang dikunci.

---

---

# Runtime Track (M-xxx)

> **Repository:** `haispace-runtime-ios`
> 100% berjalan di dalam iPad. Tidak membutuhkan backend untuk beroperasi.
> Detail lengkap: lihat `docs/ROADMAP.md` di repository Runtime.

| Milestone | Nama | Status |
|---|---|---|
| M-010 | Native Camera Foundation | ✅ Selesai |
| M-011 | Single Runtime Workflow | ✅ Selesai |
| M-012 | Frame Engine | 🔄 Engineering Complete |
| M-013 | Filter Experience | 📋 Planned |
| M-014 | Print Experience | 📋 Planned |

---

---

# Cloud Track (C-xxx)

> **Repository:** `hsp-cloud`
> Cloud menyimpan fakta. Cloud tidak pernah menentukan langkah workflow.
> Sumber: `cloud/BACKEND_IMPLEMENTATION_PLAN.md`

### 📋 C-001 — Infrastructure Foundation
**Target:** Backend siap menerima Booth pertama kali.

```
├── Device Registration   (POST /v1/devices)
├── Authentication        (JWT + apiKey)
├── Organization          (multi-tenant root)
├── Operator Management   (role: owner | manager | staff)
└── Booth Management      (status lifecycle)
```

**DoD:** Booth nyata (iPad dev) dapat register dan berautentikasi ke backend.

---

### 📋 C-002 — Content Delivery
**Bergantung pada:** C-001 selesai.
**Target:** Booth dapat mengambil Manifest dan Asset.

```
├── Event Management      (draft → active → completed → archived)
├── Manifest API          (create draft, publish, version monotonic)
├── Asset API             (upload, checksum, download-url, presigned)
└── Package API           (pricing, captureLimit, deliveryMethods)
```

**DoD:** Booth dapat fetch Manifest dan men-download Asset via presigned URL.

---

### 📋 C-003 — Runtime Data Ingest
**Bergantung pada:** C-002 selesai.
**Target:** Runtime dapat mengirim fakta ke Cloud.

```
├── Session Archive API   (ingest SessionSnapshot, idempotent)
├── Domain Event Upload   (batch, append-only)
└── Audit Upload          (batch, 7-year retention)
```

**DoD:** Runtime dapat upload SessionSnapshot setelah payment + upload DomainEvent batch.

---

### 📋 C-004 — Operations & Analytics
**Bergantung pada:** C-003 selesai.
**Target:** Operator dapat memantau platform via Mission Control.

```
├── Read-only endpoints untuk Mission Control
├── Analytics projections (DailySessionSummary, RevenueByEvent, dll.)
└── Booth health dashboard
```

---

---

# Integration Track (I-xxx)

> Runtime dan Cloud bertemu di sini.
> **Bergantung pada:** C-002 (Manifest + Asset API) sudah production-ready **DAN** M-012+ sudah stable.
> **Repository:** `haispace-runtime-ios` (consumer dari Cloud API)

### 📋 I-001 — Runtime Asset Sync

Runtime membaca Manifest dari Cloud dan men-cache Asset secara lokal.

```
App Launch / setiap 1 jam
    ↓
ManifestService.fetchLatest(boothId)
    ↓
Bandingkan assetRefs vs cache lokal (by checksum)
    ↓
AssetDownloader.download(diff)   ← hanya asset yang berubah
    ↓
~/Library/Caches/HaispaceAssets/{assetId}/
    ↓
CoreImageEditingRuntime baca dari cache
    ↓
Session berjalan offline
```

**Yang dibangun:**
- `ManifestService` — fetch + version pinning per session
- `AssetDownloader` — checksum validation + delta download
- `AssetCache` — LRU, crash-safe, disk-backed
- `ManifestVersionPin` — session aktif tidak mendapat manifest baru

**Invariant kritis:**
- Session yang sedang berjalan **tidak boleh** mendapat manifest baru di tengah jalan
- Wajib online hanya saat: Device Registration + Manifest Fetch pertama

> **Catatan Arsitektur: I-001 AssetManager**
>
> Runtime tidak boleh bergantung pada proses ekstraksi manual. Seluruh lifecycle `.hspasset` (download, verifikasi, ekstraksi, cache, cleanup) menjadi tanggung jawab penuh `AssetManager`. `CoreImageEditingRuntime` hanya bertugas menerima folder matang yang berisi `frame.png` dan `template.json`, tanpa perlu mengetahui detail kompresi, jaringan, maupun validasi integritas.

**Dampak ke Runtime:** Zero breaking change. `CoreImageEditingRuntime` tetap menerima `frameRef.assetPath` berupa direktori. Yang berubah hanya source-nya (sideload manual → `AssetManager` cache).

---

---

# Authoring Track (A-xxx)

> Tools untuk tim desain — bukan untuk customer.
> **Bergantung pada:** I-001 selesai (format asset di Cloud sudah final, output Authoring langsung compatible).
> **Repository:** TBD

### 📋 A-001 — Asset Authoring Platform

```
Upload file aset (PNG transparan, LUT, dll.)
    ↓
AssetSlotDetector (auto-detect slot dari alpha channel — BFS + PCA)
    ↓
Visual Preview + Fine-tune
    ↓
AssetPackageWriter (generate package + checksum)
    ↓
Publish ke HaiBackend (POST /v1/assets)
    ↓
Asset tersedia di Mission Control untuk di-assign ke Manifest
```

**Prinsip:**
> "Designer cukup membuat PNG transparan. Selesai. Sisanya pekerjaan sistem."

**Algoritma yang sama berlaku untuk:** Frame, Overlay, Sticker, Border, Watermark, Filter LUT, Branding.

---

---

## Kutipan Arsitektural

> *"Cloud stores facts. Runtime executes behavior."*
> — Platform DNA, `cloud/CLOUD_CONTRACT.md`

> *"Asset Sync bukan milik Runtime. Asset Sync adalah implementasi Cloud Platform."*
> — Chief Product Architect, 2026-08-02

> *"Haispace bukan lagi satu aplikasi, melainkan satu platform yang terdiri dari beberapa produk. Roadmap kita juga harus mencerminkan struktur tersebut."*
> — Chief Product Architect, 2026-08-02

> *"Runtime dan Cloud boleh berkembang paralel. Integration baru dimulai ketika Cloud sudah menyediakan API yang dibutuhkan dan Runtime sudah siap menggunakannya."*
> — Chief Product Architect, 2026-08-02
