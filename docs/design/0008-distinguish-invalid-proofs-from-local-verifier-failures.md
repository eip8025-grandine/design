---
status: Accepted             # Draft | Accepted | Superseded
authors: [creese]
workstream: verification     # gossip | verification | converged
eip_repo: frisitano/EIPs
eip_sha: 4855dbeb9a99702a8c4d948ceceb865fb3289759
consensus_specs_sha: b834fa8411cb2152563e8835c5b72c28f7503e7a
ssz_specs_sha: b42af0a2265c25f5353cace185d066e8d39dbd6e
grandine_upstream_sha: a63876249043020d8ebe3d410c5cf31b19be1cc2
superseded_by:               # Doc number, if any
---

# 0008 — Distinguish invalid proofs from local verifier failures

## Context

Grandine `feature/eip8025` (`40119ea8`) has merged
[0006](0006-make-proof-engine-validation-only.md)'s trait and
[0007](0007-verify-execution-proofs-in-the-proof-engine.md)'s
`ZkvmProofEngine` (PRs #21 and #22). The trait method is
`verify_execution_proof(ExecutionProof) -> bool`, and
`ZkvmProofEngine` implements it as `check(..).is_ok()`. Every
`Rejection` therefore becomes `false`.

In
[`validate_execution_proof_gossip`](https://github.com/ethereum/consensus-specs/blob/b834fa8411cb2152563e8835c5b72c28f7503e7a/specs/_features/eip8025/p2p-interface.md#new-execution_proof),
`false` means REJECT, and REJECT lowers the peer's score. The spec's
`ProofEngine` has no way to report a local failure. Its model assumes
that every node verifies every type in `get_supported_proof_types()`
(`{1, 2, 3}`). The EIP instead makes proof-type support "a per-node,
dynamic configuration". Some `Rejection`s say nothing about the proof:

| `Rejection` | Caused by | Reachable from gossip |
| --- | --- | --- |
| `UnsupportedProofType` | Type in `{1, 2, 3}` with no configured backend | Yes, on any node that configures fewer than three types |
| `Panic` | Defect in the verifier stack, caught by `catch_unwind` | Yes, for any input that triggers it |
| `UnsuccessfulValidation` | Caller-built `PublicInput` | No: preparation always sets `true` |
| `MockOnMainnet` | Caller-built `PublicInput` | No: preparation sets `chain_id` from the Store's `deposit_chain_id`, and construction refuses mock when that is 1, provided both read the same `Config` |

On a devnet node that configures only type 1, honest proofs of types 2
and 3 would be REJECTed, and every peer that relays them would be
downscored.

Nothing calls `verify_execution_proof` outside `proof_engine` tests
yet. `ProcessExecutionProofTask` (`fork_choice_control/src/tasks.rs`)
is a stub that returns IGNORE. One of its tests builds
`MockProofEngine::new(bool)`. The pipeline is designed in
[0004](https://github.com/eip8025-grandine/design/pull/9) (open, not
ours). 0004 keeps the `bool`, and its open question asks for a design
where "boolean `false` must not be overloaded with local
unavailability". This doc is that design. 0007 deferred "whether
gossip treats `Panic` as IGNORE". This doc settles the engine's side
of that question: a panic is `Undetermined`, never `Invalid`.

**Provisional upstream.** `proof-engine.md` is marked work in
progress. consensus-specs at this pin does not yet have Grandine's
progressive versioned hashes (open #5643). It also binds
`hash_tree_root(public_input)`, where 0007 binds the serialized bytes
and uses `zkboost_compatible_root()`. Neither affects this design
because root construction and the expected PublicInput are handled
upstream; 0008 changes only the verification result. The EIP-8025 spec
files are unchanged from 0007's `d9663f85`; only `pysetup` moved. If
upstream adds its own local-failure outcome, or a per-node
supported-type set, this doc's mapping should follow it.

## Goals and non-goals

**Goals**

- `verify_execution_proof` returns one of three verdicts: the proof is
  valid; the proof fails verification or binding; or this node reached
  no verdict.
- Each `Rejection` maps to exactly one verdict, and the compiler
  checks that every variant is mapped.
- Mock and SP1 verification keep the accept and reject decisions that
  0007 specifies.
- The trait stays object-safe and non-generic, keeps `is_null`, and
  still takes the `ExecutionProof` that preparation builds.

**Non-goals**

- Gossip admission, Seen bookkeeping, envelope authentication,
  building `PublicInput`, storage, and the shape of the pipeline's own
  messages. These belong to 0004.
- An early "is this type configured?" check before BLS.
- Changes to backends, construction, runtime configuration, metrics or
  networking.
- Retry of a proof that hit a local failure.

## Design

**Verdict.** `proof_engine` gains:

```rust
#[must_use]
#[derive(Clone, Copy, PartialEq, Eq, Debug)]
pub enum ProofVerdict {
    Valid,
    Invalid,
    Undetermined,
}
```

The trait method becomes `fn verify_execution_proof(&self,
ExecutionProof) -> ProofVerdict`. The name, the by-value argument,
`is_null`, `Send + Sync + 'static` and object safety stay as they are.
The doc comment states that the spec's `True` is `Valid` and its
`False` is `Invalid`, and that `Undetermined` is a Grandine extension.

**Classification rule.** `Invalid` means that, given the trusted
inputs, `proof_data` is at fault. Everything else is `Undetermined`.
The rule rests on three assumptions:

1. **Trusted inputs are correct.** The caller builds `public_input`
   from accepted local context. The operator supplies VKs and backend
   choices.
2. **`proof_type` only selects a backend.** A peer chooses
   `proof_type`, but the caller REJECTs any type outside
   `get_supported_proof_types()` before calling the engine. Inside the
   engine, a type with no backend reflects this node's configuration,
   not the proof.
3. **A backend's error is believed.** If decode or verification
   returns an error, that is a fact about `proof_data` under
   assumption 1. Only a panic counts as a verifier defect.

   On the SP1 verification path, Ere's errors and the reachable
   `CompressedError` variants (`ProofRejected`, `InvalidVkey`,
   `PublicValuesMismatch`; `sp1-verifier` 6.6.0) reflect verification
   against the proof, supplied VK, and compiled-in SP1 parameters.
   Ere's key-format errors occur at construction. Verifier panics are
   handled separately as `Undetermined`.

When assumption 1 or 3 fails, a local fault shows up as `Invalid`
(*Security and compatibility*).

| `Rejection` | Verdict | Log level |
| --- | --- | --- |
| none (`Ok`) | `Valid` | — |
| `Decode` | `Invalid` | `debug!` |
| `Verify` (bad proof, wrong kind, non-zero exit code) | `Invalid` | `debug!` |
| `PublicValuesMismatch` | `Invalid` | `debug!` |
| `UnsupportedProofType` | `Undetermined` | `debug!` (expected on partial configurations) |
| `UnsuccessfulValidation` | `Undetermined` | `warn!` (caller bug) |
| `MockOnMainnet` | `Undetermined` | `warn!` (caller bug) |
| `Panic` | `Undetermined` | `warn!` (verifier defect) |

`Rejection::verdict(&self) -> ProofVerdict` holds this table as a
`match` with no wildcard arm. A new variant will not compile until
someone classifies it.

`ZkvmProofEngine::check` keeps its `Result<(), Rejection>` signature
for tests and diagnostics. The trait method returns
`check(..).map_or_else(|r| r.verdict(), |()| ProofVerdict::Valid)` and
logs as the table says, never with the proof bytes.

**Other implementations.**

- `NullProofEngine` returns `Undetermined`. It used to return `false`.
  It still fails closed, because it never returns `Valid`, but it no
  longer claims the proof is invalid. Callers still check `is_null`
  first, so this is not normally reached.
- `MockProofEngine::new(ProofVerdict)` returns the verdict it was
  given, so caller tests can drive all three paths.

**Contract.** The engine returns a verdict and nothing more. It does
not touch Seen, storage or gossip outcomes; those belong to 0004.

| Verdict | Spec equivalent | What it implies for the caller |
| --- | --- | --- |
| `Valid` | `True` | May ACCEPT and store, under 0004's rules |
| `Invalid` | `False` ("execution proof is invalid") | REJECT |
| `Undetermined` | none | No verdict about the proof. Must never be treated as `True`, and must not be reported as REJECT |

The contract also applies to `on_execution_proof`, including proofs
submitted through the HTTP API: `Undetermined` establishes neither
successful verification nor proof invalidity, and the caller is
responsible for handling that outcome.

The expected mapping of `Undetermined` to IGNORE matches 0004's
existing "separately reported local fault → IGNORE; retain Seen" row.
This doc does not change Seen: the spec records both keys before it
calls the engine, so they are consumed whatever the verdict.

## Trade-offs and alternatives

**Keep `bool` and add `supports_proof_type(ProofType) -> bool`.** This
is the alternative a reviewer will raise. Called before BLS, it also
saves work. It is rejected as the fix because it covers configuration
only. A verifier panic would still be `false`, so it would still be
REJECT. It also needs a second method and an extra pipeline step. It
can still be added later as an admission optimisation (*Open
questions*).

**`Result<bool, VerifierError>`.** This keeps the spec's `bool`
visible. It is rejected because `unwrap_or(false)` and `is_ok()` both
compile, and both collapse a local failure into a verdict. Avoiding
that collapse is the point of this design. An enum with three variants
forces an exhaustive `match`.

**Expose `Rejection` through the trait.** This is rejected because it
puts `ZkvmProofEngine`'s internal taxonomy into the object-safe trait,
and every caller would have to classify it. The engine knows which of
its failures are local, so it should classify them.

**Treat `Panic` as `Invalid`**, since a panic can be input-triggered.
This is rejected. A panic is a defect in our stack, and REJECTing on
it would penalise honest relayers for that defect. Abuse is still
bounded: reaching the engine needs a valid signature from an active
validator, and the spec's Seen keys allow one attempt per (B, T, V)
and per envelope root.

**Name.** The roadmap draft used `Unverifiable`. That word describes
the proof, but the verdict is about this node. `Unverified` is taken:
Grandine's `SignatureStatus::Unverified` means "not yet checked".

**Cost.** This amends 0006's signature and 0007's
"`check(..).is_ok()`". It changes every implementation and one
`MockProofEngine` test in `fork_choice_control`.

## Security and compatibility

**Adversarial model.** As in 0007: `proof_data` and `proof_type` are
attacker input, while `public_input`, VKs and backend choices are
trusted. Under the three assumptions above, the engine reports
`Invalid` only for faults in attacker input. A peer can cause
`Undetermined` only by triggering a verifier panic, or by sending a
supported type this node does not run. Under the contract, neither is
reported as REJECT.

**DoS and resource use.** Unchanged per proof. An unconfigured type
still costs one BLS verification before the engine answers
`Undetermined`, as it does today before `false`. Seen caps this at one
attempt per (B, T, V). The early check that would avoid that BLS is
deferred. If the pipeline IGNOREs `Undetermined`, gossipsub does not
forward the proof, so a node never relays a proof it could not verify.

**Local faults that still look invalid.** These are the cases where
assumption 1 or 3 fails. A wrong VK, an SP1 version that cannot verify
a peer's honest proof, or a `PublicInput` built under a root or
binding that has since moved upstream, all yield `Verify` or
`PublicValuesMismatch`. These are `Invalid`, so honest peers get
REJECTed. The engine cannot tell them apart from bad proofs one at a
time: under the trusted inputs, the proof really does fail. The VK and
the statement are trust roots that the operator configures. 0007
already flags the root and the binding as provisional.

**Invalid or withheld proofs.** Unchanged. Proofs are non-consensus
artifacts. No verdict affects state, payload status or fork choice.

**Nodes that do not opt in.** Unchanged. `NullProofEngine` stays the
default, `is_null` short-circuits before any work, and the feature
stays default-off. If the Null engine is reached anyway, it now leads
to IGNORE instead of REJECT.

**Compatibility.** This is an API change inside the fork's
`proof_engine` only. No SSZ type, root, wire format, signing domain or
fixture changes. Spec test engines (`helpers/proof_engine.py`) return
`bool`, and they map onto `Valid` and `Invalid` with no change.

## Implementation and testing

One PR against `feature/eip8025`, touching `proof_engine` and the
`MockProofEngine` construction in `fork_choice_control/src/tasks.rs`.
It has no dependency on 0004's PRs. If 0004's pipeline lands first, the
pipeline must take up the three-way match. Otherwise 0004 adopts it
when it rebases.

**Tests.**

- The `check` helper in `zkvm_engine.rs` asserts that the engine,
  upcast to `Arc<dyn ProofEngine>`, returns `ProofVerdict::Valid` on
  `Ok` and `rejection.verdict()` otherwise. Every existing 0007 case
  therefore also checks its verdict.
- A table test calls `verdict()` on every `Rejection` variant and
  compares it with the table above.
- At trait level: an unconfigured type, and types 0 and 255, give
  `Undetermined`. `PanickingBackend` gives `Undetermined`.
  `FailingBackend` (`Verify`) gives `Invalid`.
- With `proof_engine/sp1`: the primary fixture gives `Valid`; the other
  VK and the tampered byte give `Invalid`; a changed root gives
  `Invalid`.
- `NullProofEngine` gives `Undetermined`. `MockProofEngine` returns each
  verdict it is given. Both are checked behind `Arc<dyn ProofEngine>`
  in `tests.rs`.

**Run.**

- `cargo test --release -p types -p proof_engine -p bls --features
  bls/blst --lib`, with and without `proof_engine/sp1`.
- `cargo test --release -p fork_choice_control -p bls -p kzg_utils
  --features bls/blst,kzg_utils/blst --lib`. Read the per-crate
  results.
- `scripts/ci/clippy.bash --deny warnings`, plus clippy on
  `-p proof_engine` with `proof_engine/sp1`.
- `cargo fmt -- --check` in each touched crate.

No spec-conformance or devnet test applies. No vectors cover the
engine, and nothing calls it outside tests.

## Open questions

**Before Accepted**

- **0004 coordination.** 0004's author needs to agree to replace its
  `bool` (and its `ProofVerifier` and `hash_tree_root` wording) with
  `ProofVerdict`, and to map `Undetermined` to IGNORE with Seen
  retained, as described above. The mutator's current `Err → REJECT`
  handler (`mutator.rs`, `handle_execution_proof`) is one path that
  would need to keep `Undetermined` out. The other workstream reviews
  this doc.
- **0006 and 0007.** Whether to add a one-line pointer to 0008 in each
  Accepted doc when this one is accepted, or to leave them as written.

**Deferred**

- An early `supports_proof_type` check that skips BLS for unconfigured
  types (an admission optimisation for 0004).
- A node-level signal, such as a metric, for faults that look invalid,
  for example every proof of one type failing `Verify` after a VK
  change.
- Whether a type that is configured but repeatedly `Undetermined`
  should be unsubscribed or reported in a future
  `ExecutionProofStatus`.
- Upstream adoption of a local-failure outcome.
