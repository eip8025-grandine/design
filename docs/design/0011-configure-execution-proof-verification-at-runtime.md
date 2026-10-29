---
status: Draft                # Draft | Accepted | Superseded
authors: [creese]
workstream: verification     # gossip | verification | converged
eip_repo: frisitano/EIPs
eip_sha: 4855dbeb9a99702a8c4d948ceceb865fb3289759
consensus_specs_sha: b834fa8411cb2152563e8835c5b72c28f7503e7a
ssz_specs_sha: b42af0a2265c25f5353cace185d066e8d39dbd6e
grandine_upstream_sha: a63876249043020d8ebe3d410c5cf31b19be1cc2
superseded_by:               # Doc number, if any
---

# 0011 — Configure execution-proof verification at runtime

## Context

Grandine `feature/eip8025` (`67195ede`) has the parts that verify a
proof, but nothing at runtime uses them:

- [0006](0006-make-proof-engine-validation-only.md) and
  [0008](0008-distinguish-invalid-proofs-from-local-verifier-failures.md)
  define `ProofEngine { is_null, verify_execution_proof -> ProofVerdict }`.
  `NullProofEngine` is the opt-out.
- [0007](0007-verify-execution-proofs-in-the-proof-engine.md) defines
  `ZkvmProofEngine::new(&Config, (ProofType, BackendSpec)…)`. Its
  construction rules are no empty list, no duplicate type, an SP1 VK
  that decodes, and no `ZkboostMock` when `deposit_chain_id == 1`.
  `BackendSpec::Sp1` exists only with the non-default `proof_engine/sp1`
  feature. 0007 left the CLI and runtime construction out of scope.
- [0009](0009-add-execution-proof-gossip-to-eth2-libp2p.md) gives
  `eth2_libp2p` one startup switch,
  `NetworkConfig::enable_execution_proof`, which defaults to `false`.
  It is merged on the fork (`7401336`), but Grandine's submodule
  pointer is still at `55cdfba`, which does not have the field.

None of these reach a running node. `runtime` has no option for proof
types, and no `grandine` feature turns on `proof_engine/sp1`.
`Controller::new` takes no engine. `spawn_execution_proof_task` takes
one as an argument and is dead code.

This design covers milestone R1 of the EIP-8025 roadmap. The roadmap
divides the remaining integration work into three milestones:

- **P1, the proof pipeline.** This covers the Store lifecycle (open
  Grandine #25, design
  [0010](https://github.com/eip8025-grandine/design/pull/9)) and the
  Controller and task wiring, which is in a PR not yet published. In
  [0004](https://github.com/eip8025-grandine/design/pull/9), the
  pipeline's null-engine guard IGNOREs a proof before preparation.
- **N1, networking.** This covers p2p routing and the
  `execution_proof` subscription.
- **R1, runtime configuration (this design).** This covers building
  the engine and deciding whether verification is enabled. R1 takes
  on none of the P1 or N1 responsibilities.

P1 and N1 are owned by the other workstream. Neither consumer
interface is in a published design: 0010 at `43d05db1` covers only the
Store lifecycle. Both interfaces have been agreed with the P1 and N1
owner, and this design records them (*Hand-off to P1* and *Hand-off to
N1*).

**Spec.** In
[`p2p-interface.md`](https://github.com/ethereum/consensus-specs/blob/b834fa8411cb2152563e8835c5b72c28f7503e7a/specs/_features/eip8025/p2p-interface.md),
a node that is execution proof-aware subscribes to `execution_proof`.
Gossip validation REJECTs any type outside
`get_supported_proof_types()`, which is `{1, 2, 3}`. The EIP calls the
set of types a node handles "a per-node, dynamic configuration".

**Provisional upstream, and the impact if it moves.**

- **Type assignments.** `beacon-chain.md` says the assignments are
  provisional. The parser reads `get_supported_proof_types()`, so if
  the set changes, the parser follows it with no code change.
- **[#5726](https://github.com/ethereum/consensus-specs/pull/5726)**
  (open, `4ff41108`) would replace the topic with subnets
  `proof_type % 8`. N1 would then need the set of configured types
  rather than a single enabled flag. R1 already keeps that list as
  plain data in `BeaconNodeConfig`, where N1 can read it. The engine
  trait is unaffected.
- **[#5729](https://github.com/ethereum/consensus-specs/pull/5729)
  and [#5753](https://github.com/ethereum/consensus-specs/pull/5753)**
  (open) change the statement a proof binds. They would require new
  guests and new VKs, which operators supply through the same option.
  R1 is unaffected.
- **EIP divergence.** The EIP advertises supported types through
  `ExecutionProofStatus` and the `eproof` ENR key. consensus-specs has
  the ENR key and no status message. Both are deferred.

## Goals and non-goals

**Goals**

- With no option, the node runs `NullProofEngine`, makes no
  subscription and behaves as it does today (acceptance 1).
- `--execution-proof-type <TYPE>=mock|sp1:<VK>`, repeated, builds one
  `ZkvmProofEngine` with exactly those mappings (acceptance 2).
- One engine value, plus one "verification enabled" predicate taken
  from it, reaches P1 and N1. Neither has to re-derive it
  (acceptance 3).
- Every misconfiguration stops startup before genesis or anchor
  loading. That covers mock on mainnet, `sp1` without the feature, an
  unsupported or duplicate type, and a malformed VK (acceptance 4).
- Any supported type, backend and guest VK can be configured on any
  network, Sepolia included. The only network-specific refusal is
  0007's mock-on-mainnet rule.
- A `grandine` Cargo feature turns on SP1 verification. Release builds
  succeed with and without it (acceptance 5).

**Non-goals**

- The pipeline, Seen handling, gossip outcomes (P1, 0004, 0010) and
  routing or subscription (N1, 0009).
- A configuration file, built-in VKs or named VK presets.
- Changing configuration after startup, although the EIP describes
  the supported set as dynamic.
- The `eproof` ENR key, `ExecutionProofStatus`, metrics and HTTP
  ingress.
- Any change to 0007's construction rules or to the `ProofEngine`
  trait.
- The devnet image and recipe (T1).

## Design

```text
--execution-proof-type 1=mock --execution-proof-type 2=sp1:0x0066…
   │ grandine_args (parse time)
   │   syntax, TYPE ∈ get_supported_proof_types(), VK = 32 bytes hex,
   │   sp1 needs runtime/execution-proof-sp1
   ▼
BeaconNodeConfig.execution_proof_backends: Vec<(ProofType, BackendSpec)>
   │ run_beacon_node, after the command dispatch, before genesis/anchor load
   │   []    → NullProofEngine
   │   pairs → ZkvmProofEngine::new(&chain_config, pairs)?   (0007's rules)
   ▼
proof_engine: Arc<dyn ProofEngine>  ── built once, cloned to each consumer
   ├──▶ Controller::new(…, proof_engine)              P1: pipeline, null guard
   └──▶ network_config.enable_execution_proof
          = !proof_engine.is_null()                   N1: before Network::new
```

**Option.** `BeaconNodeOptions` in `runtime/src/grandine_args.rs` gets
`--execution-proof-type`, with `value_name = "TYPE=BACKEND"`. It may be
repeated, with one pair per occurrence and no comma splitting. The
`FromStr` parser produces `(ProofType, proof_engine::BackendSpec)`:

| Input | Result |
| --- | --- |
| `<TYPE>=mock` | `BackendSpec::ZkboostMock` |
| `<TYPE>=sp1:<VK>` | `BackendSpec::Sp1 { program_vk }`, only with the feature |
| `TYPE` | Decimal `u8` that must be in `get_supported_proof_types()` |
| `VK` | Exactly 32 bytes of hex, with or without `0x`, as `cargo prove vkey` prints it |

The parser uses `BackendSpec` directly, so there is no second enum to
keep in step with it. Without the feature, `Sp1` does not exist, and
the parser returns "built without `execution-proof-sp1`" for any `sp1:`
value. `mock` maps to zkboost mock verification, not to the test
double `MockProofEngine`. The option's help text states this
distinction, and states that mock verification performs no
cryptographic check.

**Where each error is raised.** Each rule is checked in one place:

| Error | Raised by | When |
| --- | --- | --- |
| Bad syntax, unknown backend, `TYPE` not a `u8` | parser | argument parsing |
| Type outside `{1, 2, 3}` | parser | argument parsing |
| VK not 32 bytes of hex | parser | argument parsing |
| `sp1` in a build without the feature | parser | argument parsing |
| Duplicate type | `ZkvmProofEngine::new` | runtime start |
| Mock with `deposit_chain_id == 1`: `--network mainnet`, or a custom config with chain ID 1 | `ZkvmProofEngine::new` | runtime start |
| VK that is 32 bytes but non-canonical (ere `NonCanonicalProgramVk`, for example a non-zero first byte) | `ZkvmProofEngine::new` | runtime start |

Duplicates and mainnet are left to the engine, which already owns and
tests those rules (0007). Repeating them in the parser would give two
owners for one rule.

**Construction.** `run_beacon_node` (`runtime/src/runtime.rs`) builds
the engine after the `if let Some(command)` dispatch, because commands
do not need it. It builds it before `load_anchor_checkpoint_provider`,
which can download state or wait for deposits. A misconfiguration is
therefore reported before any slow startup work. An empty list gives
`NullProofEngine` without calling `new`, so `NoBackends` cannot occur.
The error from `new` is returned with the option named in its context.
SP1's one-time verifier setup also runs here, as 0007 intends. The
engine is passed to `build_local_node` in `LocalNodeConfig`.

On success, R1 logs one `info!` line per type, naming the backend and
the VK. The VK is public, and logging it lets operators confirm which
guest the node trusts. The engine already logs a `warn!` for each mock
type. Without the option, nothing is logged and behaviour is
unchanged.

**Hand-off to P1.** As agreed with the P1 owner, `Controller::new`
takes a required `Arc<dyn ProofEngine>`, not an `Option`, named
`proof_engine` per 0006's guidance. P1 owns adding this parameter and
wiring it into the pipeline. Its Controller PR has the runtime supply
`NullProofEngine` at first, and R1 then supplies the constructed engine
instead. R1 does not change the pipeline, the task or the null guard.

**Hand-off to N1.** As agreed with the N1 owner, "verification
enabled" means `!proof_engine.is_null()`, and R1 defines no other
flag. In `build_local_node`, before `Arc::new(network_config)` and
`Network::new`, one line sets `network_config.enable_execution_proof`
from that predicate. The field exists only after the submodule bump
that N1 carries, so N1 adds this line, together with routing in `p2p`.
R1's obligation is limited to making `proof_engine` available at that
point. As 0009 notes, the same flag controls the core topic, the
whitelist and the topic-count limit.

**Feature.** The `grandine` crate gets `execution-proof-sp1 =
['runtime/execution-proof-sp1']`. The `runtime` crate gets
`execution-proof-sp1 = ['proof_engine/sp1']` and a direct `proof_engine`
dependency. It already links `proof_engine` through
`fork_choice_control`. The feature is off by default and is not part of
`default-networks`. The runtime checks it with
`cfg(feature = "execution-proof-sp1")`. If `proof_engine/sp1` is
switched on only through feature unification, the parser still refuses
`sp1`, so the node fails closed.

## Trade-offs and alternatives

**A separate on/off switch, such as `--enable-execution-proofs`, next
to the type list.** This is the most likely alternative, because it
keeps the network decision separate from the verifier. It is rejected
because it creates two sources of truth that can disagree:

- **Subscribed with a Null engine.** Every proof is IGNOREd and none is
  forwarded, so the node takes in mesh traffic without relaying it.
- **Engine configured but not subscribed.** The node verifies nothing.

`is_null` already exists for this purpose (0006), and 0004's guard
uses it.

**Pass the parsed list to the Controller and let P1 build the engine.**
This is rejected. P1 and `fork_choice_control` would then depend on
backend configuration and the Cargo feature, and their tests could no
longer inject `MockProofEngine`. Building the engine in the runtime
matches how the runtime already passes a ready `execution_engine` to
`Controller::new`.

**A configuration file**, such as JSON with one entry per type. This is
rejected for now. Each type has one backend with at most one 32-byte
argument. A flag fits Kurtosis `cl_extra_params` and needs no mount.
The roadmap adopts this form.

**Check duplicates and mainnet in the parser** to get clap-style
errors. This is rejected because those rules would then be defined in
two places. The engine's errors are still raised before anything slow
starts.

**Refuse the option on predefined networks until resource limits
exist.** This alternative follows from 0009's ingress concern
(*Security and compatibility*). It is rejected for three reasons:

- The protections that concern needs, bounded queues and rate limits,
  belong where the work is queued and done: P1, N1 or follow-up work.
  A refusal at startup would only stand in for them.
- The refusal would key on `--network`, and a custom
  `--configuration-directory` for the same network gets past it, so it
  would not be a real boundary.
- It would block legitimate testing with real guests on Sepolia.

Protocol constraints and backend safeguards still apply on every
network. The type must be in `get_supported_proof_types()`, and 0007's
mock-on-mainnet rule still holds. The cost is that nothing at startup
stops an operator from exposing an unprotected node. The prerequisites
are documented in *Security and compatibility* instead.

**Name the feature `sp1`**, as the roadmap suggested. This is rejected
because `zkvm_host` already has an `sp1` feature, which means SP1
proving through `sp1-sdk`. `execution-proof-sp1` says what the feature
switches on.

**Turn on SP1 by default.** This is rejected. 0007 keeps zkVM code out
of default builds, which leaves the Windows, macOS x64 and i686 builds
unchanged, and spares every default build ere's SP1 dependency tree.

**Cost.** Adding the option touches `grandine_args.rs`, a large,
upstream-tracked file, and it changes the order of startup. R1 also
depends on the P1 Controller parameter landing first.

## Security and compatibility

**Adversarial model.** The operator and the command line are trusted.
The VKs and backend choices are trust roots (0007). A wrong VK makes
honest proofs `Invalid` (0008). R1's objective is that a
misconfiguration produces a startup error rather than a change in
runtime behaviour. Nothing R1 adds is reachable by peers. R1 only
decides whether the gossip surface of 0004 and 0009 exists.

**DoS and resource use.**

- Opting in opens `execution_proof` ingress. This includes BLS per
  proof and up to one SP1 verification per (B, T, V), whose cost is
  unmeasured (0007, 0008).
- **Unprotected ingress (0009).** Execution-proof gossip has no
  dedicated rate limit and no bounded ingress queue. `eth2_libp2p` puts
  every decoded message straight onto the service's event stream.
  Behind that sits work that is expensive and of unmeasured cost: BLS,
  preparation, and SP1 verification. On a public network, any peer can
  drive that work, and a node that opts in can be pushed into CPU and
  memory exhaustion. Gloas activated on Sepolia on 2026-10-06, at
  epoch 353,024 (`types/src/config.rs`), so this exposure already
  applies to any Sepolia node that opts in.
- Construction costs one SP1 verifier setup at startup, shared by all
  SP1 types. Its time and memory are unmeasured.
- A partial configuration, such as type 1 only, still subscribes to
  the global topic. Proofs of types 2 and 3 cost BLS, then come back
  `Undetermined`, then IGNORE, and are not forwarded (0008). Under
  #5726 the node would subscribe only to its own subnets.

**Prerequisites for public-network deployment.** R1 allows opting in
on any network, but it neither implements nor enforces these
protections. Until they land, an operator who opts in on a public
network accepts the exposure described above.

| Prerequisite | Source | Owner |
| --- | --- | --- |
| Bounded ingress queue and rate limit for `execution_proof`, between the `eth2_libp2p` event stream and the pipeline | 0009 | N1, or follow-up |
| Finite limits on queued bytes, work and concurrency for proof tasks, including a cap on in-flight verifications | 0004; the roadmap defers the in-flight cap | P1, or follow-up |
| Topic score parameters, with at least an invalid-delivery (P4) penalty | 0009, deferred | N1, or follow-up |
| Physical pruning of stored proofs and Seen keys at finalization | 0004 and 0010 defer it; the roadmap requires it "before a node runs for long" | P1 follow-up |
| Measured SP1 verification cost and rayon contention, to size the limits above | 0007 | Follow-up |

**Invalid or withheld proofs.** Nothing changes. Proofs are
non-consensus artifacts, and no verdict affects state, payload status
or fork choice. A node can never accept a proof of an unconfigured
type, and SP1 never falls back to mock. Each type has exactly one
backend, and the runtime has no fallback path.

**Mainnet.** `ZkboostMock` is refused when the engine is built if
`deposit_chain_id == 1`. That covers `--network mainnet` and any custom
config with chain ID 1. The engine also refuses mock on chain ID 1 at
verification time. Real backends are allowed on every network.

**Nodes that do not opt in.** If the option is absent, no
`ZkvmProofEngine` is built, the engine is `NullProofEngine`, and after
N1 the topic, whitelist and limits are those of 0009's "flag off".
Default builds contain no SP1 code. Building with the feature but
passing no option behaves exactly like the default. The feature stays
default-off.

**Compatibility.** There is a new CLI option and a new Cargo feature.
No wire format, SSZ type, root or database schema changes. Existing
command lines parse as before.

**Observability.** There is one `info!` line per configured type at
startup, plus 0007's mock `warn!`. Metrics are deferred.

## Implementation and testing

One PR against `feature/eip8025`, after P1's Controller PR. If that PR
is delayed, R1 waits; it does not add the parameter itself. It touches
`grandine/Cargo.toml`, `runtime/Cargo.toml` and
`runtime/src/{grandine_args,grandine_config,runtime}.rs`. Parsing and
construction go in a small `runtime` module. None of it can be
upstreamed on its own, because `types::eip8025` and `proof_engine`
exist only in this fork.

**Tests** (`runtime`, through `config_from_args` and the construction
helper):

- No option gives an empty list and `is_null()`.
- `1=mock` on a custom config parses, and the engine is not null.
- These are rejected at parse time: `0=mock`, `4=mock`, `256=mock`,
  `1=foo`, `1`, `=mock`, `2=sp1:` with 31 or 33 bytes, and non-hex VKs.
- Without the feature, `2=sp1:<ethrex VK>` gives the "built without"
  error. With the feature it parses to `Sp1`.
- No network is refused at config time. `--network sepolia 1=mock`
  builds an engine. With the feature, `--network sepolia
  2=sp1:<ethrex VK>` and `--network mainnet 2=sp1:<ethrex VK>` also
  build engines.
- `--network mainnet 1=mock` is a startup error, `MockOnMainnet` from
  construction.
- In construction under `Config::mainnet()`, `1=mock` gives
  `MockOnMainnet`. `1=mock 1=mock` gives `DuplicateProofType`. With the
  feature, `0x01` followed by 31 zero bytes gives
  `InvalidVerifyingKey`, and the ethrex v26 VK
  (`00662ca6…c88b`) succeeds.

**Run.**

- The `runtime` tests, with and without `runtime/execution-proof-sp1`.
- `cargo build --release --bin grandine --features default-networks`
  (acceptance 5, without SP1).
- `cargo build --release --bin grandine --features
  default-networks,execution-proof-sp1` (with SP1). On a cold cache,
  ere's `build.rs` needs the network (0007). CI gains this build next
  to its `proof_engine/sp1` test step.
- `scripts/ci/clippy.bash --deny warnings`, also with the feature, and
  `cargo fmt -- --check`.

The P1 and N1 PRs test their own effects: the null guard, and
subscription only when the engine is not Null. The end-to-end check is
the roadmap's devnet recipe (T1), which uses `--execution-proof-type
1=mock --execution-proof-type 2=sp1:0x00662ca6…c88b`. No spec vectors
cover configuration.

## Open questions

**Before Accepted:** none.

**Deferred**

- The public-network prerequisites in *Security and compatibility*,
  owned by P1, N1 or follow-up work.
- Per-type subscription if #5726 merges.
- The `eproof` ENR key and `ExecutionProofStatus`.
- Metrics for configured types and per-type verdicts.
- Built-in or named VKs, and a configuration file.
- Changing configuration after startup.
- Mock policy on public testnets.
- The devnet image with the feature (T1).
