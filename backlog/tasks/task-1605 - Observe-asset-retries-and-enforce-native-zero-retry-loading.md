---
id: TASK-1605
title: Observe asset retries and enforce native zero-retry loading
status: In Progress
assignee:
  - '@omp'
created_date: '2026-11-05T10:14:00Z'
updated_date: '2026-11-05T11:25:00Z'
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
- [x] #3 Cold and warm native external CSS, script, fonts and favicon loading succeeds without fallback or JavaScript retries under normal browser concurrency
- [x] #4 Injected faults and genuine overload produce truthful diagnostics and recover without leaking resources
- [x] #5 ESP32-combo builds and footprint checks pass; device settings and earlier fixes are preserved during field validation
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

ESP32-combo app, safeboot and LittleFS build succeeded for 2.0.0-beta.73+e62c403. Native font faces remain external; staged assets still need 656 KiB and preserve the 96 KiB runtime reserve within the 768 KiB partition. Private current-settings/UI backup and beta.72 app/filesystem rollback images are ready before field deployment.

Deployed 2.0.0-beta.73+e62c403 to OTGW2; all 175 exposed settings stayed unchanged and MQTT, boiler and thermostat reconnected. App image 2144992 bytes, +1184 versus beta.72, 93.5% utilization with about145.3 KiB app headroom.
Independent acceptance probe: 18 native loads (three cold/warm pairs each in Chrome, Firefox and WebKit), zero asset retries/failed attempts, one first attempt per critical asset, matching network observations and console summaries. Sixteen recorded paint samples followed CSS; two had no paint timing. A six-socket direct burst returned all200 without JS or client retries, and again passed after overload/cancellation.
A deliberately bounded eight-socket overload returned four503s with Retry-After1. Pending cancellation was recorded once; queue drained with active HWM2 and pending HWM4. Final counters: queued93, admitted92, cancelled1, timeouts0, pending0, max wait3410ms. All four file503s came from deliberate overload. One broader REST connection-cap refusal was observed, so no claim that every API request is retry-free.
Browser-side initial503 injection for CSS/JS/font produced retry/recovered/summary events, complete recovery with usable fonts, cleanFirstAttempt=false and acceptance=false. CSS/JS each retried once; font case retried twice. Device files were not altered for fault injection.
Final 21-minute snapshot: free heap57012, max block31732, minima16212/8692, no low/warning/critical entries or MQTT/WS drops; no reboot observed. Live UI visually verified through transparent SSH tunnel because automated Chrome cannot reach the unit directly on this Mac. Full-suite blockers remain the unrelated settings-help byte limit and Windows vswhere.exe; task is not marked Done without those gates.
<!-- SECTION:NOTES:END -->

## Final Summary

<!-- SECTION:FINAL_SUMMARY:BEGIN -->
Made the normal CSS/script/font path browser-native and observable. Per-navigation otgwAssetLoad records attempts, retries, failures and clean-first-attempt completion; structured console snapshots expose retry, recovery, terminal failure and summary events. Added an independent acceptance CLI that rejects fallback, refusal, duplicate requests and incomplete evidence.

Corrected sender admission to exclude only live response-owned zero-TX waiters, matching PCB and AsyncClient identity. Kept two active files, four pending entries, sender reservation and existing TX-bearing connection protection. Added queue lifecycle metrics to existing REST/telnet/reset diagnostics. Proposed ADR-210 records the deliberate amendment.

Deployed beta.73+e62c403 to OTGW2 with prior fixes and settings preserved. Eighteen native cold/warm browser loads passed with zero asset retries. Six sockets now succeed; bounded overload and cancellation recover. Deliberate failures produce truthful non-clean diagnostics. The app grew by 1184 bytes and remains at 93.5% flash.

Local changed-path tests, sanitizers, mutation, staging, evaluation, governance and embedded builds passed. Existing full-suite blockers: settings-help size assertion and Windows-only toolchain. One REST refusal was observed outside the asset-only criterion. Short-run one-unit evidence does not constitute a long soak or general production approval.
<!-- SECTION:FINAL_SUMMARY:END -->
