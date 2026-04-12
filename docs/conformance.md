# CapConnect v1 Conformance Test Runner

Status: initial conformance plan

This document defines the first conformance tests for CapConnect v1
implementations. The goal is to make protocol behavior testable before session
logic grows complex.

The canonical protocol spec is `docs/spec.md`. This document is a test plan, not
a replacement for the spec.

## Scope

The v1 conformance runner MUST cover:

- Binary frame header parsing.
- Frame flag rejection.
- Handshake ordering.
- Capability lineage and rights validation.
- `RESOLVE` ordering and binding semantics.
- Recursive `REVOKE` inference.
- C3 broken promise cascade.
- Flow-control overflow behavior.

The v1 conformance runner MUST NOT require:

- Streaming rights.
- Resumable sessions.
- Compression negotiation.
- Codegen.

Those are deliberately out of scope for v1 and may be added in v2 through the
wire `version` field and handshake `features` negotiation.

## Open Questions Beyond v1 Core

The current v1 docs solve protocol-core concerns. The remaining gaps are mostly
ecosystem and rollout concerns that should be tracked in parallel with v2
feature work.

Ecosystem and operational gaps:

- Adoption model across existing service fleets.
- Tooling maturity (debuggers, inspectors, protocol-aware CLI tooling).
- Observability standards (metrics, tracing, structured event taxonomy).
- Migration path from gRPC deployments.
- Large-scale operational compatibility (proxies, load balancers, edge setups).

Protocol features intentionally deferred to v2:

- Streaming rights.
- Resumable sessions.
- Compression negotiation.
- Codegen.

## Runner Shape

A runner MAY test an implementation through either:

- Black-box transport: open a WebSocket using `Sec-WebSocket-Protocol:
  capconnect.v1`, send raw CapConnect frames, and inspect returned frames.
- In-process harness: call the implementation's frame parser, registry, and
  session actor directly.

Both runner styles MUST enforce the same expected outcomes.

Recommended phases:

1. Frame parser vectors.
2. Handshake behavior.
3. Capability registry transitions.
4. End-to-end session scenarios.

## Required Test Cases

### T1: Frame Parser Vectors

Use the binary vectors in `docs/spec.md` section `4.2`.

Expected outcomes:

- Vector 1 `PING`: accepted.
- Vector 2 `CALL` with `ACK_REQUIRED`: accepted by frame parser.
- Vector 3 `BROKEN` with payload: accepted by frame parser.
- Vector 4 `REVOKE` with empty reason map `A0`: accepted.
- Vector 5 unknown flag `0x8000`: rejected with `E_BAD_FRAME`.
- Vector 6 `RESERVED_STREAM`: rejected with `E_BAD_FRAME`.

Assertions:

- Header length is exactly 40 bytes.
- `cap_id` is decoded as `u64` big endian.
- `payload_len` equals the actual payload byte length.
- Parser never mutates session state.

### T2: Handshake

Setup:

1. Open transport.
2. Client sends `HELLO` on `chain_id=0`, `cap_id=0`, `seq=0`.
3. Server sends `WELCOME` on `chain_id=0`, `cap_id=0`, `seq=0`.

Assertions:

- Server rejects unsupported WebSocket subprotocols.
- Server closes or rejects if first CapConnect frame is not `HELLO`.
- Server SHOULD close the transport if valid `HELLO` does not arrive within 30
  seconds.
- Next control-plane frame from the same endpoint on `chain_id=0` uses `seq=1`.

### T3: Successful Resolve With Pipelined Call

This test corresponds to Sequence A in `docs/spec.md` section `4.1`.

Setup:

1. Client receives promised capability `cap_id=41`, `state=promised`,
   `chain_id=7`.
2. Client sends `CALL` to `cap_id=41` with `seq=1`.
3. Server has existing concrete capability `cap_id=40`.
4. Server sends `RESOLVE` binding `promised=41` to `concrete=40` on `chain_id=7`
   with the next sequence number.

Assertions:

- `RESOLVE` is sent on the promised capability's stored `chain_id`.
- `RESOLVE` changes existing `cap_id=41` from `PROMISED` to `LIVE`.
- `RESOLVE` does not create a new entry for `cap_id=41`.
- `RESOLVE` does not consume `cap_id=40`.
- Buffered `CALL` is processed after resolve and returns exactly once.
- `RETURN.reply_to` equals the original `CALL.msg_id`.

Negative variant:

- Send `RESOLVE` for `cap_id=41` on any chain other than `7`.
- Expected: receiver rejects with `E_PROTOCOL`.

### T4: Recursive Revoke Inference

This test corresponds to Sequence B in `docs/spec.md` section `4.1`.

Setup:

1. Server owns root capability `cap_id=50`.
2. Client knows derived descendants `cap_id=51` and `cap_id=52` through lineage.
3. Server sends exactly one `REVOKE` for `cap_id=50`.

Assertions:

- Receiver infers revocation for `50`, `51`, and `52`.
- Receiver does not wait for descendant `REVOKE` frames.
- Sender does not emit descendant `REVOKE` frames in the same cascade.
- New `CALL` to `51` or `52` is rejected with `E_CAP_REVOKED`.
- Pending calls to affected caps are rejected with `E_CAP_REVOKED`.

Negative variant:

- Clear lineage for a descendant before applying `REVOKE`.
- Expected: implementation fails conformance because receiver-side lineage is
  mandatory.

### T5: C3 Broken Promise Cascade

This test corresponds to Sequence C in `docs/spec.md` section `4.1`.

Setup:

1. Server grants promised root `cap_id=70`.
2. Descendants `cap_id=71` and `cap_id=72` exist in the same session.
3. Client pipelines calls to `70` and `71`.
4. Server detects failure on `70` and triggers C3.

Assertions:

- Registry marks `70`, `71`, and `72` as `BROKEN` atomically.
- Pending calls to affected caps are rejected with `E_PROMISE_BROKEN`.
- Server emits `BROKEN` frames for every remotely visible affected capability.
- Emitted `BROKEN` frames are in chain order.
- Later `CALL` to `70`, `71`, or `72` is rejected with `E_PROMISE_BROKEN`.
- Peer never observes a partial cascade.

### T6: Rights Attenuation

Setup:

1. Parent capability has rights `CALL | GRANT`.
2. Sender attempts to grant child with `CALL | GRANT | REVOKE`.

Assertions:

- Receiver rejects the child grant with `E_RIGHTS`.
- Receiver does not install the invalid child capability.
- Receiver keeps the parent capability state unchanged.

Negative flag check:

- Sender attempts to grant any unassigned rights bit.
- Expected: receiver rejects with `E_RIGHTS`.

### T7: Pipeline Overflow

Setup:

1. Receiver advertises `max_pipeline_depth=2` and `max_total_in_flight=3`.
2. Sender sends more than two in-flight calls on one chain.
3. Sender sends more than three total in-flight calls across all chains.

Assertions:

- Per-chain overflow is rejected with `E_PIPELINE_OVERFLOW`.
- Global in-flight overflow is rejected with `E_PIPELINE_OVERFLOW`.
- Earlier accepted calls keep their ordering.
- Receiver SHOULD keep the session open for isolated overflow.
- Receiver MAY close the session for persistent or abusive overflow.

## Result Format

A runner SHOULD emit machine-readable results:

```json
{
  "protocol": "capconnect",
  "version": 1,
  "implementation": "name/version",
  "results": [
    {
      "id": "T1",
      "name": "Frame Parser Vectors",
      "status": "pass"
    }
  ]
}
```

`status` MUST be one of:

```text
pass
fail
skip
```

Skipped tests MUST include a reason. A v1 implementation MUST NOT skip T1, T2,
T3, T4, or T5 and still claim full CapConnect v1 conformance.

## Implementation Order

Recommended implementation order:

1. Build binary frame encoder/decoder and pass T1.
2. Build handshake and pass T2.
3. Build capability registry with lineage and pass T3/T4/T6.
4. Build message ordering and pass T3/T7.
5. Build C3 cascade and pass T5.

The conformance runner should exist before the full reference session runtime.
That keeps C3 cascade and recursive `REVOKE` inference honest while the runtime
is still small enough to debug.

## TODO Before Major Coding

- [ ] Add a traceability table: `Spec Clause -> Test ID (T1..T7)` for coverage
  auditing.
- [ ] Store test fixtures in separate files (`docs/fixtures/*.json` and hex
  vector files) so runners can load cases automatically.
- [ ] Define conformance claim levels:
  `core` (without HTTP fallback) and `full` (with HTTP fallback).
