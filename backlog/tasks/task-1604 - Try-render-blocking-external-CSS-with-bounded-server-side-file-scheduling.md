---
id: TASK-1604
title: Try render-blocking external CSS with bounded server-side file scheduling
status: In Progress
assignee:
  - '@omp'
created_date: '2026-11-05T01:11:00Z'
updated_date: '2026-11-05T09:11:00Z'
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
- [x] #4 Cold loads and overload recovery are measured with required existing fixes preserved
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

User approved live OTA. OTGW2 now runs beta.72+895b46c; filesystem hash/complete v2.js verified and all175 exposed settings unchanged. MQTT/boiler/thermostat reconnected.
Concurrency2/3/4: all assets200 and hash/length correct, file HWM2. Six-connection overload: all6 refused by unchanged connection cap with Retry-After1; recovery successful. Cancelling pending CSS behind two slow streams recovered to three successful parallel downloads. Heap returned to57204 bytes with31732 max block; probe minima20644/11252, no low/warning/critical entries or drops.
Live Chrome through transparent SSH tunnel: cold CSSend/FCP543/572ms and527/556ms; warm422/452ms; oneCSSfetch each, no failing resources/JS exceptions. Direct automated Chrome was blocked by Mac network routing/access; tunnel changed no device configuration.
Do not remove all JS script/font orchestration: the native six-connection burst still gets503s. Keep ADR-209 Proposed; this is one-unit short-run evidence. Task remains In Progress because full repository test gates have unrelated existing blockers.

Final nine-minute device snapshot: 56,392 bytes free heap, 31,732-byte largest block; recorded minima across HTTP and browser checks were 20,204 and 11,252 bytes. File high-water mark remained two; there were zero file-gate refusals and exactly six deliberate connection-cap refusals. All 175 settings remained unchanged, and MQTT, boiler and thermostat were connected. Test browser tabs and SSH tunnels were closed.
<!-- SECTION:NOTES:END -->

## Final Summary

<!-- SECTION:FINAL_SUMMARY:BEGIN -->
Implemented and field-tested a native external render-blocking stylesheet and bounded server-side static-file scheduling. The server retains two active streams (one under low heap), four pending responses, a 10-second admission deadline, 11-second idle cleanup, and the existing connection cap. Added production-path host regressions and proposed ADR-209.

Deployed 2.0.0-beta.72+895b46c to OTGW2 with the earlier hook, MQTT naming and Connection animation fixes preserved. All 175 settings were unchanged. Two-to-four concurrent downloads succeeded; cancellation and overload recovered. Live cold/warm browser first paint followed stylesheet completion, with one CSS fetch and no resource failures.

Validation: Chrome, Firefox and WebKit loading probes, scheduling/body regressions, sanitizers, quick evaluation, ADR governance, and the ESP32-combo app/filesystem build passed. The existing Web UI suite has one unrelated settings-help byte-budget failure (18,682 versus 18,432 bytes); full unittest discovery requires Windows vswhere.exe. Six simultaneous connections still receive safety 503 responses, so script/font sequencing and bounded retries remain. This is short-run one-unit evidence, not production rollout approval.
<!-- SECTION:FINAL_SUMMARY:END -->
