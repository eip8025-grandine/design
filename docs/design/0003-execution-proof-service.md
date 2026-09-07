---
status: Draft
authors: [keshavsharma25]
workstream: verification
spec_base: 7d6bd46a0
spec_tip: 6946894b0
grandine_base: feature/sign-execution-proofs@9af0859
superseded_by:
---

# 0003 — Verifier-only execution-proof service: gossip pipeline and ProofEngine boundary

## Context

EIP-8025 lets a consensus client validate execution payloads by verifying
gossiped execution proofs instead of re-executing them. The verifier side
splits into two components: the **execution-proof service**, the
consensus-layer orchestrator between gossip and proof verification, and the
**`ProofEngine`**, the implementation-dependent boundary that delegates
cryptographic verification to an external verifier
(`proof-engine.md`, "Proof engine").

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

## Goals and non-goals

**Goals**

- A new `proof_engine` crate exposing the spec's full 3-method
  `ProofEngine` trait as an `ExecutionEngine` twin (trait + `&`/`Arc`/`Mutex`
  forwards + `NullProofEngine` + `MockProofEngine`), prover methods
  default-rejecting.
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
                        → proof_engine.verify_execution_proof()
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

**Engine trait** (`proof_engine/src/engine.rs`) — mirrors
`execution_engine/src/execution_engine.rs:22`, full 3-method surface per
`proof-engine.md:26-40`:

```rust
pub trait ProofEngine<P: Preset> {
    const IS_NULL: bool;

    fn verify_execution_proof(&self, execution_proof: ExecutionProof) -> bool;

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

- Generic over `Preset`, mirroring `ExecutionEngine<P>`. The proof
  structs (`ExecutionProof`, envelopes, `ProofData`) are non-generic,
  but `request_proofs` takes the full `SszNewPayloadRequest<P>`, which
  wraps `ExecutionPayload<P>` / `ExecutionRequests<P>`
  (`types/src/eip8025/containers.rs:190`, `types/src/gloas/containers.rs:191`),
  so the trait must thread `P`. A generic _method_ on a non-generic
  trait would forbid `Arc<dyn ProofEngine<P>>`, which the task struct
  below needs. All call sites (`Store<P>`, `MutatorMessage<P>`, the
  task itself) already carry `P`, so this costs nothing new.
- `ProofAttributes` is new in `types::eip8025` (spec
  `proof-engine.md:59-65`: `proof_types` sequence; Grandine: a
  `Vec<ProofType>`-shaped container — prover methods are reject-stubs
  in the skeleton, so the bound is deferrable). Added in PR1.
- Named `ExecutionProofAction`, not a bespoke `ProofOutcome`.
- Forwarding impls for `&E` / `Arc<E>` / `Mutex<E>`, as with
  `ExecutionEngine`.
- `NullProofEngine`: `IS_NULL = true`, `verify` returns `false`
  (fail-closed), `request_proofs`/`get_proof` return
  `Err(unsupported)`. The task short-circuits on `IS_NULL` to `Ignore`
  before any pipeline work, so the fail-closed `false` is unreachable in
  practice.
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
  engine.rs        # ProofEngine trait + forwarding impls + error type
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
    pub proof_engine: Arc<dyn ProofEngine<P>>,
    pub mutator_tx: Sender<MutatorMessage<P, W>>,
    pub signed_proof: Arc<SignedExecutionProofEnvelope>,
    pub origin: ExecutionProofOrigin,
}
```

- Carrier is the **signed** envelope end to end (task → `Accept` →
  mutator). Both `Accept` variants end with the identical bare envelope in
  `Store`; the signed carrier keeps `validator_index`/`signature` one hop
  longer for `Seen` marking, logging, and events.
- Entry point `Controller::spawn_execution_proof_task`; nothing calls it
  until gossip wiring lands.
- Queued via a `LowPriorityTask::ExecutionProof` variant (proofs up to
  4 MiB plus BLS and potentially slow external verification).
- `run()` opens with the `IS_NULL` → `Ignore` short-circuit. Skeleton
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
- **Full 3-method trait vs verify-only trait.** Full trait keeps
  spec-fidelity (`proof-engine.md` defines one protocol); prover methods
  reject per the spec's explicit permission. A verify-only trait would
  fork from the spec surface and complicate future prover work.
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

| PR                      | Scope                                                                                                                                                                                                                                                                                                                                                                                        | Tests in-PR                                                                                                                                   | Gate                                              |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| 1. `proof_engine` crate | `engine.rs` (`ProofEngine<P>` trait + `&`/`Arc`/`Mutex` forwards + `ProofEngineError`), `null_engine.rs`, `mock_engine.rs`, `lib.rs`, `Cargo.toml`, workspace registration; `ProofAttributes` in `types::eip8025`. Prover methods reject per spec.                                                                                                                                           | Null fail-closed (`verify` → `false`, prover methods → `Err`); Mock configured `verify` (true/false) + canned prover proof/error.             | `cargo check/test -p proof_engine` (+ `-p types`) |
| 2. State shapes         | `fork_choice_store`: `Store.execution_proofs` field + `get_forkchoice_store` init; `Seen` maps (`execution_proof_roots`, `execution_proof_provers`), Store-owned. Spec shapes verbatim, no behavior, no eviction.                                                                                                                                                                            | Store-init test (`execution_proofs` empty at genesis); `Seen` maps default-empty.                                                             | `cargo check/test -p fork_choice_store`           |
| 3. Task plumbing        | `fork_choice_control`: `ExecutionProofAction`, `ExecutionProofOrigin`, `MutatorMessage::ExecutionProof` variant, `ProcessExecutionProofTask` with stubbed `run()` (`IS_NULL` → `Ignore`, else stub `Ignore` + `TODO(eip8025-grandine)`), `spawn_execution_proof_task`, `LowPriorityTask::ExecutionProof` variant, mutator arm (stub: no store insert yet, signals p2p per `origin.split()`). | Task smoke test: stub pipeline against `MockProofEngine` (both settings) asserting the emitted outcome message; `NullProofEngine` → `Ignore`. | `cargo check/test -p fork_choice_control`         |

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
