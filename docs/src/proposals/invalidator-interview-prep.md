---
title: Invalidator Interview Preparation
---

## Goal

This note is a compact interview-prep document for discussions about validator
resilience, current Agave behavior, and likely Invalidator-style work.

The goal is not to sound broad. The goal is to sound grounded:

- separate consensus-validator issues from RPC-heavy issues
- reason from current code and recent public work
- focus on failure domains that affect correct forward progress

## The best framing

The strongest public framing for Invalidator comes from Anza's blog post
["Strengthening Solana: Meet the Invalidator Team"](https://www.anza.xyz/blog/strengthening-solana-the-invalidator-team).
The important point is that Invalidator is framed as resilience engineering,
not offensive research for its own sake.

The most useful framing for the interview is:

Solana resilience should be analyzed as a set of validator failure domains.
What matters is whether the node keeps making correct forward progress under
adversarial or pathological conditions such as bursty traffic, hot-account
contention, packet loss, malformed input, bad leaders, restart events, and
control-plane degradation.

## Two-minute answer

If asked what I would focus on first, the cleanest answer is:

I would separate consensus-validator behavior from RPC-heavy or hybrid-node
behavior first. Then I would look at the validator failure domains that affect
correct forward progress: TPU ingress and scheduler behavior under bursty load
and hot-account contention, TVU and replay correctness under lag or malformed
data, gossip and control-plane degradation, blockstore and restart safety, and
vote processing. I would validate hypotheses against current Agave and local
experiments instead of assuming old incidents still describe present behavior.

## What I know about current Agave

### Transaction status service

Current Agave `master` is back to a single-threaded transaction status service
in `rpc/src/transaction_status_service.rs`. That matters because public history
around TSS includes a multithreaded redesign, but that redesign was reverted.

The practical point is that current behavior has to be anchored to current
code, not to older design attempts.

### Transaction history defaults

The production-relevant default is still to keep transaction history off unless
it is explicitly enabled.

Relevant code paths:

- `rpc/src/rpc.rs`
- `validator/src/commands/run/args/json_rpc_config.rs`
- `validator/src/bin/solana-test-validator.rs`

The key distinction is that `solana-test-validator` enables history and
extended metadata as developer conveniences. That is not a realistic default
for a consensus validator.

This matters because many performance intuitions become wrong when people mix:

- consensus validator behavior
- RPC-heavy behavior
- local developer-node behavior

## What the local experiments actually showed

### Direct TSS pressure did not reproduce the old story cleanly

Direct transaction-status-service harnesses at high equivalent rates did not
reproduce a convincing runaway-backlog failure on current `master`.

Artificially slowing the notifier could create drift, but that is not a good
proxy for realistic validator behavior.

The conclusion is not that TSS never matters. The conclusion is that the old
"TSS is the bottleneck" story does not reproduce cleanly enough on current code
to use as a default explanation.

### Realistic consensus-validator setups stayed healthy

In a single-validator run with transaction history off, no deliberately heavy
RPC load, and sustained TPU or QUIC pressure, the validator stayed healthy:

```text
achieved_tps=14192
funding_elapsed_secs=40.124
bench_elapsed_secs=70.416
max_slot_gap_secs=1
stalled_seconds=0
ledger_bytes=215833987
```

In a two-validator LocalCluster run with equal stake, history off, no
deliberately heavy RPC load, and conflict-heavy traffic, the cluster also
stayed healthy:

```text
achieved_tps=19450
duration=45s
funding_elapsed_secs=6.162
bench_elapsed_secs=45.187
max_slot_spread=1
max_slot_gap_secs=1
stalled_seconds=0
```

The honest conclusion is:

- current Agave stayed healthy in the realistic setups tested so far
- the strongest resilience questions seem broader than TSS-only pressure
- TPU ingress, scheduler behavior, conflicts, and slot-boundary effects are
  more promising next stress axes

## The failure domains worth discussing

### TPU ingress and scheduler behavior

This is the strongest place to start. It touches:

- packet admission and forwarding pressure
- sigverify path behavior
- leader-window constraints
- scheduler policy
- account locking and conflict handling
- whether non-conflicting traffic gets starved by hotspots

This is also the best candidate for the next round of targeted stress tests.

### TVU and replay

This is where correctness under lag, malformed input, duplicate or conflicting
data, and replay timing matters. If TPU is about landing and throughput under
pressure, TVU is about staying correct while keeping up.

### Gossip and control plane

Gossip and vote-processing issues are high leverage because they can degrade a
validator without looking like a classic "throughput" problem.

### Blockstore, snapshots, and restart

Restart safety and persistence bugs are some of the most operationally painful
failure modes. They are also a place where cross-client differences can matter.

### Vote processing

Vote path problems can convert a healthy-looking node into one that fails to
participate correctly in consensus. This is less flashy than ingress pressure
but just as important.

## What to say about Faycel Kouteib

The strongest public signal is code. The public work that stood out most was:

- TSS and history pipeline work
- notification ordering and correctness
- ingress and throttling work
- gossip testability
- restart and cross-client correctness

The useful interpretation is not "he only worked on RPC." The better
interpretation is that his public work shows interest in the places where
validator correctness, observability, ingress pressure, and restart behavior
intersect.

Useful examples:

- [#3026 Batch status and memo writes to DB](https://github.com/anza-xyz/agave/pull/3026)
- [#4032 Make transaction status service multi-threaded](https://github.com/anza-xyz/agave/pull/4032)
- [#4875 Revert #4032](https://github.com/anza-xyz/agave/pull/4875)
- [#5894 Order slot and transaction notification using work dependency tracking](https://github.com/anza-xyz/agave/pull/5894)
- [#7032 Streamer unstaked throttle rate proposal](https://github.com/anza-xyz/agave/pull/7032)
- [Alpenglow #736 Restart-safety race fix](https://github.com/anza-xyz/alpenglow/pull/736)
- [Firedancer #3494 `wait_for_supermajority` slot 0 semantics](https://github.com/firedancer-io/firedancer/pull/3494)

## Good questions I should be ready for

- If you joined Invalidator, what would you test first?
- How would you separate a consensus-validator issue from an RPC-node issue?
- What did you learn from current Agave rather than from historical incidents?
- Why is transaction history a bad default assumption when reasoning about
  validator behavior?
- What did your local experiments show, and what did they fail to show?
- Where do you think the most interesting resilience questions still are?

## Crisp answers I should be ready to give

### What would you test first

I would start with TPU ingress and scheduler behavior under bursty traffic and
hot-account contention, because that is where landing, fairness, slot pressure,
and lock conflicts all interact. It is also one of the most realistic ways to
stress correct forward progress without hiding the result behind RPC-heavy
noise.

### What did you learn from the code

Current Agave is not the same as the older public stories around TSS. TSS is
single-threaded again, and the old multithreaded redesign was reverted. Also,
production-relevant validators keep transaction history off by default, while
developer test-validator defaults are much heavier. That changes how I think
about likely bottlenecks.

### What did your experiments show

They showed that the easy story is probably wrong. I did not reproduce a clean
TSS-only failure on current `master`, and realistic single-node and two-node
setups stayed healthy. That pushes me toward ingress, scheduler, conflict, and
slot-boundary behavior as the next stress axis.

## Best next experiment

The next useful experiment is not more passive reading. It is a targeted stress
test around TPU ingress and scheduler behavior.

The clean first sequence is:

1. establish a steady conflict-light baseline
2. inject short bursts against a single hot writable account
3. move to a small hot set instead of only one hot account
4. concentrate bursts near slot transitions or leader changes
5. repeat in a small LocalCluster

The key metrics are:

- slot progress and stalled seconds
- max slot gap or spread
- submitted versus landed work
- whether non-conflicting background traffic is starved
- latency percentiles if capture is practical

## Public references worth revisiting

- [Strengthening Solana: Meet the Invalidator Team](https://www.anza.xyz/blog/strengthening-solana-the-invalidator-team)
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
