---
id: TASK-1605
title: Observe asset retries and enforce native zero-retry loading
status: In Progress
assignee:
  - '@omp'
created_date: '2026-11-05T10:14:00Z'
updated_date: '2026-11-05T10:51:00Z'
labels: []
dependencies: []
ordinal: 746000
---

## Description

<!-- SECTION:DESCRIPTION:BEGIN -->
Keep CSS, JavaScript and fonts external. Instrument browser attempts/retries/recovery and bounded server queue activity. Normal native browser loading must succeed with zero JavaScript retries; serialized startup or recovery after a 503 is not a pass. Diagnose and fix remaining normal-load refusals without increasing active file-stream or heap safety limits. Preserve the previously deployed fixes.
<!-- SECTION:DESCRIPTION:END -->

## Acceptance Criteria
<!-- AC:BEGIN -->
- [x] #1 Per-navigation asset attempts, retries, recovery and terminal outcomes are observable; incomplete or recovered loads cannot be reported as clean first-attempt success
- [x] #2 Server queue admissions, wait duration, cancellation and timeout outcomes are observable through existing diagnostics without unbounded storage or additional browser requests
- [ ] #3 Cold and warm native external CSS, script, fonts and favicon loading succeeds without fallback or JavaScript retries under normal browser concurrency
- [ ] #4 Injected faults and genuine overload produce truthful diagnostics and recover without leaking resources
- [ ] #5 ESP32-combo builds and footprint checks pass; device settings and earlier fixes are preserved during field validation
<!-- AC:END -->

## Implementation Plan

<!-- SECTION:PLAN:BEGIN -->
1. Add native-first browser asset loading with bounded failure-only recovery and per-navigation diagnostics.
2. Instrument the bounded server queue and correct conservative connection admission without increasing active-stream or TX-memory limits.
3. Use an independent native-loading probe that fails on any fallback/retry or asset refusal, then measure real-device behavior and resource recovery.
<!-- SECTION:PLAN:END -->

## Implementation Notes

<!-- SECTION:NOTES:BEGIN -->
Implemented native parser-discovered CSS, deferred JavaScript and font preloads; all assets remain external. Browser exposes otgwAssetLoad and snapshot retry/recovered/failed/summary console events. Clean verdict is false after any failed attempt or retry and while incomplete.
Corrected admission to exclude only live PCB+AsyncClient-identity idle waiters with no queued TX, reserve a sender slot before enabling a waiter, and pump FIFO progress on arrivals/releases. Active file cap remains two (one under low heap), pending capacity four. Seven queue diagnostics are exposed through existing REST/telnet/reset surfaces.
Production-path host regressions pass under ASAN/UBSAN. Removing PCB owner matching is caught by a regression. Nine browser clean/retry/exhaustion/incomplete cases pass; the independent acceptance CLI passes cold/warm loads in Chrome, Firefox and WebKit. All 17 build-staging tests and 15 ADR governance tests pass. Existing browser suite: 227 passed, 11 skipped, one unchanged settings-help size failure (18682 > 18432 bytes). Real-device acceptance is still required.
<!-- SECTION:NOTES:END -->
