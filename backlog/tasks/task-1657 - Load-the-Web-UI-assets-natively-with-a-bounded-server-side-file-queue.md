---
id: TASK-1657
title: Load the Web UI assets natively with a bounded server-side file queue
status: In Progress
assignee:
  - '@andrebrait'
created_date: '2026-11-05T13:12:00Z'
updated_date: '2026-11-05T13:14:00Z'
labels:
  - webui
  - web-server
dependencies: []
ordinal: 781000
---

## Description

<!-- SECTION:DESCRIPTION:BEGIN -->
Upstream beta.94 (TASK-1643) loads v2-bundle.css with a render-blocking link and retries 503 responses from JavaScript, then loads v2.js and the fonts one at a time. Replace that loader with native browser loading (stylesheet link, deferred script, font preloads) and let the server hold a small, bounded queue of file requests instead of answering 503 while its two file-stream slots are busy.
<!-- SECTION:DESCRIPTION:END -->

## Acceptance Criteria
<!-- AC:BEGIN -->
- [x] #1 A normal single-tab cold or warm page load completes with zero asset retries in Chrome, Firefox and WebKit
- [x] #2 No more than two file streams are active at any time
- [x] #3 Queue overflow, deadline expiry and genuine memory pressure still return 503 with Retry-After
- [x] #4 A same-build A/B comparison against beta.94 is recorded, including heap minima
- [ ] #5 The two-tab heap dip (largest free block 7,668 B vs 13,812 B on beta.94) is fixed or explained before the ADR is accepted
<!-- AC:END -->

## Implementation Notes

<!-- SECTION:NOTES:BEGIN -->
- Ported from the TASK-1604/1605 experiment branches onto beta.94; replaces the TASK-1643 loader.
- A/B on OTGW2 (same unit, beta.94 vs beta.94 + change): single-tab 30/30 clean vs 29/30; cold median 2.2-2.7 s vs 3.3-4.2 s; two-tab 10/10 vs 5/10 complete.
- Open: two-tab largest free block 7,668 B (beta.94 13,812 B), below the 8,192 B device/info guard. Under investigation.
<!-- SECTION:NOTES:END -->
