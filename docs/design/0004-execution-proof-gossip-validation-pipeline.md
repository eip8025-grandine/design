---
status: Draft
authors: [keshavsharma25]
workstream: verification
eip_repo: ethereum/EIPs
eip_sha: 9207c6011f526bd40abd79649484a1a342585bd4
consensus_specs_sha: 12c8b24bc09b12f62688d439ba20ef00b1fdfe1f
grandine_upstream_sha: 34da987dc6e1eec7be6afb6fb3c6edf317f1c32c
grandine_stack_tip: df9f361167a645dac569802e8dd9d69952e4a761
superseded_by:
---

# 0004 — Execution-proof gossip validation pipeline

## Context

Execution proofs are supplementary, non-consensus artifacts. This design
covers their authentication, binding to accepted payloads, verification and
storage without changing payload validity or fork choice.

The pipeline separates **prepare → admit → verify → complete**. Workers
establish trusted input and verify proofs; the mutator owns live admission,
proof bookkeeping and storage.

The pinned consensus-specs baseline uses bounded `ByteList` proof data,
`NewPayloadRequest` and runtime proof-type checks. Grandine retains
[#18](https://github.com/eip8025-grandine/grandine/pull/18)'s selected extension:
progressive versioned hashes from [consensus-specs #5643](https://github.com/ethereum/consensus-specs/pull/5643)
at `b9130f1c`, with [#5642](https://github.com/ethereum/consensus-specs/pull/5642)'s
selected progressive-list limit semantics.

`grandine_stack_tip` pins the reference revision; `grandine_upstream_sha` is
its underlying `upstream/develop` merge-base. The proof-engine boundary and
Store/control shapes are dependencies of this three-PR design.

## Goals and non-goals

**Goals:** Trusted preparation, atomic pre-engine deduplication, correct
verdicts, first-valid storage and consistent retirement. Keep expensive work
off the mutator and bookkeeping writes on it.

**Non-goals:** New topics, subscriptions, RPC/HTTP ingress; unselected
container/spec migrations; restart recovery or a new cache-retention policy.

## Design

```text
internal submission → low-priority preparation worker (coherent Store snapshot)
  → mutator: live checks + atomic paired Seen admission
      decline → terminal origin result
      admit   → publish bookkeeping snapshot; schedule engine worker
  → engine worker: verify the exact admitted ExecutionProof
  → mutator: applicability check + completion/storage
      → publish changed bookkeeping; terminal origin result
```

Workers must not hold a mutator lock or wait for admission replies. Expensive
BLS, state materialization, payload hashing and engine calls stay off the
mutator; only short live-store decisions are serialized.

### Preparation establishes trust, not admission

Preparation returns a protected prepared carrier, IGNORE, protocol rejection
or local error. It cannot authorize engine entry from a potentially stale
snapshot or change proof bookkeeping. State/public-key cache warming is allowed.

**Ordered checks** follow the pinned
[gossip validator](https://github.com/ethereum/consensus-specs/blob/12c8b24bc09b12f62688d439ba20ef00b1fdfe1f/specs/_features/eip8025/p2p-interface.md#new-execution_proof).
Let B = target block root, T = proof type, V = validator index and
R = the bare `ExecutionProofEnvelope` root, not its signed wrapper.

| Order | Check/action                                              | Outcome on failure       |
| ----- | --------------------------------------------------------- | ------------------------ |
| 1     | Non-empty proof                                           | REJECT                   |
| 2     | Supported proof type                                      | REJECT                   |
| 3     | Known B                                                   | IGNORE                   |
| 4     | No verified proof for (B, T)                              | IGNORE                   |
| 5     | Compute R; require unseen (B, R)                          | IGNORE                   |
| 6     | Require unseen (B, T, V)                                  | IGNORE                   |
| 7     | Accepted payload and exact trusted target state available | IGNORE                   |
| 8     | Authenticate signed envelope                              | Typed protocol rejection |
| 9     | Bind engine input; construct protected carrier            | Local error              |

**Trusted context:** a peer supplies proof bytes, not trusted public input.
Require the payload-acceptance marker plus matching full envelope for B/slot;
the marker alone supplies neither payload data nor an EL VALID verdict.
Use B's exact post-state, not head state. Check resident/cache-hit association,
ancestry seeds before replay, and output before cache publication so a local
reconstruction error cannot become trusted context. Unavailable/inconsistent
context is local IGNORE, not evidence against the peer.

**Authentication/binding:** index → non-empty data → supported type `{1, 2, 3}`
→ activity → BLS, using the target state's epoch and `DOMAIN_EXECUTION_PROOF`.
Reconstruct `NewPayloadRequest` from accepted payload/parent/requests and
state-derived versioned hashes. `PublicInput` contains its root, success=true,
configured deposit chain ID and schema `0x1501`, never peer-chosen input.

**Protected provenance:** authentication of one envelope cannot authorize
another input. Carry the authenticated envelope, cached R, prover key and
immutable bound proof together. Context identity is B plus **complete**
`PublicInput`, not an allocation identity. Preparation implies neither
engine validity nor final gossip ACCEPT; binding/consistency errors are local.

### Serialized admission closes the snapshot race

Workers validate against immutable Store snapshots. Two workers can observe
the same keys as unseen and both enter the verifier before either completion
updates the Store. Snapshot-local Seen updates cannot serialize that decision;
engine entry therefore requires admission against the live Store. This addresses
the [Seen-timing concern discussed in PR10](https://github.com/eip8025-grandine/grandine/pull/10#discussion_r4308234602).

For two candidates sharing either duplicate key, prepared from snapshot S0:

| Step                 | Without live admission          | With serialized admission                  |
| -------------------- | ------------------------------- | ------------------------------------------ |
| Workers A/B check S0 | Both see unseen keys            | Both may prepare                           |
| A reaches live Store | No pre-engine transition        | Check and insert both keys                 |
| B reaches live Store | Still free to enter engine      | Observe A's key; decline without insertion |
| Engine work          | Duplicate verification possible | Only the admitted competitor enters        |

One mutator transition checks retention, trusted context, verified (B, T)
storage and **both** Seen keys, then inserts (B, R) and (B, T, V) together
before engine scheduling. A decline inserts neither key; partial insertion
could suppress another genuine attempt. Authentication precedes insertion
so forged signatures cannot consume another validator's key.

Seen records **authenticated attempts**, not successful proofs. Both keys
remain consumed until retirement to suppress repeated invalid proofs and
changed bytes under the same prover key. This sacrifices same-key retry after
local/scheduling failure, even if work never starts; no pending ownership,
rollback or retry-release protocol is promised.

**Live applicability:** admission/completion require retained B and unchanged
trusted statement provenance. Use block-root invariants and content identity,
not expensive reconstruction on the mutator. Unrelated publications,
equivalent allocations and another proof finishing first do not change the
statement. Never silently rebind to different input. Inapplicability is local
IGNORE, not panic/REJECT; cache residency alone is not context identity.

### Completion separates attempt verdicts from verified storage

Completion accepts only a verdict tied to the exact admitted engine input,
never an arbitrary envelope plus a claimed verification result.

Same R/different V and different R/same (B, T, V) are duplicates. Distinct R
and prover keys may overlap before verified storage; an exclusive (B, T) gate
would exceed the spec's two-key rule. Completion rechecks applicability before
interpreting the verdict:

| Completion                      | Gossip outcome | Storage                                                        |
| ------------------------------- | -------------- | -------------------------------------------------------------- |
| Engine true                     | ACCEPT         | Store the first valid bare message for (B, T); never overwrite |
| Engine false                    | REJECT         | None; retain Seen                                              |
| Separately reported local fault | IGNORE         | None; retain Seen                                              |
| Retired/inapplicable target     | IGNORE         | No insertion or resurrection                                   |

If another valid attempt stores first, an admitted true still ACCEPTs without
overwriting; an admitted false still REJECTs. Only **new admissions** IGNORE
occupied storage. Proof bytes are not canonical. Protected input/result
provenance permits one off-mutator engine call instead of repeat verification
on apply.

### Retirement is shared by admission, storage and Seen

Keep precisely the latest finalized anchor and roots in
`unfinalized_locations`. During finalization pruning, after updating that
domain and removing orphans, apply it to `execution_proofs`,
`execution_proof_roots` and `execution_proof_provers` together. Independent
Seen eviction could reopen duplicate admission while work still runs.
Keeping verified proofs after their target blocks are retired would retain
history this pipeline no longer uses.

Remove older finalized ancestors and discarded branches, including recent-slot
orphans. Retirement follows block identity, not an epoch-wide slot cutoff.

Admission/completion use the same domain, so historical chain-link access
cannot readmit retired targets and late results cannot recreate pruned records.
The local retirement IGNORE must not preempt early empty/type REJECTs.
A skipped epoch-boundary slot may leave the retained anchor without a cached
full payload envelope; unavailable context is IGNORE, not substitution.

### Non-blocking orchestration preserves the authority boundary

Preparation sends its result and exits; admission schedules another worker,
rather than occupying shared workers waiting for mutator replies. Handoffs
preserve provenance and origin; snapshots published after admission and
storage/pruning avoid redundant work but do not replace live checks.

Keep `ProofVerifier::verify_execution_proof(ExecutionProof) -> bool`, using
`hash_tree_root(public_input)`. The null-engine guard returns IGNORE **before**
preparation/admission: disabled is not invalid. Enabled true/false are proof
verdicts; separately reported context/scheduling faults are local IGNORE,
never generic peer REJECT. No backend error/cancellation API is added.

Transfer origin ownership between stages for one normal-path terminal gossip/
API result. Declines finish without engine work. Scheduling failure after
admission reports local IGNORE and leaves Seen consumed.

## Trade-offs and alternatives

| Choice                                          | Rejected alternative                       | Deciding trade-off                                                    |
| ----------------------------------------------- | ------------------------------------------ | --------------------------------------------------------------------- |
| Live pre-engine paired admission                | Snapshot-only or completion-only Seen      | Extra handoff buys atomic duplicate exclusion before expensive work   |
| Permanent Seen until retirement                 | Retryable pending ownership/release        | Less lifecycle/recovery machinery, but no same-key local-fault retry  |
| Two-key exclusion                               | Exclusive per-(B, T) in-flight gate        | Preserves independent attempts; needs separate finite resource limits |
| Content-based trusted context                   | Pointer/global-generation identity         | Equivalent allocations/unrelated updates do not discard valid work    |
| Non-blocking workers, short mutator transitions | Waiting workers or verification on mutator | More message stages, but no verifier latency on the serialized path   |

## Security and compatibility

`ProofData` enforces its 4 MiB bound at construction/decoding. Ingress must
enforce the 4,194,449-byte signed-envelope cap before decoding; defining the
constant alone does not enforce a network limit. Proof type remains a u8 with
runtime support checks, not a decode guard.

Cheap prechecks avoid unnecessary proof hashing/BLS; live admission prevents
duplicate engine calls, not duplicated pre-admission BLS or unlimited distinct
attempts. Invalid/withheld proofs cannot change payload status, beacon state
or fork-choice weights. No durable Seen/restart or abrupt-stop recovery is
promised; retained-domain pruning is not a hard memory cap during non-finality.

Finite queued-byte/work and concurrency limits are prerequisites for live
ingress. Low priority is not isolation or DDoS protection. Preparation includes
O(proof bytes) hashing plus BLS/replay/binding; engine cost is backend-specific.
Map checks are expected-average O(1), subject to copy-on-write costs; pruning
scans bookkeeping. Existing logs/metrics should distinguish stage outcomes,
duplicates, local faults, pruning and queue/engine latency without proof data
or high-cardinality labels.

## Implementation and testing

| PR                                                                | Responsibility                                                                                                       | Required evidence                                                                                                           |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| PR1 ([#17](https://github.com/eip8025-grandine/grandine/pull/17)) | Authentication/binding helpers, ordered preparation, trusted context and protected prepared provenance               | Precedence, signature/type/bounds, replay/cache safety, carrier consistency and unchanged proof bookkeeping                 |
| PR2                                                               | Live applicability, paired permanent-Seen admission, admitted/completion provenance, first-valid storage and pruning | Atomic exclusion/no partial insertion, local-fault/invalid replay suppression, distinct verdicts, retention/no resurrection |
| PR3                                                               | Messages, non-blocking workers, engine scheduling, snapshots, opt-out and terminal signalling                        | Controlled interleavings, actual engine-call counts, published bookkeeping and single terminal replies                      |

Each PR builds/tests independently. PR1 has no lifecycle dependency; PR2 keeps
admission/completion/pruning together; PR3 connects the full enforced lifecycle.

Boundary cases include early empty/type precedence; both-key races; true/true
and mixed verdict orders; local faults; anchor/skipped-slot retention; recent
orphans and late results. PR3 must prove actual engine-call exclusion and
single replies with controlled interleavings; Store-only tests cannot.

Independent selected-target binding vectors establish interoperability;
Grandine-derived roots are regression coverage, not independent oracles.

## Deferred spec and type alignment

Use the pinned global gossip model, not type-mapped subnets. #5729's
payload-dependency removal and unselected #5642 changes require separate
alignment; this design requires accepted payload context and retains the
size caps. Rebaseline roots/bounds together if the request shape changes.
Historical proof serving, durable recovery and live ingress are separate
workstreams.

## Open questions

- The integration must demonstrate the sufficient cheap live-context predicate
  from B's trusted acceptance/provenance; pointer equality, cache residency and
  global snapshot generations are not substitutes.
- Real-backend operational-failure semantics and measured resource budgets
  require separate integration/exposure design; boolean false must not be
  overloaded with local unavailability.
