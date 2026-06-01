---
layout: post
title: "Governing a codebase as a commons"
date: 2026-06-01
---

*What two Nobel economists, Elinor Ostrom and Oliver Williamson, get right about keeping a codebase coherent when AI agents write a lot of it.*

A while ago I realized I had been rebuilding, by accident, something Elinor Ostrom won a Nobel Prize for studying.

I build software, and most of it now gets written with AI coding agents. Across three different codebases I noticed I had grown the same extra layer each time: a file called `CONSTITUTION.md`, a folder of numbered decision records, a checklist the agents have to follow, an audit playbook, and a CI check that gets stricter over time and never loosens. None of it was planned. Each part was a reaction to some specific way a codebase had gone bad on me. But when I put the three side by side, they have the same shape, and that shape has a name. It is a way of governing a shared resource.

This post is about what happened when I went back and read the two economists who actually understand this: Elinor Ostrom and Oliver Williamson, who shared the 2009 Nobel. I checked their ideas against the thing I had built. Some of it fit very well. Some of it broke in useful ways. And one part of my setup turned out to be a clean example of an idea Williamson worked on his whole career, which is the part I like most, so I put it in the middle.

## The accidental institution

The problem that started all of this is common and boring. A codebase grows, every single change looks reasonable on its own, and the whole thing drifts anyway. Conventions split. The same idea gets three different names in three files. An agent told to fix a bug does the smallest thing that closes the ticket and leaves the code slightly worse than it found it. Do that a thousand times. One of my repos opens its constitution by naming the result, "systemic brittleness," and then says the real point plainly: the problem was never any single bug, it was that there was no constitution. Nothing said what good meant here, so there was nothing to check a change against.

So I wrote one. Then the other two followed, because once you have the first part the rest start to feel necessary:

- A constitution. Ten or twelve principles, each with a reason, examples of what breaks it, and a test that tells you when it has been violated. Versioned. You can only change it through a written process.
- A decision log. Append-only records of the decisions we made (ADRs). You never delete a decision, you mark it as replaced and leave the old one in place. The point is that nobody, person or agent, has to reargue a settled question later, because the reasoning is still there.
- An agent protocol. A short before, during, and after checklist that applies to whoever is doing the work.
- An audit playbook. How to look for the problems a machine cannot catch, with a few set responses: fix it, write down an exception, change the rule, accept it as tracked debt, or rewrite the thing.
- Enforcement in steps. A warning when you commit, a hard failure in CI.

In one of the three this went past documents. There is a small compiled `governance` program that scans the code on every pull request. It only catches the problems a regex can catch, and the constitution is honest that this is "a floor, not a ceiling." It reads its own list of exceptions: a line of code that points at a decision record (`// Per DEC-007, ...`) gets downgraded from a failure to a note. And it has one feature I will defend to anyone: a ratchet. Once a kind of violation has been cleaned up to zero, the check turns into a hard block, for good. It looks about like this:

```
# while a violation class still has open cases, the check only warns
no_skipped_tests: warn    # 12 existing, tracked as debt

# once you have driven it to zero, you flip it, for good
no_skipped_tests: block   # 0 left, and now it cannot come back
```

That ratchet is where this stops being documentation and starts being governance. Keep it in mind, I come back to it.

## Two economists who already solved this

Ostrom and Williamson shared the 2009 economics Nobel for their work on governance, Ostrom mostly on the commons and Williamson mostly on the boundaries of the firm. They worked on problems that look opposite and were, underneath, the same question.

Ostrom studied how real communities, like inshore fishers, valley irrigators, and alpine herders, manage a shared resource for centuries without either privatizing it or handing it to a central government. The standard theory said they could not. The "tragedy of the commons" said a shared resource gets destroyed by self-interest unless someone owns it or polices it. Ostrom went and looked, and found thousands of cases where ordinary people ran it themselves and did fine. The ones that lasted share a set of design rules: clear limits on who is in, rules that fit local conditions, the people affected get to set and change the rules, monitoring, sanctions that start small and grow, cheap and fast ways to settle disputes, a recognized right to organize, and, for big systems, smaller groups nested inside larger ones.

Williamson asked why firms exist at all. If markets work so well, why is so much of economic life run inside companies by management instead of bought on the open market? His answer is transaction costs. When an asset is specific, worth a lot in one relationship but hard to reuse elsewhere, the two sides get locked into each other, and a plain market contract cannot protect against the cheating that lock-in invites. You need governance, meaning ongoing management of the relationship after the fact, not just a price.

Underneath, both of them study one thing: how a group of self-interested people with limited information write and enforce the rules that keep something valuable from going bad. That is more or less my problem too.

## A codebase is not a fishery

The idea that reorganized my thinking starts with an objection that almost kills the whole comparison.

Ostrom's commons can be used up. The fish I catch, you cannot catch. The water I take does not reach your field. That is the whole reason a commons can be overused and needs governing. Code is the opposite. You can copy it as many times as you want and take nothing from anyone. By Ostrom's own definition, a codebase is not a common resource at all, even though that is exactly what my constitutions say they govern.

So the real question is this: in a software project, what is actually scarce and can be used up? Ask it that way and the answer comes fast, and it changes everything.

- Maintainer and reviewer time. Always limited. Every review costs someone an afternoon. This is the clearest real commons in the building.
- The coherence of the design. Every change spends a little of it, and it does not come back on its own. Erosion is not a metaphor here, it is real.
- Trust, in the numbers, in the build, in the team. One bad change can drain it for everyone. (Ask anyone who lived through the xz backdoor.)

The code is free to copy. The coherence of the code, the time it takes to keep it coherent, and the trust that it does what it says, those can run out. The constitution was never protecting the source code. It was protecting the maintainers' time and the system's integrity. That one change in framing rewired how I think these documents should be written: aim them at the merge gate and the reviewer's time, not at "the code."

## Ostrom's principles, mapped, and where they break

With the resource named correctly, the fit is strong in some places and broken in others. The short version:

| Ostrom's principle | The thing in my repo | Verdict |
|---|---|---|
| Clear limits on who can take | who is allowed to merge (CODEOWNERS, commit rights) | Holds well |
| Monitoring | code review plus the CI scanner | Stronger than in nature |
| Sanctions that grow | warn, then block in CI, then revoke access | Almost exact |
| The affected make the rules | the versioned amendment process | Holds |
| Cheap dispute resolution | PR threads, written exceptions | Holds |
| Nested units | module, repo, portfolio | Holds, underused |
| Give back in proportion to taking | "you touch it, you maintain it" | Breaks (free riding) |

Two things stand out.

First, the parts of my setup I was most proud of, the scanner, the ratchet, the audit modes, are just Ostrom's monitoring and graduated sanctions, and they work better in software than in a fishery. Monitoring is the expensive, boring part of governing a real commons. Someone has to go out and count the nets. In a codebase, git and CI make monitoring constant and nearly free. The hard part for a fishing cooperative is the easy part for my laptop.

Second, the part I had filed as "just documentation," the decision log, maps to something Ostrom names directly as a reason a commons survives: the group's history of past dealings. The log is the project's memory, and memory carries weight. It is what lets a newcomer, person or model, inherit a decision instead of reopening it.

The clean break is the rule that people who take from the commons should give back in proportion. In open source this is the free-rider problem, and it is rough. For a small trusted team, or a team of agents I direct, it mostly does not apply, because there is no anonymous crowd dropping in a change and leaving. Worth knowing that limit before someone bolts this onto a large public project and expects it to hold.

## The Williamson part, which is the one I like

Williamson's deepest idea is the credible commitment. A promise is worth nothing if you can break it later at no cost, so the way you make cooperation stick is to take away your own ability to break it. You leave a hostage. You burn the boat behind you. The promise becomes believable because going back on it is no longer something you can do.

The ratchet does this to me. When I let a check turn into a permanent block, I am taking away my own future option to be lazy. Future me, at 2am, wanting to merge a quick hack that sneaks back a banned pattern, gets stopped. Not by willpower, which I do not have at 2am, but by a wall I built on purpose while I was being sensible. I turned "I promise not to backslide" into "backsliding does not build." That is Williamson's move exactly, and once you can see it you start looking for other places to leave a hostage against your own worst habits.

One fair caveat. Most of what Williamson studied was two sides fighting over the value of a locked-in relationship, a buyer and a supplier, each tempted to squeeze the other. Inside one team there is no other side taking money from me. The fight is between me today and me in six months. That makes a codebase more of an Ostrom problem, a shared resource looked after over time, than a Williamson standoff. But the ratchet itself is pure Williamson, and it is the most original part of my setup.

## A commons with robots in it

What makes this current, and not just a neat comparison, is that more and more of the contributors are AI agents. An agent is, in the plainest economic sense, a limited-information actor whose goal does not match yours. Its context window is finite, which is bounded rationality, the same thing Simon and Williamson meant. Its goal is to close the task, which is not my goal of keeping the code coherent. It will, by its nature and with no bad intent, take the move that is best right here and slightly worse overall. That is not a flaw in the model. It is the shape of the situation.

Which is freeing, because it tells you to stop governing agents by hoping they are wise. You govern an agent the same way Ostrom's villages govern a fishery and Williamson's firms govern a supplier: with limits, monitoring, sanctions that grow, written precedent, and commitments that do not depend on anyone's good intentions. The agent reads the constitution for context. The CI gate enforces what context cannot. Advice shapes behavior, and hard gates stop it.

That split, advice plus hard enforcement, is one I reached by trial and error, and it turns out to be where the whole industry is heading. GitHub's spec-kit ships a per-repo constitution.md with enforcement gates. The AGENTS.md convention now spreading across coding tools is the same idea. Claude Code describes its own design the same way: rules are advice, hooks and permissions are enforcement. I find it reassuring that something I built for people turns out to be close to the right setup for a mixed team of people and agents, with almost no change. The economics did not care which kind of contributor showed up.

## The honest ledger

None of this is free, and a post that pretends otherwise is not worth reading.

Every governing document you make an agent read is tokens it pays on every task, for a gain that is real but modest. The public work on agent instruction files points the same way: a small improvement that can turn negative when the context is machine-generated or simply too long. Context rot is part of why. Chroma's 2025 report, "Context Rot," showed that models get worse as you fill the window, well before it is actually full. A 40KB constitution read on every turn is not care, it is a mistake. So the discipline needs its own discipline: keep the part the agent reads small. Full text for the people, a short version loaded only when the model needs it.

The deeper point is the rule that makes the whole thing work. An out-of-date governance document is worse than none, because it is wrong with authority. So the one rule you cannot break is that keeping the documents true is part of the job. Letting them drift counts as a violation. Governance you do not maintain is just theater with overhead.

## "Isn't this just ADRs and a linter with extra reading?"

Mostly, yes. The parts are old. Decision records go back to Michael Nygard in 2011. Policy-as-code and lint gates are everywhere. A repo "constitution" already exists, GitHub's spec-kit ships one. I did not invent any single organ.

What I think is worth your attention is three things the usual stack does not put together. The decision log is treated as memory the agent reads, not as docs a human files and forgets. The enforcement ratchets, so cleanups are permanent instead of slowly undone. And the whole thing is aimed at a contributor that is partly machine, which changes what you can lean on (advice) and what you have to enforce (gates). The economics is the reason the combination holds, not the novelty of any one piece.

## What to steal, and the question I am chasing next

If you want to try a piece of this without taking on the whole process, start with the two cheapest, highest-value parts:

1. A decision log. Append-only. Every non-obvious choice gets three sentences: what, why, and what you turned down. This is the part most people are worst at, and it pays for itself the first time an agent or a new teammate tries to reopen something you settled in March.
2. One ratchet. Pick one thing your codebase has finally gotten right, say zero of some lint class, no banned import, no stray `console.log`, and write a CI check that blocks its return for good. Leave one hostage and see how it feels.

Everything else, the constitution, the audit modes, the scanner, is extra you can grow into if the first two earn their place.

That leaves the open question, and it is the one I am working on now. Does the full setup actually make the agents better, measured on the things it is meant to protect, like sticking to conventions, avoiding regressions, and keeping the design coherent, and not just on whether tickets close? Most of the existing studies measure the wrong thing, raw issue resolution, and find a small, expensive gain. I happen to have three codebases carrying this machinery and a habit of measuring things. The next post is the test: the same tasks, run with and without the governance, scored on the things the governance is supposed to defend. If it comes back "this is theater," I would rather be the one who found that out.

---

### Further reading

- Elinor Ostrom, *Governing the Commons* (Cambridge, 1990), and her 2009 Nobel lecture, "Beyond Markets and States: Polycentric Governance of Complex Economic Systems."
- Cox, Arnold, and Villamayor-Tomás, "A Review of Design Principles for Community-based Natural Resource Management," *Ecology and Society* 15(4), 2010. The revised, split version of the design principles.
- Oliver Williamson, *The Economic Institutions of Capitalism* (Free Press, 1985); "The Theory of the Firm as Governance Structure: From Choice to Contract," *Journal of Economic Perspectives* 16(3), 2002; and his 2009 Nobel lecture.
- Frischmann, Madison, and Strandburg, *Governing Knowledge Commons* (Oxford, 2014). Ostrom's framework adapted for non-rival, knowledge-like resources. The right tool if you want to take "a codebase is a commons" seriously instead of as a metaphor.
- Kelly Hong, Anton Troynikov, Jeff Huber, "Context Rot: How Increasing Input Tokens Impacts LLM Performance," Chroma, 2025.
- Michael Nygard, "Documenting Architecture Decisions" (2011). The original ADR.
- GitHub spec-kit, and the AGENTS.md convention. The closest things in the wild to a per-repo constitution written for AI agents.

*Part of an occasional series. Next post: do the rules actually work, or do they just feel good?*
