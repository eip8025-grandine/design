---
status: Draft
authors: [keshavsharma25]
workstream: verification
eip_repo: ethereum/EIPs
eip_sha: 9207c6011f526bd40abd79649484a1a342585bd4
consensus_specs_sha: 12c8b24bc09b12f62688d439ba20ef00b1fdfe1f
ssz_specs_sha: b42af0a2265c25f5353cace185d066e8d39dbd6e
grandine_upstream_sha: a63876249043020d8ebe3d410c5cf31b19be1cc2
grandine_pr_base: 1c0cc89793d760cb4e75b6c8d5732c8ba318bcb0
grandine_stack_tip: c4e1bdbeb323df63fca2c9524c398727606c60cf
superseded_by:
---

# 0013 — Integrate the execution-proof verification service

## Context

This change connects the [execution-proof validation pipeline](0004-execution-proof-gossip-validation-pipeline.md)
and [Store lifecycle](0010-execution-proof-store-lifecycle.md) to Grandine's
Controller, worker pool and mutator. It verifies supplementary execution proofs
and returns gossip/internal API results without changing payload validity or
fork choice.

The implementation branch is `feature/proof-service-integration`, based on
`feature/proof-store-lifecycle`. EIP, consensus-specs and SSZ pins retain 0010's
selected baseline and binding conventions, not latest-master alignment.
`grandine_upstream_sha` identifies the underlying upstream merge-base;
`grandine_pr_base` and `grandine_stack_tip` delimit this integration.

## Goals and non-goals

**Goals:** explicit verifier ownership; expensive work off-mutator; authenticated
input preserved through live admission/completion; Seen publication before engine
entry; storage publication before ACCEPT; local failures distinguished from
peer-invalid proofs.

**Non-goals:** networking subscriptions/routing, new HTTP ingress, production
backend configuration, proof generation, new Store/consensus policy, physical
pruning or restart recovery.

## Design

```text
Controller: signed proof + origin; use owned engine
  -> preparation worker: Null guard; otherwise snapshot authentication/binding
       IGNORE/rejection -> mutator -> terminal reply
       prepared -> mutator: live admission
                     decline -> terminal IGNORE
                     admit   -> publish Seen; dispatch verification worker
  -> verification worker: verify admitted input; return protected completion
  -> mutator: live completion; publish storage before ACCEPT
  -> existing gossip validation response or internal API oneshot reply
```

Preparation can also return IGNORE or a typed rejection through the mutator
without entering the verifier.

### Controller owns verifier selection

`Controller::new` requires an `Arc<dyn ProofEngine>`, retained once and cloned
internally for work. Submission cannot choose a different verifier. Runtime and
ordinary test callers explicitly supply `NullProofEngine`; private service-test
constructors inject configured engines. Null bypass returns IGNORE before
preparation/admission, without consuming Seen keys. Backend selection remains
separate; this branch does not activate a production verifier.

### Payload context capture follows verifier selection

Controller derives proof-context capture policy from `!proof_engine.is_null()`.
Initial payload workers receive that boolean; Mutator retains the same
Controller-derived policy for the common retry path used by block-, state- and
data-delayed payloads. Verifier selection is fixed for this Controller; this is
not a new runtime configuration or per-envelope policy.

Store accepts an explicit `capture_proof_context` argument rather than depending
on the engine or selecting verification policy. Disabling capture bypasses neither
ordinary payload validation nor its execution-layer notification. Only when capture
is enabled, after successful notification, does Store best-effort reconstruct the proof
`PublicInput` and carry it with the accepted payload. Capture failure or disabled
capture leaves that evidence absent, without changing acceptance, replies,
events or payload application. Applying a replacement without paired evidence
still clears any obsolete proof context; opting out must not preserve stale
authority. Enabled capture retains its existing ordering and binding semantics.

### Workers compute; the mutator authorizes and publishes

Preparation uses a coherent snapshot. Only recognized, typed envelope-validation
errors become peer rejection; unavailable context and local binding/preparation
failures become IGNORE.

A protected prepared carrier cannot authorize engine entry. The mutator consumes
it through live Store admission, closing stale-snapshot races. Successful admission
records both duplicate keys—`(block, bare-envelope root)` and
`(block, proof type, validator)`—and publishes their snapshot **before** dispatch.
The worker then verifies the exact admitted input and returns a consuming
completion with its origin. Authority is neither reconstructed nor cloned, and
workers do not wait for admission replies.

Engine `Valid`, `Invalid` and `Undetermined` map exhaustively to Store `Valid`,
`Invalid` and `LocalFault`. Completion reuses existing Store policy: retention
and full accepted-context applicability precede occupancy; the verdict matters
only while `(block, proof type)` is vacant.

| Live completion | Terminal result | Storage effect |
| --- | --- | --- |
| Retired or changed accepted context | IGNORE | No insertion |
| Occupied, regardless of report | IGNORE | Preserve first resident proof |
| Applicable and vacant; Valid | ACCEPT | Store bare envelope, then publish |
| Applicable and vacant; Invalid | REJECT | None |
| Applicable and vacant; LocalFault | IGNORE | None |

Distinct admitted attempts may overlap; there is no exclusive per-block/type
gate. Later completions cannot replace the first applicable Valid envelope or
undo permanent Seen entries. Gossip/API replies reflect the live Store result,
not the raw engine verdict.

### Failed dispatch preserves authority and replies

A stopped pool returns the original verification task, including admitted carrier
and origin. The mutator consumes it into a local-fault completion through the same
live path: no verifier call, terminal IGNORE, Seen retained. Successful enqueue
provides no panic/crash-safe delivery guarantee. No production drain or worker-join
mechanism is introduced; ordinary processing waits are not proof-drain barriers.

### Targeted traces expose outcomes

Completion traces identify block root, proof type and validator index for ACCEPT,
REJECT and discarded IGNORE, including rejection/discard reasons. Preparation
IGNORE and admission `AlreadyVerified` also emit identifiable traces. Existing
debug diagnostics remain available. This is not a promise of one self-contained
trace for every submission.

## Trade-offs and alternatives

- **Mandatory Controller-owned engine over optional/per-submission selection:**
  explicit policy ownership, at the cost of Null arguments and narrow caller updates.
- **Controller-derived capture boolean over a Store engine dependency:** disabled
  nodes avoid proof-statement reconstruction/storage on ordinary payloads, at the
  cost of explicitly forwarding the same policy to initial and retry workers.
- **Worker stages around live admission over snapshot-only verification:** extra
  messages close duplicate-key races without expensive work on the mutator.
- **Ownership-returning dispatch over dropping failed work:** preserves authority
  and replies; only the stopped-pool error path boxes the returned task.
- **Permanent Seen and first-valid storage over retries or exclusive attempts:**
  preserves Store semantics and concurrency, but local faults do not release keys.

## Security and compatibility

Authenticated, payload-bound carriers prevent substitution of peer-chosen public
input. Local failures are not peer-invalidity evidence. Early IGNORE trace identity
describes the submitted envelope, not an authenticated validator. Proof results
change neither payload status nor beacon state/fork-choice weights. Null callers
opt out before expensive proof validation and proof-context capture, not ordinary
payload validation or EL notification; the required constructor argument is an
intentional internal API change, not new network behavior.

Low-priority scheduling and deduplication do not bound distinct malicious attempts,
queue growth or retained snapshots. Resource limits, physical reclamation and
crash-safe/durable recovery remain separate work.

## Implementation and testing

| Surface | Implementation |
| --- | --- |
| `fork_choice_control/src/controller.rs`, `specialized.rs` | Engine ownership, gossip submission seam and explicit constructor/test injection. |
| `tasks.rs`, `messages.rs` | Snapshot preparation, typed error classification, protected stage messages and consuming verification. |
| `mutator.rs`, `thread_pool.rs` | Live transitions, publication ordering, origin replies, ownership-returning failed dispatch and traces; retained capture policy for payload retries. |
| `fork_choice_store/src/store.rs` | Explicit capture argument; enabled-only best-effort evidence paired with ordinarily accepted payloads. |
| `runtime`, `http_api`, root `Cargo.lock` | Explicit Null constructor arguments and direct workspace `proof_engine` dependency edges; no new backend/dependency version or HTTP endpoint. |

Tests in `tasks.rs`, `extra_tests.rs` and private mutator fixtures cover:
- Null bypass, configured engine selection, typed errors and exact origin replies.
- Authenticated accepted-payload flow with concrete `ZkboostMock` encoding:
  malformed proofs, mismatched public values and supported but unconfigured types.
- Both duplicate keys, concurrent completion orders, independent proof types and
  preservation of the actual first resident envelope.
- Accepted-context replacement and retirement through live Store transitions;
  trusted chain fixtures model lifecycle behavior, not full block-import validity.
- Gossip/API verdict and stopped-dispatch matrices, retained Seen and exact terminal
  accounting. Gated workers, bounded waits and closed-producer completion batches
  avoid sleep-based assertions or treating ordinary waits as proof completion.
- Ordinary payload submission with enabled and Null engines, failed capture and
  failed submission, preserving replies, events and payload application. The
  shared retry seam checks both capture policies. Store regressions check the
  exact EL request, zero disabled capture attempts, ordinary rejection, and
  clearing old evidence when a capture-disabled replacement is applied.

Broader verification covers Store/engine regressions, runtime/HTTP compilation,
workspace formatting and strict CI lint. These tests establish orchestration and
mock-backend consistency, not independent cryptographic vectors, real-SP1
verification or devnet/network interoperability.

## Open questions

No further service-boundary decision is required for this integration. Networking,
backend configuration, independent binding/backend vectors, deployment resource
limits and physical proof-bookkeeping reclamation remain follow-up work.
