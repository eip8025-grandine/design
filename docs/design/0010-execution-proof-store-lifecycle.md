---
status: Draft
authors: [keshavsharma25]
workstream: verification
eip_repo: ethereum/EIPs
eip_sha: 9207c6011f526bd40abd79649484a1a342585bd4
consensus_specs_sha: 12c8b24bc09b12f62688d439ba20ef00b1fdfe1f
ssz_specs_sha: b42af0a2265c25f5353cace185d066e8d39dbd6e
grandine_upstream_sha: a63876249043020d8ebe3d410c5cf31b19be1cc2
grandine_pr_base: 67195ede016e20f2febcf9ed1941da5898976b3d
grandine_stack_tip: c4e1bdbeb323df63fca2c9524c398727606c60cf
superseded_by:
---

# 0010 — Admit and complete execution proofs against live Store context

## Context

The enabled-only capture boundary below is revised against integration source
`c4e1bdbeb323df63fca2c9524c398727606c60cf` plus its working capture-gate change;
see [service integration](0013-execution-proof-service-integration.md). The PR2
base remains a lineage pin; EIP/spec/SSZ pins remain the selected historical
targets, not the current spec checkout. The completion-amendment verification
record below is historical and does not verify this revision.

**PR25 completion-policy amendment:** after retention and full-context
applicability, occupied `(B, T)` storage gates every report (Valid, Invalid, or
LocalFault) to
`ExecutionProofCompletionOutcome::Discarded(ExecutionProofDiscard::AlreadyVerified)`.
This is local IGNORE, does not change storage or Seen, and uses the existing
`Discarded` result rather than a new top-level completion outcome. It is distinct
from the unchanged `ExecutionProofAdmissionDecline::AlreadyVerified`, which
continues to decline a new admission without Seen writes. If storage is vacant,
Valid stores and ACCEPTs, Invalid REJECTs, and LocalFault locally IGNOREs. This
supersedes the earlier policy that interpreted Invalid/LocalFault after a Valid
had stored. The local Grandine source baseline is `87e5e8f8499532e98f834714c610029967bf11a0`;
the Store/test changes are present in that PR25 branch working tree, not
committed at this baseline. This representation correction is awaiting fresh
independent source review; the earlier Sol review covered the superseded
completion-enum shape. Scoped formatting/whitespace checks pass. The current
bounded locked/offline focused Store test stopped before compilation because the
pinned `universal-precompiles` `sha2` Git source is unavailable offline; earlier
check/strict-Clippy attempts on the prior representation hit the same resolver
blocker and were not rerun for this correction. Exact commands and exit statuses are recorded in
[`PR25-ALREADY-VERIFIED-AMENDMENT.md`](../../../scratchpad/execution-proof-pr2-master-update/PR25-ALREADY-VERIFIED-AMENDMENT.md);
no executable test/check/lint pass is claimed. EIP/spec/SSZ target pins remain
the selected historical targets below; this note makes no claim about current
spec or remote PR state.

This document covers **PR2: the Store lifecycle**
([Grandine #25](https://github.com/eip8025-grandine/grandine/pull/25)) within the
[execution-proof validation pipeline](0004-execution-proof-gossip-validation-pipeline.md).
The selected local Grandine baseline includes PR1
([#17](https://github.com/eip8025-grandine/grandine/pull/17)) and its
authenticated, payload-bound preparation.
PR2 adds live admission and completion; PR3 will connect them to engine
workers and terminal gossip/API replies. PR3 must map the existing
`ExecutionProofCompletionOutcome::Discarded(_)` arm to terminal IGNORE, including
`ExecutionProofDiscard::AlreadyVerified`; the separate
`ExecutionProofAdmissionDecline::AlreadyVerified` remains a pre-engine decline.
PR2 does not enable live proof processing or implement that PR3 mapping.

The spec pins retain the selected progressive-list semantics and zkBoost
binding conventions described in 0004, not a claim of alignment with latest
spec master. `grandine_upstream_sha` pins the underlying upstream merge-base;
`grandine_pr_base` and `grandine_stack_tip` delimit the implementation.

## Goals and non-goals

**Goals**
- Authorize verification only against applicable live payload context.
- Record both duplicate keys atomically before engine work.
- Preserve exact proof/context association through a consuming handoff.
- Apply an admitted report only while `(B,T)` is vacant; once storage is occupied,
  ignore every later report and preserve the first valid proof.
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

Ordinary payload submission remains authoritative. Control owns verifier policy:
Controller derives capture enablement from its engine's `!is_null()` and forwards
it to initial payload tasks and Mutator's shared delayed/retry path. Store has no
proof-engine dependency; its public validator takes an explicit capture boolean.
Only when enabled, after successful execution-layer notification, does validation
capture the proof statement best-effort and carry it with the accepted payload
to Store application. Disabled capture skips proof-context reconstruction/storage, not
ordinary validation, EL notification, acceptance, replies, events or application.
Capture failure cannot veto ordinary acceptance; accepting a replacement without
proof context, including disabled capture, clears the previous authority.
Custom-state validation cannot manufacture this live authority.

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

### Completion checks applicability and occupancy before the report

Only a report paired with the admitted input can complete. Store remains
independent of the proof-engine crate; PR3 must map engine `Valid`, `Invalid`
and `Undetermined` exhaustively to Store `Valid`, `Invalid` and `LocalFault`.

| Live completion | Result | Verified storage |
| --- | --- | --- |
| Retired or changed context, regardless of report | Discard locally | No insertion |
| Applicable report, occupied block/type | Ignore (`Discarded(AlreadyVerified)`) | Preserve the first bare envelope |
| Applicable Valid, vacant block/type | Accept | Store the first bare envelope |
| Applicable Invalid, vacant block/type | Reject | Unchanged |
| Applicable LocalFault, vacant block/type | Local outcome, not peer rejection | Unchanged |

After retention and full-Q applicability, completion checks verified storage
before reading/interpreting its report. If `(B, T)` is occupied, every report
returns `ExecutionProofCompletionOutcome::Discarded(ExecutionProofDiscard::AlreadyVerified)`
/ IGNORE; no bytes or `Arc` are replaced and neither attempt's Seen bookkeeping
is rolled back. This discard means the completion is no longer needed, unlike
`Retired` or `ContextInapplicable`, which mean it is no longer applicable. New
admissions independently return the unchanged
`ExecutionProofAdmissionDecline::AlreadyVerified` without Seen writes. If vacant,
Valid stores and ACCEPTs, Invalid REJECTs, and LocalFault locally IGNOREs. Thus
an Invalid result is rejected only when no valid proof has already occupied the
pair. No completion changes payload status, beacon state or fork-choice weights.
Historical resident entries cannot authorize retired targets; physical
reclamation is deferred.

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
- **First-valid storage rather than canonical proof selection:** the first live
  Valid completion chooses stored bytes; later admitted reports are ignored once
  occupied, so no result overwrites that evidence. Proof bytes are not canonical.
- **Best-effort context capture rather than coupled payload acceptance:** proof
  bookkeeping cannot break ordinary acceptance, but enabled capture has synchronous
  cost and a failed or disabled capture leaves no applicable proof context.

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
context capture succeeds, fails, is disabled or submission fails. Store coverage
compares enabled/disabled EL requests, verifies no disabled capture attempt, and
checks that capture-disabled replacement revokes old evidence. Integration also
checks policy propagation through the shared payload retry seam.

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
