---
status: Draft
authors: [creese]
workstream: verification
eip_repo: frisitano/EIPs
eip_sha: 4855dbeb9a99702a8c4d948ceceb865fb3289759
consensus_specs_sha: d9663f85be248c4fd0ae95806da6070e17c2442b
ssz_specs_sha: b42af0a2265c25f5353cace185d066e8d39dbd6e
grandine_upstream_sha: 34da987dc6e1eec7be6afb6fb3c6edf317f1c32c
superseded_by:
---

# 0007 — Verify execution proofs in the proof engine

## Context

The milestone is shown through direct
`ProofEngine::verify_execution_proof` calls. Grandine must verify a
zkboost-compatible mock proof, and cryptographically verify a genuine
execution proof under a trusted verifying key (VK), with both paths
enforcing the expected `PublicInput`. It also includes a
zkboost-compatible `NewPayloadRequest` root. Direct tests can supply
any `PublicInput`, but one that Grandine builds from a payload matches
no zkboost proof without that root.

**Grandine.**

- **Merged.** PR #19 merged
  [0006](0006-make-proof-engine-validation-only.md) into
  `feature/eip8025`, which is now at `2b71f2dc`. The branch has the
  object-safe trait `ProofEngine { is_null,
  verify_execution_proof(ExecutionProof) -> bool }` and two
  implementations. Neither inspects the proof:
  - `NullProofEngine` is the opt-out;
  - `MockProofEngine::new(bool)` is a test double that returns the
    `bool` it was given.

  Nothing on the branch builds a `PublicInput` from a payload yet.
- **Pending.** Open PR #17 (`a5f27d52`) adds that construction, in
  `Store::get_execution_proof` (`fork_choice_store/src/store.rs`). It
  sets `new_payload_request_root` to
  `NewPayloadRequest::new(..)?.hash_tree_root()`, which is Grandine's
  canonical root. PR 1 below changes that value.

**Upstream.**

- **Guests.**
  [ere-guests](https://github.com/eth-act/ere-guests/blob/a5af6f69ff4fe9410664e73345033fdeefe12d4d/crates/stateless-validator-common/src/guest/output.rs)
  guests commit the 43-byte SSZ serialization of `PublicInput`.
- **ere.**
  [ere](https://github.com/eth-act/ere/tree/c0c411ae9215e0f72a00d52e32e267b30ef48606/crates/verifier)
  v0.19.2 has one crate per zkVM, each implementing
  `zkVMVerifier::verify(&Proof) -> PublicValues` against one VK.
- **zkboost.**
  [zkboost](https://github.com/eth-act/zkboost/blob/b83a23f91809dd4674327c194279229eb2a37e8d/crates/types/src/execution_proof.rs)
  v0.12.0 sends ere-encoded proofs. In mock mode it sends `MockProof {
  public_values: List[byte], proof: List[byte] }`, which
  [ethereum-package
  #1492](https://github.com/ethpandaops/ethereum-package/pull/1492)
  also uses.
- **Other CLs.**
  [Lighthouse](https://github.com/eth-act/lighthouse/tree/274af519780614b622ff8af7f180dbc1cf0f22c1/beacon_node/proof_engine)
  and [Prysm #17690](https://github.com/OffchainLabs/prysm/pull/17690)
  (`ad0d0929`, open) use ere's prebuilt C library.

**Where the sources disagree.** There are two separate
interoperability issues, and a proof must clear both:

1. **Request root.** Which root goes into
   `PublicInput.new_payload_request_root`. This depends on how the
   `NewPayloadRequest` container is hashed. It is fixed on the caller
   side, in PR 1.
2. **Binding.** What the proof's committed public values are compared
   against: `hash_tree_root(public_input)` or the serialized
   `PublicInput`. This is decided inside the engine, in PR 2.

The first two data columns of the table concern the request root, and
the last column concerns the binding. The inner payload and requests
are progressive containers in every source, so the request roots
differ only in the outer container and in `versioned_hashes`.

| Source | Outer container | `versioned_hashes` | Proof binds |
| --- | --- | --- | --- |
| consensus-specs `d9663f85` | Progressive | `List[_, 4096]` | `hash_tree_root(public_input)` |
| Grandine `2b71f2dc` (open #5643) | Progressive | Progressive | not implemented |
| [execution-specs `tests-zkevm@v21.0.6`](https://github.com/ethereum/execution-specs/blob/59c1473ddfba341babaeca1c97a50e996f639a0b/src/ethereum/forks/amsterdam/execution_engine/types.py), ere-guests, zkboost, Prysm | Ordinary | Progressive | 43 serialized bytes |

The three shapes serialize identically, but their roots differ. On
devnet-8 block 93354 they are:

- `ae5c1a4a…` (spec);
- `b3adcfb6…` (Grandine);
- `8c3a8902…a481c6` (zkboost).

The EIP's guest also returns serialized output. Following interim
guidance from TingHan Jian and TaoLepton, PR 1 targets the zkboost
root and PR 2 targets the serialized binding. Both are labelled
deviations from consensus-specs.

**Provisional upstream, and what breaks for us.**

- **Binding and root** may be revised. A move to binding the
  `hash_tree_root` changes PR 2's binding and every fixture and VK. If
  consensus-specs adopts the zkboost shape, PR 1's method becomes the
  canonical root.
- **#5643** (open) provides the progressive field root. PR 1 needs
  that root whether or not #5643 merges.
- **[#5729](https://github.com/ethereum/consensus-specs/pull/5729)**
  (open, `40a3a0f1`) swaps `new_payload_request_root` for
  `beacon_block_root`. Its new guest checks the payload envelope
  against a partial `BeaconState`, so it brings new VKs and new
  fixtures. PR 1's root would leave `PublicInput`. The engine is
  unaffected, because the layout stays 43 bytes and the engine never
  reads the root.
- **Proof types** are unassigned. zkboost and Lighthouse use 1–3 for
  ethrex and 4–6 for reth, each on OpenVM, SP1 and ZisK.
- **EIP divergence.** The EIP's 400 KiB `MAX_PROOF_SIZE` differs from
  consensus-specs, which Grandine follows. `MockProof` has no spec
  status.

## Goals and non-goals

**Goals**

- A zkboost-compatible request root, used wherever Grandine builds
  `PublicInput`.
- A `ProofEngine` that verifies genuine SP1 proofs in process and
  verifies zkboost mock proofs, binding the exact `PublicInput` on
  both paths.
- Default off: default features compile no zkVM code, mainnet never
  accepts mock proofs, and real verification never falls back to mock.
- `proof_data` stays at most 4 MiB.

**Non-goals**

- Gossip, Req/Resp, the processing pipeline and proof generation.
- JSON configuration, CLI options and runtime construction.
- Sepolia proofs and Sepolia's prover configuration.
- OpenVM, ZisK and LambdaVM backends, and built-in VKs.
- Changing the 0006 trait, or the canonical `NewPayloadRequest` that
  [0005](0005-update-execution-proof-data-model.md) owns.
- #5729.
- Latency targets.

## Design

The figure follows one proof from payload to verdict. Step numbers
match the verification contract below.

```
  PR 1 · types              caller: Store::get_execution_proof (#17)
  ──────────────────────────────────────────────────────────────────
  NewPayloadRequest<P>          (canonical progressive type, unchanged)
        │ zkboost_compatible_root()
        ▼
  PublicInput { root, true, chain_id, schema_id }
        │
        ▼
  ExecutionProof { proof_data, proof_type, public_input }
        │ verify_execution_proof()  through Arc<dyn ProofEngine>
        ▼
  PR 2 · proof_engine
  ──────────────────────────────────────────────────────────────────
  ZkvmProofEngine::check()
    1–3  type configured? validation true? mock not on chain 1?
        │
        │ backend for proof_type (exactly one, fixed at construction)
        ├──────────────────────────────┐
        ▼                              ▼
  EreBackend<SP1Verifier>        ZkboostMockBackend
    4  decode SP1Proof             4  decode MockProof
    5  verify against VK              (no cryptography)
        │                              │
        └──────────────┬───────────────┘
                       ▼ public values
    6  equal to ssz_serialize(public_input)?
                       │
                       ▼
            Ok | Rejection  ──▶  bool
```

Without a constructed engine, callers keep `NullProofEngine`.

**Request root (PR 1).** `NewPayloadRequest<P>` stays the spec's
progressive container. PR 1 adds
`NewPayloadRequest::zkboost_compatible_root(&self) -> H256` in
`types/src/eip8025/container_impls.rs`. It merkleizes the four field
roots as an ordinary depth-2 container: `hash(hash(payload,
versioned_hashes), hash(parent_root, requests))`.

This gives zkboost's root because a container's root is computed in
two stages. First each field is hashed to its own root. Then the four
field roots are combined. Grandine and zkboost agree on every field
root:

- `execution_payload` and `execution_requests` are progressive
  containers with the same fields in both, so their roots match.
- `versioned_hashes` is a progressive list in both. Grandine's carries
  a limit, but the limit affects only decoding, not the root.
- `parent_beacon_block_root` is a `Bytes32`, which is its own root.

The two differ only in the second stage. Grandine's progressive
container merkleizes the field roots progressively and mixes in the
active-field bitmap. zkboost's ordinary container pads the four roots
to a power of two, here exactly four, and hashes them pairwise, with
no mix-in. The method performs that ordinary combination over field
roots Grandine already computes. It does not touch the canonical type,
its SSZ encoding, or its `hash_tree_root`. PR 1's fixture test checks
the result against an independently computed root.

The method's doc comment names the deviation from consensus-specs. It
replaces `hash_tree_root()` in #17's `get_execution_proof`, the only
place that builds a request root for `PublicInput`.

**Construction (PR 2).** The engine is built in Rust code. There is
no configuration file or CLI flag yet. The caller passes two things:

- **The chain configuration** (`&Config`). The engine reads only
  `deposit_chain_id`, which it uses to refuse mock backends on mainnet.
- **A list of `(ProofType, BackendSpec)` pairs.** Each pair says how
  proofs of one type are verified. The list becomes the engine's fixed
  map from type to backend. A proof whose type is not in the map is
  rejected (step 1 of the verification contract).

```rust
impl ZkvmProofEngine {
    pub fn new(
        config: &Config,
        backends: impl IntoIterator<Item = (ProofType, BackendSpec)>,
    ) -> Result<Self, ConstructionError> {
        // ...
    }
}
```

`BackendSpec` names the verifier for one proof type:

| Variant | What it does | Availability |
| --- | --- | --- |
| `Sp1 { program_vk: [u8; 32] }` | Real verification: decodes an ere SP1 proof and checks it cryptographically against this guest's VK | Only with the `sp1` Cargo feature |
| `ZkboostMock` | Mock verification: decodes zkboost's `MockProof` container and checks only its public values. No cryptography | Always compiled, but refused on mainnet |

Take a devnet that proves ethrex on SP1 and mocks reth, using
zkboost's provisional numbering (2 is ethrex on SP1, 5 is reth on
SP1). It would pass:

`[(2, Sp1 { program_vk: ETHREX_VK }), (5, ZkboostMock)]`

Any other type, such as 1, would then be rejected as unsupported.

`new` checks the whole list before it returns an engine. It returns a
`ConstructionError` when:

- **the list is empty.** An engine that accepts nothing should be
  `NullProofEngine` instead.
- **a proof type appears twice.** Each type has exactly one backend.
- **an SP1 VK does not decode.**
- **any entry is `ZkboostMock` and `deposit_chain_id == 1`**, meaning
  mainnet. There is no override.

When construction succeeds, the map cannot change, and the engine logs
a `warn!` for each mock type, so a node with mock types enabled shows
it in its logs. Each proof type maps to exactly one backend. A real
proof that fails verification is rejected.

**Verification contract.** `ZkvmProofEngine::check(&self,
&ExecutionProof) -> Result<(), Rejection>` is public, and the trait
method returns `check(..).is_ok()`. Steps run cheapest first:

| # | Step | Rejection |
| --- | --- | --- |
| 1 | `proof_type` is configured | `UnsupportedProofType` |
| 2 | `public_input.successful_validation` | `UnsuccessfulValidation` |
| 3 | Mock only: `public_input.chain_id != 1` | `MockOnMainnet` |
| 4 | Decode `proof_data` with the backend codec | `Decode` |
| 5 | Real only: verify against the VK, including ere's exit-code check | `Verify` |
| 6 | Public values match `ssz_serialize(public_input)` under the backend's binding | `PublicValuesMismatch` |
| — | Panic in steps 4–5 (under `catch_unwind`) | `Panic` |

The engine does not derive `public_input`. It reads only
`successful_validation` and, for mock, `chain_id`, so it is coupled
neither to `new_payload_request_root` nor to #5729.

**Backends.** A private trait, `Backend { public_values(&[u8]);
binding() }`, sits behind the map.

- **Real backends** share one `EreBackend<V:
  ere_verifier_core::zkVMVerifier>`, which decodes with
  `V::Proof::decode_from_slice` and calls `V::verify`. A new zkVM
  needs a Cargo feature, a `BackendSpec` variant and a binding rule.
- **SP1.** ere's
  [`SP1Proof`](https://github.com/eth-act/ere/blob/c0c411ae9215e0f72a00d52e32e267b30ef48606/crates/verifier/sp1/src/proof.rs)
  codec is legacy bincode. It decodes any SP1 proof kind, rejects
  trailing bytes, and bounds allocations (ere #423). Verification
  accepts only compressed proofs, so any other kind gives `Verify`,
  not `Decode`. The verifier checks the embedded public values against
  the proof digest. The exit-code check needs ere ≥ v0.19.1 (#446).
  Binding is exact.
- **Mock.** Two `ByteList<MaxProofSize>` fields with standard offsets;
  `proof` is ignored. It must decode zkboost's vector `08000000
  0b000000 aaaaaa bbbb`. Binding is exact, because zkboost commits the
  unpadded 43 bytes. Standard offsets give `proof` every byte after
  `public_values`, so appended bytes extend `proof` rather than
  failing decode. Malformed offsets give `Decode`.
- **OpenVM and ZisK (future)** zero-pad public values. They will need
  the zero-padding binding of Prysm's `publicValuesMatch`.

**Trusted VKs.** The VK is the trust root. For SP1 it is the `bytes32`
digest of one guest ELF, as `cargo prove vkey` prints it. VKs come
only from the caller.

**State and failure.** There is no per-proof state. ere keeps one SP1
compressed-proof verifier per process, in a lazily initialized static
that every SP1 backend shares. `SP1Verifier::new` initializes it
(`LazyLock::force`), and the engine calls that while it is being
constructed. The one-time setup cost therefore falls on
`ZkvmProofEngine::new`, not on the first proof. Later SP1 backends and
every verification reuse the same read-only verifier. A rejection
returns `false` and logs the type and `Rejection` at `debug!`, never
the proof bytes.

**`MockProofEngine::new(bool)`** stays as the stub for caller tests
where proof contents do not matter, such as #11's. Its doc comment
will say it is not zkboost mock verification.

**Dependencies.**

- Behind the non-default `proof_engine/sp1` feature:
  `ere-verifier-sp1` and `ere-verifier-core` at git tag `v0.19.2`,
  pulling SP1 `v6.6.0`.
- Always: the workspace `ssz` crate.

## Trade-offs and alternatives

**Request root.**

| Alternative | Rejected because |
| --- | --- |
| Make the canonical `NewPayloadRequest` ordinary | Diverges from consensus-specs, breaks Gloas `ssz_static` conformance and moves 0005's model |
| Separate `ZkboostNewPayloadRequest<P>` deriving `Ssz` | Duplicates a layout that must track Gloas, and clones the payload |
| A Cargo or configuration switch | A hidden global mode, when tests need both shapes |

**One engine with per-type backends, or separate `ProofEngine`
implementations.** Separate SP1 and mock engines would each repeat
steps 1–3, step 6 and the panic containment. Mixing real and mock
types would still need a dispatcher over them, which is this engine.
The easiest way to compose two engines is the forbidden "mock, else
real" chain.

**ere's C library, as Lighthouse and Prysm use.** It isolates
dependencies (ere #431 localizes its symbols), builds fast and is
proven on these fixtures. It stays the fallback if native integration
proves impractical. It is rejected here because:

- a panic across `extern "C"` aborts, and ere has shipped an
  input-triggered abort (#423);
- `build.rs` downloads a prebuilt binary, which hurts reproducible
  builds;
- it lacks the windows-x64 and macos-x64 targets that Grandine
  releases;
- Grandine would have to maintain unsafe FFI.

**Why SP1 first.** Fixture fit decides it; the dependency spike shows
only that SP1 links.

- **SP1.** zkboost's 6.4.0 fixtures are one minor version behind ere's
  6.6.0, and ere's own SP1 fixture survived that bump. Its 1.27 MB
  proofs exceed the EIP's 400 KiB but fit the spec's 4 MiB.
- **ZisK.** zkboost's fixtures are v1.1.0-alpha against ere's 1.3.1,
  and ere regenerated its own ZisK fixture four times between v0.17
  and v0.19.
- **OpenVM.** ere's verifier comes from a personal fork branch.

**Other options.**

| Alternative | Rejected because |
| --- | --- |
| Umbrella `ere-verifier` crate | Compiles all four backends, with no features to trim |
| `sp1-verifier` from crates.io | Re-implements ere's codec, which is zkboost's wire format |
| External verifier service | None is maintained, consensus-specs #5639 moved verification into the CL, and it adds an availability dependency |
| Sniff every proof for `MockProof` (zkboost's mock beacon node, Prysm) | Makes every real type forgeable once mock is on |
| Prysm's stand-in real verification per mock proof | Simulates devnet cost. Deferred |
| Built-in VKs | Every VK changed after ere-guests v0.17.0. Deferred |
| Follow consensus-specs: spec root, `hash_tree_root` binding | No zkboost proof would verify |

## Security and compatibility

**Adversarial model.** `proof_data` and `proof_type` are attacker
input: from direct callers now, from gossip later. `public_input` is
local. VKs and backend choices are trusted caller input. Soundness
rests on SP1, a correct VK and a correct guest. The engine adds the
binding and steps 1–3.

**DoS and resource use.**

- `ProofData` caps input at 4 MiB, and ere bounds decoder allocations.
- Steps 1–3 reject before any decoding or cryptography.
- A panic becomes `false`. An abort on allocation failure cannot be
  contained; bounded decoding mitigates it.
- A call costs at most one SP1 verification. That cost, and contention
  between SP1's rayon pool and Grandine's, are unmeasured.
- There is no caching; deduplication belongs in front of the engine.

**Invalid or withheld proofs.** An invalid proof gets `false` and
nothing else. Proofs are non-consensus artifacts, with no effect on
state, payload status or fork choice. A withheld proof means no call.

**Mock proofs** are accepted only when all of these hold:

- the type was constructed as `ZkboostMock`;
- `deposit_chain_id != 1`, which covers mainnet and its shadow forks;
- `public_input.chain_id != 1`;
- the public input matches exactly.

A `MockProof` sent under an SP1 type fails the SP1 codec.

**Nodes that do not opt in.** Default builds contain no zkVM code, so
the Windows, macOS and i686 builds are unchanged. Nothing constructs
`ZkvmProofEngine` at runtime.

**Compatibility.** `PublicInput` never goes on the wire; the envelope
carries `beacon_block_root`. PR 1 changes which proofs verify locally,
but no wire format, signing domain or canonical root.

## Implementation and testing

There are two stacked PRs against `feature/eip8025` (`2b71f2dc`).

PR 1's method needs one caller change. #17's `get_execution_proof`
must call `zkboost_compatible_root()` instead of `hash_tree_root()`.
That function does not exist on `feature/eip8025` yet, so who makes
the change depends on landing order:

- **#17 merges first.** PR 1 makes the one-line change. It also
  updates #17's `get_execution_proof_binds_the_payload_request_root`
  test to expect the compatible root.
- **PR 1 merges first.** #17 makes the same change when it rebases
  onto PR 1.

PR 1 remains independently testable through its `types` tests. If #17
has already merged, PR 1 also updates and runs the caller test.

**PR 1 — Request-root compatibility.** All tests are in `types`.

**Fixture.**

- Source: zkboost `b83a23f9`,
  `crates/server/tests/fixture/stateless_input_amsterdam.ssz`, blob
  `5c0a53f9`.
- Extraction: after the 2-byte big-endian schema ID `0x1501`, the
  body's first two offsets (20 and 125,425) delimit the request.
- Checked in: those 125,405 bytes (sha256
  `e405a037fbc4878d4378025bdeb592678c3186c9d4c4bf6e8a0044ce11321e0a`).
  They are devnet-8 block 93354, chain 7091047534, with six versioned
  hashes, and are identical to Prysm's
  `zkboost_new_payload_request.ssz`.

**Reference root.**
`0x8c3a890206a189727e151767653f846ccddbd269eb29fb0a2f97371f23a481c6`.

- Prysm's Go implementation pins it.
- The ssz-specs Python oracle at `b42af0a2` reproduced it on
  2026-10-20.
- PR 1 re-runs the oracle in a build task under `oracle/setup.sh`:
  declare an ordinary `Container` of pyspec Gloas `ExecutionPayload`,
  `ProgressiveList[VersionedHash]`, `Root` and `ExecutionRequests`
  (unchanged from `aa16bb4c` to `d9663f85`), then decode the bytes.
  Record the command and full root in the test comment.

**Tests.**

- The fixture decodes under `Mainnet` and `Minimal` and re-encodes
  identically.
- Under both presets, `zkboost_compatible_root()` equals the
  reference, and `hash_tree_root()` differs from it.
- With empty versioned hashes, the method equals the depth-2
  composition of the field roots.

**PR 2 — Execution-proof verification.** This PR adds the engine,
backends, the `sp1` feature, fixtures and tests, plus a CI step with
`proof_engine/sp1`.

**Fixtures.** From zkboost `b83a23f9`,
`crates/mock-beacon-node/tests/fixture/`. The blobs equal Lighthouse's
copies. Each VK matches its guest's GitHub release, ere-guests
v0.17.0's `vk_sha256` and Lighthouse's hex.

| Artifact | Role | sha256 |
| --- | --- | --- |
| `stateless-validator-ethrex-sp1-v6.4.0.proof` | Primary candidate | `bc57e1a62930472a8a45d0b0490105deb3c10e4c9eedeca0f226a6409c4e8237` |
| `public_values.bin`: root `68e22507…235e7bb7`, `true`, chain 1, `0x1501` | Checked in | `2bb490a0d717c8b53c9001b112a523bbaf58348f27a9d75d979e5d3aa66ae0e8` |
| ethrex v26.0.0 VK `00662ca6c9db4ecb22d9ac4c320612caeac9c320f92e3ab019af4f544e35c88b` | Constant | `5e614b666d4f9a60a68d826833b94e2c5afa029b2dc501a45f3656059811a4d7` |
| reth v0.1.0-rc.3 VK `00a03cbfa95559cfee3b45ef925f3f7a631181e35e774e92b040277d893511dd` | Constant | `420933f79a82fcb40880a97f0efb7780e6f939cbf6a0bbec716de24b8033df28` |
| `stateless-validator-reth-sp1-v6.4.0.proof` | Fallback candidate | `e7e4b6ab19665c560b8274764b866e395904dbf430adea4c2dc2b7c5874bbbdf` |

Each proof is 1,272,611 bytes. Both embed the same 43 bytes as
`public_values.bin` at byte 1,272,554, after an 8-byte length prefix.
The tamper test edits them there.

No fixture carries the request behind `68e22507…`. So real
verification uses `public_values.bin`, and only the mock path
exercises PR 1's root.

**Acceptance.** PR 2 needs one *primary fixture* to verify under ere
v0.19.2. A primary fixture is a genuine zkboost execution proof, its
committed public values and its guest's VK. Candidates are tried in
order, and the first that verifies is checked in:

1. the ethrex proof;
2. the reth proof, with the same `public_values.bin`;
3. another genuine SP1 execution-proof fixture, with its provenance
   and checksums recorded.

A generic SDK proof never qualifies. If no SP1 fixture verifies, stop
and revise this design.

The *other VK* is any different guest's VK: reth's for candidate 1,
and ethrex's otherwise.

Grandine reviewers accept the in-tree size, about 1.4 MB, in PR 2.

**Tests.** *Verification cases* build an engine and hold it as
`Arc<ZkvmProofEngine>`. Each asserts the `Rejection` (or `Ok`) from
`check`. It also asserts the matching `bool` from the same engine,
upcast to `Arc<dyn ProofEngine>`. Real cases use `Config::mainnet()`,
which shows the mainnet guard does not affect real verification. Mock
cases use a non-mainnet `Config`.

| Case | Expect |
| --- | --- |
| Primary proof, its VK, its committed public values as input | `Ok` |
| Primary proof under the other VK | `Verify` |
| One embedded public-value byte flipped, input flipped to match | `Verify` |
| Truncated, empty or garbage `proof_data` | `Decode` |
| Root, `chain_id` or `schema_id` changed | `PublicValuesMismatch` |
| `successful_validation = false` | `UnsuccessfulValidation` |
| Types 0 and 255, and an unconfigured type | `UnsupportedProofType` |
| Mock with matching, changed or zero-padded values | `Ok`, then `PublicValuesMismatch` twice |
| Mock with bad offsets; zkboost's vector | `Decode`; decodes |
| Mock with bytes appended after `proof` | `Ok`: they extend `proof` |
| Mock with `chain_id = 1` | `MockOnMainnet` |
| `MockProof` under an SP1 type | `Decode` |
| A `cfg(test)` backend that panics or errors | `Panic` or `Verify` |
| **Combined:** PR 1's fixture → `zkboost_compatible_root()` → `PublicInput` (chain 7091047534) → zkboost `MockProof` | `Ok`; with the canonical root, `PublicValuesMismatch` |

The combined test needs no gossip, networking or pipeline.

*Construction cases* assert on the `Result` of `ZkvmProofEngine::new`,
because a failure yields no engine. An empty list, a duplicate type,
an undecodable SP1 VK, and a `ZkboostMock` entry under
`Config::mainnet()` each return their `ConstructionError`. The same
mock entry under a non-mainnet `Config` returns `Ok`.

**Run.**

- `cargo test --release -p types -p proof_engine -p bls --features
  bls/blst --lib`, with and without `proof_engine/sp1`.
- With the caller change: `cargo test --release -p fork_choice_store
  -p bls -p kzg_utils --features bls/blst,kzg_utils/blst --lib`. Read
  the per-crate results, because `kzg_utils`'s own tests fail there.
- `scripts/ci/clippy.bash --deny warnings`, plus clippy on `-p types
  -p proof_engine` with `bls/blst,proof_engine/sp1`.
- `cargo fmt -- --check` inside each touched crate.
- PR 2: `cargo build --release -p grandine -p proof_engine --features
  grandine/default-networks,proof_engine/sp1`.

On a cold cache, ere's `build.rs` runs `cargo metadata` over ere's
workspace: about 500 packages from 19 git repositories. CI therefore
needs the network, not `--offline`. No spec-conformance or devnet test
applies, because there are no vectors and nothing is wired.

## Open questions

**Before Accepted:** none.

**Deferred**

- Upstream convergence on the root and the binding.
- JSON configuration, the CLI flag, runtime construction and metrics.
- Whether gossip treats `Panic` as IGNORE.
- Other backends, and built-in VKs.
- #5729.
- Mock policy on public testnets, and stand-in cost.
- Latency and rayon contention.
- The proposal's "external verifier" wording, which needs a `proposal`
  task.
