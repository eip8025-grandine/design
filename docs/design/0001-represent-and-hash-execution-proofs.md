---
status: Draft
authors: [creese]
workstream: verification
eip_repo: frisitano/EIPs
eip_sha: 4855dbeb9a99702a8c4d948ceceb865fb3289759
consensus_specs_sha: 7d6bd46a015a7dd316c5df855bd89e57c4aa6700
grandine_upstream_sha: eaf220e60699cd63d4223ad2481e42fd15f67802
superseded_by:
---

# 0001 — Represent and hash execution proofs for payload binding

## Context

EIP-8025 adds the consensus-layer proof types — `ProofType`,
`PublicInput`, `ExecutionProof`, `ExecutionProofEnvelope`,
`SignedExecutionProofEnvelope` — defined in
[`specs/_features/eip8025/beacon-chain.md`](https://github.com/ethereum/consensus-specs/blob/7d6bd46a015a7dd316c5df855bd89e57c4aa6700/specs/_features/eip8025/beacon-chain.md).
This doc covers those types, their SSZ codec and merkleization, and
the design of the payload-binding root computation.

`hash_tree_root(ExecutionProofEnvelope)` is the gossip de-duplication
key, and also the `object_root` that the domain-separated signing root
is built from. The signing root itself is `SigningData { object_root,
domain }.hash_tree_root()`, a distinct value this milestone does not
compute. `ExecutionProof` is neither signed nor gossiped: it is the
proof-engine input a verifier assembles locally.

The envelope binds a proof to a payload by `beacon_block_root`; the
verifier derives `public_input.new_payload_request_root` from the
stored payload and `state.latest_execution_payload_bid`.

Eight things matter to interoperability:

- **Gossip object shape.** consensus-specs gossips
  `SignedExecutionProofEnvelope`; the EIP still gossips
  `SignedExecutionProof` containing `ExecutionProof`. The signed
  object and de-duplication key therefore differ.
- **`PublicInput` shape.** consensus-specs now has four fields in a
  `ProgressiveContainer`: `new_payload_request_root`,
  `successful_validation`, `chain_id`, and `schema_id`. The EIP text
  still uses `chain_config: ChainConfig`. The sources therefore
  disagree on the proof system's public input, but the consensus-specs
  shape keeps `ChainConfig` and its sub-containers out of `types`.
- **`STATELESS_INPUT_SCHEMA_ID`.** `0x1501` in consensus-specs,
  `0x0001` in the EIP. In consensus-specs it is the value of a
  `PublicInput` field; in the EIP it prefixes the serialized guest
  input.
- **`MAX_PROOF_SIZE`.** 4 MiB in consensus-specs, explicitly marked
  "not definitive"; 400 KiB in the EIP.
- **`DOMAIN_EXECUTION_PROOF` differs between sources.**
  consensus-specs uses `0x0F000000`; the EIP uses `0x0D000000`, which
  is already assigned to Gloas `DOMAIN_PROPOSER_PREFERENCES` at the
  pinned revision.
- **`ProofData` uses `ProgressiveList[Byte]`.** The type itself is
  unbounded, so `MAX_PROOF_SIZE` must be enforced separately by an
  explicit check and the gossip size cap. `PublicInput` and
  `SSZNewPayloadRequest` also require `ProgressiveContainer`. The
  pinned Grandine baseline supports neither progressive lists nor
  progressive containers.
- **No `ssz_static` vectors exist for EIP-8025.** The consensus-spec
  test generator does not currently include EIP-8025.
- **Progressive SSZ is only transitively pinned.** consensus-specs
  pins `eth-ssz-specs==0.0.1.dev2`, but the design metadata does not
  name an `ssz-specs` revision.

Where consensus-specs and the EIP disagree, this design follows
consensus-specs for the CL.

## Goals and non-goals

This milestone defines the EIP-8025 consensus-layer proof types, their
SSZ encoding and merkleization, and the payload-binding design in
Grandine.

Signing, gossip, `ProofEngine`, retention, proof generation, and
defining the required Gloas types are outside this milestone.

## Design

The containers live in `types` as a feature module alongside the
per-fork ones. EIP-8025 does **not** become a `Phase` variant: it
changes no consensus validity rule and is opt-in, so it must not enter
fork scheduling or state-transition dispatch.

**Containers.** All five types are preset-independent:
`MAX_PROOF_SIZE` is a protocol constant rather than a preset value,
and none of the container fields depends on a preset.
`SignedExecutionProofEnvelope.message` is wrapped in
`ssz::Hc<ExecutionProofEnvelope>`, as `SignedBeaconBlock` wraps its
message, so the object root is merkleized once and serves both
consumers: the dedup key, and the signing root's `object_root`. The
cache avoids repeated merkleization of the same message but cannot
help across distinct messages, which is why the pre-dedup hashing cost
below still stands.

**Progressive merkleization.** `proof_data` requires
`ProgressiveList[Byte]` and `PublicInput` requires
`ProgressiveContainer`, neither of which the pinned Grandine baseline
supports. Using a bounded `ByteList` for the proof instead would make
the proof root depend on `MAX_PROOF_SIZE`, which is still provisional.
Progressive roots are limit-independent, so the implementation needs
progressive-list and progressive-container support. A general
`ProgressiveList<T>` is also required for Gloas payload binding.

**Decode bound.** Because `ProgressiveList[Byte]` is unbounded at the
type level, Grandine must enforce proof-size limits explicitly. Gossip
MUST bound the encoded message at
`MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE` (4,194,449 bytes) before
decoding — a requirement this milestone places on the later gossip
work — and Grandine rejects `proof_data` larger than `MAX_PROOF_SIZE`
bytes.

**Payload binding.** The target is `SSZNewPayloadRequest`, a
`ProgressiveContainer` containing the Gloas `ExecutionPayload`
([`specs/gloas/beacon-chain.md`](https://github.com/ethereum/consensus-specs/blob/7d6bd46a015a7dd316c5df855bd89e57c4aa6700/specs/gloas/beacon-chain.md#L939)),
`versioned_hashes`, `parent_beacon_block_root`, and the Gloas
`ExecutionRequests`. Grandine has neither: its Gloas containers reuse
the Deneb `ExecutionPayload` and the Electra `ExecutionRequests`
(`types/src/gloas/containers.rs`), and `combined::ExecutionPayload`
stops at Deneb. Grandine's Gloas `ExecutionPayloadEnvelope` also lacks
`parent_beacon_block_root`. Payload binding depends on the
implementation baseline providing the required Gloas types.
Until those types are available, this milestone can implement the
proof types and progressive SSZ support but not payload binding.
Payload binding is preset-independent under Gloas because
`withdrawals` is progressive and the remaining preset-derived bounds —
`MAX_BLOB_COMMITMENTS_PER_BLOCK`, `BYTES_PER_LOGS_BLOOM`, and
`MAX_EXTRA_DATA_BYTES` — are identical across Mainnet and Minimal.
Compile-time assertions should guard those bounds against future
divergence.

## Security and compatibility

The proof data is attacker-controlled because
`SignedExecutionProofEnvelope` arrives over public gossip. Explicit
decode bounds prevent unbounded allocation, but
`hash_tree_root(proof_envelope)` must still be computed before
de-duplication, so later gossip handling must account for the cost of
hashing a maximum-sized proof.

Execution proofs remain an optional, supplementary validity signal.
Invalid, missing, or withheld proofs do not affect payload validity or
fork choice, and nodes that do not opt in are unaffected. EIP-8025
introduces no consensus rule change or fork; compatibility therefore
depends on staying aligned with the consensus-spec baseline described
in *Context*.

## Open questions

- Which `PublicInput` shape is normative? consensus-specs uses
  `chain_id` and `schema_id`; the EIP uses `chain_config`.
- Are consensus-specs `schema_id` and the EIP's
  `STATELESS_INPUT_SCHEMA_ID` intended to identify the same schema?
- Which payload-binding request shape is normative? consensus-specs
  and the EIP currently produce different payload-binding roots, so
  Grandine cannot both follow consensus-specs and match the current
  prover. This requires upstream reconciliation.
- Which `DOMAIN_EXECUTION_PROOF` value is correct: `0x0F000000` or
  `0x0D000000`? The latter collides with the pinned Gloas domain
  assignment and needs upstream resolution.
- Should the design pin an `ssz-specs` revision directly?
- How are Grandine's SSZ roots to be cross-checked? No `ssz_static`
  vectors exist for EIP-8025.
