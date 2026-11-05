---
id: TASK-1604
title: Try render-blocking external CSS with bounded server-side file scheduling
status: In Progress
assignee:
  - '@omp'
created_date: '2026-11-05T01:11:00Z'
updated_date: '2026-11-05T01:31:00Z'
labels: []
dependencies: []
ordinal: 745000
---

## Description

<!-- SECTION:DESCRIPTION:BEGIN -->
Replace the initial JavaScript-inserted stylesheet with a native external link in head. Keep two active file streams and existing heap/connection safety caps, while holding a bounded number of file requests instead of returning ordinary-concurrency 503s. This branch includes the existing hook, MQTT entity-name, and Connection animation fixes. Experimental; no production rollout without memory and recovery evidence.
<!-- SECTION:DESCRIPTION:END -->

## Acceptance Criteria
<!-- AC:BEGIN -->
- [x] #1 The initial successful page paint is styled using an external CSS link in head
- [x] #2 Ordinary competing file requests wait within a bounded server-side queue and never exceed active stream limits
- [x] #3 Cancellation, timeout, queue overflow, and invalid file sources release resources safely
- [ ] #4 Cold loads and overload recovery are measured with required existing fixes preserved
<!-- AC:END -->

## Implementation Plan

<!-- SECTION:PLAN:BEGIN -->
1. Keep the current memory and connection caps; defer bounded static responses on existing AsyncTCP callbacks.
2. Use a native render-blocking external stylesheet link and retain bounded failure recovery.
3. Run delayed-CSS, cancellation, saturation and recovery checks; build with existing fixes preserved.
<!-- SECTION:PLAN:END -->

## Implementation Notes

<!-- SECTION:NOTES:BEGIN -->
Delayed-CSS probe failed before the change (paint before stylesheet). After native head link and real load/error event tracking: delayed, normal one-fetch, initial503 recovery, and terminal failure scenarios all pass in installed Chrome. No CSS embedded.

Server host regression now passes, including cap/FIFO/cancellation, low-memory cap1, 10s admission deadline plus 11s native idle cleanup, null allocation fallback, no-store errors and native request-abort semantics. ASAN/UBSAN also pass for file scheduling and existing request-body cleanup cases. Inverted deadline mutation is caught.
Chrome, Firefox and WebKit all pass delayed, normal, initial503 recovery and terminal failure probes. Existing Web UI suite: 218 passed, 11 skipped, one unrelated settings-help size failure (18682 > 18432 bytes; file untouched). Full unittest discovery stops at missing Windows vswhere.exe. Quick evaluation: 82 passes, 0 failures.
ESP32-combo app, safeboot and LittleFS build succeeded using established PlatformIO Python environment; on-device memory/recovery measurement not yet performed.
<!-- SECTION:NOTES:END -->
