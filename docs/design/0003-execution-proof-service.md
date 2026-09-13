---
status: Draft
authors: [keshavsharma25]
workstream: verification
spec_base: 7d6bd46a0
spec_tip: 6946894b0
grandine_base: feature/sign-execution-proofs@9af0859
superseded_by:
---

# 0003 — Verifier-only execution-proof service: gossip pipeline and ProofVerifier/ProofProver boundary

## Context

EIP-8025 lets a consensus client validate execution payloads by verifying
gossiped execution proofs instead of re-executing them. The verifier side
splits into two components: the **execution-proof service**, the
consensus-layer orchestrator between gossip and proof verification, and the
**proof engine boundary**, the implementation-dependent layer that delegates
cryptographic verification to an external verifier
(`proof-engine.md`, "Proof engine"). Grandine spells that boundary as two
traits, `ProofVerifier` + `ProofProver<P>` (Design).

This doc re-baselines the stale week-8 skeleton against two pins:

- Grandine `feature/sign-execution-proofs@9af0859` — non-generic
  `ExecutionProof` / `ExecutionProofEnvelope` /
  `SignedExecutionProofEnvelope` in `types::eip8025`
  (`types/src/eip8025/containers.rs`), `Hc`-wrapped signed message,
  `ProofData` bound, `MAX_PROOF_SIZE` / `STATELESS_INPUT_SCHEMA_ID`
  (`0x1501`) / `MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE` /
  `DOMAIN_EXECUTION_PROOF` (`0x0F000000`) in
  `types/src/eip8025/consts.rs`, envelope signing via
  `SignForSingleForkAtSlot` (`helper_functions/src/signing.rs:530`).
- consensus-specs `feat/simplify-eip8025` — spec `7d6bd46a0`, tip
  `6946894b0`: `proof-engine.md` (3-method surface), `beacon-chain.md`
  (envelope auth, `get_execution_proof`, `process_execution_proof`),
  `p2p-interface.md` (gossip-only, `Seen` maps,
  `validate_execution_proof_gossip`), `fork-choice.md`
  (`Store.execution_proofs`, `on_execution_proof`), `prover.md`
  (envelope signature, prover flow).

Signing primitives are doc 0002's deliverable; SSZ containers already landed
on the base branch. This doc fixes shapes and ownership, not bodies.

One shape decision is re-baselined here: the engine boundary is a
non-generic, object-safe `ProofVerifier` (the thing the task holds and
erases) plus a generic `ProofProver<P>` (the prover methods, kept as
reject-stubs), not the single `ProofEngine<P>` trait of the week-8
skeleton. The split exists because the erased handle cannot carry `P` or a
`const IS_NULL`; see Design for the dissection.

## Goals and non-goals

**Goals**

- A new `proof_engine` crate exposing the spec's full 3-method protocol as
  a non-generic, object-safe `ProofVerifier` (opt-out + verify) plus a
  generic `ProofProver<P>` (request + get), with `NullProofEngine` and
  `MockProofEngine`. Prover methods reject. The spec surface stays whole;
  the split exists so the erased verifier handle carries no `P`.
- A `ProcessExecutionProofTask` in `fork_choice_control` implementing the
  spec-ordered gossip pipeline verbatim, returning a bid-style
  `MutatorMessage::ExecutionProof` (no `wait_group`).
- Spec-fidelity for the stored shape: `Store.execution_proofs` mirrors
  `fork-choice.md` exactly; surrounding plumbing follows Grandine
  convention.

**Non-goals**

- Anything prover-side: `request_proofs`/`get_proof` exist as reject-stubs
  only, no prover wiring, no `request_execution_proofs` flow.
- Subscription wiring, `ExecutionProofStatus`/`ByRange`, k-of-n counting,
  recursive anchors, pruning/re-derivation design beyond finalization-tied
  retention, `client.rs`, notify-history or zkboost wire-body
  stories.

## Design

The service is not a loop or thread; in Grandine everything is a task
struct on the controller's thread pool. The lifecycle clones the
`ExecutionPayloadBidTask` precedent (`tasks.rs:701`):

```
eth2_libp2p (gossip) → p2p router → Controller
                                      │ spawn_execution_proof_task()
                                      ▼
                      ProcessExecutionProofTask::run()
                        spec-ordered pipeline (§ below)
                        → proof_verifier.verify_execution_proof()
                                      │
                                      ▼
                MutatorMessage::ExecutionProof { result, origin }
                                      │
                    mutator arm → apply to Store + signal p2p
```

Validation (including the potentially slow engine call) runs concurrently
on the thread pool, off the critical network path, while state changes
apply serially in the controller's mutator loop. This is the standard
low-priority gossip invariant: snapshot-validate → `Result<Action>` +
`Origin` → mutator applies to `Store` + signals p2p. `PayloadBid`
(`messages.rs:172`) is the precedent for omitting `wait_group` on a
low-priority gossip result.

**Engine traits** (`proof_engine/src/engine.rs`) — the spec's single
`ProofEngine` protocol, split into a non-generic verifier half and a
generic prover half:

```rust
/// Verification half: non-generic and object-safe. It is what the task
/// holds and erases.
pub trait ProofVerifier: Send + Sync + 'static {
    fn is_null(&self) -> bool;

    fn verify_execution_proof(&self, execution_proof: ExecutionProof) -> bool;
}

/// Generation half: generic over `P` because `request_proofs` takes the
/// full `SszNewPayloadRequest<P>`. Nothing wires it in Grandine yet.
pub trait ProofProver<P: Preset>: Send + Sync + 'static {
    fn request_proofs(
        &self,
        new_payload_request: SszNewPayloadRequest<P>,
        chain_id: u64,
        schema_id: u16,
        proof_attributes: ProofAttributes,
    ) -> Result<H256, ProofEngineError>;

    fn get_proof(
        &self,
        new_payload_request_root: H256,
        proof_type: ProofType,
    ) -> Result<ExecutionProof, ProofEngineError>;
}
```

- The split is an implementation detail of one spec protocol; the union
  of `ProofVerifier` + `ProofProver<P>` is the spec surface, kept whole
  so a future prover PR does not fork from `proof-engine.md`.
- `verify_execution_proof` takes an already-reconstructed
  `ExecutionProof`, so it needs no `P`. That is what makes
  `dyn ProofVerifier` legal and lets the erased handle carry no preset.
  `P` is needed only by `request_proofs`, which the verifier path never
  calls.
- Opt-out is the `is_null` method, not `const IS_NULL`. The handle is
  runtime-selected, so a compile-time const could not fold where it
  matters, and a const is not `dyn`-compatible (`E0038`). `is_null` is on
  the non-generic half, so it is callable on concrete engines without
  naming a preset — avoiding the `E0283` a `fn is_null` on
  `ProofEngine<P>` hits in the unit tests.
- No `&E` / `Arc<E>` / `Mutex<E>` forwarding impls (the `ExecutionEngine`
  twin convention). The task holds `Arc<dyn ProofVerifier>` and calls
  through `Deref`, and every method is `&self`, so no `Mutex` is needed.
- `ProofAttributes` is new in `types::eip8025` (spec
  `proof-engine.md:59-65`: `proof_types` sequence; Grandine: a
  `Vec<ProofType>`-shaped container — prover methods are reject-stubs
  in the skeleton, so the bound is deferrable). Added in PR1.
- Named `ExecutionProofAction`, not a bespoke `ProofOutcome`.
- `NullProofEngine`: `is_null` returns `true`; implements both halves;
  `verify` returns `false` (fail-closed), `request_proofs`/`get_proof`
  return `Err(unsupported)`. The task short-circuits on `is_null` to
  `Ignore` before any pipeline work, so the fail-closed `false` is
  unreachable in practice.
- `MockProofEngine`: constructor takes `execution_proof_valid: bool`
  (mirroring `MockExecutionEngine`), canned prover proof/error for the
  stubbed methods.
- Prover methods default-reject: the spec explicitly permits
  implementations without generation support to reject
  (`proof-engine.md:39-40`). Verifier-only scope means these are stubs,
  never wired.

Crate layout:

```
proof_engine/src/
  lib.rs
  engine.rs        # ProofVerifier + ProofProver<P> traits + error type
  null_engine.rs   # NullProofEngine
  mock_engine.rs   # MockProofEngine
```

Registered as a workspace member. No `client.rs` — that lands with
external-verifier integration, deferred for later.

**Service task** (`fork_choice_control`) — mirrors
`ExecutionPayloadBidTask`:

```rust
pub struct ProcessExecutionProofTask<P: Preset, W> {
    pub store_snapshot: Arc<Store<P, Storage<P>>>,
    pub proof_verifier: Arc<dyn ProofVerifier>,
    pub mutator_tx: Sender<MutatorMessage<P, W>>,
    pub signed_proof: Arc<SignedExecutionProofEnvelope>,
    pub origin: ExecutionProofOrigin,
}
```

- Carrier is the **signed** envelope end to end (task → `Accept` →
  mutator). Both `Accept` variants end with the identical bare envelope in
  `Store`; the signed carrier keeps `validator_index`/`signature` one hop
  longer for `Seen` marking, logging, and events.
- Entry point `Controller::spawn_execution_proof_task`, taking
  `proof_verifier: Arc<dyn ProofVerifier>` — no `P` and no repeated
  `+ Send + Sync` (the trait carries it). Nothing calls it until gossip
  wiring lands.
- Queued via a `LowPriorityTask::ExecutionProof` variant (proofs up to
  4 MiB plus BLS and potentially slow external verification).
- `run()` opens with the `is_null` → `Ignore` short-circuit. Skeleton
  `run()` is a stub: beyond the short-circuit it returns `Ignore`
  unconditionally, marked `// TODO(eip8025-grandine): spec-ordered
pipeline` so the follow-up is greppable. The stub exists so PR3's
  plumbing (spawn fn, `LowPriorityTask` variant, message, mutator arm)
  is reviewable without behavior; the full
  `validate_execution_proof_gossip` pipeline (`p2p-interface.md:80-151`,
  verbatim spec order: seen → block-seen → block-valid → payload →
  type-new → prover-new → envelope `REJECT` → build → mark `Seen` →
  engine `REJECT`) replaces it in place, deferred for later.
- Outcome emitted as bid-style
  `MutatorMessage::ExecutionProof { result: Result<ExecutionProofAction>, origin }`.
  `Accept` carries the signed envelope; the mutator stores the bare
  `.message` per spec. The mutator arm clones `handle_payload_bid`
  (`mutator.rs:2514`): apply to `Store`, emit event, signal p2p
  accept/reject/ignore via `origin.split()`.

**`Store` changes** (`fork_choice_store/src/store.rs`) — spec shape
verbatim (`fork-choice.md:32-53`):

```rust
// [New in EIP8025]
execution_proofs: HashMap<H256, HashMap<ProofType, ExecutionProofEnvelope>>,
```

- No skeleton eviction; retention tied to finalization (pruned with blocks).
- `get_forkchoice_store` initializes `execution_proofs: {}`.
- `on_execution_proof` semantics (`fork-choice.md:100-129`) live split
  across Grandine's validate/apply boundary: the task's snapshot
  validation covers the asserts, the mutator arm performs the store insert
  (only proofs passing downstream verification).
- The eventual `Store::validate_execution_proof` takes the engine as a
  non-generic `proof_verifier: &dyn ProofVerifier` (no `impl Trait +
  ?Sized` dance, no `P` on the handle), so `fork_choice_store` can depend
  on `proof_engine` directly.

**`Seen` changes** — spec maps verbatim (`p2p-interface.md:40-67`),
Store-owned like existing `seen_gossip_*`:

```rust
// [New in EIP8025]
execution_proof_roots: Dict[Root, Set[Root]],
execution_proof_provers: Set[Tuple[Root, ProofType, ValidatorIndex]],
```

Keyed by `beacon_block_root`; the `store.execution_proofs[block][type]`
check is separate (verified-store vs seen-attempts). LRU bounding is an
implementation detail, not spec shape.

**`get_execution_proof`** is service-owned (not on the engine): all its
inputs (payload, bid, chain/schema IDs) are consensus-layer state, and
the spec defines it as a pure function of `(state, envelope, payload)`
(`beacon-chain.md:202-234`), not engine state. It reads
`store.payloads[beacon_block_root]` and
`state.latest_execution_payload_bid`, hashes the reconstructed
`SszNewPayloadRequest`, and stamps `DEPOSIT_CHAIN_ID` /
`STATELESS_INPUT_SCHEMA_ID` (`0x1501`). Notify-history context
resolution and zkboost wire-body assembly are dropped: neither is spec,
so the trait keeps the bare `verify_execution_proof(ExecutionProof)`
signature. Body deferred for later with the pipeline.

**Bounds:** `ProofData` decode bound (`MaxProofSize`) whose root is
bound-independent; envelope gate `0 < len ≤ MAX_PROOF_SIZE` inside
`verify_execution_proof_envelope`; gossip pre-decode cap
`MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE` (4194449), mandatory because
progressive-list decode allocates in proportion to attacker input; plus
the provisional `ProofType {1, 2, 3}` allowlist via
`get_supported_proof_types`. Enforcement lives in the validation
bodies, deferred for later with the pipeline.

**Origin type** — new `ExecutionProofOrigin` wrapping `GossipId`
(mirroring `ExecutionPayloadBidOrigin`), not a raw `GossipId`, so the
mutator can `origin.split()` for p2p signalling.

## Trade-offs and alternatives

- **New `proof_engine` crate vs living in `fork_choice_control`.** Chose
  the crate for parity with `execution_engine` — both are boundaries to
  an external system.
- **Engine-boundary shape: Option A vs Option B vs Option C (chose C).**
  Recorded in full, not just as a conclusion, so the decision is
  reviewable later. All three reconcile a `P`-generic engine with the
  erased handle in `ProcessExecutionProofTask`; two were built as draft
  PRs and rejected.
  - **Option A — `ProofVerifier<P>` facade** (PR #13,
    `feature/proof-service-task-plumbing-with-proofverifier`). A parallel
    trait blanket-implemented over `ProofEngine<P>`, leaving `const
    IS_NULL` and the engine untouched; the task holds
    `Arc<dyn ProofVerifier<P>>`. Rejected: it is a second trait that must
    mirror `verify_execution_proof` (and every future verifier method) and
    can silently drift; it sits in `fork_choice_control`, which
    `fork_choice_store` cannot depend on (the dependency direction is
    reversed), so the eventual `Store::validate_execution_proof` call site
    cannot name it without a cycle; and it still carries `P` on the
    handle.
  - **Option B — `fn is_null` on `ProofEngine<P>`** (PR #14,
    `feature/proof-service-task-plumbing-fn-is-null`). Object-safe, one
    trait, but `is_null` loses its preset-free home: concrete-engine unit
    tests hit `E0283` (`NullProofEngine.is_null()` cannot infer `P`).
    Worked around by either turbofish annotations or a non-generic
    `ProofEngineBase` supertrait, both symptoms of `P` riding on the
    verifier path. Rejected in favour of isolating `P`.
  - **Option C — verifier/prover split (chosen).** A single `P`-generic
    trait cannot be the erased handle the task needs: `const IS_NULL` is
    not `dyn`-compatible (`E0038`). Because `P` is needed solely by
    `request_proofs`, splitting yields a non-generic, object-safe
    `ProofVerifier` (usable from `fork_choice_store` and callable on
    concrete engines without naming a preset) and a generic
    `ProofProver<P>` that keeps the spec surface intact. A verify-only
    trait would drop the prover half and fork from `proof-engine.md`; the
    split keeps it, so a future prover PR does not look like a fork.
- **Service-owned `get_execution_proof` vs engine-internal context.**
  Service-owned: all inputs (payload, bid, chain/schema IDs) are
  consensus-layer state, and the spec defines it as a pure function of
  `(state, envelope, payload)`, not engine state.

## Security and compatibility

- Any peer can flood the `execution_proof` topic with junk up to ~4 MiB;
  spec cheapest-check-first ordering plus pre-validity `Seen` marking
  caps repeat senders at cache lookups, and the pre-decode gossip cap
  is mandatory because progressive-list decode allocates in proportion
  to attacker-controlled input.
- BLS + eligibility checks run before any cryptographic proof work;
  invalid envelopes `Reject` and feed peer scoring via the outcome. A
  failed proof never invalidates a payload accepted through the Engine
  API — proofs stay auxiliary to existing payload validation
  (`beacon-chain.md:38-39`: non-consensus artifacts).
- Opt-out nodes inject `NullProofEngine` and subscribe to nothing;
  opted-in nodes only add a parallel validity signal.

## Implementation and testing

Stacked directly on `feature/sign-execution-proofs`, no rebase onto
`develop`. Three PRs in dependency order, each independently green
(check + tests + `fmt`, clippy per 0002 §7 flags) and each carrying
its own tests. Stub bodies are marked
`// TODO(eip8025-grandine): <what replaces them>` so follow-ups are
greppable.

The chosen engine boundary (Option C, see Trade-offs) reworks the engine
wiring on top of the task-plumbing PR (#11,
`feature/proof-service-task-plumbing`); the Option A/B draft PRs
(#13/#14) close as superseded by it.

| PR                      | Scope                                                                                                                                                                                                                                                                                                                                                                                        | Tests in-PR                                                                                                                                   | Gate                                              |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| 1. `proof_engine` crate | `engine.rs` (`ProofVerifier` + `ProofProver<P>` traits + `ProofEngineError`; no forwarding impls), `null_engine.rs`, `mock_engine.rs`, `lib.rs`, `Cargo.toml`, workspace registration; `ProofAttributes` in `types::eip8025`. Prover methods reject per spec.                                                                                                                                           | Null fail-closed (`is_null` → `true`, `verify` → `false`, prover methods → `Err`); Mock configured `verify` (true/false) + canned prover proof/error.             | `cargo check/test -p proof_engine` (+ `-p types`) |
| 2. State shapes         | `fork_choice_store`: `Store.execution_proofs` field + `get_forkchoice_store` init; `Seen` maps (`execution_proof_roots`, `execution_proof_provers`), Store-owned. Spec shapes verbatim, no behavior, no eviction.                                                                                                                                                                            | Store-init test (`execution_proofs` empty at genesis); `Seen` maps default-empty.                                                             | `cargo check/test -p fork_choice_store`           |
| 3. Task plumbing        | `fork_choice_control`: `ExecutionProofAction`, `ExecutionProofOrigin`, `MutatorMessage::ExecutionProof` variant, `ProcessExecutionProofTask` with stubbed `run()` (`is_null` → `Ignore`, else stub `Ignore` + `TODO(eip8025-grandine)`), `spawn_execution_proof_task`, `LowPriorityTask::ExecutionProof` variant, mutator arm (stub: no store insert yet, signals p2p per `origin.split()`). | Task smoke test: stub pipeline against `MockProofEngine` (both settings) asserting the emitted outcome message; `NullProofEngine` → `Ignore`. | `cargo check/test -p fork_choice_control`         |

Later work replaces stubs in place, deferred for later: envelope-auth
body and `get_execution_proof` alongside `client.rs`, then gossip
wiring, outcome→p2p routing, and recursive anchors.

## Blockers

- None. Containers and signing landed on the base branch; the doc pins
  `sign-execution-proofs@9af0859`.

## Open questions

- None blocking. k-of-n threshold values deferred (client-side counter
  over distinct `ProofType`s either way); upstream `ProofType`
  assignments remain provisional.
- `ProofProver<P>` ships unwired: public trait methods do not trip
  `dead_code`, but if landing an unused trait is unwanted it can be
  deferred to the first prover PR — the verifier task is unchanged either
  way.
- Naming: `ProofEngine` was the spec's name for the whole protocol. The
  split retires it as a trait name (`ProofVerifier` + `ProofProver<P>`);
  whether to keep `ProofEngine` for a concrete handle type is a later,
  cosmetic call.
