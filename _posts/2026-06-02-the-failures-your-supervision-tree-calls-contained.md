---
layout: post
title: "The failures your supervision tree calls \"contained\""
date: 2026-06-02
---

*I built a static analyzer for OTP supervision trees and ran it over thirty-some well-known open-source Elixir projects. It found real cross-tree coupling in Livebook, TeslaMate, Teiserver, and Electric: the kind a restart turns into an error somewhere that looks unrelated.*

An OTP supervision tree is a promise about failure. It says that when this process crashes, I restart it here, and the damage stops at this branch. The promise is only as good as the boundary, and the boundary is only real if nothing reaches across it.

Things reach across it all the time.

Your supervision tree describes how your app is *structured*: what restarts what, in what order. Your code describes how your processes *actually talk*: who calls whom. Those are two different graphs, and the failures that wake you up live in the gap between them. A process in one branch synchronously depends on a process in another. The tree calls that other process's restart "contained." For the caller blocked on a `GenServer.call` to it, it isn't. It gets `:noproc` or a timeout, and it surfaces somewhere that looks unrelated to the thing that actually restarted.

I wanted to know how often that gap is real in code people actually run, so I built a tool that reads both graphs from source and reports where they cross. It's called **firebreak**. I pointed it at thirty-some open-source Elixir apps: Livebook, Electric, TeslaMate, Teiserver, Oban, Phoenix, Ash, Broadway, Sequin, Supavisor, and more. Here is what it found.

## What it reads

Two things, and it compares them.

The **supervision forest**: for each supervisor, the strategy, the restart intensity, and the child list. It reads them the way OTP does, by calling `init/1` (child specs are runtime data; `init/1` returns them without starting anything). For code it can't load, it falls back to parsing the source.

The **coupling graph**: every `GenServer.call`/`cast`, `:gen_server`/`:gen_statem` call, registered name, `Registry`, `:global`, `Process.whereis`, `:ets`, `Phoenix.PubSub`, and `:pg`, resolved to the module that owns the process on the other end. It follows wrapper functions through a context module, so a dependency routed through a public API isn't invisible.

The findings that matter are the edges of the second graph that cross the boundaries of the first, weighted by whether they're *synchronous*. A `GenServer.call` into another branch is the dangerous one, because the caller blocks on it. A `cast` doesn't, so it rates lower. No app boot, no LLM, no instrumentation, which is what lets it sit in CI.

## What it found

**Livebook.** Three LiveViews, `HomeLive`, `OpenLive`, and `SessionLive`, call `NotebookManager` synchronously, inside `mount/3`:

```elixir
# livebook_web/live/home_live.ex:18
def mount(_params, _session, socket) do
  starred_notebooks = Livebook.NotebookManager.starred_notebooks()
```

```elixir
# livebook/notebook_manager.ex:40
def starred_notebooks() do
  GenServer.call(__MODULE__, :starred_notebooks)
end
```

`NotebookManager` is a direct child of `Livebook.Application`. The LiveViews live under the Phoenix endpoint, a different branch of the tree. The supervision tree says these are independent. They are not. If `NotebookManager` crashes and a user loads the home, open, or session page during its restart window, `mount/3` hits `:noproc` and the page fails to render. The cause, an unrelated background server restarting, is invisible from the tree alone. firebreak rates it high: synchronous, cross-tree, several callers.

**TeslaMate.** `Import.FakeApi`, a `GenServer` that replays imported data, `spawn_link`s a worker to stream chunks and never traps exits:

```elixir
# teslamate/import/fake_api.ex:121  (inside the GenServer)
spawn_link(fn ->
  s |> Stream.chunk_every(500) |> Stream.with_index() |> ...
end)
```

If that stream raises mid-import, say a malformed row or an encoding error, the linked worker dies abnormally and drags `FakeApi` down with it, aborting the user's import with no isolation.

**Livebook** has the same shape in `Runtime.K8s`, a `GenServer` declared `restart: :temporary`. `handle_continue` calls a helper that `spawn_link`s the Kubernetes pod-event watcher (`k8s.ex:151`), and the module traps exits nowhere. A network blip in the watch loop crashes the watcher and takes the runtime manager down with it. Because it's `:temporary`, it isn't restarted. A supervised `Task` or `Process.flag(:trap_exit, true)` would contain either case.

**Teiserver** (a 700-module game server). When the login throttle is enabled, every login attempt synchronously calls a single top-level `LoginThrottleServer`:

```elixir
# teiserver/account/servers/login_throttle_server.ex:62
def attempt_login(pid, userid) do
  GenServer.call(__MODULE__, {:attempt_login, pid, userid})   # no :noproc fallback
end
```

The login code runs in the per-connection protocol process, a different branch from that one server. If it restarts, every in-flight login hits an unhandled exit and the connection drops. The clincher is in the same module. The monitoring path catches exactly the failure the login path doesn't:

```elixir
# login_throttle_server.ex:52
def get_queue_length do
  GenServer.call(__MODULE__, :queue_size)
catch
  :exit, {:noproc, _call} -> 0     # the dashboard handles the server being down...
end
```

The authors clearly know the server can be absent, because they guard the dashboard query for it. The login hot path has no such guard. That asymmetry is the cross-tree dependency, sitting right there in the diff.

**Electric.** `Shapes.Consumer` calls `ShapeCache.Storage.start_link/1` inside a `handle_info` clause (`consumer.ex:265`), a process started from a message handler and re-spawned on every matching message. firebreak flags it. So did the authors: there's a `# TODO: Remove. Only needed for InMemoryStorage` on the line above it. The tool found, mechanically, a line a human had already marked for cleanup.

## "But these are apps people trust"

Right, and that's the first thing I'd be suspicious of. A tool that flags four well-regarded projects is either onto something or crying wolf, and the way you tell them apart is by what it does *not* flag.

It stays quiet on the libraries. Phoenix, Broadway, Commanded, Bandit, Finch, and `phoenix_live_view` came back clean, which is the right answer. Their processes are started by *your* app, not by them, so there's no cross-tree coupling to find. changelog.com, 326 modules of Ecto and LiveView, has zero inter-process coupling, and firebreak reports nothing. A clean app should be silent.

It's also careful about severity. It flags the classic lookup-or-create registry race (read the registry, start a process if it's missing, in the same function) in Livebook's `Deployer`, in Logflare, and in Ash. It rates every one of them `:info`, the lowest tier. That's deliberate. The soft-failure path, `{:error, :already_started}`, exists; a careful author just handles it, and all three do. Ash's even documents it as "Idempotent." The check points your eye at the spot, and the severity says "fine if you handle the return." They did. A tool that screamed BUG there would be wrong.

The corpus also made it better. Running it on real code surfaced three false-positive modes I had to fix. A port that's `Port.monitor`-ed is in fact handled, through `{:DOWN, ...}`. A `whereis` then `unregister` then `register` swap isn't a create-race. A `start_link` in `handle_continue` runs once per start, like `init`, not once per message. Analyzing FLAME turned up something worse: it builds a child spec with map-update syntax, which hit an AST shape the parser didn't model and crashed the whole run. That is not acceptable for something meant to sit in CI, so now an unmodelled shape degrades a single module instead of taking the analysis down with it.

## From a finding to a proof

A finding is a claim, and I [wrote last time]({% post_url 2026-06-01-an-ai-wrote-its-own-tla-invariant-and-caught-a-real-etcd-bug %}) about turning claims into checks a model checker will confirm. firebreak does a version of this from the static side. It can project the supervision tree into a TLA+ lifecycle spec, one per supervisor, as a pure function of the graph (`mix firebreak.spec`). Run the Livebook spec through TLC and it produces a counterexample that composes two findings the report otherwise lists apart: three crashes fit the supervisor's restart budget, the fourth blows it and the supervisor escalates, and only then is the cross-tree caller left permanently on `:noproc`. The restart-intensity warning and the coupling warning turn out to be one chain of events, and the model checker finds the exact order.

That's a proof about the *model*: the failure is reachable under OTP's restart semantics, and here is a witness trace. It is not a claim your production will hit it. But it does move the finding from "this looks dangerous" to "here is the sequence that triggers it, given your declared config."

## What this proves, and what it doesn't

What it shows: the gap between the declared tree and the real coupling graph is not hypothetical. It exists in code that ships, it concentrates in applications (where processes talk to each other) rather than in libraries, and a static pass with no app boot can find it and point at the file and line.

What it doesn't: this is best-effort static analysis, not a type system. It surfaces hazards, not certainties. Metaprogramming and runtime-computed names can still hide an edge, and a flagged crossing only bites if that process actually restarts under load, which the tool can't know. It mechanizes a review a careful OTP engineer could do by hand; it does not out-think one. And "high-confidence findings cluster in apps, libraries stay quiet" is a pattern across thirty-some projects, not a law. What would convince me is more people running it on trees I've never seen.

## Try it

firebreak is on Hex and [open source](https://github.com/b-erdem/firebreak). Add it as a dev dependency and point it at your tree:

```sh
mix firebreak                 # tiered report, coupling findings first
mix firebreak --format github # PR annotations in CI
mix firebreak --observe my_app@127.0.0.1   # fold in a live node's real shape
```

The Livebook and TeslaMate findings above each took one static run to surface. If you run a real Elixir system, point it at your own tree and see what crosses it. If it finds something useful, or cries wolf, I'd like to hear which. I'm at **brserdem@proton.me**.
