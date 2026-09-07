# Week 10 — Defining the execution proof data model

This week I focused on turning the EIP-8025 data model into a concrete
Grandine design.

I also updated the EIP-8025 proposal in response to Francesco's
feedback. The revision simplifies the consensus-layer model around a
single recursive `ExecutionProof`, with the details of constructing it
from per-payload proofs left opaque to the CL, and updates the sync
model and roadmap around that assumption.

I started [design
0001](https://github.com/eip8025-grandine/design/pull/5), covering the
consensus-layer proof types, their SSZ encoding and merkleization, and
the payload-binding root. One important part of that is progressive
SSZ. `proof_data` needs progressive merkleization so its root does not
depend on `MAX_PROOF_SIZE`. I also worked through how the signed proof
should use Grandine's existing `Hc` wrapper so the same object root
can be reused for gossip de-duplication and signing.

For payload binding, I traced the execution payload types Grandine
already has and how the Engine API request is represented. That
exposed some differences between the types in Grandine and the SSZ
shape EIP-8025 expects, which I documented in the design.

For next week, I want to start implementing the design in Grandine. I
plan to keep the work split into small stacked changes so the SSZ
support, proof containers, and payload-binding work can be reviewed
independently.
