---
id: "ADR-209"
title: "Defer bounded static-file responses so the Web UI can use a render-blocking stylesheet"
status: "Proposed"
date: "2026-11-05"
binding: false
gate: null
documents_shipped: false
verified_in: []
supersedes: []
superseded_by: null
related:
  - "ADR-139"
  - "ADR-147"
  - "ADR-165"
  - "ADR-188"
topics:
  - "web-server"
  - "backpressure"
  - "webui"
  - "first-paint"
components:
  - "webSendFile"
  - "v2.html boot loader"
context_scope: "selective"
format: "canonical"
---

# ADR-209 Defer bounded static-file responses so the Web UI can use a render-blocking stylesheet

## Status

Proposed for the TASK-1604 experiment. This is not approval to replace the production loading policy. Existing active-stream, heap and connection limits remain in force.

## Context

The Web UI creates its initial stylesheet link in JavaScript, then serializes its script and font requests. The loader retries failures, including the server's temporary 503 responses, but the initial stylesheet does not block rendering. Users see a flash of unstyled content.

A native external stylesheet link in the document head blocks initial rendering, without embedding CSS in the HTML. Its normal browser fetch does not reliably retry a 503 automatically. The current server responds immediately with 503 when its file-serving gate is full, even when an active response could soon release a slot.

ADR-147 documents failures caused by concurrent LittleFS allocations. ADR-165 retains two active file streams, with a lower cap under heap pressure. ADR-188 separately limits connections capable of retaining send data. Holding a request is not free: its socket, request metadata and response object still consume memory.

## Decision

Experiment with a native `<link rel="stylesheet" href="v2-bundle.css">` after the pre-paint theme script. Keep the CSS external and preserve sequential JavaScript/font startup and bounded error recovery.

When an ordinary static response cannot start immediately, retain at most four pending responses with a ten-second admission deadline. A waiting response must not open a file, allocate a standard file-response send buffer or consume an active-file slot. Resume it from the existing AsyncTCP response callbacks, in first-in-first-out order. Do not access HTTP request objects from the main loop or a new worker task.

The native response poll runs only when the client can send. Enforce the admission deadline there and also set an eleven-second receive-inactivity timeout while waiting, clearing that timeout when streaming starts. The one-second grace lets a send-capable poll return the ten-second deadline response before native timeout handling runs. An idle zero-window client is disconnected without a send-capable poll. A zero-window peer that continually sends data can outlive the nominal deadline; this design bounds pending memory, not absolute socket lifetime against an adversarial peer.

Keep the active-file cap of two, its heap-based reduction to one, the REST gate and the connection cap. Retry admission silently; count a refusal only when returning a real 503. Queue overflow, observed deadline expiry and connection-safety rejection still return 503 with `Retry-After: 1`; an idle stalled connection is closed by the existing timeout handler. Cancellation and failed file opens release resources exactly once. Successful responses preserve content type, ETag and cache headers.

No AsyncTCP, ESPAsyncWebServer or lwIP code or configuration is changed. No HTTPS, HTTP/2, whole-file buffering or increased stream limit is introduced. This proposal complements the existing limits; it does not supersede their production safety contracts.

## Alternatives Considered

- **Keep client-only serialization and show a loading screen.** Lowest firmware risk, but keeps initial rendering dependent on JavaScript and does not test native stylesheet loading, which the user requested.
- **Raise the file cap or remove admission gates.** Smaller code change, but contradicts the measured allocation and receive-path failures documented in ADR-147, ADR-165 and ADR-188.
- **Change the TCP/HTTP libraries or embed the assets.** Could address admission earlier or eliminate filesystem allocations, but expands the experiment substantially and changes unrelated deployment and asset-update behavior.

## Consequences

The expected benefit is a styled first paint and fewer ordinary-concurrency file refusals without extra simultaneous file streams. The cost is bounded pending-request memory, a response-lifetime state machine and additional wait latency. Saturation can still reject a native stylesheet; the bounded client failure path remains necessary.

Deferred callbacks must never retain pointers to temporary path or MIME buffers. Disconnects, timer wraparound, missing files, invalid file sources and header propagation are correctness requirements, not optional hardening. A bounded queue also cannot prevent AsyncTCP from accepting sockets before HTTP admission; it is not complete transport-level overload protection.

Deferred metadata accepts paths up to 127 bytes and MIME types up to 47 bytes. Longer values return 414 and 400 respectively rather than being truncated into another resource or type. Errors use `Cache-Control: no-store` and do not inherit successful asset validators.

## Verification

Compare the old and experimental implementations using a delayed-stylesheet browser probe and a host harness that executes the production file-serving code. Exercise queue capacity, FIFO order, cancellation, low-memory admission, deadline wraparound, real safety refusals and exactly-once slot cleanup. A normal successful browser load must request the stylesheet once, paint only after it succeeds and start the script/fonts once in order.

Before any production recommendation, measure cold and warm page loads, concurrent native asset requests, cancellation and recovery on the ESP32-S3, including heap/max-block floors, refusal counters and an unchanged active-stream high-water mark. Preserve the already deployed hook, MQTT entity-name and connection-animation fixes in the test build. A host test does not establish on-device memory safety.

Initial local evidence: the production-path host regression fails before scheduling and passes afterward, including address/undefined-behavior sanitizer runs and the existing request-body cleanup cases. A deadline-condition mutation is detected. Delayed, normal, initial-503 recovery and terminal-failure browser probes pass in Chrome, Firefox and Playwright WebKit. These checks do not replace the on-device verification above or constitute branded Safari certification.

## Related Decisions

ADR-139 keeps assets on LittleFS with cache validation. ADR-147 and ADR-165 govern active file-serving memory and parallelism. ADR-188 remains the connection-safety contract. No accepted ADR body is changed by this proposal.
