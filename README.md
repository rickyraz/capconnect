# CapConnect v1

CapConnect is a session-oriented capability protocol with a fixed binary frame
format and strict lifecycle semantics.

This repository is currently **docs-first**: the protocol contract and
conformance plan are the source of truth, and runtime implementations should
follow them exactly.

## Canonical Documents

- [Protocol Specification](docs/spec.md)
- [Conformance Test Runner Plan](docs/conformance.md)

## Current Status

- Spec status: final v1 working spec.
- Conformance status: initial runner plan with required tests `T1..T7`.
- Runtime status: expected to be implemented against the docs, then validated by
  conformance evidence.

## Non-Negotiable v1 Rules

- C3 broken promise cascade is mandatory.
- `cap_id` is `u64` and unique per session.
- Frame header is fixed at 40 bytes.
- Per-chain ordering by `(chain_id, seq)` is mandatory.
- `RESOLVE` must be sent on the promised capability's stored `chain_id`.
- `REVOKE` uses single-frame recursive inference for descendants.
- `RELEASE` payload is zero-length (`payload_len = 0`).
- `RESERVED_STREAM` must be `0` in v1.

## Conformance-First Workflow

Implement in this order:

1. Pass T1: frame parser/encoder vectors.
2. Pass T2: handshake behavior.
3. Pass T3/T4/T6: registry ordering, resolve, revoke, rights attenuation.
4. Pass T5: C3 broken promise cascade.
5. Pass T7: flow-control overflow behavior.

Do not claim protocol compliance without conformance results.

## Contribution Notes

- Keep `docs/spec.md` and `docs/conformance.md` in sync when protocol behavior
  changes.
- Add fixture-driven cases under `docs/fixtures/` when introducing new vectors
  or scenarios.
- Prefer fail-closed behavior for invalid frames or state transitions.
