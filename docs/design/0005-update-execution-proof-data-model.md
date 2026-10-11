---
status: Draft
authors: [creese]
workstream: verification
eip_repo: frisitano/EIPs
eip_sha: 4855dbeb9a99702a8c4d948ceceb865fb3289759
consensus_specs_sha: aa16bb4c156184e9548a997d53efaf2a228a6304
ssz_specs_sha: b42af0a2265c25f5353cace185d066e8d39dbd6e
grandine_upstream_sha: 34da987dc6e1eec7be6afb6fb3c6edf317f1c32c
superseded_by:
---

# 0005 — Update the execution-proof data model and payload binding

## Context

This doc supersedes [0001](0001-represent-and-hash-execution-proofs.md).
It keeps 0001's surviving decisions, listed under *Design*, and
replaces the `ProofData` representation and the payload-binding type.

Grandine `feature/eip8025` merged the EIP-8025 proof containers in
PR #5 (`2bc81575`) and payload binding in PR #6 (`2be90a1e`), against
consensus-specs `7d6bd46a`/`7fa04483`. As merged, in
[`types/src/eip8025/`](https://github.com/eip8025-grandine/grandine/tree/7bad2ca6308a33bb379d3cc0f6000a8955f03973/types/src/eip8025):

- `ProofData` is a newtype over `ProgressiveByteList<MaxProofSize>`.
  Its root does not depend on `MAX_PROOF_SIZE`.
- `ProofType` is `u8`, so decoding accepts every value.
- `SszNewPayloadRequest<P>` is a width-4 stable container.
  `versioned_hashes` is a
  `ContiguousList<VersionedHash, P::MaxBlobCommitmentsPerBlock>`,
  which needed a `MerkleElements<VersionedHash>` bound on `Preset`.

Later merged work consumes these types. PR #7 (`981aeb4c`) signs
`ExecutionProofEnvelope`, and its tests pin an envelope root and
signing root. PR #9 (`7bad2ca6`) uses `SszNewPayloadRequest<P>`,
`ProofType` and `ProofAttributes` in the `proof_engine` traits, mock
and null engine.

Upstream sources. The pin is `master` at `aa16bb4c`, tagged
`v1.7.0-beta.4`.

| Source | Status | Change relevant here |
| --- | --- | --- |
| [#5619](https://github.com/ethereum/consensus-specs/pull/5619) (`81e15d3f`) | merged | Removes `SSZNewPayloadRequest`. Gloas [`NewPayloadRequest`](https://github.com/ethereum/consensus-specs/blob/aa16bb4c156184e9548a997d53efaf2a228a6304/specs/gloas/beacon-chain.md#newpayloadrequest) is the same width-4 `ProgressiveContainer`. `VersionedHashes` is Deneb's `List[VersionedHash, MAX_BLOB_COMMITMENTS_PER_BLOCK]`. |
| [#5593](https://github.com/ethereum/consensus-specs/pull/5593) (`04cc0780`) | merged | [`ProofData(ByteList)`](https://github.com/ethereum/consensus-specs/blob/aa16bb4c156184e9548a997d53efaf2a228a6304/specs/_features/eip8025/beacon-chain.md#new-proofdata), `LIMIT = MAX_PROOF_SIZE`. `ProofType` stays a plain `Uint8`. Gossip now REJECTs empty proof data and types outside `get_supported_proof_types()` = {1, 2, 3} before any lookup or hashing. `MAX_PROOF_SIZE` stays a constant, and `MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE` stays at 4,194,449. |
| [#5639](https://github.com/ethereum/consensus-specs/pull/5639) (`e42b2493`) | merged | `ProofEngine` is validation-only. `ProofAttributes`, `request_proofs`, `get_proof` and `prover.md` are removed. |
| [#5643](https://github.com/ethereum/consensus-specs/pull/5643) @ `b9130f1c` | open | Gloas `VersionedHashes(ProgressiveList[VersionedHash])`, `LIMIT = MAX_BLOB_COMMITMENTS_PER_BLOCK`. |
| [#5642](https://github.com/ethereum/consensus-specs/pull/5642) @ `638af25c` | open | Moves to ssz-specs `b42af0a2`, where a `ProgressiveList` may declare a `LIMIT`. The limit is checked on construction and decode and never reaches the root (#225). Also makes competing changes to `ProofData`, `MAX_PROOF_SIZE` and the envelope bound. |

Following Francesco's request, the target is the pin for `ProofData`
and `ProofType`, #5643 for `VersionedHashes`, and from #5642 only the
progressive-list limit. #5643 needs that limit. Its ref pins
`eth-ssz-specs==0.1.0`, whose `ProgressiveList` refuses a declared
`LIMIT`, so #5643 does not build on its own. The pinned `ssz_specs_sha`
is #5642's revision. `master` itself still pins `0.1.0`.

**Provisional upstream risks.** None of these blocks this design.

- **#5642 is not rebased on merged #5593.** Its ref still declares
  `ProofData(ProgressiveList[Byte])`. If that form lands, every proof
  root returns to the current progressive form, and the envelope and
  signing vectors change again.
- **If #5643 is dropped,** `versioned_hashes` reverts to a list root.
- **`versioned_hashes` differs from the pin.** `master` still declares
  `VersionedHashes` as Deneb's `List`. This design uses #5643's
  `ProgressiveList`, so the resulting `NewPayloadRequest` root differs
  from the pinned specification.
- **`MAX_PROOF_SIZE` is "not definitive".** Under `ByteList` its value
  now reaches the root.
- **EIP divergences carried from 0001,** still present at the pinned
  revisions:
  - the EIP gossips `SignedExecutionProof` rather than
    `SignedExecutionProofEnvelope`;
  - `PublicInput` carries `chain_config` in the EIP versus
    `chain_id`/`schema_id`;
  - `STATELESS_INPUT_SCHEMA_ID` is `0x0001` versus `0x1501`;
  - `MAX_PROOF_SIZE` is 400 KiB versus 4 MiB;
  - `DOMAIN_EXECUTION_PROOF` is `0x0D000000`, which Gloas assigns to
    `DOMAIN_PROPOSER_PREFERENCES`, versus `0x0F000000`.

  Grandine follows consensus-specs for the CL.

## Goals and non-goals

**Goals**

- Rename `types::eip8025::SszNewPayloadRequest<P>` to
  `NewPayloadRequest<P>` in place.
- Adopt merged #5593's `ProofData`, keep `ProofType` a plain `u8`,
  and adopt #5643's progressive `VersionedHashes` with a 4,096 limit.
- Update #7 signing and #9 `proof_engine` only as far as compilation
  and vectors require.
- Bounds are unchanged: `MAX_PROOF_SIZE` = 4,194,304 bytes (131,072
  chunks, depth 17), and the gossip pre-decode bound stays at
  4,194,449 bytes.

**Non-goals**

- No broader Gloas refactor. Other `ProgressiveList`s, including
  `blob_kzg_commitments`, keep their current cap. None of #5642's
  other LIMITs or assert removals are adopted.
- None of #5642's `ProofData` form, `MAX_PROOF_SIZE` preset, or
  envelope-bound removal.
- No gossip, verification-flow, `Store`, signing-semantics,
  ProofEngine-architecture or recursive-proof changes. #5593's gossip
  changes, including the empty-proof and supported-type REJECTs,
  belong to [0004](https://github.com/eip8025-grandine/design/pull/9).
- #5639's removal of `ProofAttributes`, `request_proofs` and
  `get_proof`. See *Open questions*.

## Design

**Unchanged from 0001.**

- Containers live in `types::eip8025`. EIP-8025 is not a `Phase`: it
  is opt-in and changes no consensus validity rule.
- The proof containers are preset-independent.
- `SignedExecutionProofEnvelope.message` is `Hc<_>`, so one
  merkleization serves both the dedup key and the signing
  `object_root`.
- Gossip MUST cap encoded input at
  `MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE` before decoding.
- Payload binding builds from `CombinedExecutionPayload` and
  `ExecutionPayloadParams::Gloas`, and the preset `const_assert`s
  remain.

**Changes**

| Item | After | Root effect |
| --- | --- | --- |
| `SszNewPayloadRequest<P>` | `NewPayloadRequest<P>`, same fields, layout and `new()` | none |
| `ProofData` | `pub type ProofData = ByteList<MaxProofSize>`; the newtype is deleted | changes for every value, including empty |
| `ProofType` | unchanged, `pub type ProofType = u8` | none |
| `versioned_hashes` | `ProgressiveList<VersionedHash, P::MaxBlobCommitmentsPerBlock>` | changes for every value |

**Rename.** Doc comments and `PayloadBindingError` messages stop
citing `SSZNewPayloadRequest`. The type states that it follows #5643
rather than `master`. No other `NewPayloadRequest` exists in Grandine.

**`ProofData`.** By reading, the alias preserves all behaviour visible
outside the crate:

- **SSZ bytes:** identical.
- **SSZ decoding:** the same `ReadError::ListTooLong { maximum:
  4194304, .. }`.
- **Serde:** the same transparent `0x` hex in human-readable formats,
  raw bytes otherwise, and the same over-limit error. Today's
  `Deserialize` already fails inside `ByteList` before its own check
  runs.
- **Constructors:** `TryFrom<Vec<u8>>` and `as_bytes()` are kept.
- **Derived traits:** `Clone`, `Eq` and an empty `Default` are kept.

Two things do differ. `Debug` loses the `ProofData { bytes: .. }`
wrapper, which affects log and test-failure text only. `ByteList`'s
`From<ContiguousList<u8, MaxProofSize>>` becomes reachable, but it
is bounded by the type.

**`ProofType`.** Merged #5593 keeps `ProofType(Uint8)`, so the alias
stays. SSZ decoding and serde accept all 256 values, as the spec type
does, and the proof containers keep `derive(Default)`. Membership of
`get_supported_proof_types()` is a gossip and verification check, not
a type invariant, so it belongs to 0004. An earlier revision of this
doc specified a validated newtype from #5593's pre-merge ref
`667e8ea9`; that form did not merge.

**Versioned hashes.** This depends on an `ssz` change that stands
without EIP-8025. `ProgressiveList<T>` gains a limit parameter that
defaults to today's cap, `ProgressiveList<T, N = PrettyBigU>`, like
`ProgressiveByteList<N>`. The backing `ContiguousList<T, N>` enforces
`N` on construction and decode, and hashing ignores it, so existing
uses are unchanged. With the parameter in place, #6's
`MerkleElements<VersionedHash>` bound on `Preset` is removed.
`new()` keeps `VersionedHashesTooLong`, because Gloas
`blob_kzg_commitments` is still capped at 2^32 in Grandine.

**Merkleization.**

- `ProofData` changes from progressive to
  `mix_in_length(merkleize(chunks, limit=131072))`.
- `versioned_hashes` changes from a depth-12 list root to
  `mix_in_length(merkleize_progressive(hashes))`.
- The `PublicInput` schema and merkleization are unchanged. For a
  given payload, though, `new_payload_request_root` changes, so the
  payload-derived `PublicInput` root and the `ExecutionProof` root
  change too.
- The `ExecutionProofEnvelope` and `SignedExecutionProofEnvelope`
  roots change through `ProofData`, and so does the dedup and
  signing `object_root`.

`container_impls.rs` must name only `BytesPerLogsBloom` and
`MaxExtraDataBytes` as root-relevant. `MaxBlobCommitmentsPerBlock`
becomes decode-only.

**Downstream.**

- **#7 signing:** `SignForSingleForkAtSlot` and `0x0F000000` are
  unchanged. Only vector values change. The tests' `PROOF_TYPE = 7`
  stays: it still decodes, and is unsupported only at gossip.
- **#9 `proof_engine`:** the rename only. Trait shapes, including
  `ProofAttributes`, are unchanged.

## Trade-offs and alternatives

**Hold `versioned_hashes` at `master`'s `List`.** This would match the
beta.1 vectors. It is rejected in order to target the expected end
state.

**Unbounded `ProgressiveList<VersionedHash>`, checked only in
`new()`.** Derived decoding would then admit up to 2^32 entries where
#5643 rejects more than 4,096.

**Validate `ProofType` at decode,** as #5593's pre-merge ref did. The
merged spec keeps `Uint8` and REJECTs unsupported types in gossip, so
a decode-time check would diverge from it. It would also make a type
assigned later undecodable rather than merely unsupported.

## Security and compatibility

**DoS.** The bounds are unchanged. Gossip still needs the 4,194,449-byte
pre-decode cap. Hashing a maximum-size proof before dedup costs the
same order as before. The 4,096 versioned-hash limit now also holds
at decode.

**Invalid or withheld proofs.** These remain a supplementary signal,
with no effect on payload validity or fork choice. A prover that
derives `new_payload_request_root` differently produces proofs that
fail verification here, and that fails safe.

**Proof types.** Decoding admits every `ProofType`, as the spec does.
Merged #5593 REJECTs unsupported types and empty proof data at the top
of gossip validation, before the block lookup and before hashing the
envelope. Until 0004 adopts that, nothing here reaches gossip. The EIP
still calls the supported set per-node configuration, while the pin
fixes it in `get_supported_proof_types()`; that too is 0004's.

**Nodes that do not opt in** are unaffected. Nothing is wired into
consensus, gossip or fork choice. `NullProofEngine` stays the default,
and no Gloas consensus type outside `types::eip8025` changes root.

## Implementation and testing

**Dependencies.**

- The progressive versioned hashes need the `ssz` limit parameter
  first.
- The rename has no root effect, so the existing vectors verify it
  unchanged.
- Everything else is confined to `types::eip8025`, plus the #7 and
  #9 test fixtures.

**Tests that change.**

- **`types/src/eip8025/tests.rs`.**
  - New vectors for `execution_proof_root_matches_reference` and
    `envelope_roots_match_reference`.
  - Encodings and error assertions re-pointed at `ByteList`.
  - `public_input_root_matches_reference` keeps its vector, because
    it uses a fixed placeholder `new_payload_request_root`. It is a
    schema check, not a payload-derived one.
- **`helper_functions/src/signing/tests.rs`.** Both constants in
  `signing_root_matches_pinned_vector`. `PROOF_TYPE` stays 7.
- **`proof_engine` tests.** Rename only.

**Tests to add.**

- Mirroring #5593's
  `test_signed_execution_proof_envelope_rejects_oversize_proof_data`:
  a maximum-size `SignedExecutionProofEnvelope` encodes to exactly
  4,194,449 bytes, and appending one byte fails decoding at the
  `ProofData` limit.
- `ProofData` serde shape unchanged.
- `ssz`: the limit is enforced, and the root is independent of `N`.
- Decoding rejects 4,097 versioned hashes.
- A pinned root for a payload-derived `NewPayloadRequest`, `PublicInput`
  and `ExecutionProof`.

**Root verification.** Verified by spec vectors, on reading Grandine's
test wiring rather than running it:

- Bounded `ByteList` merkleization: pre-Gloas `ssz_static`, through
  `transactions` and `extra_data`.
- Progressive lists of per-element-hashed values: Gloas `ssz_static`,
  through `blob_kzg_commitments`.
- Plain and progressive container composition: `ssz_static` and the
  ssz-specs progressive-container fixtures.

Not verified: any composed EIP-8025 root end to end, and whether our
declarations transcribe the spec. A reference built from the same
ssz-specs primitives by the same hand checks neither, so it is a
consistency check, not a cross-check. EIP-8025 has no `ssz_static`
vectors.

At least one implementation independent of Grandine must produce
expected roots for every changed composed type: `ExecutionProof`,
`ExecutionProofEnvelope`, `SignedExecutionProofEnvelope`, and a
payload-derived `NewPayloadRequest` and `PublicInput`. A hand-written
composition over the same ssz-specs primitives does not qualify, and
neither does recording the gap. The pinned vectors are those roots,
and the implementation must match them. Choosing the oracle and
environment is left to the implementer. None of the candidates runs in
this workbench:

- The pyspec at the pin covers the proof containers, and is the only
  one that takes declarations from the spec text. No single
  upstream ref yields `NewPayloadRequest` under #5643, which cannot
  build alone.
- The ssz-specs Python reference implementation at `b42af0a2` needs
  `pydantic`.
- The independent Lean implementation in `ssz-specs/lean` reads
  declaration-carrying vectors and needs Lean `v4.33.1`.

Whichever oracle is used, the record states which one checked each
root.

The pre-#5643 Gloas `NewPayloadRequest` `ssz_static` vectors cannot
validate the new root by design. They also do not exist in this
fork's v1.7.0-beta.0 set, and upstream ignores them. That is a known
limitation of targeting an open spec change.

## Open questions

**Deferred.**

- **#5639.** `ProofAttributes`, `request_proofs` and `get_proof` are
  gone upstream but remain in `proof_engine` and `types::eip8025`.
  Removing them is a ProofEngine-architecture change for its own doc,
  or for 0004 if it owns the engine interface.
- **Supported proof types and empty proofs.** #5593's gossip REJECTs
  and `verify_execution_proof_envelope`'s non-empty check belong to
  0004. So does reconciling the pin's fixed set {1, 2, 3} with the
  EIP's per-node configuration.
- **`eth-ssz-specs` version.** `master` pins `0.1.0`; the pinned
  `ssz_specs_sha` is #5642's `b42af0a2`. This resolves when #5642
  merges or is replaced.
