# DingTalk Stream Upstream Throughput Investigation (2026-03-02)

## Scope

This document captures evidence from controlled tests focused on upstream DingTalk Stream callback delivery throughput, isolated from downstream plugin business logic.

## Test Objective

- Verify whether message loss occurs before or after local SDK callback reception.
- Isolate local handling factors (plugin processing, ACK behavior, connection errors).
- Compare behavior with and without SDK keepalive.

## Environment and Controls

- Channel plugin: `@soimy/dingtalk` (local workspace)
- Stream SDK source: local fork with instrumentation (`DWClient` debug phases and counters)
- Single-consumer validation applied during SDK-only tests:
  - `openclaw gateway stop`
  - only `scripts/dingtalk-stream-monitor.mjs` running
- Monitor options used:
  - `--sdk-debug`
  - A run: `--no-sdk-keepalive`
  - B run: `--sdk-keepalive`

## Instrumentation Points

SDK debug phases used for throughput evidence:

- `inbound-received`: raw callback frame reached SDK inbound parser
- `callback-dispatch`: callback routed to registered listener
- `socket-send`: callback ACK sent back to DingTalk server
- `socket-open`: stream connection established

Monitor summary fields used:

- `callbackReceived`, `callbackAcked`
- `socketClose`, `socketError`
- `probeOk`, `probeFail`
- `sdkTopPhases`

## Collected Results

### Run A (No SDK Keepalive)

- App messages sent: 20 (`trace-SDK-*` set)
- Real SDK callback ingress:
  - `inbound-received(real)=7`
  - `callback-dispatch(real)=7`
  - `socket-send(real)=7`
  - `trace payload matched=7`
- Connection/transport health:
  - `socketClose=0`
  - `socketError=0`
  - `probeFail=0`

Interpretation:

- Every callback that reached SDK was fully dispatched and ACKed.
- Missing messages did not fail inside local callback handling.

### Run B (SDK Keepalive Enabled)

- App messages sent: 20 (`trace-SDK-kal-*` set)
- Real SDK callback ingress:
  - `inbound=10`
  - `dispatch=10`
  - `ack=10`
  - `trace=10`
- Received trace IDs:
  - `trace-SDK-kal-001`
  - `trace-SDK-kal-006`
  - `trace-SDK-kal-007`
  - `trace-SDK-kal-008`
  - `trace-SDK-kal-010`
  - `trace-SDK-kal-011`
  - `trace-SDK-kal-013`
  - `trace-SDK-kal-014`
  - `trace-SDK-kal-016`
  - `trace-SDK-kal-017`
- Connection/transport health (from summary):
  - `socketClose=0`
  - `socketError=0`
  - `probeFail=0`

Interpretation:

- Keepalive improves observed ingress rate (7 -> 10) but does not restore full 20/20 delivery.
- Local SDK processing and ACK path remains healthy for all received callbacks.

## Core Conclusion

High-confidence conclusion:

- Message loss is occurring before local SDK callback ingress (`inbound-received`) in these tests.
- Local plugin/business processing is not the primary loss point for received callbacks.
- Local transport did not show instability during sampled windows (`socketClose=0`, `socketError=0`, `probeFail=0`).

Working hypothesis:

- Upstream DingTalk Stream push path (server-side delivery/routing/application-side push selection) is the dominant bottleneck for missed app-originated messages in this test setup.

## Suggested Escalation Package (to DingTalk Support)

Use the following concise evidence summary:

```text
Single-consumer stream tests with SDK-level instrumentation:
- Sent 20 app messages in each run.
- Without SDK keepalive: SDK inbound callbacks = 7/20.
- With SDK keepalive: SDK inbound callbacks = 10/20.
- For all callbacks received locally, dispatch and ACK are 100% successful.
- Local transport stable during windows: socketClose=0, socketError=0, probeFail=0.
Conclusion: missing messages are not dropped in local callback processing; they fail to reach SDK callback ingress, indicating upstream stream push path loss.
```

Recommended attachments:

- raw monitor logs (`/tmp/sdk-*.log`)
- sample missing trace ID list
- connection metadata (`connectionId`, `messageId` snippets)
