---
status: Draft
authors: [keshavsharma25]
workstream: verification
eip_repo: ethereum/EIPs
eip_sha: 9207c6011f526bd40abd79649484a1a342585bd4
consensus_specs_sha: 12c8b24bc09b12f62688d439ba20ef00b1fdfe1f
ssz_specs_sha: b42af0a2265c25f5353cace185d066e8d39dbd6e
grandine_upstream_sha: a63876249043020d8ebe3d410c5cf31b19be1cc2
grandine_pr_base: c736de6f05b2bdecb3d166e3e396dc6ab91d2e1f
grandine_stack_tip: e014f0fbed46046c5dfdf0d9ac1078bdd366d406
superseded_by:
---

# 0010 — Admit and complete execution proofs against live Store context

## Context

This document covers **PR2: the Store lifecycle**
([Grandine #25](https://github.com/eip8025-grandine/grandine/pull/25)) within the
[execution-proof validation pipeline](0004-execution-proof-gossip-validation-pipeline.md).
PR1 ([#17](https://github.com/eip8025-grandine/grandine/pull/17)) is merged into
the selected base and provides authenticated, payload-bound preparation.
PR2 adds live admission and completion; PR3 will connect them to engine
workers and terminal gossip/API replies. PR2 does not enable live proof processing.

The spec pins retain the selected progressive-list semantics and zkBoost
binding conventions described in 0004, not a claim of alignment with latest
spec master. `grandine_upstream_sha` pins the underlying upstream merge-base;
`grandine_pr_base` and `grandine_stack_tip` delimit the implementation.

## Goals and non-goals

**Goals**
- Authorize verification only against applicable live payload context.
- Record both duplicate keys atomically before engine work.
- Preserve exact proof/context association through a consuming handoff.
- Apply each eligible result independently and store only the first valid proof.
- Prevent stale work from reviving retired targets without changing consensus behavior.

**Non-goals**
- Engine dispatch, snapshot publication, origin replies or new proof ingress.
- Physical pruning, durable deduplication, retries or exclusive per-block/type gates.
- Real-proof interoperability, proof generation or memory/latency guarantees.

## Design

```text
PR1: authenticate and bind on a snapshot
  → PR2: live admission; consume duplicate keys together
  → PR3: verify the exact admitted input off-mutator
  → PR2: live completion; preserve first-valid storage
  → PR3: publish changes and reply to the origin
```

### Accepted context is explicit

Ordinary payload submission remains authoritative. Only after successful
execution-layer notification does validation capture the proof statement,
best-effort, and carry it together with the accepted payload to Store application.
Capture failure cannot veto ordinary acceptance; accepting a replacement without
proof context clears the previous authority. Custom-state validation cannot
manufacture this live authority.

Applicability requires the target to be the latest finalized anchor or a
retained unfinalized block, an accepted-payload marker, and equality of the
**complete public input**: request root, successful-validation flag, chain ID
and schema ID. This is not an EL VALID verdict. Cache residency, pointer
identity and unrelated snapshot updates do not determine authority. Fresh
preparation still needs payload bytes; already-prepared work need not retain them.

### Admission records attempts, not successes

A serialized live Store operation checks retention, context, existing verified
storage and both duplicate keys: `(block, bare-envelope root)` and
`(block, proof type, validator)`. It either declines without writes or records
both keys and returns protected, consuming verification authority.

Invalid proofs, local faults and dispatch failures do not release these keys.
Distinct attempts with different keys may overlap for the same block/type;
there is no additional exclusive in-flight gate.

### Completion checks applicability before the report

Only a report paired with the admitted input can complete. Store remains
independent of the proof-engine crate; PR3 must map engine `Valid`, `Invalid`
and `Undetermined` exhaustively to Store `Valid`, `Invalid` and `LocalFault`.

| Live completion | Result | Verified storage |
| --- | --- | --- |
| Retired or changed context, regardless of report | Discard locally | No insertion |
| Applicable Valid | Accept | Store the first bare envelope for block/type; never replace |
| Applicable Invalid | Reject | Unchanged |
| Applicable LocalFault | Local outcome, not peer rejection | Unchanged |

An already-admitted applicable Valid still accepts if another attempt stored
first; an Invalid still rejects. Occupancy suppresses new admissions, not
independent reports. No completion rolls back Seen or changes payload status,
beacon state or fork-choice weights. Historical resident entries cannot
authorize retired targets; physical reclamation is deferred.

## Trade-offs and alternatives

- **Live checks rather than snapshot-only authorization:** extra handoffs close
  duplicate and stale-context races before engine entry and storage.
- **Permanent Seen rather than retry ownership:** simpler attempt semantics,
  but the same keys cannot retry after a local failure.
- **Store `LocalFault` rather than `Unavailable`:** the category covers failure
  to obtain a proof verdict locally, including dispatch failure before an engine
  runs and engine `Undetermined` reports. `Unavailable` would suggest missing
  proof data or temporary availability, neither of which describes every case.
  The current engine calls this `Undetermined`, not `Unavailable`; PR3 adapts
  that report without coupling Store's vocabulary to the engine. Missing context
  has separate preparation/discard outcomes. `LocalFault` implies neither peer
  invalidity nor retry eligibility: admitted Seen keys remain consumed.
- **First-valid storage rather than canonical proof selection:** preserves
  existing evidence without erasing another admitted attempt's result.
- **Best-effort context capture rather than coupled payload acceptance:** proof
  bookkeeping cannot break ordinary acceptance, but capture has synchronous cost
  and a failed capture leaves no applicable proof context.

## Security and compatibility

Authentication precedes admission, so forged signatures cannot consume another
validator's key. Unavailable context and local verifier failures are not
peer-invalidity evidence. Withheld or invalid proofs cannot change consensus
validity. Nodes opting out must bypass preparation/admission in PR3.

Deduplication prevents duplicate admitted work, not unlimited distinct attempts
or duplicate preparation. Physical maps and old snapshots may retain data beyond
logical retirement. Queue/concurrency limits, network size enforcement and
cleanup remain separate requirements; no bounded-memory or constant-latency
claim is made. The handoff does not provide panic/crash-safe completion or
restart recovery.

## Implementation and testing

PR1 provides preparation; PR2 implements the Store transitions plus accepted
payload/context transport; PR3 supplies service integration. These changes are
EIP-8025-specific, not a standalone generic Store feature.

Unit coverage exercises atomic declines, both duplicate keys, stale snapshots,
context replacement, local faults, overlapping completion orders, first-valid
storage, skipped-boundary anchors and recent orphans. A separate Control
regression protects ordinary payload submission, replies and events when
context capture succeeds, fails or submission fails.

Post-rebase tests and strict CI lint remain verification gates. Store tests
model verification reports: they do not establish actual engine-call counts,
terminal reply uniqueness, spec conformance or devnet interoperability. PR3
must test the orchestration; independent selected-target proof vectors belong
to the backend/binding workstream.

## Open questions

No additional Store-policy decision is required for PR2. Acceptance still needs
final verification and review. Dispatch ownership, snapshot ordering and terminal
signaling are PR3 obligations; physical cleanup and live-exposure resource policy
remain follow-ups.
