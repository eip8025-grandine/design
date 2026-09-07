# Week 12 — Updating the data model for consensus-specs

This week I focused on bringing the EIP-8025 work onto current
Grandine and aligning it with current consensus-specs.

The first surprise was that upstream `develop` had already picked up a
substantial amount of progressive SSZ support. That made my original
progressive merkleization PR redundant, so I dropped and closed
[#2](https://github.com/eip8025-grandine/grandine/pull/2) rather than
carrying a duplicate implementation.

I rebased [#3](https://github.com/eip8025-grandine/grandine/pull/3)
onto the new base. Grandine now has `ProgressiveByteList<N>`, so the
PR became focused on reference root and boundary coverage rather than
adding another byte list implementation.

I then rebased
[#5](https://github.com/eip8025-grandine/grandine/pull/5) and brought
the proof containers in line with the newer consensus-specs design.
`PublicInput` is now the four-field progressive container,
`ExecutionProofEnvelope` carries the `beacon_block_root` used for
gossip binding, and `SignedExecutionProofEnvelope` wraps that
envelope. `ExecutionProof` remains the local `ProofEngine` input.

I finished rebasing the remaining payload binding work in
[#6](https://github.com/eip8025-grandine/grandine/pull/6) and updated
[design 0001](https://github.com/eip8025-grandine/design/pull/5) to
include the separate `ssz-specs` revision used by consensus-specs. I
also updated the design doc template and contributor guidance to pin
`ssz-specs` alongside the EIP, consensus-specs, and Grandine
baselines. That change was merged in [design
#1](https://github.com/eip8025-grandine/design/pull/1).

Outside the implementation work, I submitted the project proposal to
Devcon.

Next I want to move on to `ProofEngine`: implementing its verification
interface, integrating the external verifier, and wiring in the
new-payload and fork-choice notifications needed for proof-state
management.
