# Haispace Domain Events

> **Kamus event seluruh platform Haispace.**
>
> Domain Events adalah bahasa bersama yang digunakan oleh semua komponen:
> HaiBooth, HaiCamera, HaiAdmin, Analytics, Mission Control, dan AI Assistant.
>
> Tidak ada komponen yang berkomunikasi melalui API internal.
> Semua komponen berbicara melalui Domain Events.

---

## Prinsip Domain Events

1. **Events adalah fakta.** Event adalah sesuatu yang sudah terjadi, bukan perintah.
2. **Events bersifat immutable.** Event yang sudah diterbitkan tidak dapat diubah.
3. **Events memiliki urutan.** Setiap event memiliki timestamp dan sequence number.
4. **Events adalah audit trail.** Semua event disimpan secara persisten sebelum diproses.

---

## Konvensi Penamaan

```
[Domain].[Entity][PastTense]

Contoh:
  Session.Created
  Payment.CommitmentAccepted
  Delivery.Completed
```

---

## Katalog Domain Events

### Session Domain

| Event | Deskripsi | Payload |
|-------|-----------|---------|
| `Session.Created` | Session baru dimulai | `sessionId`, `boothId`, `timestamp` |
| `Session.PackageSelected` | Tamu memilih paket | `sessionId`, `packageId`, `packageName` |
| `Session.Aborted` | Session dibatalkan | `sessionId`, `reason`, `abortedBy` |
| `Session.Completed` | Session selesai dengan sukses | `sessionId`, `duration`, `timestamp` |

---

### Capture Domain

| Event | Deskripsi | Payload |
|-------|-----------|---------|
| `Capture.Started` | Proses pengambilan media dimulai | `sessionId`, `captureIndex`, `captureType` |
| `Capture.Completed` | Media berhasil diambil | `sessionId`, `captureId`, `mediaPath`, `captureType` |
| `Capture.Failed` | Proses pengambilan media gagal | `sessionId`, `captureIndex`, `reason` |
| `Capture.Retried` | Tamu mengulang pengambilan media | `sessionId`, `captureIndex`, `previousCaptureId` |

---

### Media Processing Domain

| Event | Deskripsi | Payload |
|-------|-----------|---------|
| `MediaProcessing.Started` | Proses editing/compositing dimulai | `sessionId`, `captureIds` |
| `MediaProcessing.PreviewReady` | Preview siap ditampilkan ke tamu | `sessionId`, `previewPath` |
| `MediaProcessing.ExportCompleted` | Export full-resolution selesai | `sessionId`, `outputPath`, `outputRef` |
| `MediaProcessing.Failed` | Proses gagal | `sessionId`, `reason` |

---

### Payment Domain

| Event | Deskripsi | Payload |
|-------|-----------|---------|
| `Payment.Pending` | Proses pembayaran dimulai | `sessionId`, `amount`, `method` |
| `Payment.CommitmentAccepted` | Booth menerima konfirmasi lokal | `sessionId`, `localTransactionId`, `method` |
| `Payment.CommitmentVerified` | Cloud mengkonfirmasi pembayaran | `sessionId`, `serverId`, `verifiedAt` |
| `Payment.Failed` | Pembayaran gagal | `sessionId`, `reason`, `method` |
| `Payment.Refunded` | Pembayaran dikembalikan | `sessionId`, `refundId`, `reason` |

---

### Delivery Domain

| Event | Deskripsi | Payload |
|-------|-----------|---------|
| `Delivery.Queued` | Pengiriman masuk ke antrian | `sessionId`, `deliveryId`, `channel` |
| `Delivery.Started` | Pengiriman dimulai | `sessionId`, `deliveryId`, `channel` |
| `Delivery.Completed` | Pengiriman berhasil | `sessionId`, `deliveryId`, `channel`, `recipient` |
| `Delivery.Failed` | Pengiriman gagal | `sessionId`, `deliveryId`, `channel`, `reason`, `retryCount` |
| `Delivery.Retrying` | Sistem mencoba ulang pengiriman | `sessionId`, `deliveryId`, `retryCount`, `nextRetryAt` |

---

### Booth Domain

| Event | Deskripsi | Payload |
|-------|-----------|---------|
| `Booth.Started` | Booth pertama kali menyala | `boothId`, `appVersion`, `timestamp` |
| `Booth.Ready` | Booth siap menerima tamu | `boothId`, `eventId`, `assetVersion` |
| `Booth.HealthChanged` | Status kesehatan booth berubah | `boothId`, `from`, `to`, `reason` |
| `Booth.OperatorLoggedIn` | Operator login | `boothId`, `operatorId`, `timestamp` |
| `Booth.OperatorLoggedOut` | Operator logout | `boothId`, `operatorId`, `timestamp` |
| `Booth.EmergencyStop` | Operator menghentikan booth secara darurat | `boothId`, `operatorId`, `reason` |

---

### Asset & Manifest Domain

| Event | Deskripsi | Payload |
|-------|-----------|---------|
| `Manifest.SyncStarted` | Sinkronisasi manifest dimulai | `boothId`, `localVersion`, `remoteVersion` |
| `Manifest.SyncCompleted` | Sinkronisasi selesai | `boothId`, `version`, `changedAssets` |
| `Manifest.SyncFailed` | Sinkronisasi gagal | `boothId`, `reason`, `nextRetryAt` |
| `Asset.DownloadStarted` | Download asset dimulai | `assetId`, `version`, `size` |
| `Asset.DownloadCompleted` | Download asset selesai | `assetId`, `version`, `checksum` |
| `Asset.DownloadFailed` | Download asset gagal | `assetId`, `reason` |
| `Asset.ChecksumMismatch` | Checksum asset tidak cocok | `assetId`, `expected`, `actual` |

---

### Sync Domain

| Event | Deskripsi | Payload |
|-------|-----------|---------|
| `Sync.AuditUploaded` | Audit trail berhasil dikirim ke Cloud | `boothId`, `sessionIds`, `eventCount` |
| `Sync.AnalyticsUploaded` | Analytics berhasil dikirim | `boothId`, `period`, `sessionCount` |
| `Sync.DeliveryRetried` | Delivery dari queue berhasil diretry | `deliveryId`, `sessionId` |
| `Sync.ConflictDetected` | Konflik data terdeteksi | `boothId`, `conflictType`, `localVersion`, `remoteVersion` |

---

## Event Envelope

Setiap Domain Event dibungkus dalam envelope standar:

```json
{
  "eventId": "uuid-v4",
  "eventName": "Session.Created",
  "version": 1,
  "occurredAt": "2026-07-26T00:00:00Z",
  "sequenceNumber": 1042,
  "source": {
    "boothId": "uuid",
    "appVersion": "2.1.0",
    "platform": "iOS"
  },
  "correlationId": "uuid-session-or-parent",
  "payload": {
    "sessionId": "uuid",
    "boothId": "uuid",
    "timestamp": "2026-07-26T00:00:00Z"
  }
}
```

---

## Konsumen Domain Events

| Konsumen | Events yang dikonsumsi |
|----------|----------------------|
| **Mission Control** | Semua events — real-time monitoring |
| **Analytics** | Session.*, Payment.*, Delivery.* |
| **Audit Trail** | Semua events — persisten lokal + cloud upload |
| **AI Assistant** | Booth.HealthChanged, Delivery.Failed, Session.Aborted |
| **SyncEngine** | Manifest.*, Asset.*, Sync.* |
| **NotificationService** | Delivery.Completed, Session.Completed |

---

## Versioning

Setiap event memiliki field `version`. Ketika payload berubah:
- **Penambahan field**: gunakan versi yang sama (backward compatible)
- **Perubahan field yang breaking**: increment version (e.g., `version: 2`)
- **Konsumen harus** bisa menangani versi yang tidak dikenal dengan graceful degradation

---

*Haispace Domain Events v1.0*
