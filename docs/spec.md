# CapConnect v1 Specification

Status: final v1 working spec

CapConnect is a session-oriented capability protocol for connecting clients and
servers over a small binary frame format. Its core invariant is that capability
lifecycle is part of the protocol, not only an application convention.

The most important rule in this document is **C3: broken promise cascade**. Any
implementation that does not enforce C3 is not CapConnect-compatible.

Normative terms:

- `MUST`, `MUST NOT`, `REQUIRED`: required for protocol compatibility.
- `SHOULD`, `SHOULD NOT`: recommended unless there is a documented reason.
- `MAY`: optional.

## Part 1: Protocol Identity, Transport, and Wire Format

### 1.1 Name and Version

Protocol name: `CapConnect`

Wire version: `1`

WebSocket subprotocol: `capconnect.v1`

Default structured payload encoding: deterministic CBOR

Binary fields are encoded in network byte order, also known as big endian.

The `capconnect.v1` subprotocol token is syntactically valid for
`Sec-WebSocket-Protocol`. Public deployments SHOULD register the token with the
IANA WebSocket Subprotocol Name Registry or use an origin-scoped token such as
`capconnect.example.com.v1` to avoid collisions.

### 1.2 Transport Layer

The primary transport for CapConnect v1 is WebSocket over TLS.

The client opens a WebSocket connection and requests:

```http
Sec-WebSocket-Protocol: capconnect.v1
```

The server MUST reject the upgrade if it cannot speak `capconnect.v1`.

One WebSocket connection maps to one CapConnect session. Closing the connection
closes the session unless the implementation explicitly supports resumable
sessions. Resumable sessions are out of scope for v1.

CapConnect frames MUST be sent as WebSocket binary messages. Text messages MUST
be treated as `E_BAD_FRAME`.

### 1.3 Wire Frame

Each reassembled WebSocket binary message contains exactly one CapConnect frame.
The receiver MUST parse the CapConnect frame from the WebSocket message payload,
not from raw WebSocket frame boundaries. WebSocket fragmentation, masking, and
control frames remain transport-layer concerns.

Fixed header size: 40 bytes

```text
offset  size  field
0       2     magic
2       1     version
3       1     type
4       2     flags
6       2     header_len
8       4     payload_len
12      4     chain_id
16      8     cap_id
24      8     seq
32      8     msg_id
40      n     payload
```

Field rules:

- `magic` MUST be `0x4343`, the ASCII bytes `CC`.
- `version` MUST be `1`.
- `type` is one of the message types in Part 2.
- `flags` is a bitset. Unknown flags MUST cause `E_BAD_FRAME`.
- `header_len` MUST be `40` for v1.
- `payload_len` is the number of bytes after the header.
- `chain_id` identifies the capability chain for ordering.
- `cap_id` identifies the target or subject capability for the message.
- `seq` is the sender sequence number within `chain_id`.
- `msg_id` is a sender-scoped message identifier for correlation.

Frame flags:

```text
0x0001 ACK_REQUIRED
0x0002 RESERVED_STREAM
0x0004 HAS_ERROR
0x0008 NO_PIPELINE
```

`RESERVED_STREAM` is reserved for a future streaming extension and MUST be `0`
in v1. A receiver MUST reject frames with `RESERVED_STREAM` set using
`E_BAD_FRAME`.

The default maximum frame payload is 16 MiB. A peer MAY advertise a smaller or
larger limit during handshake. A receiver MUST reject frames larger than its
advertised maximum with `E_FRAME_TOO_LARGE`.

## Part 2: Messages, Capability Table, and Semantics

### 2.1 Message Types

```text
0x01 HELLO
0x02 WELCOME
0x03 CALL
0x04 RETURN
0x05 GRANT
0x06 RESOLVE
0x07 REVOKE
0x08 RELEASE
0x09 BROKEN
0x0A ACK
0x0B PING
0x0C PONG
0x0D CLOSE
```

Message meaning:

- `HELLO`: client opens the CapConnect protocol inside the transport.
- `WELCOME`: server accepts the session and returns negotiated settings.
- `CALL`: caller invokes an operation on a capability.
- `RETURN`: callee returns a result or application-level failure.
- `GRANT`: sender introduces a new capability reference to the receiver.
- `RESOLVE`: sender resolves a promised capability to a concrete capability.
- `REVOKE`: owner invalidates a capability and its descendants.
- `RELEASE`: receiver tells the owner it no longer needs a capability.
- `BROKEN`: sender marks a promised capability as permanently broken.
- `ACK`: sender acknowledges a frame when `ACK_REQUIRED` was set.
- `PING`: liveness probe.
- `PONG`: liveness response.
- `CLOSE`: graceful protocol close.

### 2.2 Capability Reference

A capability reference is:

```text
CapRef {
  cap_id: u64
}
```

`cap_id = 0` is reserved and MUST NOT be used for an application capability.

`cap_id` MUST be globally unique within a session for the endpoint that minted
it. Reuse of a previously released, revoked, or broken `cap_id` in the same
session is forbidden and MUST be treated as `E_PROTOCOL`.

### 2.3 Capability Entry

Each session maintains a capability table. A capability table entry contains:

```text
CapEntry {
  ref: CapRef
  chain_id: u32
  parent: Option<CapRef>
  owner: Endpoint
  state: CapState
  rights: Rights
}
```

`Endpoint` is one of:

```text
CLIENT
SERVER
```

`CapState` is one of:

```text
PROMISED
LIVE
REVOKED
RELEASED
BROKEN
```

`Rights` is a bitset:

```text
0x0001 CALL
0x0002 GRANT
0x0004 RESOLVE
0x0008 REVOKE
```

Rights are attenuating. A derived capability MUST NOT have rights that are not
present on its parent unless the owner explicitly grants a new root capability.
Unassigned rights bits MUST be `0`; a receiver MUST reject any grant or use of
unassigned rights bits with `E_RIGHTS`.

CapConnect v1 does not define streaming capability rights. Streaming MUST be
specified by a future version or negotiated extension before either endpoint
uses stream-specific rights or flags.

### 2.4 Ownership Rules

R1. Exactly one endpoint owns a capability at a time.

R2. Only the owner MAY revoke a capability.

R3. Only an endpoint with `GRANT` right MAY create a derived capability visible
to the peer.

R4. A derived capability MUST NOT amplify rights from its parent.

R5. `RELEASE` is advisory and idempotent. It means the receiver is done with the
capability. It does not grant the receiver authority to revoke the capability.

R6. `REVOKE` is authoritative and recursive. A revoked capability invalidates
all descendants in the same session.

R6a. `REVOKE` uses single-frame recursive inference. The owner MUST send
`REVOKE` for the ancestor capability only; the receiver MUST infer recursive
revocation for every locally known descendant from its lineage table. The sender
MUST NOT emit additional `REVOKE` frames for descendants in the same cascade.
This is intentionally different from C3 `BROKEN`, which emits per remotely
visible capability so each pending promise observes an explicit broken reason.

R7. `RESOLVE` is valid only for a capability in `PROMISED` state.

R8. `BROKEN` is valid only for `PROMISED` or `LIVE` capabilities that can no
longer satisfy pending or future work.

### 2.5 Pipelining Semantics

CapConnect supports promise pipelining. A peer MAY send `CALL` messages to a
capability while that capability is still in `PROMISED` state, unless the frame
or negotiated settings set `NO_PIPELINE`.

Ordering is per capability chain:

- Every chain has a monotonic `seq`.
- A sender MUST increment `seq` by 1 for each frame in the same `chain_id`.
- A receiver MUST process frames for the same `chain_id` in `seq` order.
- A receiver MAY process different `chain_id` values concurrently.
- A receiver MAY buffer out-of-order frames up to the negotiated pipeline depth.
- A repeated `seq` MUST be rejected with `E_SEQ_REPLAY`.
- A sequence gap that cannot be buffered MUST be rejected with `E_SEQ_GAP`.

If a promised capability resolves, buffered calls for that capability continue in
chain order. If it breaks, C3 applies.

Chain identifier assignment:

- `chain_id` namespace is sender-scoped. The effective ordering key is
  `(sender_endpoint, chain_id)`.
- `chain_id = 0` is reserved for control-plane frames (`HELLO`, `WELCOME`,
  transport/session control) and MUST NOT be used for application capability
  traffic.
- Sender chooses non-zero `chain_id` values for new capability chains and MUST
  avoid reusing an active chain identifier.
- Receiver MUST maintain independent sequence tracking per
  `(sender_endpoint, chain_id)` to prevent collisions between peers.
- Sequence numbers begin at `0` for every `(sender_endpoint, chain_id)`.
- `chain_id = 0` is not exempt from ordering. `HELLO` and `WELCOME` both use
  `seq = 0` because they are sent by different endpoints. The next control-plane
  frame sent by the same endpoint on `chain_id = 0` MUST use `seq = 1`.

### 2.6 Rights Enforcement

Receiver-side rights validation is mandatory and MUST NOT rely on blind trust.

- Receiver MUST maintain a capability lineage table containing at least:
  `cap_id`, `parent_cap_id` (if any), `owner`, and `rights`.
- On `GRANT` with non-null parent, receiver MUST verify:
  1) parent exists in local lineage table,
  2) sender is authorized owner/delegator for that parent,
  3) granted rights are a subset of parent rights.
- If any check fails, receiver MUST reject with `E_RIGHTS` or `E_UNKNOWN_CAP`
  and MUST NOT install the new capability entry.
- On `GRANT` with null parent, receiver treats capability as new root delegated
  by sender authority; lineage starts at that root.
- Receiver MUST validate operation permissions at `CALL` time; invoking an
  operation that needs missing rights MUST fail with `E_RIGHTS`.

### 2.7 Error Codes

Protocol errors are encoded as symbolic strings in structured payloads. The
wire-level error code list for v1 is:

```text
E_PROTOCOL
E_VERSION
E_BAD_FRAME
E_FRAME_TOO_LARGE
E_UNKNOWN_TYPE
E_ENCODING
E_UNKNOWN_CAP
E_CAP_REVOKED
E_CAP_RELEASED
E_PROMISE_BROKEN
E_NOT_OWNER
E_RIGHTS
E_SEQ_GAP
E_SEQ_REPLAY
E_PIPELINE_OVERFLOW
E_TIMEOUT
E_UNAVAILABLE
E_INTERNAL
E_HTTP_FALLBACK_UNSAFE
```

Application errors returned by a capability call MUST be carried in `RETURN`,
not by violating the protocol. Protocol errors MUST use `BROKEN`, `REVOKE`, or
`CLOSE`, depending on scope.

### 2.8 Capability State Transition Table

State transitions are normative and MUST be enforced by every compliant
implementation.

```text
from       event/message                          to         allowed
PROMISED   RESOLVE(valid concrete capability)     LIVE       yes
PROMISED   BROKEN(owner or detector emits break)  BROKEN     yes
PROMISED   REVOKE(owner)                          REVOKED    yes
PROMISED   RELEASE(non-owner drops local ref)     RELEASED   yes
LIVE       BROKEN(owner or detector emits break)  BROKEN     yes
LIVE       REVOKE(owner)                          REVOKED    yes
LIVE       RELEASE(non-owner drops local ref)     RELEASED   yes
REVOKED    any                                    *          no
RELEASED   any                                    *          no
BROKEN     any                                    *          no
```

Transition guards:

- `RESOLVE` MUST fail with `E_PROTOCOL` if the source state is not `PROMISED`.
- `RESOLVE` MUST preserve the promised capability's existing `chain_id`,
  `parent_cap_id`, owner, and rights unless the resolve payload explicitly
  narrows rights.
- `REVOKE` MUST fail with `E_NOT_OWNER` if the sender is not owner.
- `RELEASE` is idempotent. Repeated `RELEASE` on already `RELEASED` state MUST
  be accepted as no-op.
- `BROKEN` on already `BROKEN` state MUST be accepted as idempotent no-op.
- Any attempt to move from terminal state (`REVOKED`, `RELEASED`, `BROKEN`) to
  `LIVE` or `PROMISED` MUST fail closed with `E_PROTOCOL`.
- Recursive transitions from `REVOKE` and C3 cascade MUST apply in deterministic
  order: parent first, then descendants in per-chain `seq` order.

### 2.9 Flow Control and Overflow Behavior

`max_pipeline_depth` is a directional hard limit per capability chain and per
sender endpoint. `max_total_in_flight` is a directional hard limit across all
chains for a sender endpoint.

- Receiver MUST count in-flight `CALL` messages that have not produced terminal
  `RETURN` or terminal capability-state transition.
- The client advertises its receive limits in `HELLO`; the server advertises its
  receive limits in `WELCOME`.
- Each sender MUST obey the peer's advertised receive limits.
- If an incoming `CALL` exceeds the receiver's advertised `max_pipeline_depth`,
  receiver MUST reject that call with
  `RETURN{ ok=false, error.code=E_PIPELINE_OVERFLOW }`.
- If an incoming `CALL` exceeds the receiver's advertised
  `max_total_in_flight`, receiver MUST reject that call with
  `RETURN{ ok=false, error.code=E_PIPELINE_OVERFLOW }`.
- Receiver SHOULD keep the session open after isolated overflow events.
- Receiver MAY close the session with `CLOSE` and `E_PIPELINE_OVERFLOW` when
  overflow is persistent or abusive.
- Overflow handling MUST be deterministic and MUST NOT reorder earlier accepted
  calls in that chain.

## Part 3: Handshake, Encoding, Fallback, and Interfaces

### 3.1 WebSocket Handshake

After the WebSocket upgrade succeeds, the client MUST send `HELLO` as the first
CapConnect frame. The server MUST respond with `WELCOME` or close the session.
Server SHOULD close the transport if it does not receive a valid `HELLO` within
30 seconds of WebSocket upgrade completion. Implementations MAY use a shorter
timeout when under resource pressure.

`HELLO` payload:

```text
{
  "protocol": "capconnect",
  "version": 1,
  "encoding": "cbor",
  "max_frame_payload": u32,
  "max_pipeline_depth": u32,
  "max_total_in_flight": u32,
  "features": [string]
}
```

`WELCOME` payload:

```text
{
  "session_id": bytes,
  "version": 1,
  "encoding": "cbor",
  "max_frame_payload": u32,
  "max_pipeline_depth": u32,
  "max_total_in_flight": u32,
  "root_caps": [CapGrant],
  "features": [string]
}
```

As initial control-plane frames, `HELLO` and `WELCOME` MUST use `chain_id = 0`,
`cap_id = 0`, and `seq = 0`.

### 3.2 Structured Encoding

The fixed frame header is always binary.

Structured payloads in v1 use deterministically encoded CBOR following the core
deterministic encoding requirements in RFC 8949 section 4.2.1. Implementations
MAY expose JSON to application code, but the protocol payload MUST remain
deterministic CBOR unless a future version negotiates another encoding.

JSON exposure is non-normative convenience only.

- On-wire payload semantics are defined by CBOR data model and CDDL schema.
- `args` and `value` MUST be valid CBOR values.
- Raw opaque bytes MUST be encoded as CBOR `bstr`.
- Implementations exposing JSON MUST document mapping rules and MUST reject
  non-bijective or lossy conversions with `E_ENCODING`.

Opaque application payloads MAY be bytes inside the structured `CALL` or
`RETURN` envelope.

`CALL` payload:

```text
{
  "op": string,
  "args": any,
  "content_type": string | null
}
```

`RETURN` payload:

```text
{
  "reply_to": u64,
  "ok": bool,
  "value": any | null,
  "error": {
    "code": string,
    "message": string,
    "details": any | null
  } | null
}
```

`GRANT` payload uses `CapGrant`:

```text
CapGrant {
  "cap_id": u64,
  "chain_id": u32,
  "parent_cap_id": u64 | null,
  "owner": "client" | "server",
  "rights": u16,
  "state": "promised" | "live"
}
```

`RESOLVE` payload:

```text
{
  "promised": CapRef,
  "concrete": CapRef,
  "rights": u16 | null
}
```

`RESOLVE` binds an existing promised capability to an existing concrete
capability. It MUST NOT create a new capability entry. The promised capability's
`chain_id`, `parent_cap_id`, owner, and rights are preserved unless `rights` is
present and narrows the existing rights. If `rights` includes any right not
already present on the promised capability, receiver MUST reject with
`E_RIGHTS`.

`RESOLVE` MUST be sent on the same `chain_id` as the promised capability's
stored `chain_id` entry. A receiver MUST reject `RESOLVE` with `E_PROTOCOL` if
the frame header `chain_id` does not match the promised capability's stored
chain.

`RESOLVE` does not consume, release, revoke, or alias away the concrete
capability. The concrete capability remains usable through its original
`cap_id` until a separate `RELEASE`, `REVOKE`, or `BROKEN` transition affects
it. After `RESOLVE`, calls to the promised `cap_id` and calls to the concrete
`cap_id` are both valid if both references remain live and have `CALL` right.

`ACK` payload:

```text
{
  "ack_msg_id": u64,
  "ack_chain_id": u32,
  "ack_seq": u64
}
```

`RELEASE` payload: zero-length. The released capability is identified by frame
`cap_id`. The frame MUST use `payload_len = 0`.

`REVOKE` payload:

```text
{
  "reason"?: {
    "code": string,
    "message": string,
    "details": any | null
  } | null
}
```

The `reason` key MAY be omitted. A `REVOKE` payload encoded as an empty CBOR map
is valid and means no reason was supplied.

`CLOSE` payload:

```text
{
  "reason": {
    "code": string,
    "message": string,
    "details": any | null
  } | null
}
```

`BROKEN` payload:

```text
{
  "reason": {
    "code": string,
    "message": string,
    "details": any | null
  },
  "cascade_root": CapRef
}
```

### 3.3 HTTP Fallback

HTTP fallback exists only for environments where WebSocket is unavailable.

Fallback endpoint shape:

```text
POST /capconnect/v1/sessions
GET  /capconnect/v1/sessions/{session_id}
POST /capconnect/v1/sessions/{session_id}/frames
GET  /capconnect/v1/sessions/{session_id}/events
DELETE /capconnect/v1/sessions/{session_id}
```

Recommended media types:

```text
application/capconnect-frame
application/capconnect-stream
text/event-stream
```

Hard constraints:

- HTTP fallback MUST preserve the same CapConnect frame byte sequence after
  decoding any HTTP transport envelope.
- Binary streaming fallback SHOULD use `application/capconnect-stream` and carry
  raw CapConnect frame bytes.
- SSE fallback MUST use `text/event-stream`; each event's `data` field MUST be
  base64url without padding for exactly one CapConnect frame. The receiver MUST
  decode it to the original bytes before protocol processing.
- HTTP fallback MUST preserve per-chain ordering exactly as WebSocket transport.
- HTTP fallback MUST provide a server-to-client event stream, such as SSE or
  streaming fetch. Plain stateless request-response polling is not compatible.
- HTTP fallback MUST use session affinity or a shared ordered session log.
- HTTP fallback MUST NOT acknowledge frames before they are durably appended to
  the session order.
- HTTP fallback MUST use standard HTTP status codes. CapConnect protocol errors
  MUST remain in CapConnect frames, not custom HTTP status codes.
- If the fallback cannot deliver a required `BROKEN`, `REVOKE`, or `CLOSE`
  frame, it MUST close the session with `E_HTTP_FALLBACK_UNSAFE`.
- On any fallback-driven termination, server MUST persist terminal metadata for a
  minimum recovery window of 60 seconds and expose it through
  `GET /capconnect/v1/sessions/{session_id}`.
- Session status response MUST include at least:
  `state` (`open` or `closed`), `reason_code`, and `final_event_id` (if any).
- Client reconnecting after stream break MUST query session status before
  retrying writes; if `state=closed` and `reason_code=E_HTTP_FALLBACK_UNSAFE`,
  client MUST treat session as terminal failure, not transient network blip.
- HTTP fallback MUST NOT weaken C3.

### 3.4 Language-Neutral Interface Model

CapConnect's normative interface contract is language-neutral. Go, TypeScript,
Gleam, Rust, or other host-language APIs are projections of this model and MUST
NOT change protocol semantics.

Core abstract types:

```text
CapRef = {
  cap_id: u64
}

CapError = {
  code: string,
  message: string,
  details: any | null
}

Call = {
  msg_id: u64,
  target: CapRef,
  op: string,
  args: any,
  content_type: string | null
}

Return = {
  reply_to: u64,
  ok: bool,
  value: any | null,
  error: CapError | null
}
```

Required session operations:

```text
session.roots() -> [CapRef]
session.call(target: CapRef, op: string, args: any) -> Return
session.grant(parent: CapRef | null, rights: Rights) -> CapRef | CapError
session.resolve(promised: CapRef, concrete: CapRef) -> void | CapError
session.revoke(target: CapRef, reason: CapError) -> void | CapError
session.release(target: CapRef) -> void | CapError
session.broken(target: CapRef, reason: CapError) -> void | CapError
session.close(reason: CapError | null) -> void
```

Language binding rules:

- A binding MAY expose `call` as a future, promise, async function, actor
  message, callback, or blocking call.
- A binding MUST preserve `msg_id`, `chain_id`, `seq`, and C3 behavior even if
  the host language hides those fields from application code.
- Session runtime, not application code, assigns `chain_id` for capabilities
  created by `session.grant`. Bindings MAY expose advanced APIs for explicit
  chain placement, but default application APIs MUST let the runtime assign it.
- A binding MUST NOT map protocol errors to host-language exceptions in a way
  that loses `code`, `message`, `details`, or the affected `CapRef`.
- A binding MUST document whether cancellation sends `RELEASE`, `REVOKE`, or
  only cancels local waiting.

### 3.5 CDDL Payload Schema

CapConnect v1 uses CDDL as the language-neutral schema notation for structured
CBOR payloads. This schema is normative for payload shape; the behavioral rules
in Part 2 and the non-negotiable constraints still apply.

```text
uint32 = 0..4294967295
uint64 = 0..18446744073709551615
rights = 0..65535

cap-ref = {
  cap_id: uint64
}

cap-error = {
  code: tstr,
  message: tstr,
  details: any / nil
}

hello = {
  protocol: "capconnect",
  version: 1,
  encoding: "cbor",
  max_frame_payload: uint32,
  max_pipeline_depth: uint32,
  max_total_in_flight: uint32,
  features: [* tstr]
}

welcome = {
  session_id: bstr,
  version: 1,
  encoding: "cbor",
  max_frame_payload: uint32,
  max_pipeline_depth: uint32,
  max_total_in_flight: uint32,
  root_caps: [* cap-grant],
  features: [* tstr]
}

call = {
  op: tstr,
  args: any,
  content_type: tstr / nil
}

return = {
  reply_to: uint64,
  ok: bool,
  value: any / nil,
  error: cap-error / nil
}

cap-grant = {
  cap_id: uint64,
  chain_id: uint32,
  parent_cap_id: uint64 / nil,
  owner: "client" / "server",
  rights: rights,
  state: "promised" / "live"
}

resolve = {
  promised: cap-ref,
  concrete: cap-ref,
  rights: rights / nil
}

ack = {
  ack_msg_id: uint64,
  ack_chain_id: uint32,
  ack_seq: uint64
}

; release has no CBOR item. The frame payload_len MUST be 0.

revoke = {
  ? reason: cap-error / nil
}

close = {
  reason: cap-error / nil
}

broken = {
  reason: cap-error,
  cascade_root: cap-ref
}
```

### 3.6 Non-Normative Host API Projections

Host-language interfaces MAY be documented separately as projections of the
language-neutral model. Examples include:

- Gleam actor API for the reference server implementation.
- TypeScript promise-based API for browser and Node clients.
- Go adapter API for teams embedding CapConnect in Go services.
- Rust trait-based API for native clients or servers.

These projections are convenience APIs. They are not the protocol spec.

## Part 4: Conformance Scenarios and Test Vectors

### 4.1 End-to-End Frame Sequences

Sequence A: successful `PROMISED -> LIVE` resolve with pipelined call.

1. Client receives a promised capability via `GRANT` (`cap_id=41`, `state=promised`, `chain_id=7`).
2. Client sends `CALL` on `cap_id=41` with `seq=1` in `chain_id=7`.
3. Server buffers the call because `cap_id=41` is still `PROMISED`.
4. Server resolves existing concrete capability `cap_id=40`.
5. Server sends `RESOLVE` binding promised `cap_id=41` to concrete `cap_id=40` with `seq=2` in `chain_id=7`.
6. Server processes buffered call in chain order and sends `RETURN` with `reply_to=CALL.msg_id`.
7. Client observes `cap_id=41` state `LIVE` and receives exactly one `RETURN`.
8. `cap_id=40` remains live and independently callable unless separately
   released, revoked, or broken.

Conformance checks:

- No frame for `chain_id=7` is processed out of `seq` order.
- `RESOLVE` does not create a new capability entry for `cap_id=41`; it changes
  the existing promised entry to `LIVE`.
- `RESOLVE` does not consume `cap_id=40`.
- Buffered call is not dropped across `RESOLVE`.
- `RETURN.reply_to` matches original `CALL.msg_id`.

Sequence B: owner revoke of subtree.

1. Server owns root capability `cap_id=50` and has derived descendants `51` and `52`.
2. Client has outstanding references to `50`, `51`, and `52`.
3. Server sends exactly one `REVOKE` for `cap_id=50` in its chain.
4. Client infers recursive revocation for descendants `51` and `52` from its
   local lineage table.
5. Server marks `50`, `51`, and `52` as `REVOKED` recursively.
6. Server rejects pending calls against those caps with `E_CAP_REVOKED`.
7. Client sending new `CALL` to `51` or `52` receives `E_CAP_REVOKED`.

Conformance checks:

- Recursion is complete: every descendant moves to `REVOKED`.
- Receiver does not wait for additional descendant `REVOKE` frames.
- New calls to revoked descendants are denied.
- No descendant remains `LIVE` after ancestor revoke.

Sequence C: C3 broken promise cascade.

1. Server granted promised root `cap_id=70`; descendants `71` and `72` were derived from it.
2. Client pipelined calls to `70` (`msg_id=1001`) and `71` (`msg_id=1002`).
3. Server detects promise failure on `70` and triggers C3.
4. Server marks `70`, `71`, and `72` as `BROKEN` atomically in registry.
5. Server rejects pending/buffered calls for affected caps with `E_PROMISE_BROKEN`.
6. Server emits `BROKEN` frames for all remotely visible affected caps in chain order.
7. Any later `CALL` for `70`, `71`, or `72` is rejected with `E_PROMISE_BROKEN`.

Conformance checks:

- Cascade includes descendants, not only root.
- Pending work is rejected, not silently dropped.
- Post-cascade calls cannot succeed.
- Peer sees a consistent state after cascade completion, never partial state.

### 4.2 Binary Frame Test Vectors

Vectors below validate binary header parsing and fail-closed behavior. All
multi-byte fields use big endian.

Vector 1: `PING` with empty payload.

```text
hex:
43 43 01 0B 00 00 00 28 00 00 00 00 00 00 00 01
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01
00 00 00 00 00 00 00 64
```

Expected decode:

```text
magic=0x4343 version=1 type=PING flags=0x0000 header_len=40 payload_len=0
chain_id=1 cap_id=0 seq=1 msg_id=100
```

Expected result: accepted.

Vector 2: `CALL` with `ACK_REQUIRED` and 1-byte payload.

```text
hex:
43 43 01 03 00 01 00 28 00 00 00 01 00 00 00 02
00 00 00 00 00 00 00 10 00 00 00 00 00 00 00 02
00 00 00 00 00 00 00 65 A0
```

Expected decode:

```text
magic=0x4343 version=1 type=CALL flags=0x0001 header_len=40 payload_len=1
chain_id=2 cap_id=16 seq=2 msg_id=101 payload=a0
```

Expected result: accepted at frame parser. Payload semantic validation is
implementation-specific and occurs after frame parsing.

Vector 3: `BROKEN` with `HAS_ERROR` and 4-byte payload.

```text
hex:
43 43 01 09 00 04 00 28 00 00 00 04 00 00 00 03
00 00 00 00 00 00 00 20 00 00 00 00 00 00 00 05
00 00 00 00 00 00 02 00 DE AD BE EF
```

Expected decode:

```text
magic=0x4343 version=1 type=BROKEN flags=0x0004 header_len=40 payload_len=4
chain_id=3 cap_id=32 seq=5 msg_id=512 payload=deadbeef
```

Expected result: accepted.

Vector 4: `REVOKE` with empty reason map following same chain.

```text
hex:
43 43 01 07 00 00 00 28 00 00 00 01 00 00 00 03
00 00 00 00 00 00 00 20 00 00 00 00 00 00 00 06
00 00 00 00 00 00 02 01 A0
```

Expected decode:

```text
magic=0x4343 version=1 type=REVOKE flags=0x0000 header_len=40 payload_len=1
chain_id=3 cap_id=32 seq=6 msg_id=513 payload=a0
```

Expected result: accepted.

Vector 5: unknown flag bit set.

```text
hex:
43 43 01 0B 80 00 00 28 00 00 00 00 00 00 00 01
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01
00 00 00 00 00 00 00 64
```

Expected decode:

```text
magic=0x4343 version=1 type=PING flags=0x8000 header_len=40 payload_len=0
chain_id=1 cap_id=0 seq=1 msg_id=100
```

Expected result: rejected with `E_BAD_FRAME` because `0x8000` is undefined in
v1 frame flags.

Vector 6: reserved streaming flag set.

```text
hex:
43 43 01 0B 00 02 00 28 00 00 00 00 00 00 00 01
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01
00 00 00 00 00 00 00 64
```

Expected decode:

```text
magic=0x4343 version=1 type=PING flags=0x0002 header_len=40 payload_len=0
chain_id=1 cap_id=0 seq=1 msg_id=100
```

Expected result: rejected with `E_BAD_FRAME` because `RESERVED_STREAM` MUST be
zero in v1.

## Non-Negotiable Constraints

C1. The fixed frame header MUST remain exactly 40 bytes for v1.

C2. Capability states are monotonic. A capability that reaches `REVOKED`,
`RELEASED`, or `BROKEN` MUST NOT become `LIVE` again.

C3. Broken promise cascade is required at the protocol level. When a promised
capability breaks, the endpoint that detects the break MUST mark the cascade
root as `BROKEN`, recursively mark every descendant capability in the same
session as `BROKEN`, reject every pending or buffered call in that cascade with
`E_PROMISE_BROKEN`, and send `BROKEN` frames for every remotely visible affected
capability in chain order. A peer MUST treat future frames targeting any affected
capability as `E_PROMISE_BROKEN`. No implementation MAY treat this as only an
application-level promise rejection.

C4. `cap_id` uniqueness is mandatory per session. Reusing any previous `cap_id`
in the same session MUST fail with `E_PROTOCOL`.

C5. Derived capabilities MUST NOT amplify rights.

C6. Per-chain ordering MUST be enforced even when the transport already appears
ordered.

C7. HTTP fallback MUST preserve protocol semantics. If it cannot preserve C3 or
per-chain ordering, it MUST fail closed.

C8. Frame encoding and decoding MUST be pure data transformation. It MUST NOT
mutate session state.

C9. Unknown message types, unknown flags, invalid frame sizes, and invalid state
transitions MUST fail closed.

## Protocol Standards Validation

CapConnect v1 was validated against the following protocol standards and prior
art.

WebSocket alignment:

- RFC 6455 supports application subprotocol negotiation through
  `Sec-WebSocket-Protocol`; `capconnect.v1` is valid as a token, but public
  deployments should register it or use an origin-scoped name.
- RFC 6455 binary messages carry arbitrary application bytes. CapConnect maps
  one CapConnect frame to one reassembled WebSocket binary message.
- RFC 6455 allows WebSocket messages to be fragmented. CapConnect MUST NOT rely
  on raw WebSocket frame boundaries.
- Client-to-server WebSocket masking is transport-layer behavior. A CapConnect
  frame encoder/decoder MUST NOT mix capability state with WebSocket masking.

CBOR alignment:

- RFC 8949 avoids the older "Canonical CBOR" term and defines deterministic
  encoding requirements. CapConnect v1 therefore specifies deterministic CBOR
  rather than ambiguous canonical CBOR.
- Map ordering, shortest integer representation, and definite lengths MUST
  follow RFC 8949 deterministic rules.
- RFC 8610 CDDL is a standard notation for expressing CBOR and JSON data
  structures, so CapConnect uses CDDL for language-neutral payload schemas
  instead of making a Go, TypeScript, or Gleam binding normative.

HTTP fallback alignment:

- RFC 9110 defines HTTP method and status semantics. CapConnect fallback uses
  ordinary `POST`, `GET`, and `DELETE`; protocol-specific failures stay inside
  CapConnect frames instead of inventing HTTP status codes.
- SSE is standardized as a text event stream in the HTML Standard. Therefore SSE
  fallback carries base64url-encoded CapConnect frames, while streaming fetch may
  carry raw binary frames.

Capability and pipelining alignment:

- Cap'n Proto RPC demonstrates promise pipelining for capability-style RPC.
  CapConnect uses the same high-level idea of sending calls against unresolved
  promised results.
- C3 is a CapConnect-specific constraint: broken promise cascade is mandatory at
  the protocol level and is stricter than treating promise rejection as only an
  application callback.

Runtime implementation alignment:

- Erlang/OTP supervision trees support isolated worker processes supervised by
  restart policies, which matches the recommended one-session-per-process model.
- `gleam_otp` actors process messages sequentially, making them a good fit for
  session-local registry mutation and per-chain ordering.
- Erlang NIF guidance treats NIFs that cannot complete in roughly 1 ms as dirty
  NIFs; Zigler frame encode/decode NIFs MUST follow that guidance when payload
  size makes work exceed the scheduler budget.

References:

- RFC 6455, The WebSocket Protocol:
  https://datatracker.ietf.org/doc/html/rfc6455
- IANA WebSocket Protocol Registries:
  https://www.iana.org/assignments/websocket/websocket.xhtml
- RFC 8949, Concise Binary Object Representation, deterministic encoding:
  https://datatracker.ietf.org/doc/html/rfc8949#section-4.2.1
- RFC 8610, Concise Data Definition Language:
  https://www.rfc-editor.org/rfc/rfc8610
- RFC 9110, HTTP Semantics:
  https://datatracker.ietf.org/doc/html/rfc9110
- HTML Standard, Server-sent events:
  https://html.spec.whatwg.org/dev/server-sent-events.html
- Cap'n Proto RPC Protocol:
  https://capnproto.org/rpc.html
- Erlang/OTP Design Principles:
  https://www.erlang.org/docs/24/design_principles/users_guide
- Erlang NIF dirty scheduler guidance:
  https://erlang.org/documentation/doc-10.1/erts-10.1/doc/html/erl_nif.html
- Gleam OTP Actor documentation:
  https://hexdocs.pm/gleam_otp/gleam/otp/actor.html
- Zigler documentation:
  https://hexdocs.pm/zigler/readme.html

## Reference Implementation Direction: Gleam + Zigler

The reference implementation for this repository SHOULD use Gleam for session
semantics and Zig through Zigler for frame-level binary work.

Responsibility split:

- Gleam: session management, capability registry, message ordering, error
  propagation, broken promise cascade, ownership lifecycle, business pipeline.
- Zig via Zigler: binary frame encoder and decoder, `CapEntry` packing,
  fixed-header parsing, and zero-copy buffer utilities. WebSocket masking helpers
  MAY live in Zig only if the project implements the WebSocket transport itself;
  if a WebSocket library is used, masking SHOULD remain in that library.

Recommended structure:

```text
capconnect/
  src/
    session/
      actor.gleam
      registry.gleam
      ordering.gleam
    pipeline/
      error.gleam
      ownership.gleam
    transport/
      websocket.gleam
    frame/
      encoder.gleam
      decoder.gleam
  native/
    frame_encoder.zig
    frame_decoder.zig
    cap_entry.zig
```

Why this direction:

- One BEAM process per session gives natural crash isolation.
- A session crash does not corrupt another session.
- OTP supervision maps well to session lifecycle.
- Capability table state fits cleanly inside a `gleam_otp` actor.
- Zig is a good fit for side-effect-free binary frame packing and parsing.

Zigler NIF rule:

Frame encoding and decoding SHOULD be normal NIFs only while the work is known to
complete under the BEAM scheduler budget. If payload size or benchmark results
show encoder or decoder work taking longer than roughly 1 ms, the NIF MUST be
declared as `dirty_cpu` before production use.

## Implementation Notes for C3

The broken promise cascade is best implemented as a registry operation, not as a
handler-local error path.

Suggested operation:

```text
break_promise(registry, root_cap, reason) ->
  affected = registry.descendants_including(root_cap)
  registry.mark_all(affected, BROKEN, reason)
  ordering.reject_pending_calls(affected, E_PROMISE_BROKEN)
  transport.emit_broken_frames_in_chain_order(affected, reason)
```

The operation MUST be atomic from the session actor perspective. Other messages
for the same session MUST observe either the state before the cascade starts or
the state after the cascade completes, never a partial cascade.
