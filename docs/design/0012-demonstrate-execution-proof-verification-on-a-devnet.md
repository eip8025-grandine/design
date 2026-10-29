---
status: Draft                # Draft | Accepted | Superseded
authors: [creese]
workstream: converged        # gossip | verification | converged
eip_repo: frisitano/EIPs
eip_sha: 4855dbeb9a99702a8c4d948ceceb865fb3289759
consensus_specs_sha: b834fa8411cb2152563e8835c5b72c28f7503e7a
ssz_specs_sha: b42af0a2265c25f5353cace185d066e8d39dbd6e
grandine_upstream_sha: a63876249043020d8ebe3d410c5cf31b19be1cc2
superseded_by:               # Doc number, if any
---

# 0012 — Demonstrate execution-proof verification on a devnet

## Context

This design covers milestone T1 of the EIP-8025 roadmap: a gossip
publisher and a reproducible Kurtosis recipe that show one Grandine
node handling execution proofs end to end. It adds test tooling only.
Every behaviour it observes belongs to another milestone:

| Milestone | Provides | Design | State on `feature/eip8025` (`67195ede`) |
| --- | --- | --- | --- |
| L1 | `execution_proof` topic and `PubsubMessage::ExecutionProof` in `eth2_libp2p` | [0009](0009-add-execution-proof-gossip-to-eth2-libp2p.md) | Merged on the fork (`7401336`); Grandine's submodule pointer is still `55cdfba` |
| P1 | Preparation, Seen admission, engine call, storage, typed outcome | [0004, 0010](https://github.com/eip8025-grandine/design/pull/9) | P1b is Grandine #25, unmerged; P1c, the Controller and task wiring, is pending |
| R1 | `--execution-proof-type`, the `execution-proof-sp1` feature, engine construction | [0011](https://github.com/eip8025-grandine/design/pull/16) | Grandine #26; the engine hand-off waits on P1c |
| N1 | p2p routing and subscription when the engine is not Null | none published | Pending; it carries the submodule bump |

The engine is [0006](0006-make-proof-engine-validation-only.md),
[0007](0007-verify-execution-proofs-in-the-proof-engine.md) and
[0008](0008-distinguish-invalid-proofs-from-local-verifier-failures.md).
The SP1 backend returns public values only after verification succeeds
(`proof_engine/src/ere_backend.rs`), and `ZkvmProofEngine::check`
compares them with the serialized `PublicInput` afterwards. An SP1
`PublicValuesMismatch` therefore means the proof verified and the
binding failed. Each milestone has its own tests; none runs the
released binary, its command line, a live chain and the wire together.

**Fixture.** `proof_engine/src/fixtures/stateless-validator-ethrex-sp1-v6.4.0.proof`
(1,272,611 bytes) is zkBoost's genuine ethrex SP1 proof. Its committed
`PublicInput`, pinned in `zkvm_engine.rs`, has root `68e22507…7bb7`,
chain ID 1 and schema `0x1501`. Its guest VK is `0x00662ca6…c88b`,
with SHA-256 `5e614b66…a4d7`.

**Spec.** At the pin,
[`p2p-interface.md`](https://github.com/ethereum/consensus-specs/blob/b834fa8411cb2152563e8835c5b72c28f7503e7a/specs/_features/eip8025/p2p-interface.md)
defines one global topic, and `get_supported_proof_types()` is
`{1, 2, 3}`.

**Provisional upstream, and the effect if it moves.**

- **[#5762](https://github.com/ethereum/consensus-specs/pull/5762)**
  merged upstream as `ec37552e`, after the pin. It assigns types 1–7
  to guest programs, still provisionally, and widens the supported set
  to `{1, …, 7}`. Type 2 is ethrex `26.0.0` on SP1 `v6.4.0` with VK
  digest `5e614b66…a4d7`, the fixture's guest, so the SP1 step uses
  type 2 as assigned. Type 1 is ethrex on OpenVM; the recipe configures
  it as mock instead (*Security and compatibility*). Types 1 and 2 are
  supported before and after #5762, so adopting it changes nothing
  here.
- **[#5726](https://github.com/ethereum/consensus-specs/pull/5726)**
  (open) would move the topic to `execution_proof_{proof_type % 8}`.
  The publisher would use subnets 1 and 2; the steps do not change.
- **[#5729](https://github.com/ethereum/consensus-specs/pull/5729) and
  [#5753](https://github.com/ethereum/consensus-specs/pull/5753)**
  (open) change the bound statement. The mock step follows Grandine's
  public-input code; the SP1 step still REJECTs.
- **zkBoost root.** Grandine binds `zkboost_compatible_root()` (0007).
  If upstream settles on another root, the mock step follows Grandine
  and does not detect the move.

## Goals and non-goals

**Goals**

1. **Mock ACCEPT.** A valid type-1 mock proof for a recent accepted
   payload B, sent over gossip, is authenticated, bound, verified,
   stored and ACCEPTed, and Grandine forwards it to an observing peer.
2. **SP1 binding REJECT.** The unchanged SP1 fixture, sent as type 2
   for the same B, passes SP1 verification. Grandine then REJECTs it
   with `PublicValuesMismatch` and does not forward it.
3. One command builds the image, starts the enclave, runs the steps and
   exits 0 (pass), 1 (a condition is contradicted) or 2 (inconclusive:
   setup or timeout). The exit code is never 0 unless every condition
   was observed.
4. A run completes within one epoch of selecting B, so B stays
   unfinalized and retained (0004's retirement rule).
5. Every run leaves an evidence bundle that is enough to re-check the
   verdict without the enclave.
6. The publisher builds against L1 alone, before P1, R1 and N1 merge.

**Non-goals**

- Generating SP1 proofs, SP1 ACCEPT over gossip, a second Grandine
  node, HTTP submission, Req/Resp, pruning and metrics.
- Exercising types 3–7, `Undetermined`, decode-level REJECTs or a node
  that does not opt in. L1, P1, R1 and N1 test those in-process; repeating
  them on a devnet adds run time without adding evidence for the two
  goals.
- Running the recipe in CI. It needs Docker, Kurtosis and network
  access to build SP1.
- Measuring verification cost.

## Design

```text
host: run.sh ─ docker build ─ kurtosis run ─ docker run ─ kurtosis service logs ─ evaluate
                                   │
 enclave ┌─────────────────────────┼─────────────────────────────────────────────┐
         │ geth ◀─engine─▶ Grandine (recipe image, execution-proof-sp1)           │
         │                 --execution-proof-type 1=mock                          │
         │                 --execution-proof-type 2=sp1:0x00662ca6…c88b           │
         │                    ▲ beacon API          ▲ │ execution_proof gossip    │
         │ publisher ─────────┘                     │ │                           │
         │ (same image)   P_a, P_b, P_c ──publish───┘ └──forward on ACCEPT──▶ O    │
         └────────────────────────────────────────────────────────────────────────┘
```

**Components.** All live in the Grandine repository:

| Path | Content |
| --- | --- |
| `execution_proof_publisher/` | Workspace binary, `publish = false`, like `ad_hoc_bench`. Subcommands `run` and `evaluate`. |
| `scripts/eip8025_devnet/Dockerfile` | One image holding `grandine`, built with `-p grandine --features default-networks,execution-proof-sp1` (0011's verified command), and the publisher. The base image follows `rust-toolchain.toml`. |
| `scripts/eip8025_devnet/network_params.yaml` | The topology below |
| `scripts/eip8025_devnet/run.sh` | The driver |

The publisher depends on `types`, `helper_functions`, `eip_2335`,
`bls`, `ssz` and `eth2_libp2p`. It does not depend on `p2p`, `runtime`
or `fork_choice_*`. That keeps it independent of the code under test
beyond the shared types, and it compiles against L1 before N1 handles
the new variant in `p2p`.

**Topology.** An ethereum-package enclave, with the package pinned to a
release:

- one participant: Grandine from the recipe image, and geth
  `ethpandaops/geth:glamsterdam-devnet-8-aa1f2fc`, the image zkBoost's
  testnet uses;
- Gloas at genesis, mainnet preset, network ID 3151908;
- `cl_extra_params`: the two `--execution-proof-type` values above;
- `cl_extra_env_vars`: `GRANDINE_LOG=proof_engine=debug,fork_choice_control=debug,p2p=debug`.
  `GRANDINE_LOG` adds directives to the default filter
  (`binary_utils/src/lib.rs`).

If Grandine alone cannot sustain a Gloas chain with revealed payloads,
the recipe adds one Lighthouse and geth participant as a block
producer. That participant does not subscribe to `execution_proof`, so
the run still concerns one Grandine node.

**Publisher identities.** One container joins the enclave's Docker
network and runs four `eth2_libp2p` instances, each with its own key:

| Identity | Role |
| --- | --- |
| O | Observer. Subscribes and never publishes. A message reaches O only if Grandine ACCEPTs it. |
| P_a | Publishes steps 1 and 2 |
| P_b | Publishes step 3 |
| P_c | Publishes step 4 |

Each REJECT gets a fresh identity. If P1 or N1 penalises a REJECTed
sender with `LowToleranceError` (−10, as on other topics), two REJECTs
from one peer reach the −20 disconnect threshold
(`eth2_libp2p/src/peer_manager/peerdb/score.rs`). Four identities on
one IP stay under gossipsub's IP-colocation threshold of 8.

Discovery is disabled. Each identity dials Grandine's address from
`/eth/v1/node/identity`, and none dials another. The run is
inconclusive if O ever has a peer other than Grandine. Grandine sends
Status only on outgoing connections (`p2p/src/network.rs`), so each
identity sends one built from Grandine's beacon API (fork digest,
finalized checkpoint, head), which `check_status` classifies as
synced. Later Status requests are answered with the request's own
Status, block and column requests with `ResourceUnavailable`;
`eth2_libp2p` answers Ping and MetaData. Every identity reports IGNORE
for messages it receives, so none re-forwards.

**Target and keys.** The publisher loads the enclave's network config
(`config.yaml` from the genesis artifact) and refuses any
`DEPOSIT_CHAIN_ID` that belongs to one of Grandine's predefined
networks. B is the most recent block that is at least one slot old,
whose envelope is returned by
`/eth/v1/beacon/execution_payload_envelopes/{B}` with
`execution_optimistic: false`, and that is not finalized. Keys come
from the Grandine participant's keystore artifact, fetched with
`kurtosis files download` and decrypted with `eip_2335`. The publisher
selects four distinct active validators, V1–V4, and maps them to
indices through the beacon API. Execution-proof signatures are not
slashable, so using validator keys that are also attesting is safe.

**Messages.** Every envelope targets B and is signed with
`helper_functions`' `ExecutionProofEnvelope` signing, the same code
Grandine verifies with.

- Mock: `MockProof { public_values: ssz(PublicInput), proof }`, with
  the input built as `Store::get_execution_proof` builds it.
  `NewPayloadRequest::new` takes the payload, parent root and requests
  from B's envelope and versioned hashes from the bid in B's block. The
  root is `zkboost_compatible_root()`, the chain ID comes from the
  config, and the schema is `0x1501`. Steps 1 and 2 use different
  `proof` bytes, so their envelope roots differ.
- SP1: the fixture's bytes, unchanged in step 3. In step 4, byte
  1,272,554 is flipped: the first committed public-value byte, as in
  #22's `tampered_public_values_fail_verification`, which pins that
  this gives `Verify`.

**Steps and pass conditions.** The steps run in order. Each step waits
for O to receive the message, or for a window W (default 2 slots).
Grandine's outcome must be logged at least 6 s before W ends, so a
forward would have reached O. A missing outcome makes the run
inconclusive, not a failure.

| Step | Sender, validator | Message | Grandine evidence | At O | Why |
| --- | --- | --- | --- | --- | --- |
| 0 | all | none | R1's startup lines: type 1 `mock`, type 2 `sp1` with VK `0x00662ca6…c88b`. Grandine subscribed to the topic, as seen by all four identities, and in O's mesh for it. | — | Without it, "O received nothing" proves nothing |
| 1 | P_a, V1 | type 1 mock for B | ACCEPT for (B, 1, V1) | Receives it from Grandine, byte-identical | Goal 1 |
| 2 | P_a, V2 | type 1 mock for B, different `proof` bytes | IGNORE, verified proof already known for (B, 1) | Nothing | Shows step 1 was stored, the only external evidence of storage |
| 3 | P_b, V3 | type 2 fixture | REJECT for (B, 2, V3); engine line `verdict: Invalid`, `committed public values do not match the public input`, `proof_type: 2` | Nothing | Goal 2 |
| 4 | P_c, V4 | type 2 fixture, one byte flipped | REJECT for (B, 2, V4); engine line `proof failed verification`, `proof_type: 2` | Nothing | Shows the running verifier discriminates, so step 3's verification is not vacuous |

Steps 1 and 3 are the goals. Steps 0, 2 and 4 are the extra criteria
that make them conclusive. The REJECTs come last, so no step depends on
a peer that was just penalised.

**Evidence sources.** O's receipts and connections come from the
publisher. Grandine's side comes from its log:

- R1's per-type `info!` line (0011), already in #26;
- the engine's verdict line (`proof_engine/src/zkvm_engine.rs`), which
  carries the verdict, the `Rejection` text and the type, but no block
  root or validator. Steps are serialized and the devnet has no other
  proof source, so the evaluator requires exactly one such line for
  the type inside the step's window;
- one terminal-outcome line per message, owned by P1. It names the
  block root, type, validator index, outcome and, for an IGNORE or
  REJECT, the reason. This line does not exist yet (*Open questions*).

T1 adds no code to `p2p`, `fork_choice_*`, `runtime` or
`proof_engine`.

**Driver and evaluation.** `run.sh` builds the image, records its ID
and the Grandine and `eth2_libp2p` commits, starts the enclave, and
waits up to 10 minutes for Grandine to stop syncing with a Gloas head
and a candidate B. `publisher run` writes a JSONL report of each
publish (time, message ID, envelope root, B, type, validator) and each
receipt or connection at O. The driver saves Grandine's
`kurtosis service logs`, and `publisher evaluate` reads the report and
log, prints the step table with each cell marked pass, fail or
inconclusive, and sets the exit code. The bundle (parameters,
versions, report, log, verdict) goes to
`target/eip8025-devnet/<UTC timestamp>/`. The enclave is removed
unless `--keep` is given.

**Sequencing.**

| Part | Needs | Can start |
| --- | --- | --- |
| Publisher identities, messages, report | L1 | Now |
| `evaluate` | P1's outcome line | Once the line is agreed |
| Recipe image | R1's feature | After R1 |
| A passing run | P1c, R1, N1 | After all three merge |

The T1 PR merges after N1, because only N1 moves the submodule pointer
to L1.

## Trade-offs and alternatives

**Use eth-act's Lighthouse and zkBoost as the proof source.** zkBoost's
`docker/example/testnet` already gossips proofs from a Lighthouse
`optional-proofs` node, with no new binary. It is rejected for T1
because it publishes whatever its provers produce: it cannot send the
SP1 fixture against an unrelated payload, a tampered proof, or a
second proof aimed at the stored (B, T) check, and it cannot isolate an
observer. Failures would also mix Grandine defects with cross-client
root differences. It suits a later interoperability milestone.

**An in-process Rust integration test** wiring a Controller to an
`eth2_libp2p` node pair. It would be deterministic and could run in
CI, but it bypasses what T1 exists to cover: the command line, the
feature build, engine construction in `runtime`, a live chain and the
wire. P1's fixture-to-ACCEPT test already covers the in-process path.

**An independent publisher** in Python or Go, with its own SSZ, would
make the mock ACCEPT a cross-check of Grandine's root. It is rejected
because it needs its own libp2p, Status and signing stack, while the
root already has an independent pinned vector (`8c3a8902…`, #20). The
cost is that the mock ACCEPT shows the pipeline is wired and
self-consistent, not that the root is right.

**One publishing identity**, as the roadmap sketched, makes steps 3
and 4 depend on P1's penalty policy and score decay.

**A debug endpoint for storage** instead of step 2 would add API
surface that only a test uses. Step 2 observes storage through
behaviour the spec defines.

**Make step 4 optional.** The step-3 reason already implies that
verification passed. Step 4 stays because it is the only evidence from
the running binary, not from reading code, that the verifier rejects
bad proofs. It costs one verification and a throwaway identity.

**Cost.** A new workspace crate, a second Dockerfile, a dependency on
ethereum-package and Kurtosis behaviour, and negative results that
rest on a timing window.

## Security and compatibility

**Adversarial model.** The publisher is a hostile peer by construction:
it sends a proof bound to the wrong payload and a tampered proof. The
run shows how Grandine treats such a peer. It holds devnet validator
keys only and refuses predefined networks' chain IDs, so it cannot be
pointed at a public network by mistake. The recipe enables no
`port_publisher`.

**DoS and resource use.** T1 adds no surface to Grandine. A run sends
four messages of at most 1.3 MB. Step 2 is IGNOREd before
authentication (0004), so Grandine performs BLS three times, mock
verification once and SP1 verification twice. The ingress
prerequisites in 0011 (bounded queues, in-flight caps, P4 scoring,
pruning) are still unmet, so the recipe is for a closed local enclave
only.

**Invalid or withheld proofs.** Proofs are non-consensus artifacts:
the REJECTed proofs change no state, payload status or fork choice. A
REJECTed or IGNOREd proof is never forwarded, which steps 2–4 check. A
withheld or lost proof gives no outcome line, and the run is
inconclusive, never a pass.

**Mock on type 1.** Under #5762, type 1 is ethrex on OpenVM. A node
configured `1=mock` accepts any type-1 `MockProof` whose public values
match, with no cryptographic check, and treats a genuine OpenVM proof
as `Invalid`, because it is not a `MockProof`. That is
acceptable only on a closed devnet where nothing else publishes type 1.
The chain ID is 3151908; on chain ID 1, 0007 refuses mock at startup.

**Nodes that do not opt in.** T1 changes no production code. The
default Dockerfile, the Makefile and default builds are unchanged, and
only the recipe image enables `execution-proof-sp1`. `make build`
builds `--bin grandine` only. The publisher is a workspace member, so
`make check` and clippy cover it.

**Compatibility.** No wire format, SSZ type, root, schema or option
changes.

**Observability.** No metrics. The run depends on the three log lines
above. The engine line is at `debug`, and the recipe enables it through
`GRANDINE_LOG`.

## Implementation and testing

One PR against `feature/eip8025`, after N1. It adds
`execution_proof_publisher/`, `scripts/eip8025_devnet/`, a workspace
member and lock entries. None of it can be upstreamed, because
`types::eip8025` exists only in this fork.

**Unit tests (publisher).**

- A mock proof built from a fixture envelope and block is `Valid`
  under `ZkvmProofEngine` with a mock type-1 backend, given the input
  `Store::get_execution_proof` builds (`proof_engine` and
  `fork_choice_store` as dev-dependencies only).
- Envelope signatures verify with Grandine's verification path.
- The step-4 tamper flips exactly byte 1,272,554.
- `evaluate`, on recorded report and log fixtures, returns pass for a
  clean run. It returns fail for a forward at O in steps 2–4, a wrong
  reason, or an outcome for the wrong (B, T, V). It returns
  inconclusive for a missing outcome, an outcome logged less than 6 s
  before W ends, or a second peer at O.
- The chain-ID guard refuses every predefined network.

**Devnet acceptance.** On a clean checkout with P1c, R1 and N1 merged,
`scripts/eip8025_devnet/run.sh` exits 0 twice in a row, each time from
a fresh enclave. Both bundles are attached to the PR. Also run
`scripts/ci/clippy.bash --deny warnings` and `cargo fmt -- --check`.

## Open questions

**Before Accepted**

- The terminal-outcome log line: its target, level and fields. The
  publisher's evaluator depends on it, and P1 owns it. It needs
  agreement with the P1 owner. N1's subscription log is not needed,
  because step 0 observes subscription on the wire.

**Settled in the implementation PR**

- The ethereum-package release, and whether it runs Grandine on Gloas
  at genesis with a custom image, `cl_extra_params` and
  `cl_extra_env_vars`.
- Whether one Grandine node sustains the chain, or the recipe needs
  the Lighthouse producer.
- W and the 6 s margin, from the first measured runs.

**Deferred**

- SP1 ACCEPT over gossip, once a devnet prover gossips proofs of live
  blocks.
- Running the ethrex SP1 guest in execute mode on one devnet block and
  comparing its 43 bytes with Grandine's `PublicInput`.
- Adopting #5762 in Grandine (types 4–7, the VK table), and whether
  type 1 stays mock in the recipe once OpenVM verification exists.
- Subnet topics if #5726 merges.
- Running the recipe in CI.
- An interoperability recipe with eth-act's Lighthouse and zkBoost.
