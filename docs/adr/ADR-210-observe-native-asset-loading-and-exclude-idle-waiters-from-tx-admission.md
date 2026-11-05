---
id: "ADR-210"
title: "Observe native asset loading and exclude bounded idle waiters from TX admission"
status: "Proposed"
date: "2026-11-05"
binding: false
gate: null
documents_shipped: false
verified_in: []
supersedes: []
superseded_by: null
related:
  - "ADR-147"
  - "ADR-165"
  - "ADR-188"
  - "ADR-209"
topics:
  - "web-server"
  - "backpressure"
  - "native-assets"
  - "observability"
components:
  - "WebQueuedFileResponse"
  - "platformTcpTxHoldingOnPort"
  - "v2.html asset diagnostics"
context_scope: "selective"
format: "canonical"
---

# ADR-210 Observe native asset loading and exclude bounded idle waiters from TX admission

## Status

Proposed for TASK-1605. This is a user-approved experiment, not approval for general production rollout. It proposes a specific amendment to ADR-188's connection accounting and replaces ADR-209's experimental client serialization. Accepted ADR-188 remains unchanged while this proposal is under evaluation; adoption must record the partial supersession explicitly rather than silently rewriting its decision.

## Context

The first experiment could serve two, three and four simultaneous downloads, but six native connections all received 503 from the separate connection cap. Six connections are an ordinary browser pattern, not automatically genuine overload. The existing JavaScript fallback could conceal that failure during browsing, so a successful serialized load did not establish native-loading reliability.

The cap counts every ESTABLISHED connection as a possible sender. A response retained in the bounded file FIFO is different: it cannot transmit until admitted, holds no file/send buffer and may safely wait without being counted as another eligible sender. Simply subtracting queue length is unsafe because a cancelled waiter may no longer be in the native PCB list, and an old PCB address may be reused by an unrelated connection.

## Acceptance Criterion

> Retries remain legitimate for network faults or genuine overload. But the acceptance criterion should be: normal native asset loading succeeds with zero JavaScript retries.

A normal cold/warm load uses parser-discovered external CSS, a deferred script and native font requests. It must not depend on the old serialized JavaScript startup. All required assets must complete on their first attempt, observed HTTP responses must be successful, and the browser diagnostic record must report zero failed attempts and zero retries. An incomplete or recovered load is not clean success. Favicon failures are also failures when observed by the browser or checked separately by the acceptance probe.

## Decision

Keep two active file streams, the heap-based reduction to one, at most four pending file responses, the ten-second admission deadline and eleven-second idle receive timeout. Keep the cap of four eligible/TX-bearing HTTP connections. No library, lwIP configuration, TLS or HTTP/2 changes are introduced.

Amend eligibility accounting only for identified response-owned idle waiters. Under the lwIP core lock, match both the live PCB address and its AsyncClient callback owner. An entry with unsent or unacked segments always counts, including a closing connection. A waiter that has disappeared or whose PCB address was reused must not hide another sender. Before enabling an excluded candidate to send, reserve its one sender slot.

When the connection cap prevents admission, hold the bounded file request instead of immediately returning 503. Pump the oldest eligible FIFO response on arrivals, polls and release, only on the existing async_tcp task. This lets normal six-connection arrivals make progress before the pending queue fills. Queue overflow, admission expiry and genuine memory/TX pressure remain bounded failure paths; do not raise active-file or send-memory limits to manufacture a pass.

Make normal CSS/script/font loads browser-native. Keep font-face rules in the external bundle and preload the three fonts with matching anonymous CORS settings, so CSS reuses those downloads. Retain bounded JavaScript recovery only after an actual load error. Do not embed CSS, JavaScript or font data into HTML.

## Diagnostics

`window.otgwAssetLoad` is one bounded per-navigation record for the five critical assets: `v2-bundle.css`, `v2.js` and the three WOFF2 fonts. It exposes `mode`, `pageId`, `complete`, `cleanFirstAttempt`, `retryCount`, `failureCount` and per-asset `attempts`, `retries`, `state`, `lastElapsedMs`. `failureCount` includes failed attempts even when recovery succeeds. Unsupported/skipped or incomplete assets cannot produce clean first-attempt success.

Console events `asset_retry`, `asset_recovered`, `asset_failed` and `asset_load_summary` carry the navigation ID and fixed asset path. Event payloads are snapshots, not references to mutable diagnostic state. Native load/error events do not identify HTTP status, so they report a generic load error; the independent network probe supplies response codes. No extra diagnostic HTTP requests or unbounded event history are added to the page.

Existing REST device-info and telnet diagnostics expose queue totals for deferred/admitted/cancelled/timed-out requests, maximum wait milliseconds, current pending depth and pending high-water mark. They are bounded integer fields and do not control admission. Resetting diagnostics must not reset or corrupt live queue state. Existing file/connection refusal counters remain available. No new MQTT topics are added.

## Alternatives Considered

- **Keep serialized startup and call successful recovery a pass.** Rejected: hides ordinary asset refusals and does not meet the acceptance criterion.
- **Raise the connection/file limits.** Rejected: the historical receive-path and LittleFS failures remain relevant, and more simultaneous file buffers are unnecessary.
- **Bundle every asset into one HTML file.** Plausible lower-server-complexity alternative, but the user selected external assets and native loading. Font embedding also adds base64 bytes and changes cache/update behavior; this experiment does not implement it.
- **Subtract the number of pending requests from the native connection count.** Rejected: stale/cancelled waiters and PCB reuse can undercount real senders, weakening the safety cap.

## Consequences

Normal native bursts can wait without creating extra file streams or being rejected solely for having accepted sockets. Retry masking becomes observable in both aggregate and per-asset form. The costs are a small bounded telemetry state and more precise eligibility/FIFO lifetime handling.

This does not guarantee unlimited clients or absolute socket lifetime against a zero-window peer that continually sends receive data. The pending queue and active resources remain bounded. Public browser diagnostics are operational evidence, not a security boundary or a promise that application JavaScript executed without a runtime exception.

## Verification

The host regression compiles production file serving, gate accounting, platform PCB walking, diagnostic state and reset code. It must demonstrate six pre-existing live asset sockets progressing without 503, FIFO/cap bounds, correct sender reservation, and no exclusion of TX-bearing, closing or reused PCB entries. Preserve cancellation, allocation failure, timer rollover, error caching and native pre-/post-send abort regressions.

Browser regressions exercise clean native requests, cached/reload state reset, genuine CSS/script/font failures, recovery, exhaustion and incomplete loads. The independent `scripts/tests/asset_loading.py` probe observes requests/statuses and the page record; it rejects managed mode, retries, failed attempts, duplicate critical-asset requests and missing response evidence even when the page claims success. Cold and warm checks must pass on the real device with settings, integrations and memory safety preserved. Local success alone is not field acceptance.

## Related Decisions

ADR-147 and ADR-165 retain their active-stream and heap constraints. ADR-188's conservative possible-sender accounting is the explicit proposed amendment. ADR-209 records the earlier experiment and its six-connection limitation; those historical measurements are preserved.
