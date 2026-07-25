---
layout: post
title: "An AI wrote its own TLA+ invariant and caught a real, unfixed etcd bug"
date: 2026-06-01
---

I gave a tool the execution traces of a small etcd program and no properties to check. It wrote a TLA+ specification, invented its own safety invariant, ran a model checker, and produced a counterexample. That counterexample matches an open, unfixed etcd issue, filed in April 2026, after the training cutoff of the model it used, Claude Sonnet 4.5.

It's less magical than that sounds. I'll walk through what happened, and the parts it doesn't prove.

## Why I built this

More of our distributed-systems code is getting written by AI agents now. The bugs that matter in that code are races, lost updates, stale reads, consensus edge cases. Those are the ones unit tests and type systems miss. Formal methods catch them, but almost nobody writes a TLA+ spec for their service, and fewer keep one current.

The question I keep coming back to is whether the gate could write the spec for you, from what the code actually does, and just tell you what breaks. AI writing the check on the code, not the code itself. The tool is called attest. This is the first finding from it I think is worth showing.

## The bug

[etcd-io/etcd#21638](https://github.com/etcd-io/etcd/issues/21638), filed April 2026 against v3.6.10, still open:

> clientv3: LeaseKeepAlive channel may yield buffered pre-revoke success after LeaseRevoke returns

The reproduction is five steps. Grant a lease, attach a key, call `KeepAlive` until the channel has buffered a response, call `Revoke`, then read the channel. That read can hand you a keepalive response with `TTL > 0` and a revision strictly older than the revoke's.

Post-cutoff is the point. The issue was filed after the model's training cutoff and is still unfixed in master, so the model couldn't have memorized a patch.

I wrote a plain reproducer against an embedded etcd that emits every observable event as JSONL. No issue numbers in the code, no comments calling anything a bug, event names that describe activity rather than judge it. One run:

```
{"event":"grant_returned","revision":1, ...}
{"event":"put_returned","revision":2, ...}
{"event":"keepalive_returned", ...}
{"event":"channel_observed","channel_len":1, ...}
{"event":"revoke_called", ...}
{"event":"revoke_returned","revision":3, ...}
{"event":"channel_yielded","revision":2,"ttl_sec":3, ...}
{"event":"end"}
```

Look at the last two lines. The revoke returns at `revision=3`, then the channel yields a response stamped `revision=2`. Four runs, all deterministic.

## What the tool did with `properties: []`

I ran attest on the code and the four traces with no properties supplied, just an empty list. It had to work out what "correct" even means here.

It generated a TLA+ spec for the lease lifecycle, with state for the lease, the channel, the revision counter, and the revision carried by the buffered response. Then it did the part I care about. It wrote a safety property of its own:

```tla
ChannelYieldConsistency ==
    [](channel_state = 2 /\ lease_state = 2 => buffered_revision >= revision)
```

That says: once the channel has yielded and the lease is revoked, the revision it handed you can't be older than the current one. TLC broke it in 55 states with a six-step counterexample, walking the same grant, put, keepalive, revoke, yield path the trace shows. One turn, about thirty cents.

attest doesn't stop at "violated." Its explanation phase writes a root-cause document: it walks the counterexample, points at the lines doing the receive, and rates severity, then suggests calling `kaCancel()` before `Revoke()` as a workaround. So what you get out of a run is a TLA+ spec, a counterexample the model checker confirmed, and a writeup you could send to a maintainer.

## "But did you lead the witness?"

Fair question, and the first one I'd ask. There are two ways this could be cheating. attest's own prompts tell the model to find bugs, and the model has obviously seen etcd's client in training. So I built the most isolated version I could.

- Ran it in a bare `/tmp` directory with no link to attest's code or docs.
- Renamed the trace events to opaque labels: `op_a_returned`, `op_b_returned`, on through `op_c_received`.
- Used no attest at all. Just `claude --print` with a neutral prompt asking what invariants a system like this should hold, and whether the traces violate any. No mention of bugs, TLA+, or anomalies, and no property suggested.

It still proposed four invariants on its own. One of them:

> Causality: Post-revoke operations cannot observe pre-revoke lease states

It then flagged the exact violation, `op_c_received: ttl=3, revision=2` arriving after `op_d_returned: revision=3`, and tied it to split-brain in leader election and distributed locks. Nine turns, nineteen cents.

## What this proves, and what it doesn't

What it shows:

- attest surfaces a real anomaly in production-library code, as a formal counterexample, on a bug it couldn't have memorized the fix for.
- It will propose its own invariant when you give it none.
- The writeup it produces is good enough to send upstream.

What it doesn't show:

- Discovery from first principles. The model knows etcd's API, and "the revision went backwards" is a textbook causality smell.
- Anything a careful human wouldn't catch from that trace. attest mechanizes the catch. It does not out-think you.
- A track record. This is one case, not a corpus. The thing that would convince me is reproducing it on several different post-cutoff bugs in different projects.

The other direction matters too. On a gossip counter I wrote to be correct, attest explored 485,401 states against six invariants it had proposed, and reported no violation. That is the right answer. A bug-finder that cries wolf is useless, and this one stayed quiet when the code was fine.

## Why a spec, and not just a flag

A model saying "this looks racy" isn't worth much on its own. A spec and a model-checked counterexample are, because the artifact doesn't depend on who wrote the code. It checks the behaviour you observed whether a human or an LLM produced it, it reproduces, and someone who doesn't trust the model can read it and rerun it for themselves. That independence is the point of a gate.

## How it works

attest is written in Elixir. It takes the source and the execution traces, has a frontier model (Claude Sonnet 4.5) propose a TLA+ spec grounded in those traces (they are treated as ground truth, so the spec has to admit every run you observed), checks it with TLC, and then either reports no violation or explains the counterexample. The model only proposes. The verdict comes from TLC.

It's early, and not open source yet. If you run distributed systems, especially if AI is writing more of that code than it used to, and you want to try it on your own traces or just compare notes, I'm at brserdem@proton.me.
