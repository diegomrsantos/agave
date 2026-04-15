---
title: Validator Resilience and Constellation Research Notes
---

## Scope

This note captures working conclusions from a research pass across five related
threads:

1. validator resilience and Invalidator interview preparation
2. current Agave behavior around transaction history and transaction status
   service
3. public code signals from Faycel Kouteib
4. local stress testing against current Agave
5. early Ideas-stage design work around Constellation proposer interfaces

This is not a formal proposal. It is a working note meant to preserve what was
learned and where the strongest next steps seem to be.

## Invalidator and interview framing

The strongest public framing for Invalidator comes from Anza's blog post
["Strengthening Solana: Meet the Invalidator Team"](https://www.anza.xyz/blog/strengthening-solana-the-invalidator-team).
The important point is that Invalidator is publicly described as resilience
engineering, not offensive exploitation. The team was formed after the February
2024 outage and focuses on adversarial testing of realistic validator failure
domains, including:

- denial-of-service pressure on validator ports
- packet loss and packet delay
- malicious leaders and malicious users
- economic stress
- synthetic load in private clusters and testnet-like environments

The best preparation angle is therefore not "how to attack Solana" but "how a
validator fails under adversarial or pathological conditions, and which
subsystems determine whether it keeps making progress."

The most important study topics are:

- consensus validator versus RPC or hybrid node responsibilities
- TPU ingress, leader windows, scheduler behavior, hot-account contention, and
  transaction landing
- TVU and replay correctness under lag and corruption
- gossip and the control plane
- blockstore, restart, and snapshot behavior
- vote processing
- RPC and hybrid-node overload as a separate, secondary track

Useful public references:

- [Validator or RPC node](https://docs.anza.xyz/operations/validator-or-rpc-node)
- [Validator anatomy](https://docs.anza.xyz/validator/anatomy)
- [TPU](https://docs.anza.xyz/validator/tpu)
- [TVU](https://docs.anza.xyz/validator/tvu)
- [Gossip](https://docs.anza.xyz/validator/gossip)
- [Blockstore](https://docs.anza.xyz/validator/blockstore)
- [Transaction landing on TPU](https://www.anza.xyz/blog/transaction-landing-on-tpu)
- [Introducing the central scheduler](https://www.anza.xyz/blog/introducing-the-central-scheduler-an-optional-feature-of-agave-v1-18)
- [January 2026 gossip and vote processing security patch summary](https://www.anza.xyz/blog/january-2026-gossip-and-vote-processing-security-patch-summary)
- [Precompile alignment bug root cause analysis](https://www.anza.xyz/blog/precompile-alignment-bug-root-cause-analysis)

## Current Agave findings

### Transaction status service

On current local `master`, transaction status service is single-threaded again
in `rpc/src/transaction_status_service.rs`. That matters because earlier public
history around transaction status service included a multithreaded redesign that
was later reverted.

The practical conclusion is that any claim about "current Agave TSS behavior"
has to be anchored to current code, not to the older multithreaded design work.

### Transaction history defaults

The current production-facing default is still to keep transaction history off
unless explicitly enabled.

Relevant code paths:

- `rpc/src/rpc.rs` contains the `JsonRpcConfig` defaults.
- `validator/src/commands/run/args/json_rpc_config.rs` documents that
  `--enable-rpc-transaction-history` increases disk usage and IOPS, and that
  extended metadata depends on it.
- `validator/src/bin/solana-test-validator.rs` enables transaction history and
  extended metadata in the local test-validator path.

The important distinction is that `solana-test-validator` defaults are
developer-convenience defaults, not realistic consensus-validator defaults. For
realistic consensus-validator experiments, transaction history should generally
be treated as off unless the test explicitly wants RPC-heavy behavior.

## Faycel Kouteib public work signals

The strongest public signal from Faycel Kouteib is code, not talks or essays.
The most relevant public work found during this pass falls into four themes.

### Transaction status service and history pipeline

- [#3026 Batch status and memo writes to DB](https://github.com/anza-xyz/agave/pull/3026)
- [#4032 Make transaction status service multi-threaded](https://github.com/anza-xyz/agave/pull/4032)
- [#4654 Clean up tests that need to shut down transaction status service](https://github.com/anza-xyz/agave/pull/4654)
- [#4861 Fix CI flakiness in TSS tests](https://github.com/anza-xyz/agave/pull/4861)
- [#4875 Revert #4032](https://github.com/anza-xyz/agave/pull/4875)
- [#5894 Order slot and transaction notification using work dependency tracking](https://github.com/anza-xyz/agave/pull/5894)
- [#7633 Draft parallelization redesign for TSS](https://github.com/anza-xyz/agave/pull/7633)

The main takeaway is that the public history is not "TSS was parallelized and
that solved it." The more accurate story is that TSS parallelization was tried,
reverted, and later followed by different correctness and ordering work.

### Ingress and networking behavior

- [#7032 Streamer unstaked throttle rate proposal](https://github.com/anza-xyz/agave/pull/7032)

This points toward interest in ingress fairness and backpressure, not just RPC
internals.

### Gossip and validator testability

- [#2367 Make `split_gossip_messages` public](https://github.com/anza-xyz/agave/pull/2367)
- [#7664 Gossip service exit unit test fix](https://github.com/anza-xyz/agave/pull/7664)

### Cross-client and restart correctness

- [Alpenglow #736 Restart-safety race fix](https://github.com/anza-xyz/alpenglow/pull/736)
- [Firedancer #3494 `wait_for_supermajority` slot 0 semantics](https://github.com/firedancer-io/firedancer/pull/3494)

Overall, the public footprint suggests a focus on validator correctness under
load, transaction-status and notification semantics, ingress behavior, and
restart or cross-client correctness.

## Local experiment results

### Direct TSS load tests

A set of direct transaction-status-service harnesses was used first because the
historical TSS work made that a plausible bottleneck candidate.

Observed outcome:

- pure direct TSS load at roughly 40k, 60k, and 80k tx/s equivalent did not
  reproduce runaway backlog on current `master`
- an artificially slow notifier could create drift, but that was rejected as
  unrealistic
- more plausible heavy TSS profiles still did not reproduce the old failure
  shape in isolation

The conclusion was that TSS-only isolation does not currently reproduce the
historical issue well enough to claim it is the dominant bottleneck in present
Agave under realistic consensus-validator conditions.

### Single-validator realistic harness

A more realistic single-validator harness was then run with:

- transaction history off
- no intentionally heavy public RPC usage
- sustained TPU or QUIC load

Observed result:

```text
achieved_tps=14192
funding_elapsed_secs=40.124
bench_elapsed_secs=70.416
max_slot_gap_secs=1
stalled_seconds=0
ledger_bytes=215833987
```

The validator remained healthy. No obvious stall or widening slot lag was seen.

### Two-node LocalCluster conflict-heavy harness

A second pass used a two-validator local cluster with equal stake, history off,
no deliberately heavy RPC load, and conflict-heavy traffic.

Observed result:

```text
achieved_tps=19450
duration=45s
funding_elapsed_secs=6.162
bench_elapsed_secs=45.187
max_slot_spread=1
max_slot_gap_secs=1
stalled_seconds=0
```

The cluster remained healthy. The observer-side transaction count appeared to
undercount relative to the load generator, so the more trustworthy signals were
bench TPS, slot spread, and lack of stalls.

### Experimental conclusion

The most honest conclusion from the local runs is:

- current Agave stayed healthy in the realistic consensus-validator setups that
  were tested
- the earlier TSS failure shape did not reproduce cleanly in present code
- validator resilience questions are still real, but the strongest stress axes
  seem broader than TSS-only pressure

## Constellation and Alpenglow design exploration

### Public materials

The main public references used for this thread were:

- [Constellation blog post](https://www.anza.xyz/blog/constellation)
- [Constellation site](https://constellation.anza.xyz/)
- [Alpenglow blog post](https://www.anza.xyz/blog/alpenglow-a-new-consensus-for-solana)
- [Internet Capital Markets roadmap](https://www.anza.xyz/blog/the-internet-capital-markets-roadmap)
- [SIMD repository](https://github.com/solana-foundation/solana-improvement-documents)
- [SIMD process](https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0001-simd-process.md)

Adjacent prior art considered:

- [Bidirectional QUIC communication channel](https://forum.solana.com/t/bidirectional-quic-communication-channel/48)
- [Add new Warning and Error fields to JSON RPC results](https://forum.solana.com/t/add-new-warning-and-error-fields-to-json-rpc-results/2015)
- [`getRecentPrioritizationFees`](https://solana.com/docs/rpc/http/getrecentprioritizationfees)

### Main proposal iterations

The initial instinct was to propose an access-layer standard covering proposer
discovery, quotes, submission, and receipts. That turned out to be too broad
and too shallow at the same time. It named interface pieces, but it did not yet
define the semantic contract strongly enough.

The better center of gravity turned out to be:

- request identity
- proposer-local service semantics
- a small number of explicit proposer obligations

That led to two stronger conclusions.

First, proposer discovery is probably worth discussing separately rather than
inside the same first Ideas post as landing-intent semantics.

Second, proposers probably should not be responsible for reporting chain
inclusion in the first version. Clients can learn inclusion from the chain
itself, which keeps the initial contract focused on proposer-local behavior.

### Current best sequencing

The current best sequence appears to be:

1. a discovery-first Ideas post
2. a separate landing-intent and service-receipt Ideas post

#### Discovery-first post

The strongest version of a discovery-first post is intentionally narrow. It
would standardize only the descriptor schema and authentication, not registry,
ranking, reputation, or distribution policy.

The reason for keeping distribution out of scope is that otherwise the
discussion quickly becomes about who controls the directory, who gets listed,
and who becomes the default ecosystem gatekeeper. That governance problem is
real, but it is different from the narrower interoperability problem of
standardizing a signed descriptor shape.

The minimal descriptor fields discussed were:

- proposer public key
- endpoint or endpoints
- API version
- genesis hash
- valid-until
- signature

#### Landing-intent and service-receipt post

The best current shape for the second post is a client contract for landing
intents and proposer-local service receipts.

The current candidate landing intent contains:

- `intent_hash`
- `transaction`
- `max_inclusion_fee`
- `max_ordering_fee`
- `expiry`
- `supersedes_intent_hash` optional
- `intent_signature`

Important semantics:

- `intent_hash` is content-bound, not an arbitrary label
- the landing intent is immutable
- the same `intent_hash` is used when the same request is sent to multiple
  proposers
- intent expiry is a proposer servicing bound, distinct from chain-level
  transaction validity
- one-to-one supersession is the simplest useful first replacement model

The current candidate service-local lifecycle contains:

- `Acknowledged`
- `Accepted`
- `Rejected`
- `ClosedExpired`
- `ClosedSuperseded`

Important semantics:

- `Acknowledged` means the proposer has parsed the request, verified enough
  structure to recognize it, and can identify the request, but has not yet made
  a strong durability-backed service commitment
- `Accepted` means the proposer has durably admitted the request into service
- `Rejected`, `ClosedExpired`, and `ClosedSuperseded` are terminal proposer-local
  states
- clients learn inclusion from the chain itself rather than from proposer
  receipts in the first version

Important obligations:

- repeated submission of the same `intent_hash` to the same proposer is
  idempotent
- after `Accepted`, the proposer must not silently drop the request
- `ClosedSuperseded` only applies when a new request explicitly names the older
  request through `supersedes_intent_hash`
- the simplest v1 authorization model is to bind the landing intent envelope to
  the transaction fee payer rather than introducing a separate
  `intent_authority`

### Design conclusions

The main conclusions from the Constellation design exploration were:

- discovery and request semantics should probably not be forced into the same
  first Ideas post
- quote APIs are useful, but they are probably a follow-on rather than a good
  v1 center
- proposer-local service semantics are the strongest first semantic target
- a weak `Acknowledged` state and a durable `Accepted` state are much cleaner
  than overloading one admission state
- proposer-reported inclusion is probably not worth including in the first
  version if it is not cryptographic

## Practical next steps

The most promising follow-up items appear to be:

1. draft a discovery-first Ideas post focused only on signed proposer
   descriptors
2. draft a separate Ideas post for landing intents plus service-local receipts
3. continue studying validator resilience through TPU, scheduler, gossip, TVU,
   and restart failure domains rather than focusing only on TSS
4. treat realistic consensus-validator versus RPC-hybrid behavior as a
   recurring distinction in future experiments and design work
