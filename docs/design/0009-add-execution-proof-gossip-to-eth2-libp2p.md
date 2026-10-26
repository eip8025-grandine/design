---
status: Draft                # Draft | Accepted | Superseded
authors: [creese]
eip_repo: frisitano/EIPs
eip_sha: 4855dbeb9a99702a8c4d948ceceb865fb3289759
consensus_specs_sha: b834fa8411cb2152563e8835c5b72c28f7503e7a
ssz_specs_sha: b42af0a2265c25f5353cace185d066e8d39dbd6e
grandine_upstream_sha: a6387624
eth2_libp2p_upstream_sha: 55cdfbaf97ebe26e5e6f7331ea1bb24def6bf02a
superseded_by:               # Doc number, if any
---

# 0009 — Add execution-proof gossip to eth2_libp2p

## Context

The pinned
[p2p spec](https://github.com/ethereum/consensus-specs/blob/b834fa8411cb2152563e8835c5b72c28f7503e7a/specs/_features/eip8025/p2p-interface.md)
defines one global topic, `execution_proof`, which carries
`SignedExecutionProofEnvelope`. Proofs travel only by gossip; the spec
defines no Req/Resp protocol. The topic has a type-specific SSZ bound,
`MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE` = 4,194,449 bytes. Under the
phase0
[gossipsub size limits](https://github.com/ethereum/consensus-specs/blob/b834fa8411cb2152563e8835c5b72c28f7503e7a/specs/phase0/p2p-interface.md#gossipsub-size-limits),
the uncompressed payload must not exceed `MAX_PAYLOAD_SIZE` (10 MiB)
or that bound, whichever is lower. Clients MUST reject anything
larger. EIP-8025 is a feature built on Gloas. It has no fork version,
so the topic name uses the node's ordinary Gloas fork digest:
`/eth2/{fork_digest}/execution_proof/ssz_snappy`.

Grandine already has the type and the constant
(`types/src/eip8025/{containers,consts}.rs`). The constant's doc
comment says "This crate does not enforce that bound". Both
[0005](0005-update-execution-proof-data-model.md) and
[0004](https://github.com/eip8025-grandine/design/pull/9) require
ingress to enforce it before decoding. The `eth2_libp2p` submodule
(`55cdfba`) has no execution-proof topic. This doc covers only the
transport in `eth2_libp2p`. Routing in Grandine's `p2p` crate, proof
processing (0004 and
[0008](0008-distinguish-invalid-proofs-from-local-verifier-failures.md))
and the runtime option that sets the flag are outside it.

`eth2_libp2p` is a fork of Lighthouse's `lighthouse_network` that tracks
upstream backports, so anything added here is permanent divergence.
For naming, this doc follows eth-act's Lighthouse `optional-proofs`
branch (`63a526aa`):
`GossipKind::ExecutionProof`, `"execution_proof"`,
`TopicConfig::enable_execution_proof` and
`PubsubMessage::ExecutionProof(Arc<SignedExecutionProofEnvelope>)`.
That branch applies no pre-decode bound.

The EIP text is stale here. It names the message `SignedExecutionProof`,
keys deduplication on `new_payload_request_root`, and defines three
Req/Resp protocols. This doc follows consensus-specs.

**Provisional upstream.**

- [#5726](https://github.com/ethereum/consensus-specs/pull/5726) (open,
  `4ff41108`) replaces the global topic with eight
  `execution_proof_{subnet_id}` topics, where `subnet_id = proof_type %
  8`, and adds a wrong-subnet REJECT. If it merges, `GossipKind` and
  `PubsubMessage` gain a subnet ID, an opted-in node subscribes to eight
  topics, and the whitelist grows by eight entries per digest. The
  message type and the bound do not change.
- [#5729](https://github.com/ethereum/consensus-specs/pull/5729)
  (`bbb8c7f7`) and [#5753](https://github.com/ethereum/consensus-specs/pull/5753)
  (`76d78302`) change only the validation rules, so the wire format is
  unaffected.
- `MAX_PROOF_SIZE` is marked "not definitive". Grandine derives the
  envelope bound from `MaxProofSize`, so if the limit moves, this design
  follows it with no code change.

## Goals and non-goals

**Goals**

- An opted-in node subscribes to `execution_proof` on every Gloas
  digest it uses. That includes BPO digests and the next fork's digest
  one epoch early. The node can publish and receive
  `SignedExecutionProofEnvelope` encoded as ssz_snappy.
- A decompressed payload over 4,194,449 bytes is rejected before SSZ
  decoding. A payload of exactly 4,194,449 bytes reaches the decoder.
- With the flag off, nothing changes: subscribed topics, the
  subscription whitelist and count limits, scoring parameters and the
  gossip cache all stay as they are today.
- Grandine turns the topic on with one config field. It receives typed
  messages through the existing `NetworkEvent::PubsubMessage` and
  publishes through the existing `publish`.

**Non-goals**

- Any check in `validate_execution_proof_gossip`, including the
  empty-proof and supported-type REJECTs. Also out: Seen, verification,
  storage and the ACCEPT/IGNORE/REJECT decision (0004, 0008).
- Routing in Grandine `p2p`, the runtime option that sets the flag,
  and proof-engine construction.
- Advertising the `eproof` ENR key, and preferring proof-aware peers.
- Topic score parameters for `execution_proof`.
- Subnets, unless #5726 merges.
- Turning the subscription on or off after startup.

## Design

```text
peer ─▶ gossipsub ─▶ SnappyTransform        (unchanged: compressed ≤ max_compressed_len(MAX_PAYLOAD_SIZE),
                     │                        uncompressed ≤ MAX_PAYLOAD_SIZE)
                     ├─ topic not subscribed ─▶ dropped by gossipsub (unchanged)
                     ▼
        PubsubMessage::decode ── digest not Gloas ─────────▶ Err ─▶ REJECT in eth2_libp2p
                     ├────────── len > 4,194,449 ───────────▶ Err ─▶ REJECT
                     ├────────── SSZ decode fails ──────────▶ Err ─▶ REJECT
                     ▼
        NetworkEvent::PubsubMessage { ExecutionProof } ─▶ Grandine p2p ─▶ 0004 pipeline
                                                         ◀─ report_message_validation_result
```

| File (`eth2_libp2p/src/`) | Change |
| --- | --- |
| `types/topics.rs` | `EXECUTION_PROOF_TOPIC = "execution_proof"`; `GossipKind::ExecutionProof` in `Display`, `decode` and `AsRefStr`. `TopicConfig::enable_execution_proof`. `core_topics_to_subscribe` adds the kind at `Phase::Gloas` and later when the flag is set. `is_fork_non_core_topic` returns `false` for it. `all_topics_at_fork` takes the flag. |
| `types/pubsub.rs` | `PubsubMessage::ExecutionProof(Arc<SignedExecutionProofEnvelope>)` with `kind`, `decode`, `encode`, and a `Display` showing block root, type and validator index, never proof bytes. |
| `types/globals.rs` | `as_topic_config` copies the flag. |
| `config.rs` | `Config::enable_execution_proof: bool`, default `false`. |
| `service/utils.rs` | `create_whitelist_filter` adds the topic for each digest only when the flag is set. |
| `service/mod.rs` | Passes the flag to `all_topics_at_fork` and the whitelist. Builds the gossip cache with no retry for this kind. |
| `service/gossip_cache.rs` | Adds an `execution_proof` timeout field and match arm. |

**Fork and topic.** `decode` accepts the topic only on a digest that
`ForkContext` maps to `Phase::Gloas`. A BPO digest maps to the phase
it falls in. Earlier digests return `Err`, as every other Gloas topic does. A
future `Phase` has to be added to the match explicitly, again like
every other Gloas topic. Fork transitions need nothing new, because the
topic is a core topic. `subscribe_new_fork_topics` subscribes the next
digest early, and `unsubscribe_from_fork_topics_except` drops the old
one.

**Size bound.** `decode` calls the existing `ensure_max_message_size`
on the decompressed `data`, with
`types::eip8025::consts::MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE`,
before it calls `from_ssz_default`. That is the same pattern as the
Gloas bid, aggregate and attester-slashing bounds. Since 4,194,449 is
less than `MAX_PAYLOAD_SIZE`, this bound is the one that applies. The
compressed and RPC-frame limits are unchanged, and a maximum-size
envelope fits within both. Snappy's worst case for 4,194,449 bytes is
4,893,555 bytes (`32 + n + n/6`, `snap` 1.1.2). The compressed limit
is `max_compressed_len(MAX_PAYLOAD_SIZE)` = 12,233,418 bytes, and the
gossipsub frame limit is `max_message_size()` = 12,234,442 bytes.
A payload over `MAX_PAYLOAD_SIZE` is refused from its snappy length
header, before it is decompressed. A payload between the two bounds is
decompressed and then refused here, before SSZ decoding. On the
outbound side, `ProofData` is a `ByteList` that enforces its limit
when constructed, so a well-typed envelope always encodes within the
bound. `publish` does not re-check it, which matches the other Gloas
types.

**Interface.** Grandine sets `NetworkConfig::enable_execution_proof`
before it starts the service. That one flag controls three things
together: the core topic, the whitelist entry and the worst-case topic
count. Grandine's existing `subscribe_to_core_topics` and fork
handling then subscribe without any new message type. Inbound messages
arrive as `NetworkEvent::PubsubMessage { id, source, topic, message
}`, and outcomes go back through the existing
`ReportMessageValidationResult`. Outbound proofs use `publish` on the
current digest. No new public functions are added.

**Gossip cache.** This kind is not retried, which matches sync
committee messages. Each entry can be 4 MiB, and a proof matters only
for a short window. A prover with no subscribed peers has a peering
problem, and a retry doesn't fix that.

## Trade-offs and alternatives

**No library flag: Grandine calls `subscribe_kind(ExecutionProof)`.**
This is the obvious alternative, because it keeps
`config.rs` and `topics.rs` almost untouched. It is rejected for two
reasons:

- The library has to change anyway. The whitelist filter refuses to
  subscribe to any topic it does not list (`SubscriptionError::NotAllowed`,
  `rust-libp2p` at `sigp@accba72b`).
- A non-core topic does not follow fork transitions on its own.
  `subscribe_new_fork_topics` re-subscribes only core topics and topics
  that `is_fork_non_core_topic` accepts. Grandine would have to manage
  the next digest itself.

The flag keeps fork handling inside existing code and matches eth-act's
Lighthouse. The cost is that the choice is made at startup only.

**Always whitelist the topic.** eth-act's Lighthouse does this: its
`all_topics_at_fork` sets `enable_execution_proof: true`. That gives a
smaller diff, but a node that did not opt in would then track peers'
`execution_proof` subscriptions and allow four more topics in its count
limit. This is rejected because non-opted nodes must stay unchanged.
The cost is one extra parameter.

**Check the bound in `SnappyTransform`, before decompression**, using
the topic and `decompress_len`. This is stronger, because an oversize
proof would never be decompressed into up to 10 MiB. It is rejected
because it changes the transform for every topic, while all the
existing Gloas bounds are checked in `decode`. Every node already pays
the cost of decompressing up to `MAX_PAYLOAD_SIZE` on every topic. It
can be revisited if proof-topic flooding is ever measured.

**Wait for #5726.** Rejected. The pinned spec has a global topic, and
moving to subnets is a mechanical change.

**Cost.** This adds permanent divergence to seven files of a
backport-tracking fork. Upstream backports that touch these match arms
will conflict. Using eth-act's names keeps those conflicts small.

## Security and compatibility

**Adversarial model.** Any gossip peer can send any bytes on the topic,
up to gossipsub's transmit limit. `eth2_libp2p` trusts nothing in the
message. It checks the topic, the fork and the size, then the SSZ
structure. Everything else is checked downstream.

**DoS and resource use.**

- For each inbound message on a subscribed node, the existing costs
  are snappy decompression (up to `MAX_PAYLOAD_SIZE`) and the
  message-ID hash. On top of that come a length comparison and at most
  one SSZ decode of up to 4 MiB. An oversize payload never reaches the
  decoder.
- The largest possible message does not grow: blocks already allow
  10 MiB on every node. Only message volume grows, and only on nodes
  that opt in.
- `eth2_libp2p` applies no rate limit and no bounded queue to the
  topic. Every decoded message goes straight onto the service's event
  stream. 0004 makes bounded queues and work a prerequisite for live
  ingress, so Grandine must not set the flag on a public network until
  those exist.
- **Scoring gap.** With no topic score parameters, gossipsub records no
  invalid-message penalty on this topic (`stats_or_default_mut` returns
  stats only for scored topics). Size and decode REJECTs
  therefore do not lower the sender's gossipsub score. Every Gloas
  topic in this fork already behaves this way. Application-level
  `report_peer` penalties belong to Grandine's `p2p` layer. See *Open
  questions*.
- Memory: gossipsub puts each inbound message in `mcache` on receipt.
  A REJECT or IGNORE removes it; otherwise it stays for
  `history_length` (12) heartbeats. Only ACCEPTed messages are
  forwarded or advertised. The spec does not limit ACCEPTs to one
  proof per (block, type). Its duplicate keys are the proof root and
  (block, type, validator), and a stored proof only stops new
  admissions, so proofs from distinct provers admitted before the
  first valid one is stored can each ACCEPT (0004). Each extra ACCEPT
  needs distinct valid proof bytes and a distinct active validator.
  Cache occupancy therefore depends on the volume of messages received
  within that window. The finite validation queue and byte limits in
  0004 bound queued work, but do not directly cap `mcache` occupancy.
  Proofs are above the 1,000-byte `IDONTWANT` threshold, which cuts
  duplicate transfers. Unpublished proofs are not held in the gossip
  cache.

**Invalid or withheld proofs.** `eth2_libp2p` gives no verdict on a
proof. A malformed message is REJECTed here. A well-formed invalid
proof reaches Grandine, which REJECTs or IGNOREs it under 0004 and
0008. A withheld proof just means no message arrives. Proofs are
non-consensus artifacts, so neither case changes state, payload status
or fork choice. The pre-decode check changes when an oversize message
is rejected, not whether it is rejected. `proof_data` is the only
variable-size field, so any input over 4,194,449 bytes would fail the
`ProofData` limit during decoding anyway.

**Nodes that do not opt in.** The flag defaults to `false`. Such a node
has no subscription, no whitelist entry, and unchanged count limits,
scoring and cache. Gossipsub drops unsolicited messages on the topic as
"not subscribed" after the transform, as it does for any unknown topic
today (gossipsub `behaviour.rs`). The whitelist still filters out
peers' `SUBSCRIBE` frames for the topic.

**Compatibility.** Other topics keep their wire format, message IDs
and Req/Resp unchanged. The topic string and SSZ type match eth-act's
Lighthouse, and both clients accept exactly the same set of valid
messages.

**Observability.** This change adds no metrics. The `Display` used in
logs leaves out the proof bytes. Per-topic counters belong to
Grandine's `p2p` layer.

## Implementation and testing

The change is confined to the `eth2_libp2p` crate at `55cdfba`. The
crate builds inside the Grandine workspace (`types = { workspace = true
}`), and its tests do not depend on Grandine's `p2p` crate.

**Integration dependency.** `p2p/src/network.rs`
`handle_pubsub_message` is an exhaustive `match`. Grandine therefore
cannot adopt this `eth2_libp2p` revision until `p2p` handles
`PubsubMessage::ExecutionProof`.

Nothing here can be upstreamed on its own, because `types::eip8025`
exists only in this fork. With the flag off the code is inert.

**Tests (in `eth2_libp2p`)**

- `topics.rs`: `ExecutionProof` added to the round-trip list.
  `as_ref()` is `"execution_proof"`. The topic string is exactly
  `/eth2/01020304/execution_proof/ssz_snappy`. With the flag set, the
  kind is subscribed at Gloas and not before. With the flag off it is
  absent at every phase. `is_fork_non_core_topic` returns `false`. The
  existing `test_core_topics_to_subscribe` is unchanged.
- `pubsub.rs`: `decode_oversized` at the bound plus one returns an error
  that names `MAX_SIGNED_EXECUTION_PROOF_ENVELOPE_SIZE`. At the bound,
  the error does not name it. An envelope with a 4,194,304-byte proof
  encodes to exactly 4,194,449 bytes and decodes back equal. With
  incompressible proof bytes, the same envelope passes
  `SnappyTransform`'s outbound and then inbound path unchanged. The
  same bytes on a Fulu digest return an error.
- `service/utils.rs`: the whitelist contains the topic hash if and only
  if the flag is set.
- `tests/`: `build_node_pair` on a Gloas config. With the flag set on
  both nodes, A publishes and B receives an equal
  `PubsubMessage::ExecutionProof`. With the flag off on B, B's
  subscriptions don't include the topic, and A sees no peer
  subscribed to it. The test asserts on subscription state, not on
  timeouts.

No spec vectors cover gossip. An end-to-end devnet run with a gossip
publisher exercises this path on the wire, but it is outside these
tests.

## Open questions

**Deferred**

- Topic score parameters, at least an invalid-delivery (P4) penalty,
  before any public testnet.
- Advertising the `eproof` ENR key and preferring proof-aware peers.
  Without them the mesh can stay empty on a network where few nodes
  opt in.
- A per-topic bound in `SnappyTransform`, if flooding is measured.
- Turning the subscription on or off at runtime.
