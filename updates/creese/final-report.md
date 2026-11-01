# EPF7 Final Report: Implementing EIP-8025 Optional Execution Proofs in Grandine

Charles Reese · EPF Cohort 7 · 1 November 2026

## Abstract

During EPF7, I worked with Keshav Sharma to add optional execution-proof verification to Grandine, an Ethereum consensus client written in Rust. [EIP-8025](https://eips.ethereum.org/EIPS/eip-8025) introduces proof verification alongside the existing execution-client validation path, allowing the ecosystem to build confidence in execution proofs without changing consensus.

Our implementation receives signed proofs over gossip, derives the expected public input from the block’s accepted payload context, verifies the proof and checks that its committed public values match that input, and decides whether to ignore, reject, or store and forward it. Verification is off by default.

My contributions cover the data model and payload binding, proof-verification backends, gossip transport, runtime configuration, and devnet testing. I demonstrated cryptographic verification of a genuine SP1 execution proof through the running Grandine pipeline on a private devnet. The remaining work toward Sepolia is to stabilize the test harness, add the backend selected for the Sepolia prover, and verify a proof on the live testnet.

The updated project proposal ([cohort-seven#297](https://github.com/eth-protocol-fellows/cohort-seven/pull/297)) describes the intended scope. Implementation and design references appear below.

## Final status and roadmap

### How the scope evolved

The original plan included proof retrieval over RPC, proof synchronization, and a broader ProofEngine interface. As the upstream specification evolved, RPC retrieval was removed in favor of recursive proofs, which later also moved outside EIP-8025. The ProofEngine interface became verification only, and proof-state management moved into the Store and processing pipeline. The proving-stack integration also changed from the initially planned zkboost interface to in-process verification through Ere.

We repeatedly revised the roadmap and rebalanced tasks as the scope of each workstream changed. In August, we assigned gossip transport to Keshav; I ultimately implemented the `eth2_libp2p` changes as his Store responsibilities grew.

The revised project targets Sepolia readiness and preserves existing behavior for nodes that do not opt in. Most of my implementation PRs are merged into our project fork. They have not been merged upstream into Grandine, and the devnet implementation PR remains open.

### Data model and payload binding

I implemented the execution-proof containers, including `ProofData`, `PublicInput`, and the signed gossip envelope ([grandine#5](https://github.com/eip8025-grandine/grandine/pull/5)). I then added the payload-request representation and root computation needed to bind a proof to the payload it certifies ([grandine#6](https://github.com/eip8025-grandine/grandine/pull/6)).

Keeping these types aligned with the evolving specifications took more work than any individual subsystem. I updated `ProofData` from a progressive byte list to a bounded SSZ byte list, renamed `SszNewPayloadRequest` to `NewPayloadRequest`, and added progressive versioned hashes with type-level limits ([grandine#18](https://github.com/eip8025-grandine/grandine/pull/18)). These choices affect hashing as well as representation.

Integration exposed another discrepancy: consensus-specs used a progressive `NewPayloadRequest` container, while zkboost and its execution-validation guests used an ordinary SSZ container. The same request bytes therefore produced different roots. I added a separate zkboost-compatible root computation while preserving Grandine’s consensus-spec representation and canonical root ([grandine#20](https://github.com/eip8025-grandine/grandine/pull/20)).

I checked both roots against the Python `ssz-specs` reference implementation using a zkboost fixture. The compatible root also matches Prysm’s reference value for the same fixture. Independent reference coverage for the other composed proof and signing roots remains unfinished.

### Proof verification

Keshav built the [initial verification and proof-generation interfaces](https://github.com/eip8025-grandine/grandine/pull/9), along with a no-op implementation for nodes with proof verification disabled. I aligned the proof engine with the revised verification-only specification and removed the unused generation interfaces ([grandine#19](https://github.com/eip8025-grandine/grandine/pull/19)).

I added configurable backends and zkboost-compatible mock verification, including checks against the expected public input ([grandine#21](https://github.com/eip8025-grandine/grandine/pull/21)). I then integrated genuine SP1 verification through Ere ([grandine#22](https://github.com/eip8025-grandine/grandine/pull/22)). The backends compare committed public values with the expected input’s SSZ bytes, matching the proving stack’s encoding rather than the public-input root described in the consensus specification. Tests use an ethrex SP1 proof borrowed from zkboost and cover the correct verifying key, an incorrect key, tampering, and mismatched expected public input.

The original boolean result could not distinguish an invalid proof from a node’s inability to verify it—for example, because no backend was configured for that proof type. Treating both cases as invalid could wrongly penalize a gossip peer. I introduced `Valid`, `Invalid`, and `Undetermined` outcomes so the processing pipeline can handle local failures separately ([grandine#24](https://github.com/eip8025-grandine/grandine/pull/24)).

### Gossip transport and runtime configuration

I added the opt-in execution-proof topic to `eth2_libp2p`, including typed SSZ/Snappy transport, message-size checks, activation gating, and tests between two gossip peers ([eth2_libp2p#1](https://github.com/eip8025-grandine/eth2_libp2p/pull/1)).

I implemented runtime configuration through a repeatable `--execution-proof-type` option, with one backend setting per proof type ([grandine#26](https://github.com/eip8025-grandine/grandine/pull/26)). SP1 verification requires an explicit build feature, and mock verification cannot be configured on mainnet. Without configuration, proof subscription and verification remain disabled. The execution client still receives and validates payloads either way.

### Code review 

Because we could not find a mentor, Keshav and I reviewed each other’s PRs. I checked whether tests covered the intended behavior and whether the changes preserved normal node operation. For example, in the [Store lifecycle review](https://github.com/eip8025-grandine/grandine/pull/25#discussion_r4411298891), I caught unnecessary proof-related work running even on nodes that had not opted in. Keshav corrected this in the subsequent integration PR.

### End-to-end devnet results

I built a Kurtosis runner with Grandine and Geth, a signed-proof publisher, and an observer that checks what Grandine forwards ([grandine#29](https://github.com/eip8025-grandine/grandine/pull/29)). I reused the genuine ethrex SP1 fixture from zkboost to test verification without setting up a prover. That proof was produced for a different payload from the one on the devnet.

By inspecting Grandine’s logs and the observer’s records across separate runs, I observed these outcomes:

| Scenario | Observed result |
| --- | --- |
| Mock proof matching the accepted payload | Grandine accepted and stored it; the observer received the same message from Grandine. |
| A second mock proof for the same block and proof type | Grandine ignored it because a verified proof was already stored. |
| Genuine SP1 proof for a different payload | Cryptographic verification succeeded; the committed public values did not match the expected public input, so Grandine rejected it. |
| SP1 fixture with one bit flipped in its public values | Cryptographic verification failed, and Grandine rejected it. |

This demonstrated genuine SP1 verification through the running pipeline on a private devnet.

The automated verdicts remain inconclusive. Three saved runs have empty captured Grandine logs. Another contains the relevant log lines but still has a saved verdict reporting missing startup and outcome evidence. Sequential runs also lost peer connections and stopped when gossip publication failed with `NoPeersSubscribedToTopic`. In the isolated SP1 runs, verification completed before the later disconnections. I still need to fix evidence collection and evaluation and keep the test peers connected throughout a complete run.

### Remaining work

To reach the Sepolia goal, I need to:

1. Make the devnet runner reliably execute and evaluate all four scenarios in one command.
2. Add OpenVM verification for the guest selected by the Sepolia proving effort, with the matching verifying key and public-input representation. That choice was announced after I implemented SP1 support.
3. Run our Grandine fork on Sepolia and verify a proof for a live payload.

Physical pruning of retained proof state and limits on concurrent verification remain follow-up work for long-running, opted-in nodes.

## Impact and next steps

Execution proofs could let Ethereum increase execution throughput without requiring every validating node to repeat all of that computation. That would help keep independent validation accessible as the gas limit rises. EIP-8025 is an optional step toward that possibility: execution clients still validate payloads, while proof verification can be exercised against real chain data.

EIP-8025 was [declined for inclusion in Hegotá at ACDC #189](https://raw.githubusercontent.com/ethereum/pm/master/.github/ACDbot/artifacts/acdc/2026-10-15_189/transcript_corrected.vtt). I want to keep momentum and demonstrate results. Optional proof verification provides a way to build confidence in the proving stack before committing to a timeline for moving away from re-execution. This implementation is a contribution toward testing that path in another consensus client.

After the Sepolia work, I plan to repackage our integration branch into a series of stacked PRs for submission to Grandine upstream. I will then ask Francesco Risitano (TauLepton) where I can contribute next. There is a lot left to do, and I want to keep working on it.

Keshav and I are also preparing to present the implementation and devnet results at [Devcon 8](https://cfp.devcon.org/devcon8/talk/8FCR7F/).

## Self-evaluation

### What I learned

I joined EPF7 permissionlessly with no previous protocol-development or client-code experience and no professional experience with Rust. I wanted to learn by making a useful contribution to Ethereum scaling.

I want Ethereum to support a more equitable financial system without giving up the decentralization that makes it valuable. For that to happen, it needs to serve many more people while keeping independent validation accessible. Execution proofs offered a concrete way to work toward both goals.

The amount of upstream design churn was the biggest surprise of the project. I expected to spend most of my time learning Rust and Grandine; instead, much of the difficulty was establishing what to implement and keeping it compatible with the proving stack. Once the data model and verification responsibilities settled, the backend, transport, and runtime work moved quickly. Seeing a genuine SP1 proof reach the verifier through a running devnet was the most exciting result of the fellowship for me.

I learned to prioritize independent reference checks early. I had investigated them without knowing how to complete them or appreciating their importance, then merged the data-model work before finishing those checks. The later request-root mismatch made the lesson concrete: agreement among tests built from my own interpretation does not establish agreement with another implementation.

I also became much more comfortable navigating an unfamiliar client codebase. At the start, I knew about EIPs but not how consensus-specs, execution-specs, and client implementations fit together. Understanding EIP-8025 required reading both Grandine and Lighthouse, following discussions, and asking developers where the current decisions lived. I learned to use Kurtosis for devnet testing and became more willing to ask core developers for clarification when the documents were incomplete. 

### Practices that helped

I created the shared `eip8025-grandine` organization so the project would feel equally owned, rather than having one of us contribute to a fork under the other’s account. I also created a design repository and proposed writing a design before implementation, with a template and review checklist refined with Keshav’s feedback ([design#1](https://github.com/eip8025-grandine/design/pull/1)). Writing down the approach forced me to resolve gaps in my understanding and gave each of us enough context to work asynchronously.

The designs also made AI-assisted review more useful. I wrote a Claude `/audit` skill to compare a PR against its design and used it on both my changes and Keshav’s. Its most consistent contribution was identifying missing test coverage. It also helped catch integration assumptions: while implementing runtime configuration, an audit exposed problems I had missed because I was developing against the wrong `eth2_libp2p` checkout.

Later, I created `/audit-pr` to help me review Keshav’s PRs in unfamiliar parts of Grandine. It extended `/audit`’s checks against the design with a file-by-file walkthrough of what changed and why. Reading it alongside the diff and asking follow-up questions helped me understand code I had not written and review it more effectively. The code and tests remained the basis for my review. I wish I had built it earlier.

## Feedback on EPF7

I greatly appreciated Mario continuing to organize our weekly standups. They gave me a reason to make steady progress and a place to meet people who cared about Ethereum as much as I did. Keshav reached out after hearing one of my updates, and that connection became the foundation of our project. Mario’s support for presenting at Devcon was another unexpected encouragement: as a permissionless participant, I had not expected a conference stipend, and having a presentation to work toward pushed me to contribute more.

The early emphasis on understanding the protocol before choosing a project was valuable. It gave me time to learn about fork choice and other foundations I would otherwise have rushed past.

One small improvement would be to limit each mentor’s proposed projects to perhaps one to three, depending on their availability. That would help mentors focus on the projects they are most excited about and support the participants who choose them.

The AMAs were valuable, though their quality and usefulness varied. I appreciated the speakers making time for us; clearer preparation around participants’ questions and experience levels could help each session deliver more. EPF should aim to raise the quality of the AMAs and maintain a consistently high standard across the program.

## Project references

- **Implementation:** [Grandine fork](https://github.com/eip8025-grandine/grandine) and [eth2_libp2p fork](https://github.com/eip8025-grandine/eth2_libp2p).
- **Specifications:** [EIP-8025](https://eips.ethereum.org/EIPS/eip-8025) and the [consensus-layer feature specifications](https://github.com/ethereum/consensus-specs/tree/master/specs/_features/eip8025). The EIP is the canonical proposal link; the project’s designs record the specific revisions and compatibility decisions used for implementation.
- **Development updates:** The [cohort index](https://github.com/eth-protocol-fellows/cohort-seven/blob/main/development-updates.md) links several of my updates. Weeks 10–12 are also preserved in the design repository ([design#6](https://github.com/eip8025-grandine/design/pull/6), [design#7](https://github.com/eip8025-grandine/design/pull/7), [design#8](https://github.com/eip8025-grandine/design/pull/8)).

## Acknowledgments

Thank you to Keshav for reaching out, sharing my enthusiasm for Ethereum scaling, and building this with me; to Mario for his steady support and for making EPF7 a welcoming community; to Francesco for clarifying the evolving scope, reviewing our proposal, and including me in testnet discussions; and to the eth-act community for answering questions and helping me find my way into the work.
