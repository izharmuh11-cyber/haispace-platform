# Authentication — Haispace Cloud

**Status:** Draft
**Versi:** 1.0
**Milestone:** 3 — Cloud Contract
**Penulis:** Antigravity (Lead Software Architect)
**Review:** GPT (Chief Product Architect)

---

> **Prinsip Utama:** Cloud mengenal **identitas device** (Booth), bukan identitas pengguna akhir (Tamu). Tamu tidak memiliki akun. Operator memiliki session. Booth memiliki registrasi permanen.

---

## 1. Aktor dan Identitasnya

Haispace memiliki tiga aktor yang berbicara dengan Cloud:

| Aktor | Identity Type | Scope |
|---|---|---|
| **Booth** (iPad) | Device Certificate / API Key per-device | Mengirim session events, snapshot, audit |
| **Operator** | Operator Session Token | Melihat status, mengkonfigurasi booth, melihat hasil |
| **HaiCamera** (iPhone/iPod) | P2P pairing dengan Booth | Tidak langsung berbicara dengan Cloud — hanya via Booth |

**Tamu tidak punya identitas Cloud.** Tamu hanya dikenal sebagai data dalam Session.

---

## 2. Booth Identity (Runtime Identity)

### Konsep

Setiap iPad yang menjalankan HaispaceBooths memiliki:

```
BoothIdentity
    boothId         UUID permanen — di-generate saat instalasi pertama
    runtimeId       "booth-runtime-ios" (dari RuntimeDescriptor)
    deviceClass     "Booth"
    platform        "iOS"
    publicKey       RSA public key — digunakan untuk verifikasi request
    registeredAt    timestamp registrasi pertama
    lastSeenAt      timestamp terakhir aktif
```

`boothId` disimpan secara permanen di **iOS Keychain** — tidak hilang meski app di-reinstall.

### Device Registration Flow

```
Booth pertama kali dijalankan
    ↓
Cek Keychain: apakah boothId sudah ada?
    ↓ (tidak ada)
Generate UUID sebagai boothId
Generate RSA key pair
    ↓
POST /devices/register
{
  "boothId": "...",
  "runtimeId": "booth-runtime-ios",
  "architectureVersion": "1.0.0",
  "platform": "iOS",
  "deviceClass": "Booth",
  "publicKey": "-----BEGIN PUBLIC KEY-----..."
}
    ↓
Cloud merespons dengan:
{
  "boothId": "...",
  "apiKey": "hsp_booth_...",
  "registeredAt": "..."
}
    ↓
Simpan apiKey ke Keychain
    ↓
Booth siap beroperasi
```

### Re-Registration

Booth harus re-register jika:
- `apiKey` di-revoke oleh Admin (keamanan)
- `boothId` tidak ditemukan di database Cloud (device baru / device reset)
- Error `AUTHENTICATION_ERROR` dengan `reason: DEVICE_NOT_REGISTERED`

---

## 3. Request Authentication

Setiap request dari Booth ke Cloud menggunakan dua header:

```http
X-Booth-Id: <boothId>
X-Api-Key: <apiKey>
```

Untuk request yang membawa payload (POST, PATCH), tambahkan:

```http
X-Request-Id: <UUID> ← untuk idempotency
X-Runtime-Version: 1.0.0
X-Architecture-Version: 1.0.0
```

**Tidak menggunakan Bearer Token** untuk Booth → API Key lebih sederhana dan tidak memerlukan token refresh di tengah session aktif.

---

## 4. Operator Session

Operator login ke Mission Control (bukan ke Booth secara langsung).

### Flow

```
Operator membuka Mission Control (iPad / Mac)
    ↓
Input username + password
    ↓
POST /auth/operator/login
{
  "username": "...",
  "password": "..."
}
    ↓
Respons:
{
  "operatorId": "...",
  "sessionToken": "op_sess_...",
  "expiresAt": "...",
  "permissions": ["view_sessions", "manage_booths", "view_audits"]
}
    ↓
sessionToken digunakan untuk semua request Mission Control
Authorization: Bearer op_sess_...
```

### Token Lifecycle

| Parameter | Value |
|---|---|
| Token TTL | 8 jam (shift kerja operator) |
| Refresh | Sliding window — diperpanjang jika aktif |
| Revoke | Admin dapat revoke kapan saja |

---

## 5. Manifest Authorization

Manifest berisi asset frame dan filter — beberapa bersifat eksklusif per event.

```
GET /manifest?booth={boothId}&event={eventId}
X-Booth-Id: <boothId>
X-Api-Key: <apiKey>
```

Cloud akan memvalidasi:
1. Apakah booth ini terdaftar dan aktif?
2. Apakah booth ini di-assign ke event yang diminta?
3. Apakah manifest untuk event ini tersedia?

Jika booth tidak di-assign ke event, Cloud merespons `PERMISSION_DENIED`.

---

## 6. Boundary Keamanan

### Yang Boleh Dilakukan Booth

| Operasi | Deskripsi |
|---|---|
| Register device | Mendaftarkan diri ke Cloud |
| Upload session events | Mengirim DomainEvent dari session |
| Upload audit records | Mengirim audit trail |
| Upload session snapshot | Mengirim recovery data |
| Fetch manifest | Mengambil manifest yang sesuai event |
| Report delivery status | Melaporkan hasil delivery |

### Yang TIDAK Boleh Dilakukan Booth

| Operasi | Alasan |
|---|---|
| Membuat/mengubah Event (jadwal) | Hanya Admin |
| Membuat/mengubah Package | Hanya Admin |
| Melihat session booth lain | Data isolation per booth |
| Mengakses data billing/revenue | Hanya Admin |
| Merevoke API Key booth lain | Hanya Admin |

---

## 7. Security Boundaries yang Belum Didefinisikan (Future)

Hal berikut **sengaja tidak didefinisikan** di Milestone 3 — akan masuk ke ADR tersendiri saat implementasi backend dimulai:

- JWT / OAuth2 untuk Operator (jika diperlukan SSO)
- Mutual TLS untuk Booth (jika keamanan enterprise diperlukan)
- Rate limiting per booth
- IP allowlist untuk Admin API
- Audit log untuk operator actions

---

*AUTHENTICATION.md v1.0 — Milestone 3 Cloud Contract*
*Ref: constitution/PLATFORM_RUNTIME_V1.md, cloud/SYNC_STRATEGY.md*
