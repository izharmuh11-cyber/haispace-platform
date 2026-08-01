# Product DNA Inventory — Haispace Platform

**Status:** ACTIVE
**Phase:** M-001 — Platform Independence
**Maintained by:** Lead Software Architect (Antigravity)
**Approved by:** Chief Product Architect (GPT)
**Created:** 2026-08-01

> This document is the authoritative backlog of all Product Knowledge extracted from historical products (SnapBooth, hsp-internal). Every DNA entry represents a business decision — not a code artifact. When all entries reach `Migrated` or `Rejected`, Product Archaeology is complete and `PRODUCT_ARCHAEOLOGY_CLOSURE.md` can be signed.

---

## How to Read This Document

| Field | Description |
|-------|-------------|
| `DNA-ID` | Unique identifier |
| `Origin` | Where this knowledge was discovered |
| `Category` | Domain area |
| `Business Value` | Why this matters to operators and guests |
| `Platform Status` | `Pending` / `Migrated` / `Deprecated` / `Rejected` |
| `Decision` | `Accepted` / `Rejected` / `Deferred` |
| `Reason` | Why this decision was made |
| `Target ADR` | Which ADR will formalize this into Platform Contract |
| `Target Capability` | Which Capability will implement this |

---

## Editing Domain

### DNA-001 — Multi-Slot Frame Composition

| Field | Value |
|-------|-------|
| **Origin** | SnapBooth `server/template-engine.js` |
| **Category** | Editing / Composition |
| **Description** | A single frame can contain multiple photo slots. Each slot has its own position, size, and photo source. The composition engine places each photo into its designated slot and overlays the frame. |
| **Business Value** | Enables strip layouts (2x2, 4x1), collage frames, multi-person booths. Directly increases product quality and pricing potential. |
| **Platform Status** | `Pending` |
| **Decision** | `Accepted` |
| **Reason** | Core differentiator for premium photobooth. Required for all future booth types (Photo, Video, AI, 360). |
| **Target ADR** | ADR-014 — Composition Capability Contract |
| **Target Capability** | `CompositionCapability` |
| **Notes** | SnapBooth had 8+ template configs. Each slot defined by `x, y, width, height`. |

---

### DNA-002 — Gravity-Based Crop with Zoom

| Field | Value |
|-------|-------|
| **Origin** | SnapBooth `server/template-engine.js` — `cropGravityX`, `cropGravityY`, `cropZoom` |
| **Category** | Editing / Composition |
| **Description** | Users adjust crop position using gravity point (0.0–1.0 on both axes) and zoom multiplier. Engine computes crop region based on gravity relative to available room after zoom. |
| **Business Value** | Eliminates bad automatic cropping. Guests ensure face is centered. Operators fine-tune before printing. |
| **Platform Status** | `Pending` |
| **Decision** | `Accepted` |
| **Reason** | Battle-tested in SnapBooth field deployments. Gravity model is more intuitive than pixel-offset for non-technical operators. |
| **Target ADR** | ADR-014 — Composition Capability Contract |
| **Target Capability** | `CompositionCapability` via `SlotTransform` |
| **Notes** | GPT expanded to `SlotTransform { gravity, zoom, rotation, mirror, offset }` for future extensibility. Today: implement gravity + zoom only. |

---

### DNA-003 — Frame Asset Cache (Bounded In-Memory)

| Field | Value |
|-------|-------|
| **Origin** | SnapBooth `server/template-engine.js` — `pngCache`, `svgCache` |
| **Category** | Editing / Performance |
| **Description** | Template PNG frames cached in-memory (bounded, max 50) after first load. SVG-generated frames cached by content key. Prevents repeated disk I/O on every capture. |
| **Business Value** | Frame overlay processing becomes near-instant after first use. Critical for multi-guest events using the same frame hundreds of times. |
| **Platform Status** | `Pending` |
| **Decision** | `Accepted` |
| **Reason** | On-device CoreImage/Metal on iOS handles this natively. Asset caching policy needs formalization in Manifest. |
| **Target ADR** | ADR-014 — Composition Capability Contract |
| **Target Capability** | `CompositionCapability` + Asset caching layer |
| **Notes** | Maps to `CIImage` caching and `MTLTexture` reuse in Metal pipeline on iOS. |

---

### DNA-004 — Graceful Placeholder on Missing Photo

| Field | Value |
|-------|-------|
| **Origin** | SnapBooth `server/template-engine.js` — grey fallback slot |
| **Category** | Editing / Resilience |
| **Description** | If a photo for a slot cannot be found, engine renders a solid placeholder rectangle instead of crashing. Composition continues and produces a valid output file. |
| **Business Value** | A partially-failed composition is infinitely better than a crash. Operator can reprocess. Guest does not experience hard failure. |
| **Platform Status** | `Pending` |
| **Decision** | `Accepted` |
| **Reason** | Consistent with Haispace Principle: capability failure must not kill the Workflow. Maps to RG-004. |
| **Target ADR** | ADR-014 — Composition Capability Contract |
| **Target Capability** | `CompositionCapability` error handling policy |
| **Notes** | Placeholder color should be configurable in FrameManifest. |

---

## Runtime Domain

### DNA-005 — Device Readiness State per Physical Device

| Field | Value |
|-------|-------|
| **Origin** | SnapBooth `server/sockets/signaling.js` — `operatorState` Map |
| **Category** | Runtime / Device Management |
| **Description** | Each operator session tracks readiness of every connected device separately: `cameraReady`, `photoReady`, `selectReady`. Workflow queries device readiness through a single abstraction. |
| **Business Value** | Workflow stays clean and decoupled from hardware topology. Adding a new device type does not change Workflow logic. |
| **Platform Status** | `Pending` |
| **Decision** | `Accepted` |
| **Reason** | GPT elevated to `DeviceRegistry` as a Domain Service sitting between CapabilityManager and hardware layer. |
| **Target ADR** | ADR-013 — DeviceRegistry Domain Service |
| **Target Capability** | `CapabilityManager` (extended) |
| **Notes** | `canCapture()` replaces all `cameraConnected && previewConnected && ...` checks. |

---

### DNA-006 — Disconnect Grace Period with Reconnect Cancellation

| Field | Value |
|-------|-------|
| **Origin** | SnapBooth `server/sockets/signaling.js` — `cleanupTimers` Map |
| **Category** | Runtime / Resilience |
| **Description** | When a device disconnects, a grace-period timer starts. If device reconnects before timer fires, timer is cancelled and session continues uninterrupted. Only if timer fires does session treat device as truly gone. |
| **Business Value** | Brief WiFi drops (common at event venues) do not destroy in-progress sessions. Operator does not need to restart. |
| **Platform Status** | `Pending` |
| **Decision** | `Accepted` |
| **Reason** | MultipeerConnectivity on iOS handles transport layer. But session-level grace period policy needs formalization. |
| **Target ADR** | ADR-013 — DeviceRegistry Domain Service |
| **Target Capability** | `P2PCapability` + `DeviceRegistry` |
| **Notes** | Grace period duration should be configurable in Manifest. Suggested default: 15 seconds. |

---

## Operator Domain

### DNA-007 — Kiosk Mode Edge Cases (Field-Validated)

| Field | Value |
|-------|-------|
| **Origin** | SnapBooth `src/app/KioskGuard.js` (577 lines) |
| **Category** | Operator / Kiosk |
| **Description** | 577 lines of battle-tested kiosk lockdown covering: iOS rubber-banding, status bar pull-down blocking, fullscreen with fallbacks, operator PIN exit, gesture detection for accidental swipes, download route exclusion. |
| **Business Value** | A kiosk guests can accidentally exit destroys an operator's event. These scenarios were discovered through real field deployments. |
| **Platform Status** | `Pending` |
| **Decision** | `Accepted` — Knowledge only, not code |
| **Reason** | Native iOS handles most via Guided Access and UIApplication APIs. Specific scenarios must become QA checklist. |
| **Target ADR** | N/A — maps to `KIOSK_RUNTIME_CHECKLIST.md` |
| **Target Capability** | `RootView` + `OperatorDashboardView` |
| **Notes** | See `KIOSK_RUNTIME_CHECKLIST.md` for full scenario extraction. |

---

## Observability Domain

### DNA-008 — Circular In-Memory Buffer for Live Log Query

| Field | Value |
|-------|-------|
| **Origin** | SnapBooth `server/activity-logger.js` — 5000-entry circular buffer |
| **Category** | Observability |
| **Description** | Bounded in-memory circular buffer stores all runtime events. Mission Control queries this buffer for fast, real-time log display without disk reads. Log file rotation handles persistence separately. |
| **Business Value** | Mission Control displays live logs with zero I/O latency. Operators see what is happening right now. |
| **Platform Status** | `Pending` |
| **Decision** | `Accepted` |
| **Reason** | `SessionAuditTrail` uses JSONL for persistence. Circular buffer is the missing fast-query layer for Mission Control. |
| **Target ADR** | Extends ADR-008 (DomainEventPublisher) |
| **Target Capability** | `ObservabilityModule` — new `LiveEventBuffer` component |
| **Notes** | Buffer size for iOS: 1000–2000 entries (memory constraints). Auto-flush oldest when full. |

---

### DNA-009 — Payload Sanitization in Logging

| Field | Value |
|-------|-------|
| **Origin** | SnapBooth `server/activity-logger.js` — `summarizePayload()` |
| **Category** | Observability / Security |
| **Description** | Before logging any payload, sensitive fields (passwords, base64 images) are redacted. Large strings are summarized with their length. Arrays logged as item count only. |
| **Business Value** | Logs remain useful for debugging without becoming a security liability. Guest photo data never appears in log files. |
| **Platform Status** | `Pending` |
| **Decision** | `Accepted` |
| **Reason** | `HaispaceLogger` exists in hsp-internal but payload sanitization policy is not formalized. |
| **Target ADR** | N/A — implementation guideline for `HaispaceLogger` |
| **Target Capability** | `ObservabilityModule` |
| **Notes** | Always redact: passwords, API keys, base64 image data, guest PII (name, Instagram, phone). |

---

## Session Domain

### DNA-010 — Stale Session Cleanup on Reconnect

| Field | Value |
|-------|-------|
| **Origin** | SnapBooth `server/sockets/signaling.js` — stale session detection |
| **Category** | Session / Recovery |
| **Description** | When a device reconnects and requests its active session, system checks if session is still valid. If session was completed or deleted, stale reference is cleared and device is informed gracefully. |
| **Business Value** | Prevents ghost sessions from blocking new guests. Operator does not need to manually clear state. |
| **Platform Status** | `Migrated` ✅ |
| **Decision** | `Accepted` |
| **Reason** | Already implemented in hsp-internal as `OrphanedSessionDetector`. First confirmed migration. |
| **Target ADR** | ADR-002 (Operational Resilience) — already accepted |
| **Target Capability** | `OrphanedSessionDetector` (existing) |
| **Notes** | First `Migrated` entry — confirms Product Archaeology has produced concrete results. |

---

## Summary

| Status | Count |
|--------|-------|
| `Pending` | 9 |
| `Migrated` | 1 |
| `Deprecated` | 0 |
| `Rejected` | 0 |
| **Total** | **10** |

> **Archaeology Complete Condition:** All entries must reach `Migrated` or `Rejected` before `PRODUCT_ARCHAEOLOGY_CLOSURE.md` can be signed.

---

*Product DNA Inventory v1.0*
*M-001 — Platform Independence Phase*
*Last Updated: 2026-08-01*
*Approved by: GPT (Chief Product Architect) + Antigravity (Lead Software Architect)*
