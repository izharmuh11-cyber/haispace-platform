# Runtime API Contract

This document acts as the definitive contract for any hardware runtime (e.g. Swift iPad app) communicating with the Haispace Cloud API.
**Version:** v1.0.0-rc1

## Global Constraints

- **Base URL:** `https://api.haispaceproject.my.id`
- **Authentication:** Most runtime endpoints require the `x-booth-api-key` header.
- **Content-Type:** `application/json`
- **Idempotency:** Creation endpoints (POST) are generally idempotent or use upsert mechanics to handle duplicate payloads from offline queue retries.
- **Retry Policy:** The Runtime MUST implement Exponential Backoff (e.g., 2s, 4s, 8s, 16s, max 60s) for network timeouts or 5xx errors. DO NOT retry on 4xx errors (except 401 if a key refresh is implemented, though Booth API Keys are static).

---

## 1. Capabilities & Synchronization

### 1.1 Runtime Capabilities
Retrieves versioning and compatibility matrices for the runtime before it attempts to register or sync.

- **Endpoint:** `GET /v1/runtime/capabilities`
- **Auth:** None
- **Response (200 OK):**
  ```json
  {
    "minimumRuntime": "1.0.0",
    "supportedApiVersion": "v1",
    "manifestSchema": 3,
    "maintenance": false
  }
  ```
- **Error Codes:** 
  - `503 Service Unavailable` (If maintenance mode is active at the load balancer level)

### 1.2 Clock Synchronization
Retrieves the authoritative server time to ensure local device clocks do not drift.

- **Endpoint:** `GET /v1/runtime/time`
- **Auth:** None
- **Response (200 OK):**
  ```json
  {
    "serverTime": "2026-07-26T15:22:38.000Z",
    "timestamp": 1785054158000
  }
  ```

---

## 2. Hardware Lifecycle

### 2.1 Device Registration
Registers a physical hardware device to a specific Booth ID. Returns the Booth API Key for future authentication.
*Idempotency:* Calling this with an existing `deviceId` + `boothId` will return the existing record and key, rather than duplicating.

- **Endpoint:** `POST /v1/devices`
- **Auth:** None (currently).
- **Request Body:**
  ```json
  {
    "deviceId": "UUID-FROM-APPLE",
    "boothId": "bth_123",
    "platform": "ios",
    "deviceClass": "ipad-pro-12.9"
  }
  ```
- **Response (201 Created):**
  ```json
  {
    "deviceId": "UUID-FROM-APPLE",
    "boothId": "bth_123",
    "apiKey": "bth_key_abc123"
  }
  ```
- **Error Codes:**
  - `404 Not Found` (If Booth ID does not exist)

### 2.2 Heartbeat (Descriptor Upload)
Uploads the current health and state of the hardware. The Cloud uses this to calculate `lastSeenAt` and track physical status.

- **Endpoint:** `PATCH /v1/devices/{deviceId}/descriptor`
- **Auth:** `x-booth-api-key`
- **Request Body:**
  ```json
  {
    "runtimeVersion": "1.0.0-rc1",
    "manifestVersion": "man_456",
    "batteryLevel": 85,
    "storageFree": 45000000000,
    "networkType": "wifi",
    "pendingUploads": 0,
    "state": "idle"
  }
  ```
- **Response (200 OK):**
  ```json
  {
    "deviceId": "UUID-FROM-APPLE",
    "lastSeenAt": "2026-07-26T15:22:38.000Z"
  }
  ```

---

## 3. Mission Data

### 3.1 Download Manifest
Retrieves the current Manifest that this booth must execute (determined by the Event the Booth is assigned to).

- **Endpoint:** `GET /v1/runtime/manifest`
- **Auth:** `x-booth-api-key`
- **Response (200 OK):**
  ```json
  {
    "manifestId": "man_456",
    "schemaVersion": 3,
    "architecture": "arm64",
    "packages": [...],
    "configuration": {...}
  }
  ```
- **Offline Behaviour:** If unreachable, Runtime MUST boot using the locally cached manifest.

---

## 4. Session & Telemetry Uploads

### 4.1 Sync Session
Uploads a completed (or archived) session metadata payload.

- **Endpoint:** `POST /v1/sessions`
- **Auth:** `x-booth-api-key`
- **Request Body:**
  ```json
  {
    "sessionId": "ses_789",
    "eventId": "evt_123",
    "boothId": "bth_123",
    "manifestId": "man_456",
    "state": "completed",
    "startedAt": "2026-07-26T14:00:00.000Z",
    "completedAt": "2026-07-26T14:05:00.000Z"
  }
  ```
- **Response (201 Created):**
  ```json
  { "success": true, "sessionId": "ses_789" }
  ```

### 4.2 Sync Session Events (Domain Events)
Uploads an array of chronological events that occurred during a session.

- **Endpoint:** `POST /v1/session-events`
- **Auth:** `x-booth-api-key`
- **Request Body:**
  ```json
  {
    "events": [
      {
        "eventId": "sev_1",
        "sessionId": "ses_789",
        "eventType": "capture_start",
        "payload": {},
        "timestamp": "2026-07-26T14:01:00.000Z"
      }
    ]
  }
  ```
- **Response (207 Multi-Status):**
  ```json
  {
    "accepted": ["sev_1"],
    "rejected": []
  }
  ```

### 4.3 Sync Audit Events (Hardware Events)
Uploads system/hardware level events (e.g. Printer offline, Camera disconnected).

- **Endpoint:** `POST /v1/audit-events`
- **Auth:** `x-booth-api-key`
- **Request Body:**
  ```json
  {
    "events": [
      {
        "auditId": "aud_1",
        "boothId": "bth_123",
        "category": "hardware",
        "action": "camera_connect",
        "outcome": "success",
        "timestamp": "2026-07-26T13:50:00.000Z"
      }
    ]
  }
  ```
- **Response (207 Multi-Status):**
  ```json
  {
    "accepted": ["aud_1"],
    "rejected": []
  }
  ```
