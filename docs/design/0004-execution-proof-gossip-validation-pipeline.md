---
status: Draft
authors: [keshavsharma25]
workstream: verification
eip_repo: frisitano/EIPs
eip_sha: 9207c6011f526bd40abd79649484a1a342585bd4
consensus_specs_sha: 6946894b02f6e95bf5b1daf4ce39a3c3244a1e41
grandine_upstream_sha: e896cbd41bf885852c501ac0de2a6ff4e7376cc1
superseded_by:
---

# 0004 — Execution-proof gossip validation pipeline

## Context

The proof-engine boundary, store shapes, and control task exist. This design
replaces the validation stub in `fork_choice_control/src/tasks.rs:752` and the
bare-message storage stub in `fork_choice_control/src/mutator.rs:2641`.

Implement the `feat/simplify-eip8025` consensus-specs snapshot pinned above,
on top of Grandine's `feature/proof-service-task-plumbing-verifier-prover-split`
(PR #15). It uses progressive `ProofData`, `SSZNewPayloadRequest`, and runtime
proof-type checks. Later spec and peer-owned type changes need a separate PR.

## Goals and non-goals

**Goals:** Authenticate the envelope, bind it to the accepted payload, verify
the proof, and return ACCEPT / REJECT / IGNORE. Store verified bare messages
and Seen keys; prune all three maps at finalization. Keep the existing
read-only validation / mutator-only apply split.

**Non-goals:** New P2P topics, subscriptions, RPC or HTTP ingress; proof
generation; changes to `ProofProver<P>`; peer-owned container changes;
restart recovery or a new cache-retention policy.

## Design

```text
existing ingress → Controller::spawn_execution_proof_task [low priority]
  → ProcessExecutionProofTask (snapshot; Null engine → Ignore)
  → Store::validate_execution_proof_gossip (read-only)
  → MutatorMessage::ExecutionProof { result, origin }
  → Mutator::handle_execution_proof
      Accept → Store::apply_execution_proof + update_store_snapshot()
      Ignore / Err → existing origin.split() signalling
```

`fork_choice_store` adds a `proof_engine` dependency:

```rust
pub fn validate_execution_proof_gossip(
    &self, signed_proof: Arc<SignedExecutionProofEnvelope>,
    proof_verifier: &dyn ProofVerifier,
) -> Result<ExecutionProofAction>;

pub fn apply_execution_proof(
    &mut self, signed_proof: Arc<SignedExecutionProofEnvelope>,
) -> bool; // false for a duplicate racing snapshot validation → Ignore
```

**Gossip order** — `specs/_features/eip8025/p2p-interface.md`,
`validate_execution_proof_gossip`:

| #   | Check                                           | Result                             |
| --- | ----------------------------------------------- | ---------------------------------- |
| 0   | Hash the bare envelope                          | Before any IGNORE                  |
| 1   | Proof root already Seen for this block          | IGNORE                             |
| 2   | Block unknown                                   | IGNORE                             |
| 3   | Known block lacks valid post-state              | REJECT                             |
| 4   | Payload unavailable                             | IGNORE                             |
| 5   | Verified proof already stored for (block, type) | IGNORE                             |
| 6   | Prover already Seen for (block, type, index)    | IGNORE                             |
| 7   | `verify_execution_proof_envelope` fails         | REJECT                             |
| 8   | `get_execution_proof`                           | Build engine input                 |
| 9   | Mark authenticated root and prover Seen         | Before engine verification; see D3 |
| 10  | Proof-engine verification fails                 | REJECT; otherwise ACCEPT           |

**Envelope authentication** — `beacon-chain.md`,
`verify_execution_proof_envelope`: check (1) envelope/payload block-root
equality, (2) validator index bounds, (3) `0 < len(proof_data) <= MAX_PROOF_SIZE`
(4 MiB), (4) proof type in `{1, 2, 3}`, (5) active validator at
`get_current_epoch(state)` in the proven block's post-state, then (6) BLS
signature over the envelope's signing root with the validator pubkey and
`DOMAIN_EXECUTION_PROOF` at `compute_epoch_at_slot(state.slot)`. Reuse
`SignForSingleForkAtSlot::verify`; do not duplicate PR #7's signing code.

**Binding** — `beacon-chain.md`, `get_execution_proof`: build
`SszNewPayloadRequest<P>` from the accepted envelope's payload, parent root,
and execution requests, plus versioned hashes of
`state.latest_execution_payload_bid.blob_kzg_commitments` (using the existing
`kzg_commitment_to_versioned_hash` helper). Hash the request into
`PublicInput.new_payload_request_root`. Set `successful_validation = true`,
`chain_id = DEPOSIT_CHAIN_ID`, and `schema_id = STATELESS_INPUT_SCHEMA_ID`
(`0x1501`); carry the envelope's proof type and data into `ExecutionProof`.
Do not replace the existing Grandine request type in this series.

**Storage and retention** — Store the bare `.message` under `(block root,
proof type)` using `entry().or_default()`. Seen maps hold proof roots per block
and prover keys `(block root, type, validator index)`. Only the mutator writes;
a duplicate racing validation becomes IGNORE. Prune all three maps together
with the existing finalization sweep.

**Grandine mappings and deviations**

- **D1 — payload:** `store.payloads` is a set, not the spec's envelope map.
  Fetch the accepted envelope from `ExecutionPayloadEnvelopeCache` after the
  payload-availability check; a cache miss is IGNORE, never bind other data.
- **D2 — post-state:** use the proven block's chain-link state. If a block is
  known but its post-state is unavailable, REJECT, not IGNORE.
- **D3 — Seen timing:** the spec marks authenticated attempts before engine
  verification; our validate/apply split needs a mutator path for engine-invalid
  attempts. Decide before PR2 (below).
- **D4 — helper split:** store helpers separate authentication from binding;
  their doc comments link to the corresponding spec functions.
- **D5 — decode bound:** existing `ProofData` rejects oversized bytes at
  construction/decoding. Keep the runtime size and type checks in envelope
  authentication; do not add a proof-type decode guard. Test oversized input
  at decoding, since it cannot become an in-memory `ProofData`.

## Trade-offs and alternatives

Snapshot validation keeps slow proof-engine work off the mutator, but apply
must reject a racing duplicate. Carry `Arc<SignedExecutionProofEnvelope>` on
Accept so the mutator can record the prover; persist only `.message`.

**D3 is open:** marking Seen only on Accept lets the same authenticated,
engine-invalid proof repeatedly consume BLS and engine work. The spec marks it
Seen before the engine call. One option is a `RejectVerified(signed)` result:
the mutator marks that attempt Seen, then signals REJECT. Authentication
failures must not mark Seen.

## Security and compatibility

Keep the pre-decode `MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE` cap and the
`ProofData` bound. The pinned check order hashes an envelope before the early
IGNOREs; dedup only saves repeat BLS/engine work if D3 is resolved. An invalid
proof does not invalidate an accepted payload or change fork-choice weights.
A null proof engine returns IGNORE before validation. Cache eviction yields
IGNORE, not verification against an unrelated payload.

## Implementation and testing

Stack on `feature/proof-service-task-plumbing-verifier-prover-split`. Use the
pinned spec and current Grandine types for all three PRs; run affected tests,
formatting, and clippy (`scripts/ci/clippy.bash`).

| PR  | Scope                                                                          | Tests                                                                                                                                                                              |
| --- | ------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PR1 | IGNORE/REJECT errors; `verify_execution_proof_envelope`; `get_execution_proof` | Valid signature; root, empty data, type, index, activity, and BLS failures; oversize decode rejection; pre-Gloas binding error; fields/root against a separately assembled request |
| PR2 | Ordered gossip validation; apply, Seen, mutator Accept, pruning; settle D3     | Every IGNORE/REJECT path including missing post-state; duplicate race; engine-invalid replay; pruning                                                                              |
| PR3 | Replace task stub; finish controller/mutator wiring                            | Mock engine: Accept, Reject, Null Ignore, duplicate Ignore; replay engine-call count                                                                                               |

## Deferred spec and type alignment

- A fixed expected binding root derived from the pinned pyspec or another
  implementation would strengthen PR1's tests. The pinned tests compute it
  dynamically; PR1's separate request assembly is sufficient for this series.
  Re-derive a golden root if the request shape changes.
- [#5593](https://github.com/ethereum/consensus-specs/pull/5593) proposes
  `ByteList` proof data, a proof-type decode guard, a different request shape,
  and changed gossip/auth checks. Re-pin and re-test SSZ roots, envelope size,
  and binding when the peer-owned type work lands; do not anticipate it here.
- Draft [#5639](https://github.com/ethereum/consensus-specs/pull/5639) would
  remove proof production from the consensus-layer interface. Keep the current
  verifier/prover boundary for this series.

## Blockers and open questions

- Decide D3's engine-invalid Seen path before PR2.
- Confirm cache lifetime versus retained blocks/proofs; cache miss remains IGNORE.
- Check whether a known Grandine block can lack post-state, and how to retain
  the spec's REJECT behavior in that case (D2).
