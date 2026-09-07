# Week 11 — Implementing the execution proof data model

This week I started implementing the EIP-8025 data model in Grandine.

I split the work into stacked PRs, starting with progressive SSZ
merkleization, then `ProgressiveByteList`, the execution proof
containers, and finally payload binding
([#2](https://github.com/eip8025-grandine/grandine/pull/2),
[#3](https://github.com/eip8025-grandine/grandine/pull/3),
[#5](https://github.com/eip8025-grandine/grandine/pull/5),
[#6](https://github.com/eip8025-grandine/grandine/pull/6)).

Partway through that work, Keshav pointed me to Francesco's newer
consensus-specs branch. Comparing it with my implementation turned up
several important changes. The gossiped object is now an
`ExecutionProofEnvelope` bound to a `beacon_block_root`, `PublicInput`
has moved to a four-field progressive container, and payload binding
now targets a progressive `SSZNewPayloadRequest`. Progressive SSZ had
also moved out of consensus-specs into the separate `ssz-specs`
repository.

That meant both the implementation and design needed to be updated. I
spent the rest of the week updating [design
0001](https://github.com/eip8025-grandine/design/pull/5) for the new
types and documenting the remaining consensus-specs/EIP differences
around payload binding and the proof-system public input.

For next week, I want to sync the Grandine fork with current upstream
`develop` and rebase, then update the affected PRs to match the
revised design.
