---
status: Draft
authors: [creese]
workstream: verification
eip_repo: frisitano/EIPs
eip_sha: 4855dbeb9a99702a8c4d948ceceb865fb3289759
consensus_specs_sha: c489a99077c16bad7d75053257c50633d12d314d
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
signing root. PR #9 (`7bad2ca6`) uses `SszNewPayloadRequest<P>` and
`ProofType` in the `proof_engine` traits, mock and null engine.

Upstream sources:

| Source | Status | Change relevant here |
| --- | --- | --- |
| [#5619](https://github.com/ethereum/consensus-specs/pull/5619) (`81e15d3f`) | merged | Removes `SSZNewPayloadRequest`. Gloas [`NewPayloadRequest`](https://github.com/ethereum/consensus-specs/blob/c489a99077c16bad7d75053257c50633d12d314d/specs/gloas/beacon-chain.md#newpayloadrequest) is the same width-4 `ProgressiveContainer`. `VersionedHashes` is Deneb's `List[VersionedHash, MAX_BLOB_COMMITMENTS_PER_BLOCK]`. |
| [#5593](https://github.com/ethereum/consensus-specs/pull/5593) @ `667e8ea9` | open | `ProofData(ByteList)`, `LIMIT = MAX_PROOF_SIZE`. `ProofType` admits only `ASSIGNED_VALUES`, rejecting others on construction and deserialization. `MAX_PROOF_SIZE` stays a constant, and `MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE` stays at 4,194,449. |
| [#5643](https://github.com/ethereum/consensus-specs/pull/5643) @ `b9130f1c` | open | Gloas `VersionedHashes(ProgressiveList[VersionedHash])`, `LIMIT = MAX_BLOB_COMMITMENTS_PER_BLOCK`. |
| [#5642](https://github.com/ethereum/consensus-specs/pull/5642) @ `638af25c` | open | Moves to ssz-specs `b42af0a2`, where a `ProgressiveList` may declare a `LIMIT`. The limit is checked on construction and decode and never reaches the root (#225). Also makes competing changes to `ProofData`, `MAX_PROOF_SIZE` and the envelope bound. |

Following Francesco's request, the target is #5593 for `ProofData`
and `ProofType`, #5643 for `VersionedHashes`, and from #5642 only the
progressive-list limit. #5643 needs that limit. Its ref pins
`eth-ssz-specs==0.1.0`, whose `ProgressiveList` refuses a declared
`LIMIT`, so #5643 does not build on its own. The pinned `ssz_specs_sha`
is #5642's revision. `master` itself still pins `0.1.0`.

**Provisional upstream risks.** None of these blocks this design.

- **#5593 and #5642 conflict on `ProofData`.** If #5642's form lands,
  every proof root returns to the current progressive form, and the
  envelope and signing vectors change again.
- **If #5643 is dropped,** `versioned_hashes` reverts to a list root.
- **Released vectors already disagree.** v1.7.0-beta.1 Gloas
  `NewPayloadRequest` `ssz_static` vectors use the list root, so our
  root deliberately differs from them and from `master` until #5643
  merges.
- **`ProofType` assignments are provisional.** Changing the set is a
  code change.
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
- Adopt #5593's `ProofData` and `ProofType`, and #5643's progressive
  `VersionedHashes` with a 4,096 limit.
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
  changes belong to [0004](https://github.com/eip8025-grandine/design/pull/9).

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
| `ProofType` | validated newtype over `u8` | none for admitted values |
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

**`ProofType`.** A `u8` newtype; `ASSIGNED_VALUES` lives in
`eip8025`. Every entry point checks membership:

- **`TryFrom<u8>` and `SszRead`:** via the existing
  `ReadError::Custom`, following the `BooleanInvalid` precedent, so
  `ssz` needs no change.
- **`Deserialize` and `FromStr`:** `Display` prints the decimal value,
  so the field's existing `string_or_native` and `ProofAttributes`'
  `string_or_native_sequence` still apply.

This keeps today's representations: one byte in SSZ, the `uint8` root,
a decimal string in JSON (native integers accepted), and native `u8`
in binary serde. `ProofType` has no `Default`, because 0 is
unassigned, so the proof containers drop the `derive(Default)` that
nothing uses. Only membership matters here; what each assigned value
denotes does not.

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
  unchanged. Only vector values change, and the tests'
  `PROOF_TYPE = 7` no longer decodes.
- **#9 `proof_engine`:** the rename plus `ProofType` construction in
  the fixtures. Trait shapes are unchanged.

## Trade-offs and alternatives

**Hold `versioned_hashes` at `master`'s `List`.** This would match the
beta.1 vectors. It is rejected in order to target the expected end
state.

**Unbounded `ProgressiveList<VersionedHash>`, checked only in
`new()`.** Derived decoding would then admit up to 2^32 entries where
#5643 rejects more than 4,096.

## Security and compatibility

**DoS.** The bounds are unchanged. Gossip still needs the 4,194,449-byte
pre-decode cap. Hashing a maximum-size proof before dedup costs the
same order as before. The 4,096 versioned-hash limit now also holds
at decode.

**Invalid or withheld proofs.** These remain a supplementary signal,
with no effect on payload validity or fork choice. A prover that
derives `new_payload_request_root` differently produces proofs that
fail verification here, and that fails safe.

**Fixed proof types.** The EIP calls proof types per-node
configuration, while #5593 fixes them in the type. A node rejects
envelopes for later-assigned types at decode, which counts as REJECT
for peer scoring.

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
  - `PROOF_TYPE` 7 → an assigned value.
  - New vectors for `execution_proof_root_matches_reference` and
    `envelope_roots_match_reference`.
  - Encodings and error assertions re-pointed at `ByteList`.
  - `public_input_root_matches_reference` keeps its vector, because
    it uses a fixed placeholder `new_payload_request_root`. It is a
    schema check, not a payload-derived one.
- **`helper_functions/src/signing/tests.rs`.** `PROOF_TYPE`, and both
  constants in `signing_root_matches_pinned_vector`.
- **`proof_engine` tests.** Rename and `ProofType` construction.

**Tests to add.**

- `ProofType` cases mirroring #5593's `test_proof_type.py`: 0, 4 and
  255 rejected by `TryFrom`, by SSZ decoding of an envelope whose
  proof-type byte is mutated, and by serde. JSON keeps the decimal
  string form.
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

**Required before Accepted.** At least one implementation independent
of Grandine must produce expected roots for every changed composed
type: `ExecutionProof`, `ExecutionProofEnvelope`,
`SignedExecutionProofEnvelope`, and a payload-derived
`NewPayloadRequest` and `PublicInput`. A hand-written composition
over the same ssz-specs primitives does not qualify, and neither does
recording the gap. The pinned vectors are those roots, and the
implementation must match them. Choosing the oracle and environment
is left to the implementer. None of the candidates runs in this
workbench:

- The pyspec at the #5593 ref covers the proof containers, and is the
  only one that takes declarations from the spec text. No single
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
