# batteryguard

A macOS menu-bar app that maximises MacBook battery lifespan by giving users fine-grained control over charging limits, discharge cycles, heat protection, and battery health monitoring — inspired by the AlDente feature set.

---

## Table of Contents

1. [Scope & Purpose](#scope--purpose)
2. [User Stories](#user-stories)
3. [Implementation Plan](#implementation-plan)

---

## Scope & Purpose

### Problem

MacBook batteries are lithium-ion cells whose capacity degrades over time. Two of the biggest accelerators of that degradation are:

* **Chronic over-charge** — sitting at or near 100 % while plugged in stresses the chemistry continuously.
* **Heat** — charging at elevated temperatures compounds voltage stress and accelerates capacity loss.

macOS's built-in "Optimised Battery Charging" helps somewhat, but it is opaque, offers no user control, and only works on a fixed schedule. Power users, developers, and anyone who works predominantly at a desk need a transparent, configurable solution.

### Purpose of batteryguard

batteryguard puts the user in control of every aspect of the charging lifecycle:

| Goal | How batteryguard helps |
|------|------------------------|
| Limit maximum charge | Set a hard upper bound (e.g. 80 %) so the battery never sits fully charged |
| Drain to a healthy level | Discharge mode lets the Mac run on battery while still plugged in |
| Prevent heat-induced wear | Heat Protection pauses charging above a configurable temperature |
| Reduce micro-cycling at the limit | Sailing Mode cycles gently within a user-defined range |
| Keep capacity readings accurate | Calibration Mode runs a controlled full charge/discharge cycle |
| Surface real-time health data | Stats panel exposes capacity, temperature, cycle count and power draw |
| Integrate with automation | Apple Shortcuts support for event-driven or scheduled rules |

The app lives entirely in the macOS menu bar, requires no persistent background services beyond a privileged helper, and communicates with hardware through the System Management Controller (SMC) and standard macOS battery APIs.

---

## User Stories

Each story follows the format: **"As a [persona], I want [capability] so that [benefit]."**

### Epic 1 — Charge Limiting (Free tier)

**US-01** — As a **desk-bound MacBook user**, I want to set a maximum charging percentage (e.g. 80 %) so that my battery is never kept at a high voltage for extended periods, reducing long-term capacity loss.

**US-02** — As a **user preparing for travel**, I want a one-tap "Top Up" override that charges to 100 % temporarily, so that I can get a full day's range without permanently changing my limit.

**US-03** — As a **new user**, I want the app to show me my current charge limit in the menu-bar icon so that I always have at-a-glance confirmation it is active.

### Epic 2 — Discharge Mode (Free tier)

**US-04** — As a **user who left the Mac plugged in too long and is now above my target**, I want a Discharge Mode that draws power from the battery while the charger is still connected, so that I can bring the battery down to my desired level without unplugging.

**US-05** — As a **user in Discharge Mode**, I want the mode to stop automatically once the charge limit is reached so that I do not need to monitor it manually.

### Epic 3 — Sailing Mode (Pro tier)

**US-06** — As a **power user who is always plugged in**, I want to define a charge window (e.g. 75 %–80 %) so that the battery gently oscillates within that range instead of micro-cycling at a single threshold, reducing wear from repeated small charge events.

**US-07** — As a **Sailing Mode user**, I want to see the lower and upper boundaries displayed in the status panel so that I can confirm the active range at a glance.

### Epic 4 — Heat Protection (Pro tier)

**US-08** — As a **user running intensive workloads**, I want charging to pause automatically when the battery temperature exceeds a configurable threshold (e.g. 35 °C) so that heat-induced degradation is prevented without requiring my attention.

**US-09** — As a **user with Heat Protection enabled**, I want a visual or notification alert when charging is paused due to heat so that I am aware of why my Mac appears to have stopped charging.

**US-10** — As a **user with specific hardware constraints**, I want to customise the temperature threshold so that I can tune the behaviour to my environment and use-case.

### Epic 5 — Calibration Mode (Pro tier)

**US-11** — As a **user whose battery health indicator shows inconsistent readings**, I want a Calibration Mode that automates a full charge/discharge cycle so that the battery management system recalibrates its capacity estimate without requiring manual intervention.

**US-12** — As a **user running Calibration Mode**, I want the app to temporarily suspend all other limits and protections for the duration of the cycle so that calibration completes accurately and predictably.

**US-13** — As a **user who runs calibration infrequently**, I want the app to remind me to calibrate every 4 weeks (configurable from 2–12 weeks) so that I do not forget to maintain accuracy.

### Epic 6 — Battery Health Statistics (Free & Pro tiers)

**US-14** — As a **user who wants to understand their battery's condition**, I want a Stats panel showing current charge %, design capacity vs. actual capacity, cycle count, temperature, and current power draw so that I can make informed decisions about usage and replacement.

**US-15** — As a **technically inclined user**, I want raw SMC data (voltage, amperage, cycle count) available in the Stats panel so that I can diagnose battery behaviour in detail.

**US-16** — As a **user tracking degradation over time**, I want historical capacity charts stored locally so that I can observe trends without relying on third-party cloud services.

### Epic 7 — Automatic Discharge (Pro tier)

**US-17** — As a **user who regularly returns home and plugs in above the charge limit**, I want Automatic Discharge to trigger without me having to open the app so that my battery is always managed to my set level passively.

### Epic 8 — Apple Shortcuts Integration (Pro tier)

**US-18** — As a **macOS automation user**, I want batteryguard actions (set limit, enable/disable sailing, trigger top-up) exposed as Apple Shortcuts actions so that I can build event-driven workflows (e.g. raise the limit when connecting to power at the office, lower it at home).

### Epic 9 — Onboarding & Settings

**US-19** — As a **first-time user**, I want an onboarding flow that explains why charging limits improve battery health and recommends sensible defaults so that I can start with a good configuration immediately.

**US-20** — As a **privacy-conscious user**, I want the app to operate entirely locally with no telemetry or network calls so that no personal or device data leaves my machine.

**US-21** — As a **macOS administrator**, I want batteryguard to install a minimal privileged helper (via SMJobBless) that is the only component requiring elevated privileges so that the attack surface is as small as possible.

---

## Implementation Plan

The plan is broken into four phases, ordered by user value and technical dependency.

### Phase 1 — Foundation & Core Charging Control (MVP)

**Goal:** Ship the minimum viable product: charge limiting and discharge, with a menu-bar UI.

| # | Task | Details |
|---|------|---------|
| 1.1 | Project setup | Swift / SwiftUI macOS app target (macOS 13+). Xcode project with a menu-bar `NSStatusItem`. |
| 1.2 | SMC communication layer | Implement a Swift wrapper around the SMC I/O Kit interface to read/write battery registers (BCLM, AC-W, charging inhibit flags). Extract to a reusable `SMCKit` module. |
| 1.3 | Privileged helper (XPC) | Create a `com.batteryguard.helper` Launch Daemon using SMJobBless. All SMC write operations go through XPC so the main app runs without root. |
| 1.4 | Charge Limiter UI | Slider + text field in the popover to set max charge %. Persist setting in `UserDefaults`. Apply limit via the helper on launch and on change. |
| 1.5 | Discharge Mode | Toggle in popover. When active, instruct the SMC to inhibit AC charging while the adapter is connected. Auto-disable when charge drops to the limit. |
| 1.6 | Top-Up override | Button in popover. Temporarily sets limit to 100 % and re-applies the user's limit when charge reaches 100 % (or user cancels). |
| 1.7 | Menu-bar status icon | Show current charge % and a coloured indicator (green / amber / red) reflecting battery state. |
| 1.8 | Basic Stats panel | Read-only view: charge %, current capacity (mAh), cycle count, plugged-in state. Sourced from `IOPSCopyPowerSourcesInfo`. |
| 1.9 | Unit tests — SMCKit | Test register encode/decode logic with mock I/O responses. |
| 1.10 | CI pipeline | GitHub Actions: build, test, and notarise on every PR targeting `main`. |

### Phase 2 — Heat Protection & Sailing Mode (Pro v1)

**Goal:** Add the two most impactful Pro features; introduce licence/entitlement gating.

| # | Task | Details |
|---|------|---------|
| 2.1 | Licence management | Implement a local licence key validator or StoreKit 2 in-app purchase for the Pro feature set. |
| 2.2 | Battery temperature monitor | Poll `IOPSCopyPowerSourcesInfo` / SMC temperature keys on a 30-second timer. Expose as a `@Published` property in a `BatteryMonitor` observable. |
| 2.3 | Heat Protection logic | When temperature > user threshold, call the helper to inhibit charging. Resume automatically when temperature drops below threshold − hysteresis (e.g. 2 °C). |
| 2.4 | Heat Protection UI | Settings row: enable/disable toggle + threshold stepper. Status indicator in popover when protection is active. Menu-bar badge on heat-pause. |
| 2.5 | Notifications | `UserNotifications` alert when Heat Protection pauses or resumes charging. |
| 2.6 | Sailing Mode logic | User sets `[lowerBound, upperBound]`. The `BatteryMonitor` charges to upper, then discharges to lower using Discharge Mode, cycling automatically. |
| 2.7 | Sailing Mode UI | Range slider in popover (only visible when Pro licence is active). Current phase indicator (charging / sailing / discharging). |
| 2.8 | Pro feature gating | All Pro UI elements visible but disabled with an upgrade prompt when no licence is present. |
| 2.9 | Integration tests | Simulate temperature events and verify charging-inhibit calls to the mock helper. |

### Phase 3 — Calibration, Automatic Discharge & Advanced Stats (Pro v2)

**Goal:** Complete the Pro feature set and enrich the stats experience.

| # | Task | Details |
|---|------|---------|
| 3.1 | Calibration Mode | State machine: (1) discharge to ~15 %, (2) charge to 100 % with Heat Protection suspended, (3) hold at 100 % for 2 hours, (4) restore original settings. Progress indicator in UI. |
| 3.2 | Calibration reminder | Store last calibration timestamp. Show a notification and menu-bar badge after N weeks (user-configurable). |
| 3.3 | Automatic Discharge | On power-source-change event (charger plugged in), if charge > limit, automatically enter Discharge Mode. |
| 3.4 | Advanced Stats panel | Add: design capacity, full-charge capacity, temperature graph (24-hour sparkline), voltage, amperage. Store readings in a local SQLite database via GRDB or CoreData. |
| 3.5 | Capacity history chart | SwiftUI `Charts` view showing capacity trend over time (weeks / months). |
| 3.6 | Export | CSV/JSON export of historical stats for power users. |
| 3.7 | Unit & integration tests | Test calibration state machine transitions and automatic discharge trigger logic. |

### Phase 4 — Shortcuts, Onboarding & Polish

**Goal:** Automate, educate, and delight.

| # | Task | Details |
|---|------|---------|
| 4.1 | Apple Shortcuts intents | Define `AppIntent` actions: `SetChargeLimitIntent`, `EnableSailingModeIntent`, `TopUpIntent`, `GetBatteryStatsIntent`. |
| 4.2 | Onboarding flow | First-launch sheet explaining the science behind charge limits, walking through initial setup, and recommending the 20–80 % rule. |
| 4.3 | Accessibility | Full VoiceOver support for all controls; honour `reduceMotion` and `increaseContrast` settings. |
| 4.4 | Localisation | Extract all user-facing strings into `.xcstrings`; ship English + at least two additional locales. |
| 4.5 | Menu-bar customisation | Let users choose between icon-only, percentage-only, or icon + percentage display. |
| 4.6 | System settings integration | Register as a Login Item via `SMAppService` so the app starts at login without a legacy helper. |
| 4.7 | Performance hardening | Audit timer frequency; switch to `IOPSNotificationCreateRunLoopSource` for event-driven updates where possible to reduce CPU wake-ups. |
| 4.8 | Security audit | Validate XPC entitlements, review SMC write paths, run static analysis (Xcode Analyzer + CodeQL). |
| 4.9 | App Store / direct distribution | Prepare for Mac App Store submission (sandboxing constraints vs. direct distribution decision) or notarised DMG with Sparkle auto-update. |
| 4.10 | End-to-end tests | UI tests covering the onboarding flow, limit changes, and Shortcuts execution. |

### Architecture Overview

```
batteryguard (main app — sandboxed, menu bar)
│
├── UI Layer (SwiftUI)
│   ├── MenuBarView          — status icon + popover
│   ├── ChargeLimiterView    — slider, top-up button
│   ├── SailingModeView      — range picker (Pro)
│   ├── HeatProtectionView   — toggle + threshold (Pro)
│   ├── CalibrationView      — progress + trigger (Pro)
│   ├── StatsView            — live & historical data
│   └── OnboardingView       — first-launch wizard
│
├── Domain Layer
│   ├── BatteryMonitor       — @Observable, polls IOKit / SMC
│   ├── ChargeLimitManager   — applies/removes limits via XPC
│   ├── SailingController    — state machine for range cycling
│   ├── HeatProtectionEngine — temperature-threshold logic
│   └── CalibrationEngine    — full-cycle state machine
│
├── Data Layer
│   ├── SettingsStore        — UserDefaults-backed preferences
│   └── BatteryHistoryStore  — SQLite / CoreData time-series
│
└── batteryguard-helper (XPC Launch Daemon — privileged)
    └── SMCKit               — I/O Kit SMC read/write wrapper
```

### Technology Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Language | Swift 6 | Native performance, concurrency model, type safety |
| UI | SwiftUI + AppKit interop | Declarative menu-bar popover, Charts framework |
| Privilege escalation | SMJobBless XPC helper | Minimal surface area; no persistent root process |
| Battery APIs | IOKit / `IOPSCopyPowerSourcesInfo` | Apple-sanctioned access to power source data |
| SMC access | I/O Kit `IOServiceGetMatchingService` | Required to read/write charge-limit registers |
| Persistence | UserDefaults + SQLite (GRDB) | Lightweight; no server dependency |
| Automation | AppIntents (Apple Shortcuts) | First-class macOS 13+ integration |
| Updates | Sparkle 2 (direct) or Mac App Store | TBD based on sandboxing requirements |
| CI/CD | GitHub Actions + Xcode Cloud | Automated build, test, and notarisation |

### Milestones & Rough Timeline

| Milestone | Deliverable | Target |
|-----------|------------|--------|
| M1 | Phase 1 complete — public beta (free features) | 8 weeks |
| M2 | Phase 2 complete — Pro v1 launch | +6 weeks |
| M3 | Phase 3 complete — Pro v2 launch | +6 weeks |
| M4 | Phase 4 complete — App Store / v1.0 release | +4 weeks |