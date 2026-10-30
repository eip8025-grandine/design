---
status: Draft
authors: [keshavsharma25]
workstream: gossip
eip_repo: ethereum/EIPs
eip_sha: 9207c6011f526bd40abd79649484a1a342585bd4
consensus_specs_sha: 460fd37c3a58f6baf8a9a285d2adf71c4f818929
ssz_specs_sha: b42af0a2265c25f5353cace185d066e8d39dbd6e
grandine_upstream_sha: a63876249043020d8ebe3d410c5cf31b19be1cc2
grandine_pr_base: 54c0bf88c315139ca4a2fb5fd6553e3925b01351
grandine_stack_tip: 54c0bf88c315139ca4a2fb5fd6553e3925b01351
eth2_libp2p_sha: 55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a
eth2_libp2p_fork_sha: 7401336463a80fd1d1742f1a268021a9870d8c6c
superseded_by:
---

# 0014: Route execution-proof gossip to the controller

## Context

[Service integration](0013-execution-proof-service-integration.md) gives the
Controller an `Arc<dyn ProofEngine>` and an ingress method,
`on_gossip_execution_proof(Arc<SignedExecutionProofEnvelope>, GossipId)`
(`fork_choice_control/src/controller.rs:804`). Payload-context capture already
uses `!proof_engine.is_null()` at lines 185 and 1068. Runtime supplies a
`NullProofEngine` placeholder until backend configuration is available.

The worker/mutator pipeline returns typed `P2pMessage::Accept`, `Ignore` and
`Reject` outcomes. `p2p/src/network.rs:486` handles these replies;
`report_outcome` at line 3155 forwards them to gossipsub. Authentication,
admission and outcome policy remain those of the
[validation pipeline](0004-execution-proof-gossip-validation-pipeline.md) and
[Store lifecycle](0010-execution-proof-store-lifecycle.md).

The missing connections are subscription enablement and decoded-message routing.
The `eth2_libp2p` fork provides the `execution_proof` topic,
`PubsubMessage::ExecutionProof`, a pre-decode envelope-size check, and
`Config::enable_execution_proof` (default false), with subscription gated on
Gloas. Grandine still uses the library revision in `eth2_libp2p_sha`, without
these additions. The fork revision in `eth2_libp2p_fork_sha` adds them directly
on that baseline.

The EIP and consensus-specs pins identify the local networking references, not
an alignment of the existing validation service to every current spec change.
SSZ and upstream Grandine pins retain 0013's baseline. EIP-8025 and its feature
specifications remain provisional; a topic-scheme change requires re-pinning
and codec review.

## Goals and non-goals

**Goals:** fetchable library support; exactly-once routing of each decoded
proof and its `GossipId`; unchanged asynchronous outcome reporting; subscription
policy derived from the Controller's engine; default-off dormancy.

**Non-goals:** the separate `--execution-proof-type` runtime option, real backend
selection, proof generation/publication, HTTP ingress, req/resp, `eproof` ENR
advertising, peer preference, topic scoring, pruning or new resource budgets.

## Design

```text
run_beacon_node: create engine -> LocalNodeConfig -> build_local_node
  -> inspect !is_null() before moving engine -> Controller owns engine
  -> network_config.enable_execution_proof -> Gloas-gated subscription

eth2_libp2p: transport bounds -> envelope-size check -> decoded ExecutionProof
  -> p2p handle_pubsub_message: metrics + metadata trace
  -> Controller.on_gossip_execution_proof(envelope, GossipId)
  -> existing workers/mutator -> P2pMessage::{Accept, Ignore, Reject}
  -> p2p report_outcome -> gossipsub
```

### 1. Use the fork revision and a fetchable URL

Update the `eth2_libp2p` gitlink from `eth2_libp2p_sha` to
`eth2_libp2p_fork_sha` and change its `.gitmodules` URL from
`https://github.com/grandinetech/eth2_libp2p.git` to
`https://github.com/eip8025-grandine/eth2_libp2p.git` together. The fork-only
revision must be reachable through the declared URL, not just a local checkout.
This is required for fresh clones and CI's `submodules: true` checkout.
Restore the upstream URL once it serves the fork revision.

### 2. Hoist engine ownership in runtime

`runtime/src/runtime.rs` keeps `NullProofEngine` in its imports at line 75
and adds `ProofEngine`. Replace the inline Null argument to `Controller::new`
with this binding in `run_beacon_node`, before anchor loading at line 1466:

```rust
let proof_engine: Arc<dyn ProofEngine> = Arc::new(NullProofEngine);
```

Carry it as a `proof_engine: Arc<dyn ProofEngine>` field on `LocalNodeConfig`
(line 130), supplied at its construction (line 1472). Destructure it as the
bare `proof_engine` binding in `build_local_node` (line 202), not as an unused
`_proof_engine`. Before moving it into `Controller::new`, compute
`let enable_execution_proof = !proof_engine.is_null();`. Pass that same engine
to the Controller unchanged. Make the extracted `network_config` mutable and
set `network_config.enable_execution_proof = enable_execution_proof` and log
the assigned field with `tracing::debug!` before its `Arc::new(network_config)`
wrap at line 689.

The Null initializer leaves the flag false. The later runtime-configuration
change replaces only the initializer expression; no accessor, second engine
or independent subscription setting is needed.

### 3. Route the decoded envelope in p2p

Add one `PubsubMessage::ExecutionProof(envelope)` arm to
`handle_pubsub_message` (`p2p/src/network.rs:2517`), following the
execution-payload arm at line 2722:

- Register `gossip_object` with label `execution_proof`, following the existing
  `execution_payload_envelope` metric convention.
- Trace only metadata: block root, proof type, validator index and source
  peer. Never format the envelope or proof bytes, unlike the neighboring
  payload trace.
- Construct `GossipId { source, message_id }` and call
  `controller.on_gossip_execution_proof(envelope, gossip_id)` exactly once.

Submission is not a validation verdict. Results return through the existing
`P2pMessage::Accept/Ignore/Reject` channel and `report_outcome`, with no new
outcome channel, synchronous waiting or wildcard conversion of local failures
to REJECT. No new proof cache or retained routing state is introduced.

| Engine                                | `enable_execution_proof` | Topic subscription         | Dispatch                              |
| ------------------------------------- | ------------------------ | -------------------------- | ------------------------------------- |
| Null placeholder (default)            | false                    | None                       | No subscribed proof traffic           |
| Non-Null engine (later configuration) | true                     | Enabled under Gloas gating | Envelope and `GossipId` to Controller |

## Trade-offs and alternatives

- **Fork URL rather than waiting for upstream:** temporary source divergence
  buys a fetchable library revision now. The URL returns upstream when that
  revision is available there.
- **Hoist rather than park `_proof_engine` or expose a Controller accessor:**
  explicit ownership transfer derives subscription from the actual retained
  engine and leaves later configuration as one initializer replacement.
- **Existing handler rather than a proof-specific event path:** one arm reuses
  metrics, identity and outcome reporting without duplicating channels.
- **Defer `eproof` ENR awareness:** the pinned EIP describes awareness for peer
  discovery; the feature's `p2p-interface.md` makes peer preference advisory
  (MAY). Deferral forgoes preference for proof-aware peers, not validation
  correctness. It does not make proof-unaware peers subscribe to the topic.

## Security and compatibility

The Null default adds no subscription, ENR change or proof-gossip exposure;
ordinary node behavior is unchanged without opt-in. Enabled ingress inherits
compressed/decompressed transport bounds and rejects envelopes larger than
`MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE` (4,194,449 bytes) before SSZ decoding
(`eth2_libp2p/src/types/pubsub.rs:671` at the fork revision). This bounds one
envelope, not aggregate traffic, queue memory or verifier concurrency; deployment
resource limits remain separate. Metadata-only traces avoid logging proof data.

Outcome honesty stays in the pipeline: local faults are IGNORE, not evidence
for peer REJECT, with live applicability and first-valid storage policy owned
by [0010](0010-execution-proof-store-lifecycle.md). Invalid or withheld proofs
cannot change payload validity, beacon state or fork-choice weights, as in
[0004](0004-execution-proof-gossip-validation-pipeline.md). No consensus rule or
wire-container change is introduced here. A provisional spec topic-scheme move
requires re-pinning and reviewing the codec, not redesigning Controller routing.

## Implementation and testing

Success requires all of the following:

1. A fresh clone and CI fetch the declared fork pin through `.gitmodules`.
   The library's execution-proof topic, opt-in subscription/whitelist, codec
   round-trip, maximum-size and oversized-envelope tests pass at that pin.
2. A handler test submits a decoded envelope and observes exactly one
   `on_gossip_execution_proof` call with the same envelope and complete
   `GossipId`; metrics record `gossip_object` label `execution_proof` once.
3. ACCEPT, IGNORE and REJECT each reach gossipsub through the existing channel
   with the original `GossipId` and unchanged propagation/peer-reporting
   semantics. Local failures remain IGNORE, with no new rejection mapping.
4. With the Null default, the topic is absent, ENR contents are unchanged and
   there is no subscribed proof dispatch. The release build succeeds without
   new warnings.
5. Injecting a non-Null engine enables subscription under Gloas gating and
   routes decoded envelopes to the Controller. Before Gloas, subscription
   remains absent even when the flag is true.
6. Workspace `cargo fmt --check --all` and the existing strict lint invocation,
   `scripts/ci/clippy.bash --deny warnings`, pass.

These checks cover the three work items through the dispatch boundary, retaining
service outcome regressions from 0013. End-to-end delivery over a real network
belongs to later publisher/devnet work and is not established by these tests.

## Open questions

- Repository hosting for this work. Runtime configuration is assumed to build
  on this ownership layout and follow it, activating only the initializer.
- When the upstream library will serve the fork revision so the `.gitmodules`
  URL can return upstream.
- Whether proof-topic scoring is required before deployment or deferred with
  ENR awareness and peer-preference work.
