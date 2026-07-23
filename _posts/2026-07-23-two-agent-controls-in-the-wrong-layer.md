---
layout: post
title: "The two agent controls that live in the wrong layer"
date: 2026-07-23
---

*Two controls every production agent needs, a hard per-run spend cap and a human gate in front of irreversible actions, keep getting bolted on at the LLM gateway or the observability dashboard. Both are the wrong layer. Here is why they belong on the job, with the pattern in [Belay](https://github.com/b-erdem/belay).*

Autonomous agents fail in production two ways that a normal background job never does. They take an action you cannot take back, like sending the email or running the migration, without a person seeing it first. And they loop, calling a paid API over and over until someone notices the bill.

Both failure modes are well documented now. There is a [widely shared writeup](https://credyt.ai/blog/4k-api-bill-overnight-production-agents) of a LangChain pipeline where two agents started ping-ponging requests, ran for eleven days, and cost $47,000 before anyone caught it. Threads full of "I burned $6,000 of Claude overnight with one command" are common. The [FinOps Foundation's 2026 survey](https://www.infoq.com/news/2026/07/ai-agents-billing-guardrails/) found 98% of practices now manage some form of AI spend, up from 31% two years earlier. This is no longer an edge case.

So teams reach for two controls: a hard cap on what a run can spend, and a human gate in front of irreversible actions. The problem is not that these controls are hard to want. It is that they keep getting built at the wrong layer.

## Where the spend cap usually goes, and why it is too coarse

The common advice is to put budgets at the LLM gateway. Tools like LiteLLM and Portkey sit inline and enforce spend, and they are good at it. But they enforce per API key or per team. A gateway can tell you that the `outreach` key has spent its monthly $500. It cannot tell you that *this one outreach job* is allowed to spend two dollars and no more.

That gap matters because the runaway cases are almost always a single run gone wrong: one loop, one job that re-reads its whole context on every step, one agent that keeps retrying a failing tool. A per-key budget is the wrong unit. By the time a shared key trips its limit, the bad run has already spent most of the damage, and every other job behind that key is now blocked too.

The spend cap wants to live on the run, because the thing you are trying to bound is the run.

## Where the human gate usually goes, and why it is fragile

For approvals, the honest options today are a framework primitive or a bolt-on service. [HumanLayer](https://www.humanlayer.dev/) is a whole company built on this: an API that lets an agent pause and require human approval before a risky tool call. LangGraph ships [`interrupt()`](https://docs.langchain.com/oss/python/langgraph/interrupts) for the same purpose.

`interrupt()` works, but its own docs carry a warning that tells you where the difficulty really is: when the graph resumes, the node re-executes from the top, so any charge or write before the interrupt has to be made idempotent or it happens twice. You also inherit the coordination problem. A review might take thirty seconds or three days, and while it waits, you either hold a process open and pay for compute that does nothing, or you build the machinery to tear the run down and bring it back at the right step when the approval arrives.

That machinery, durable pause, resume at the exact point, and not paying for finished work twice, is not an agent problem. It is a durable execution problem, which is the layer both controls actually belong to.

## Put both on the job

If your durable job engine already journals each step and can park a job at zero cost, the two controls stop being bolt-ons. The run's budget is a column on the job. The approval is a signal the job waits for. Here is the whole pattern in Belay, a durable job engine for Elixir on Postgres, as one worker:

```elixir
defmodule MyApp.OutreachAgent do
  use Belay.Worker, queue: :ai, max_attempts: 5

  @impl Belay.Worker
  def run(ctx) do
    # Paid steps. Each commits to the journal, so a retry, or a resume after
    # approval, replays them instead of paying for them again.
    contacts =
      Belay.step(ctx, :research, fn -> find_contacts!(ctx.job.input) end,
        cost: [usd: 0.30, tokens: 18_000])

    drafts =
      Belay.step(ctx, :draft, fn -> draft_messages!(contacts) end,
        cost: [usd: 0.10, tokens: 6_000])

    # The irreversible action waits for a person. The job parks at zero cost
    # until your review UI sends the decision.
    case Belay.await(ctx, :approval, timeout: 3 * 86_400) do
      %{"approved" => true} ->
        Belay.step(ctx, :send, fn -> send_all!(drafts) end)
        {:ok, %{"sent" => length(drafts)}}

      _ ->
        {:cancel, :not_approved}
    end
  end
end

# The whole run is bounded. Checked before each step against durable spend.
Belay.insert(MyApp.Belay,
  MyApp.OutreachAgent.new(%{"segment" => "trial-expired"}, budget: [usd: 2.00]))
```

Three things fall out of putting it here that the gateway and the dashboard cannot give you.

The budget is per run. `budget: [usd: 2.00]` bounds this job and nothing else. A loop or a bad retry inside it stops at two dollars, and the rest of the queue keeps moving.

The cap is enforced, not observed. The check reads durable spend before each step runs, so the job stops before the next paid call rather than showing up on a dashboard after the invoice. This is the piece every runaway post-mortem names as missing: they had observability, they did not have something that could halt the run before the next call.

The approval wait costs nothing. The job is a row in Postgres marked awaiting, not a process burning compute for three days. When the signal arrives it resumes, the research and drafting replay from the journal, and only the send runs. The idempotency footgun is gone because finished steps are not re-executed.

## The honest part

None of this is magic, and it is not free of tradeoffs. Belay is new, it is Postgres only, and it is Elixir, so it is not for you if you are on another stack. Step bodies are at-least-once until their result commits, so an external charge made in the crash-before-journal window can still repeat. You handle that the way you handle it anywhere, with the provider's idempotency key.

The narrower point stands regardless of the tool. If you are running agents in production, the two controls that keep you out of the incident channel, a hard per-run spend cap and a durable human gate, are properties of the run. Enforcing them at the gateway is too coarse, and watching them on a dashboard is too late. They belong in the layer that owns the run's lifecycle. If that layer is a durable job engine, you already have the place to put them.
