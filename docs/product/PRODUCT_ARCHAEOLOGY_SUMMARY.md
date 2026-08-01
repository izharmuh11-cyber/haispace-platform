# Product Archaeology Summary — Haispace Platform

**Status:** ACTIVE
**Phase:** M-001 — Platform Independence
**Maintained by:** Lead Software Architect (Antigravity)
**Approved by:** Chief Product Architect (GPT)
**Created:** 2026-08-01

> This document tells the story of how Haispace came to be. It is written for engineers who join the project in the future and need to understand not just *what* Haispace is, but *why* it was built the way it was. Read this before reading any ADR.

---

## The Beginning — SnapBooth

Haispace did not start from a blank slate.

Before Haispace existed, there was **SnapBooth** — a self-service photobooth system built with Next.js, Express, Socket.io, and WebRTC. SnapBooth ran at real events: outdoor beaches, city streets, university graduation ceremonies.

SnapBooth worked. Guests registered their names and Instagram handles, chose from photo packages, took photos via WebRTC camera streaming between an iPhone and two iPads, selected a decorative frame, paid via QRIS, and downloaded their photos via a personal QR code.

But SnapBooth had limits that became clear through field experience:

- **Camera latency:** WebRTC introduced 150–400ms preview lag. AVFoundation on native iOS is ~16ms.
- **Offline vulnerability:** If the server went down, the entire system stopped. No session could be created, no photo taken.
- **Kiosk fragility:** Guests could exit the browser, trigger Safari navigation bars, and break the kiosk experience in dozens of ways.
- **Performance ceiling:** Image processing happened server-side via Sharp, taking 2–5 seconds per photo. On-device GPU processing in iOS takes under 500ms.
- **No recovery:** A crash meant calling the operator. No automatic session restoration existed.

These were not bugs. They were the structural limits of a web-based kiosk running on hardware designed for a native experience.

---

## The Decision — Build Native

The decision to build a native iOS application was made after a clear-eyed comparison of web vs. native capabilities. The score was approximately **40% vs. 100%** across camera control, offline capability, device integration, performance, and reliability.

The most important gap was not a feature. It was **reliability in the field**.

A photobooth at a graduation ceremony runs for 6–8 hours without an operator watching every interaction. It must recover from WiFi drops, battery warnings, thermal stress, and guest confusion — silently, automatically, without requiring a restart.

That reliability is only achievable with a native platform.

---

## The Archaeology Process

Rather than discarding SnapBooth's codebase, the team treated it as a **Product Fossil** — something that had survived field deployment and therefore contained proven knowledge.

A formal **Product Archaeology** process was defined:

1. **Forensic Investigation** — Read the code not for syntax, but for decisions. Why was gravity-based cropping implemented? Why was the operator state registry built per-device rather than per-session? These questions led to Product DNA.

2. **DNA Extraction** — Each meaningful decision was extracted and documented as a DNA entry in `PRODUCT_DNA_INVENTORY.md`. Not the JavaScript code. The *intent* behind the code.

3. **Legacy Evaluation** — Each DNA entry was evaluated: Does it still apply to a native iOS context? Should it become a Platform Contract? Or should it be deprecated in favor of a better native approach?

4. **Migration to Platform** — Accepted DNA entries become ADRs, Capability Contracts, or QA checklists. They enter the Platform Constitution. The JavaScript that originally expressed them becomes irrelevant.

---

## What Was Discovered

The Archaeology process revealed that SnapBooth's most valuable knowledge was **not in its feature set**, but in its **failure modes**:

- The 577-line `KioskGuard.js` was not a clever feature — it was a record of every way an iPad kiosk could fail in the field. Rubber-banding. Status bar pull-down. Phantom sessions. Each line represented a real guest experiencing a real problem.

- The `operatorState` registry in `signaling.js` was not an optimization — it was the discovery that device readiness must be tracked per physical device, not per session, because devices reconnect independently.

- The `activity-logger.js` circular buffer was not clever engineering — it was the realization that Mission Control cannot read a log file every second without killing performance.

Each of these discoveries informed a Platform decision.

---

## What Was Built on the Foundation

Between SnapBooth and `haispace-runtime-ios`, a significant intermediate step occurred: `hsp-internal`.

`hsp-internal` served as the **architectural laboratory** where Platform Contracts were tested against real Swift/SwiftUI implementation. It was never intended to be a shipping product under that name. Its purpose was to prove that the architecture decisions captured in `haispace-platform` could be implemented on real iOS hardware.

The key foundations established in `hsp-internal` that carry forward:

| Foundation | Status | Will Live In |
|-----------|--------|-------------|
| `WorkflowOrchestrator` — single state machine | ✅ Frozen | `haispace-runtime-ios` |
| `HaispaceSession` Aggregate Root | ✅ Frozen | `haispace-runtime-ios` |
| `RuntimeContainer` — Module composition root | ✅ Frozen | `haispace-runtime-ios` |
| `SessionAuditTrail` — append-only JSONL | ✅ Frozen | `haispace-runtime-ios` |
| `OrphanedSessionDetector` — pure recovery | ✅ Frozen | `haispace-runtime-ios` |
| `DomainEventPublisher` — priority ordering | ✅ Frozen | `haispace-runtime-ios` |
| `CapabilityManager` — Registry + Resolver + Policy | 🔄 In Progress | `haispace-runtime-ios` |
| View Migration (7/8 screens) | 🔄 In Progress | Completes in `haispace-runtime-ios` |

---

## The Transition

The transition from `hsp-internal` to `haispace-runtime-ios` is not a rewrite. It is a **transplantation**.

The Core layer — everything in `Core/Runtime/`, `Core/Domain/`, `Core/Workflow/`, `Core/Audit/`, `Core/Observability/` — moves intact. The App layer that has already been cleaned follows. The project shell (Xcode project configuration, bundle identifier, signing) is created fresh.

What stays behind in `hsp-internal`:
- `docs/legacy/` — the Product Archaeology records, which served their purpose
- The Xcode project history tied to the old bundle identifier
- Any prototype experiments that were not promoted to Platform Contract

`hsp-internal` does not get deleted. It becomes an **Historical Development Repository** — readable, referenced, but no longer the site of active development.

---

## Architecture Discoveries

Three unexpected discoveries emerged from the Archaeology process that shaped the Platform in ways not originally anticipated:

### Discovery 1: Composition is a Business Capability, not a Feature

Initially, "add a frame to a photo" seemed like a simple feature. The Archaeology revealed it was a **Business Capability** with a formal contract:

```
Input Media → Layout → Placement → Transform → Overlay → Output
```

This contract applies equally to Photo Booth, Video Booth, AI Booth, and 360 Booth. The frame changes. The contract does not. This led to the definition of the Composition Capability as a Platform-level concern, documented in ADR-014.

### Discovery 2: Device Readiness is a Runtime Concern, not a P2P Concern

The SnapBooth `operatorState` registry seemed like a WebSocket implementation detail. It was actually revealing a deeper truth: **the Workflow must never query hardware directly**. The Workflow asks `canCapture()`. Something else answers. That something is `DeviceRegistry` — a Domain Service that sits between hardware and Workflow, absorbing all the complexity of reconnection, grace periods, and readiness states.

### Discovery 3: Archaeology has a Natural Completion Point

Product Archaeology is not an infinite process. It ends when every DNA entry has been either Migrated to a Platform Contract or explicitly Rejected with a reason. That completion is marked by `PRODUCT_ARCHAEOLOGY_CLOSURE.md` — a frozen document that declares the platform independent of its historical origins.

---

## The Moment of Independence

When `PRODUCT_ARCHAEOLOGY_CLOSURE.md` is signed, the following becomes true:

> **Haispace is no longer an evolution of SnapBooth. It is an independent platform whose future is governed by its own Constitution, Product Knowledge, and Platform Contracts.**

From that moment, no engineer needs to read SnapBooth's code to understand why a decision was made. The Platform Constitution, the ADRs, the Product DNA Inventory, and this document contain everything.

That is the purpose of Product Archaeology.

---

*Product Archaeology Summary v1.0*
*M-001 — Platform Independence Phase*
*Written: 2026-08-01*
*Approved by: GPT (Chief Product Architect) + Antigravity (Lead Software Architect)*
