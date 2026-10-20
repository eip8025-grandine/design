---
status: Accepted
authors: [creese]
workstream: verification
eip_repo: frisitano/EIPs
eip_sha: 4855dbeb9a99702a8c4d948ceceb865fb3289759
consensus_specs_sha: d9663f85be248c4fd0ae95806da6070e17c2442b
ssz_specs_sha: b42af0a2265c25f5353cace185d066e8d39dbd6e
grandine_upstream_sha: 34da987dc6e1eec7be6afb6fb3c6edf317f1c32c
superseded_by:
---

# 0006 — Make the proof engine validation-only

## Context

Grandine `feature/eip8025` (`d24d6d84`) has a `proof_engine` crate,
merged in PR #9 (`7bad2ca6`) and updated by PR #18 (`d24d6d84`) for
[0005](0005-update-execution-proof-data-model.md). Its
[`engine.rs`](https://github.com/eip8025-grandine/grandine/blob/d24d6d84c231be39e1a444a71c15776e9cb004b5/proof_engine/src/engine.rs)
splits the spec's `ProofEngine` protocol into two traits:

- `ProofVerifier` is non-generic and object-safe. It has `is_null` and
  `verify_execution_proof`.
- `ProofProver<P>` has `request_proofs` and `get_proof`, which return
  `ProofEngineError::Unsupported`.

`NullProofEngine` and `MockProofEngine` implement both traits.
`types::eip8025::ProofAttributes` exists only to feed
`request_proofs`. On the baseline, nothing outside `proof_engine` and
the `types` tests uses any of these. Unmerged Grandine PRs #11 and
#17 hold `Arc<dyn ProofVerifier>`, but they are not baseline.

The split comes from
[0003](https://github.com/eip8025-grandine/design/pull/4) (open, not
ours). Its deciding premise was that a verify-only trait "would drop
the prover half and fork from `proof-engine.md`". It retired the
`ProofEngine` trait name for that reason.

Upstream has since removed that premise.
[#5639](https://github.com/ethereum/consensus-specs/pull/5639)
(`e42b2493`, merged and already inside 0005's `aa16bb4c` pin) made
[`proof-engine.md`](https://github.com/ethereum/consensus-specs/blob/d9663f85be248c4fd0ae95806da6070e17c2442b/specs/_features/eip8025/proof-engine.md)
validation-only. It removed `ProofAttributes`, `request_proofs`,
`get_proof` and `prover.md`, and the pyspec's `NoopProofEngine` now
implements only `verify_execution_proof`. Every spec call site passes
a parameter named `proof_engine: ProofEngine`:
`process_execution_proof`, `on_execution_proof` and
`validate_execution_proof_gossip`. 0005 deferred this removal to its
own doc, and this is that doc.

**Provisional upstream.** The relevant open EIP-8025 PRs were reviewed
as of 2026-10-19. None changes the `ProofEngine` interface.

**EIP divergence.** The EIP still describes proof-generation and
`notify_*` methods. Grandine follows consensus-specs for the CL.

## Goals and non-goals

**Goals**

- One trait, `proof_engine::ProofEngine`, whose methods are the spec
  protocol at the pin plus Grandine's `is_null` opt-out.
- Delete everything #5639 removed: `ProofProver<P>`,
  `ProofEngineError`, `ProofAttributes`, the prover stubs and their
  tests.
- Re-point the spec links in `proof_engine` from `7fa04483` to the
  pin.

**Non-goals**

- Changes to `verify_execution_proof`'s signature or semantics,
  including by-value versus `&ExecutionProof`.
- The EIP's `notify_*` lifecycle methods. consensus-specs has no such
  methods.
- Any prover role or proof-node client.
- #5729's `PublicInput` change, #5726's subnets, and EIP-8440.
- Editing unmerged Grandine PRs #11 and #17, or design docs 0003 and
  0004.

## Design

**Trait.** `ProofVerifier` is renamed `ProofEngine` and is otherwise
unchanged:

- non-generic, object-safe, `Send + Sync + 'static`;
- `fn is_null(&self) -> bool`;
- `fn verify_execution_proof(&self, ExecutionProof) -> bool`.

The doc comment states that it is the whole spec protocol, and that
`is_null` is a Grandine extension that the spec does not have.

**Removed.**

| Item | Where |
| --- | --- |
| `ProofProver<P>`, `ProofEngineError` | `engine.rs`, `lib.rs` re-exports |
| prover impls, `canned_proof`, `with_canned_proof` | `mock_engine.rs`, `null_engine.rs` |
| `ProofAttributes` | `types/src/eip8025/containers.rs` |
| `thiserror` dependency | `proof_engine/Cargo.toml` (only `ProofEngineError` used it) |

**Kept.**

- `NullProofEngine` and `MockProofEngine` keep their names and
  behaviour. `NullProofEngine` reports `is_null() == true` and fails
  closed on verify. `MockProofEngine::new(bool)` drives verify.
- The crate now depends on `types` alone. It no longer needs `Preset`,
  `NewPayloadRequest` or `H256`.

The result follows Grandine's existing pattern:
`execution_engine::ExecutionEngine` with `NullExecutionEngine` and
`MockExecutionEngine`, and now `ProofEngine` with `NullProofEngine`
and `MockProofEngine`.

**Guidance for callers.** This is not a change in this doc. Callers
should hold `Arc<dyn ProofEngine>` in a field or parameter named
`proof_engine`, as the spec does. #11 and #17 adopt this on rebase.

## Trade-offs and alternatives

**Keep the name `ProofVerifier` and delete only `ProofProver<P>`.**
This is the alternative a reviewer will raise. It is accurate, and it
spares #11 and #17 a rename. It is rejected for three reasons:

- 0003 retired `ProofEngine` only because the protocol had been split,
  and that is no longer true.
- Every spec call site says `proof_engine`, so the name makes Grandine
  easier to read against the spec.
- The name completes the `*Engine` and `Null*Engine` pattern that the
  implementors already follow.

**Keep `ProofProver<P>` as unwired stubs** for a future prover, since
the EIP still describes `request_proofs`. Rejected. consensus-specs no
longer defines these methods, and they are its authority for the CL. A
Grandine-only interface with no spec text invites drift. A future
prover, under EIP-8025 or EIP-8440, should be designed against
whatever the spec then says.

**Re-export a deprecated `ProofVerifier` alias.** Rejected.
`#[deprecated]` on a `use` re-export compiles but warns nowhere. This
was checked with rustc 1.95, implementing and using the alias as a
`dyn` trait. The crate has no consumers outside this fork, and the two
open PRs must rebase onto the removal anyway.

## Security and compatibility

**Adversarial model and DoS.** Nothing changes. No gossip, Req/Resp or
HTTP path reaches `proof_engine` on the baseline. Verification still
takes a reconstructed `ExecutionProof` whose `ProofData` is bounded at
4 MiB by 0005's types. The removed prover methods were unreachable
reject-stubs, so deleting them removes no exposed surface.

**Invalid or withheld proofs.** Nothing changes.
`verify_execution_proof` returning `false` remains the only rejection
signal. Proofs remain non-consensus artifacts, per
[`beacon-chain.md`](https://github.com/ethereum/consensus-specs/blob/d9663f85be248c4fd0ae95806da6070e17c2442b/specs/_features/eip8025/beacon-chain.md),
so a withheld proof affects no payload status or fork choice.

**Nodes that do not opt in.** These are unaffected. `NullProofEngine`
stays the default opt-out: `is_null` short-circuits, and verify fails
closed if reached. The feature stays default-off.

**Compatibility.** This is a breaking API change inside the fork's
`proof_engine` and `types::eip8025` only. No SSZ type, root, vector,
wire format or signing domain changes. `ProofAttributes` was
serde-only and never exposed on any API, so no JSON shape that a peer
or tool could see disappears.

## Implementation and testing

One PR against `feature/eip8025`, touching `proof_engine` and
`types::eip8025`.

**Merge ordering.** If #11 or #17 merges first, this change updates
its callers. Otherwise, those PRs adopt the renamed interface when
they rebase.

**Tests removed.** Remove tests for the deleted proof-generation APIs
and `ProofAttributes`.

**Tests kept.** The null and mock `is_null` and verify tests, renamed
only.

**Test added.** In `proof_engine`, put `NullProofEngine` and
`MockProofEngine` behind `Arc<dyn ProofEngine>` and call both methods
through it. Object safety is the property this design exists to keep.
No merged caller exercises it, so without this test only #11 would
catch a regression.

**Run.**

- `cargo test --release --features bls/blst -p types -p proof_engine`
- `scripts/ci/clippy.bash` (already lints `proof_engine`)
- `cargo fmt -- --check` in both crates

No spec-conformance or devnet test applies: no EIP-8025 vectors cover
the engine, and nothing is wired.

**Deferred.**

- **0003 and 0004** This design supersedes their `ProofEngine`
  interface decisions, but leaves the rest unchanged. Their author can
  update those documents separately.
