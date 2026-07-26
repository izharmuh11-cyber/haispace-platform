# Runtime Failure Matrix

This document defines the strict engineering behaviors the iPad Runtime must implement during failure modes. **The goal is absolute resilience:** a booth must never crash, and must continue making money even if the cloud is completely offline.

| Scenario | Detection | Immediate Action | Recovery Action |
| :--- | :--- | :--- | :--- |
| **API Unavailable (5xx or Timeout)** | Any HTTP request fails after max timeout (15s). | Enter `OfflineQueue` state. Cache payloads locally. Serve user from local manifest. | Exponential backoff retry (2s, 4s, 8s, 16s, max 60s). Drain queue when 200 OK is received. |
| **Invalid API Key (401)** | Cloud returns HTTP 401. | Enter `Error` state. Stop accepting new sessions. | Operator must re-authenticate the device via Mission Control or physically input a new OTP. |
| **Manifest Download Fails** | `GET /v1/runtime/manifest` fails during Boot. | Fallback to last cached manifest. If no cache exists, enter `Error` state. | Retry manifest download every 60 seconds. |
| **Printer Disconnected** | AirPrint or USB printer stops responding. | Show "Printing Unavailable" warning. Log `Audit Event`. Continue capturing but skip print state. | Poll printer status every 10s. Resume print options when restored. |
| **Camera Unavailable** | iOS AVFoundation throws hardware error. | Enter `Error` state immediately. Log `Audit Event`. | Requires Operator physical intervention / app restart. |
| **Storage Full (Critical)** | Local device storage < 500MB. | Enter `Error` state. Prevent new sessions to avoid data corruption. | Force flush `OfflineQueue`. Operator must manually clear storage. |
| **Storage Warning** | Local device storage < 2GB. | Log `Audit Event` (Storage Warning). Continue operations. | Cloud Dashboard will show Alert based on Heartbeat telemetry. |
| **Battery Critical** | Device battery < 10%. | Log `Audit Event` (Battery Critical). Continue operations. | Operator should plug in device based on Cloud Dashboard alert. |
| **Clock Drift** | `GET /v1/runtime/time` differs from local clock by > 5 seconds. | Calculate offset delta. Apply offset to all outgoing `timestamp` fields. | Do not attempt to change iOS system clock. |
| **Version Mismatch** | `capabilities` API returns `minimumRuntime` > current app version. | Enter `Error` state. Display "Update Required". | App update via MDM or App Store required. |
