---
status: Draft
authors: [creese]
workstream: verification
eip_repo: frisitano/EIPs
eip_sha: 4855dbeb9a99702a8c4d948ceceb865fb3289759
consensus_specs_sha: a08d8a6e2b45f0b8c0d379abc15583427c643689
grandine_upstream_sha: eaf220e60699cd63d4223ad2481e42fd15f67802
superseded_by:
---

# 0001 — Represent and hash execution proofs for payload binding

## Context

EIP-8025 adds four consensus-layer SSZ types — `ProofType`,
`PublicInput`, `ExecutionProof`, `SignedExecutionProof` — defined in
[`specs/_features/eip8025/beacon-chain.md`](https://github.com/ethereum/consensus-specs/blob/a08d8a6e2b45f0b8c0d379abc15583427c643689/specs/_features/eip8025/beacon-chain.md).
This doc covers those types, their SSZ codec and merkleization, and
the computation of the payload-binding root.

`hash_tree_root(ExecutionProof)` is the gossip de-duplication key, and
also the `object_root` that the domain-separated signing root is built
from. The signing root itself is `SigningData { object_root, domain
}.hash_tree_root()`, a distinct value this milestone does not compute.
`public_input.new_payload_request_root` is the only link between a
proof and the payload it certifies. Grandine already holds all four
`NewPayloadRequest` fields at the `notify_new_payload` boundary, as
`ExecutionPayload<P>` plus `ExecutionPayloadParams<P>`
(`types/src/combined.rs`).

Five things in the pinned baseline are provisional and matter to
interoperability:

- **`PublicInput` shape.** consensus-specs has one field,
  `new_payload_request_root`; the EIP text adds a
  `successful_validation` boolean and a `chain_config: ChainConfig`.
  Every `ExecutionProof` root and every signature over one differs
  between the two, and the EIP shape would pull `ChainConfig` and its
  sub-containers into `types`.
- **`MAX_PROOF_SIZE`.** 4 MiB in consensus-specs, explicitly marked
  "not definitive"; 400 KiB in the EIP.
- **`DOMAIN_EXECUTION_PROOF` differs between sources.**
  consensus-specs uses `0x0F000000`; the EIP uses `0x0D000000`, which
  is already assigned to Gloas `DOMAIN_PROPOSER_PREFERENCES` at the
  pinned revision.
- **`ProofData` uses `ProgressiveByteList`.** The type itself is
  unbounded, so `MAX_PROOF_SIZE` must be enforced separately by an
  explicit check and the gossip size cap. Grandine does not yet
  support progressive lists.
- **No `ssz_static` vectors exist for EIP-8025.** The consensus-spec
  test generator does not currently include EIP-8025, so Grandine must
  cross-check its SSZ roots against the pyspec at the pinned revision
  instead of generated static fixtures.

Where consensus-specs and the EIP disagree, this design follows
consensus-specs for the CL.

## Goals and non-goals

This milestone defines the EIP-8025 consensus-layer proof types, their
SSZ encoding and merkleization, and payload binding in Grandine.

Signing, gossip, `ProofEngine`, retention, and proof generation are
outside this milestone.

## Design

The containers live in `types` as a feature module alongside the
per-fork ones. EIP-8025 does **not** become a `Phase` variant: it
changes no consensus validity rule and is opt-in, so it must not enter
fork scheduling or state-transition dispatch.

**Containers.** All four types are preset-independent:
`MAX_PROOF_SIZE` is a protocol constant rather than a preset value,
and none of the container fields depends on a preset.
`SignedExecutionProof.message` is wrapped in
`ssz::Hc<ExecutionProof>`, as `SignedBeaconBlock` wraps its message,
so the object root is merkleized once and serves both consumers: the
dedup key, and the signing root's `object_root`. The cache avoids
repeated merkleization of the same message but cannot help across
distinct messages, which is why the pre-dedup hashing cost below still
stands.

**Progressive merkleization.** `proof_data` requires
`ProgressiveByteList`, which Grandine's `ssz` crate does not yet
support. Using a bounded `ByteList` instead would make the proof root
depend on `MAX_PROOF_SIZE`, which is still provisional. Progressive
roots are limit-independent, so Grandine needs progressive
merkleization support. Only `ProgressiveByteList` is in scope: it does
not close Gloas's separate `BlobKZGCommitments` divergence
(`specs/gloas/beacon-chain.md:238`), so a general `ProgressiveList<T>`
remains deferred.

**Decode bound.** Because `ProgressiveByteList` is unbounded at the
type level, Grandine must enforce proof-size limits explicitly. Gossip
MUST bound the encoded message at `MAX_SIGNED_EXECUTION_PROOF_SIZE`
(4,194,449 bytes) before decoding — a requirement this milestone
places on the later gossip work — and `ExecutionProof` decoding
rejects `proof_data` larger than `MAX_PROOF_SIZE`.

**Payload binding.** Grandine reconstructs the spec's
`NewPayloadRequest` from the existing `(ExecutionPayload<P>,
ExecutionPayloadParams<P>)` pair and computes
`new_payload_request_root` as its `hash_tree_root`. The spec uses
fixed bounds while Grandine reuses preset-derived types; those bounds
match on Mainnet but not Minimal, where only
`MaxWithdrawalsPerPayload` diverges (4 against 16), enough on its own
to change the root. Binding is therefore Mainnet-only. Reusing the
existing types avoids duplicating the payload type tree, with
compile-time assertions guarding against future bound divergence.

Grandine does not yet support EIP-7928's `block_access_list`, which is
part of `SszExecutionPayload`. Until that support lands,
`new_payload_request_root` cannot match a prover's root, so payload
binding remains provisional.

## Security and compatibility

The proof data is attacker-controlled because `SignedExecutionProof`
arrives over public gossip. Explicit decode bounds prevent unbounded
allocation, but `hash_tree_root(proof)` must still be computed before
de-duplication, so later gossip handling must account for the cost of
hashing a maximum-sized proof.

Execution proofs remain an optional, supplementary validity signal.
Invalid, missing, or withheld proofs do not affect payload validity or
fork choice, and nodes that do not opt in are unaffected. EIP-8025
introduces no consensus rule change or fork; compatibility therefore
depends on staying aligned with the consensus-spec baseline described
in *Context*.

## Open questions

- Which `PublicInput` shape will be normative, and does the CL need to
  model `ChainConfig`? This must be settled before Accepted.
- Which `DOMAIN_EXECUTION_PROOF` value is correct: `0x0F000000` or
  `0x0D000000`? The latter collides with the pinned Gloas domain
  assignment and needs upstream resolution.
- Is the EL SSZ schema intentionally preset-independent? If so,
  reusing Grandine's preset-derived payload types makes payload
  binding Mainnet-only.
