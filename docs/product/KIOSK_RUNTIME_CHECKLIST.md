# Kiosk Runtime Checklist — Haispace Platform

**Status:** ACTIVE
**Phase:** M-001 — Platform Independence
**Origin:** Extracted from SnapBooth `KioskGuard.js` (577 lines of field-validated edge cases)
**Maintained by:** Lead Software Architect (Antigravity)
**Approved by:** Chief Product Architect (GPT)
**Created:** 2026-08-01

> This document is not a static specification. It is a living record of kiosk runtime scenarios, discovered through real field deployments of SnapBooth and refined for native iOS implementation in `haispace-runtime-ios`. QA uses this document directly for device testing.

---

## How to Read This Document

| Column | Description |
|--------|-------------|
| `Scenario` | The specific edge case or user action |
| `Risk` | What goes wrong if this is not handled |
| `Expected Behaviour` | What the system must do |
| `Native Strategy` | The iOS-native mechanism that handles this |
| `Verification Method` | How to manually test this |
| `Automation Possible?` | Whether a UI test can cover this |
| `Status` | `Pending` / `Verified` / `Not Applicable` |

---

## Field Validation Log

Use this section to record real-world test results.

| Date | Venue | Device | iOS Version | Operator | Result | Notes |
|------|-------|--------|-------------|----------|--------|-------|
| — | — | — | — | — | — | First entry when field test occurs |

---

## 1. Accidental System Gesture — Pull Down from Top

| Field | Value |
|-------|-------|
| **Scenario** | User or guest swipes down from top edge of screen |
| **Risk** | iOS Notification Center or Control Center opens, disrupting active session |
| **Expected Behaviour** | Swipe is blocked entirely. Session continues uninterrupted. |
| **Native Strategy** | Apple Guided Access locks edge swipes. Additionally: `UIApplication.shared.isIdleTimerDisabled = true`. If Guided Access not active: block via `preferredScreenEdgesDeferringSystemGestures` in UIViewController. |
| **Verification Method** | With session active, swipe down from top 20 times rapidly. Confirm notification center never appears. |
| **Automation Possible?** | No — system gesture cannot be triggered by XCUITest |
| **Status** | `Pending` |

---

## 2. Accidental System Gesture — Swipe from Bottom

| Field | Value |
|-------|-------|
| **Scenario** | User swipes up from bottom edge to access Home Bar or App Switcher |
| **Risk** | App moves to background, session is paused or lost |
| **Expected Behaviour** | Swipe blocked. Single swipe shows Home indicator but does not dismiss. Double swipe does not open App Switcher. |
| **Native Strategy** | Guided Access. Additionally: `prefersHomeIndicatorAutoHidden = true` when in active session. |
| **Verification Method** | Swipe up from bottom once and twice during active session. App must not go to background. |
| **Automation Possible?** | No |
| **Status** | `Pending` |

---

## 3. Screen Sleep / Auto-Lock

| Field | Value |
|-------|-------|
| **Scenario** | iPad auto-lock timer fires during a session |
| **Risk** | Screen locks mid-session. Guest cannot continue. Payment may be lost. |
| **Expected Behaviour** | Screen never sleeps while Kiosk mode is active. Auto-lock disabled for the entire event. |
| **Native Strategy** | `UIApplication.shared.isIdleTimerDisabled = true` — set in `AppDelegate.setupAppearance()`. Must be re-verified after app returns from background. |
| **Verification Method** | Set iPad auto-lock to 30 seconds. Start a session. Wait 35 seconds without touching screen. Screen must not lock. |
| **Automation Possible?** | Partially — can verify `isIdleTimerDisabled` is `true` via unit test |
| **Status** | `Pending` |

---

## 4. Operator Secret Exit from Kiosk

| Field | Value |
|-------|-------|
| **Scenario** | Legitimate operator needs to exit kiosk mode (e.g., configure settings, end event) |
| **Risk** | No exit path = operator locked out. Easy exit path = guest can accidentally exit. |
| **Expected Behaviour** | Triple-tap in top-right corner opens PIN entry overlay. Correct PIN unlocks operator dashboard. Wrong PIN logs the attempt and shows error. |
| **Native Strategy** | `.onTapGesture(count: 3)` on a clear 80×80 overlay in `KioskRouterView` top-right corner. PIN entry via `PINEntryView`. |
| **Verification Method** | Triple-tap top-right corner. Confirm PIN modal appears. Enter wrong PIN 3 times. Confirm error. Enter correct PIN. Confirm operator dashboard opens. |
| **Automation Possible?** | Yes — XCUITest can simulate triple-tap and PIN entry |
| **Status** | `Pending` |

---

## 5. App Moved to Background

| Field | Value |
|-------|-------|
| **Scenario** | iPad receives a phone call, or operator accidentally presses Home button if Guided Access is not enabled |
| **Risk** | Session timer continues, but guest interaction is lost. Camera session may be interrupted by iOS. |
| **Expected Behaviour** | App pauses gracefully. Camera session suspended cleanly. When app returns to foreground, session resumes from the exact same state. |
| **Native Strategy** | `UIApplication.didEnterBackgroundNotification` triggers state preservation. `UIApplication.didBecomeActiveNotification` triggers `handleAppBecomeActive()`. Camera `AVCaptureSession` must be suspended and resumed correctly. |
| **Verification Method** | Start capture session. Press Home (with Guided Access disabled in test). Wait 5 seconds. Return to app. Camera must resume. |
| **Automation Possible?** | Partially |
| **Status** | `Pending` |

---

## 6. Download Page Must Not Enter Kiosk Mode

| Field | Value |
|-------|-------|
| **Scenario** | Guest opens their personal download QR link on a separate device (or the photobooth device shows the download screen) |
| **Risk** | Download page enters kiosk mode — guest cannot scroll, download, or interact normally |
| **Expected Behaviour** | Download/gallery screens are explicitly excluded from kiosk lockdown. Normal browser/app interaction allowed. |
| **Native Strategy** | `RootView` routing logic: kiosk mode is only active when `isKioskModeActive == true`. Download screens are reached via `DeliveryView`, which is within kiosk flow but with scroll enabled. |
| **Verification Method** | Navigate to delivery view. Confirm scrolling works. Confirm fullscreen tap-block is not applied to download link area. |
| **Automation Possible?** | Yes |
| **Status** | `Pending` |

---

## 7. iOS Rubber-Banding (Overscroll)

| Field | Value |
|-------|-------|
| **Scenario** | Guest drags on a non-scrollable screen, causing the entire UI to bounce/rubber-band |
| **Risk** | Breaks kiosk illusion. Guest feels the app is a web app, not a native kiosk. May expose URL bar or navigation chrome. |
| **Expected Behaviour** | Zero rubber-banding on all kiosk screens. Only intentionally scrollable areas (photo selection grid) allow scroll. |
| **Native Strategy** | SwiftUI views do not have overscroll by default. Verify `ScrollView` usage has `.scrollBounceBehavior(.basedOnSize)`. No `UIScrollView` subclasses with `bounces = true` unless intentional. |
| **Verification Method** | On each kiosk screen, drag aggressively in all 4 directions. UI must not bounce. |
| **Automation Possible?** | Partially |
| **Status** | `Pending` |

---

## 8. Battery Critical Warning

| Field | Value |
|-------|-------|
| **Scenario** | iPad battery drops below 10% during event |
| **Risk** | Sudden shutdown mid-session. Payment confirmed but delivery not completed. Operator unaware. |
| **Expected Behaviour** | Mission Control receives battery warning. Operator notified. Session can continue but operator is alerted to plug in device. |
| **Native Strategy** | `UIDevice.current.isBatteryMonitoringEnabled = true` (set in AppDelegate). `UIDevice.batteryLevelDidChangeNotification` triggers `HealthAggregator` update. |
| **Verification Method** | Set `UIDevice.current.batteryLevel` mock to `0.08`. Confirm `HealthAggregator` reports degraded status. Confirm `MissionControlView` shows warning. |
| **Automation Possible?** | Yes — mockable via `HealthAggregator` in testing environment |
| **Status** | `Pending` |

---

## 9. Thermal Throttling

| Field | Value |
|-------|-------|
| **Scenario** | iPad overheats during long outdoor event (direct sunlight, no shade) |
| **Risk** | CPU/GPU throttled. Camera framerate drops. Processing becomes slow. App may be killed by OS. |
| **Expected Behaviour** | `ProcessInfo.thermalState` monitored. At `.serious`: notify operator via Mission Control. At `.critical`: gracefully pause capture, show operator warning, do not crash. |
| **Native Strategy** | Subscribe to `ProcessInfo.thermalStateDidChangeNotification`. Feed state into `HealthAggregator`. |
| **Verification Method** | Cannot easily simulate. During PRR: test in direct sunlight for 2 hours and monitor Mission Control. |
| **Automation Possible?** | No — requires physical hardware in thermal stress |
| **Status** | `Pending` |

---

## 10. Airplane Mode / Network Loss Mid-Session

| Field | Value |
|-------|-------|
| **Scenario** | WiFi disconnects or venue internet goes down during an active session |
| **Risk** | Session sync to cloud fails. If payment is confirmed, delivery data may not reach cloud. |
| **Expected Behaviour** | Session continues fully offline. All events queued locally. On reconnect, offline queue is uploaded automatically. Guest experience is not interrupted. |
| **Native Strategy** | `InfrastructureModule` offline queue. `URLSession` background upload tasks. `NWPathMonitor` for connectivity monitoring. |
| **Verification Method** | Start session. Enable Airplane Mode after registration. Complete full flow through payment. Disable Airplane Mode. Confirm cloud receives session data. |
| **Automation Possible?** | Yes — can mock `NWPathMonitor` in testing |
| **Status** | `Pending` |

---

## 11. Physical Home Button Press (iPad with Home Button)

| Field | Value |
|-------|-------|
| **Scenario** | Older iPad with physical Home button — guest or operator accidentally presses it |
| **Risk** | App exits to home screen. Session lost. |
| **Expected Behaviour** | Guided Access prevents this. If Guided Access is not active (not enrolled), app re-launches and `OrphanedSessionDetector` recovers session. |
| **Native Strategy** | Primary: Apple Guided Access via MDM or Settings. Secondary: `OrphanedSessionDetector` on next launch. |
| **Verification Method** | With Guided Access disabled: press Home during active session. Re-launch app. Confirm `OrphanedSessionDetector` detects orphan and presents recovery options. |
| **Automation Possible?** | Partially — recovery detection testable |
| **Status** | `Pending` |

---

## 12. Split View / Stage Manager (iPad)

| Field | Value |
|-------|-------|
| **Scenario** | User activates Split View or Stage Manager, pulling another app alongside HaiBooth |
| **Risk** | HaiBooth loses full-screen exclusive control. Kiosk illusion broken. |
| **Expected Behaviour** | HaiBooth must always run fullscreen. Split View must be disabled for this app. |
| **Native Strategy** | `Info.plist`: set `UIRequiresFullScreen = YES`. This disables Split View and Slide Over for the app entirely. |
| **Verification Method** | Attempt to drag HaiBooth into Split View. Confirm it refuses and stays fullscreen. |
| **Automation Possible?** | No — system-level behavior |
| **Status** | `Pending` |

---

*Kiosk Runtime Checklist v1.0*
*M-001 — Platform Independence Phase*
*Last Updated: 2026-08-01*
*Approved by: GPT (Chief Product Architect) + Antigravity (Lead Software Architect)*
