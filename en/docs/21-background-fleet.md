# Chapter 21: A Background Agent Fleet — Detaching from the Terminal and Daemon Supervision

> The earlier chapters gave you autonomy along two axes. Chapter 17's `/loop` is the time axis: the same session comes back to do another round after a while. Chapter 19's Dynamic Workflows is the space axis: one script fans out dozens to hundreds of subagents right now. But both still live inside a session bound to a terminal — close the terminal and they're gone. This chapter is about how Claude Code cuts that last cord: how a session detaches from the terminal and keeps running once the terminal is closed, and how, when you have several such background sessions at once, they get looked after as a fleet. What triggers it is one command, `/bg`; what manages it is one table, `claude agents`.
>
> The evidence for this chapter splits two ways, and very unevenly. The in-session substrate for detached sessions — the background-task UI, `run_in_background` inside `AgentTool` — can be read line by line in the leaked snapshot (labeled v2.1.88) source. But the daemon supervisor that actually treats multiple background processes as a fleet never leaked with the snapshot: there `grep -rlF 'tengu_bg_'` hits zero files, and it survives only as strings in the 2.1.202 binary. So this chapter keeps making one thing explicit: which parts are mechanisms readable in source, which are a feature map assembled from nearly eighty event names in the binary strings, which are logic inferred from those names, and which are the local process model — a blind spot that simply doesn't show through.

## 21.1 The third thing beyond the two axes: detaching from the terminal

Start by picking those two axes back up. `/loop` extends a session in time and a Workflow splits it in space, but both rest on one premise neither breaks: the session and the terminal are on the same cord. Your shell is alive, so the session is alive; the shell exits, the process group gets a signal, and the session goes with it. For a long task meant to run tens of minutes to hours — the kind whose launcher should long since have moved on to something else — that cord is in the way: you can't keep a terminal open forever just to keep it running.

The background fleet exists to solve exactly this, and the first move it introduces is "detach." `/bg` lifts the current session off the terminal and drops it into the background to persist; your terminal is freed at once for something else, while that session keeps running elsewhere. The line in the strings that states this most counterintuitively is "Sessions keep running if you close the terminal." — the session is no longer an appendage of the terminal; close the terminal and it lives on. This is orthogonal to the earlier two axes: `/loop` and Workflow are about "how work is arranged inside one session," while detaching is about "how the session itself survives your absence." Stack all three and you get the full meaning of "unattended" — you don't have to watch it, and you don't have to stay connected to it.

Seen from another angle, what this chapter describes is really the substrate under the earlier ones. `/loop`'s round-after-round continuation needs something keeping it from dropping; the batch of subagents a Workflow fans out, and the parallel members of a multi-agent team, all likewise rest on a layer beneath where "processes don't just vanish." The earlier chapters each covered their own orchestration — when to dispatch, how many, how to collect — but none turned that layer over to look at it closely. This chapter is about it: how a detached session, with you absent and the terminal closed, still has something watching whether it lives or dies. Subagents belong to the same background-capability landscape as detached sessions; it's just that whether they are supervised by this `/bg` daemon is a boundary the evidence can't bridge (see 21.6) — so the chapter reads less like a new feature and more like turning over the ground the earlier chapters have been standing on without looking at closely.

## 21.2 `/bg`: lifting the session off the terminal

`/bg`'s one-line self-introduction says everything it needs to. Verbatim in the strings: `/bg` detaches this session to run in the background, and then `claude agents` gathers every backgrounded session into one table, each row carrying a status color — a glance tells you which one is waiting on you; space to fire off a reply, enter to take it over. The companion onboarding hint is even blunter: `/bg` this session, then run `claude agents` in a new terminal. One move pushes the session to the background, the other fishes it back up to look at from a different terminal.

Once detached, the session doesn't go mute. It's still running and may still need you — say, to ask a question only you can rule on — you just aren't in front of it right now. So detaching isn't "fire and forget," it's "fire and switch to watching asynchronously": the session keeps advancing, flashes a status in that table when it needs you, and you go back when you're free. This also explains why the whole thing threads into the phone-remote-control note in this box's CLAUDE.md — a local `/bg` to detach, attach back from your phone, the process kept alive by the background the whole time; the same session passes among local, phone, and persistent, all on the strength of "the session isn't bound to a terminal."

## 21.3 `claude agents`: one table for all background sessions

Detaching solves "how one session stays alive"; `claude agents` solves "how to watch several at once." In the CLI it's a proper Commander subcommand, its description reading verbatim Manage background agents. It carries a small set of actions: `claude agents` lists every background session, `claude attach {id}` opens one in the current terminal to take over, `claude logs {id}` pages through its recent output. Together the three are a plain fleet panel — the list for the whole picture, attach for a single one, logs for reviewing the record after the fact.

The key to that table is the status color. It compresses "which session needs your involvement" into a single column you can scan at a glance — you don't have to attach into each one to look; a glance tells you who to deal with first. As for which color maps to which state, the strings say only that the status color is for spotting which ones need you; they don't pin down a color-to-state mapping, and neither will this text. There's also a rule buried here that bites: if a session is already running as a background agent, you can't just `--resume` it elsewhere. The strings put it plainly — it's a background agent now, open `claude agents` to attach to it, or stop it there first to resume here. The same session isn't allowed to be taken over in two places at once; attach and resume are mutually exclusive entrances. The constraint itself reveals that a background session isn't a piece of state you can copy freely, but a live process with a single owner that has to be taken over exclusively.

## 21.4 Below the surface: a process-fleet supervisor running locally

Everything up to here is still the layer the user can see. What actually holds up "close the terminal and it doesn't drop, crash and you can still come back" is a supervisor assembled from the nearly eighty event names prefixed `tengu_bg_` in the binary strings. Say it up front: this section has the most lopsided evidence in the chapter. The event names themselves are read verbatim from the 2.1.202 strings, solid; but under what conditions each event fires, and how they chain into a state machine, has not one line of matching source in the snapshot (`grep -rlF 'tengu_bg_'` hits zero files) — it can only be inferred from the names plus the surrounding minified code. So the description below of "what it's doing" has hard names and inferred logic, and I write to that measure.

Cluster these events by name and five or six subsystems surface, each minding its own patch, together looking like a small-scale process supervisor. At the top is a single entry point `bg_agent_action` and a `bg_dispatch` family — responsible for dispatching a task, first running `bg_classify` to categorize it, and along the way it can be rescued (`_dispatch_rescued`), dropped for being too old (`_stale_drop`), escalated all the way to SIGKILL (`_sigkill_escalate`), or routed down a degraded path under memory pressure (`_low_mem`/`_fallback`). Below that is the daemon's life and death: `_daemon_spawn_failed`, `_daemon_install`, restarting a wedged one (`_daemon_zombie_restart`) without mistakenly killing a false zombie (`_zombie_false_positive`), asking you first on a cold start (`_daemon_cold_start_ask`), with a dedicated path each for Windows (`_daemon_wmi_fallback`) and macOS (`_daemon_macos_aqua_wrap`).

Further down it gets interesting. There's a `spare` family — `_spare_spawn`, `_spare_claim`, `_spare_enable`, `_prewarm_per_sweep` — which, going by the names, looks like it maintains a pool of pre-warmed spare workers: rather than spinning up a process only when a task arrives, it keeps a few warm ahead of time, and an arriving task claims one directly, saving the few seconds of cold start. The rise and fall of real workers is another family: `_worker_spawn`/`_exit`/`_vanished`/`_stalled` — born, exited cleanly, vanished for no clear reason, wedged and not moving, each with its own event. The most supervisor-like is the respawn / adopt / orphan group: a crashed worker respawns automatically (`_respawn`), guarding against duplicates before it does (`_respawn_unconfirmed_bail`); an ownerless process gets adopted (`_adopt`), respawning if its token was lost in the process (`_adopt_token_lost_respawn`); an unclaimed orphan gets reaped (`_orphan_reap`), while the fleet's whole roster — called just that — has orphans registered back in (`_roster_orphan_adopted`) and can also fail to parse (`_roster_parse_failed`). The attach/detach family minds the moment you take over — the first frame (`_first_frame`), respawn-on-stall (`_stall_respawn`), a kick (`_attach_kick`); and at the very bottom is a transport layer minding pipe-level mishaps like a PTY it can't get, auth that doesn't match, a protocol mismatch, a truncated bridge.

Walk one worker through the whole thing and you feel the density of this machinery. It's born (`_worker_spawn`) and entered into that roster; running along, if it disappears for no clear reason (`_vanished`) or wedges (`_stalled`), then — going by the names `_respawn` and `_respawn_unconfirmed_bail` — the supervisor doesn't pretend not to notice but judges whether to respawn it, and before respawning confirms it's really dead and not a network blip, so the same job doesn't get two copies. When the machine is short on memory it's a different set of moves: it retires workers that hold memory and aren't so critical (`_retire_pinned_low_mem`, `_low_mem_mb`), making room for the critical ones. If the supervisor has just taken over and finds a pile of ownerless processes still running (say the previous daemon didn't exit cleanly), it adopts these orphans back (`_adopt`, `_orphan_reap`, `_roster_orphan_adopted`) rather than letting them turn into zombies no one minds. Chain these and you have what a supervisor should be: keep the books, check for life, patch a crash, retire under pressure, take in the ownerless.

Put these patches together and the conclusion is clear: what's behind `/bg` isn't "spin up a background process," which would be light. It's a whole process-fleet supervisor with health checks, crash respawn, orphan adoption, memory-pressure retirement, and a pre-warmed spare pool — roughly a subset of systemd or pm2 in scale — except the whole machine runs inside your local client, so that the detached sessions have something watching whether they live or die while you're not looking. This supervisor is the chapter's hardest bone and also the part that can only speak through strings: the names are read verbatim from the binary, but what the state machine they chain into actually looks like is inferred by me from the names and surrounding logic, not read out of any source — a measure I'll keep for you.

## 21.5 Started on demand, exits when idle, killable with one switch

You might picture such a supervisor as a resident daemon, eating memory from boot. It's not. In the strings its stance is on-demand — started only when used. With no work holding it, it says of itself nothing holding this daemon open and then idle-exits; the next time you type `claude agents` or `claude --bg`, a new one is pulled up. This is a different temperament from Chapter 17's "resident, watching a goal" autonomy: the supervisor exists only when there really are background sessions to watch, walks off on its own when there's no work, and doesn't sit on resources.

Starting on demand also brings a pitfall to guard against: what if two daemons start at the same instant. The strings have a clean yielding rule — an on-demand daemon never displaces a running one, and only a transient one may be displaced. Flipped around: a daemon already properly watching a batch of sessions has priority, and a freshly on-demand one may not shove it aside, avoiding two supervisors fighting and scrambling the fleet's books.

The whole thing also keeps a master switch. A setting reads verbatim: Disable agent view (`claude agents`, `--bg`, `/background`, and that on-demand daemon, all off together), typically pushed down in managed settings, equivalent to the environment variable `CLAUDE_CODE_DISABLE_AGENT_VIEW=1`. That is, the entire block of detaching-plus-fleet-supervision can be pressed off with one switch — for an organization that doesn't want background resident processes cropping up on employee machines, this is a path to shut it all down uniformly from above.

## 21.6 Another layer of the same landscape: subagents default to the background

There's a layer to add here that sits alongside detached sessions but rests on entirely different evidence: the backgrounding of subagents. What you `/bg` out is a whole session; inside a single session, meanwhile, the subagents the main session dispatches also run in the background by default. This picks right back up on the `run_in_background` from [Chapter 8](07-multi-agent.md) and [Chapter 19](19-dynamic-workflows.md): in the current version, subagents run in the background by default. The `AgentTool` description is readable in the snapshot source, and the strings also keep a few slightly worded variants, all meaning the same thing: Subagents run in the background by default, notifying you when done; only when you must have its result before going on do you pass `run_in_background: false` for a synchronous run. In the schema `run_in_background` defaults to true, and it carries an `isolation: 'worktree'` field governing filesystem isolation — this `run_in_background` appears in ten files in the snapshot (`AgentTool`, `BashTool`, `PowerShellTool`, a bundled batch, and more); this layer has source you can read line by line, not inference from strings.

How does a background subagent come back when done? Via `<task-notification>`. On finishing, it doesn't stuff its result back into your current utterance but re-enters the main loop as a task-notification — in the minified code it carries a `promptOrigin:{kind:"task-notification"}` marker, is flagged a meta message by `promptIsMeta`, and enters a dedicated command-queue mode (`new Set(["task-notification"])`). In other words, "some background agent finished" has its own class of entrance in the main loop, handled separately from a line you typed by hand — so the main loop can treat "human input" and "background report" as two distinct sources rather than jamming the report in as a disguised user message. This works together with that `isolation: 'worktree'` field: the schema exposes worktree as an isolation option, so background subagents running in parallel can take independent git worktrees and not clobber each other's changes — exactly the isolation substrate from [Chapter 8](07-multi-agent.md). Whether every subagent gets its own worktree, and what the default is, the schema itself doesn't establish, and this chapter won't draw that conclusion for it.

This notification mechanism also comes with a heavy behavioral constraint; the harshest of the three variants reads verbatim: when an agent runs in the background, you're notified automatically when it completes — do NOT sleep, poll, or proactively check on its progress. Don't sleep, don't poll, don't go poke at it. That line nails "async" from advice into discipline: once you dispatch, let go and wait for the notification instead of sitting there re-checking — which is, as it happens, the same source as the "no sleep-poll" clause in this box's runtime discipline. So "background" in Claude Code is really two layers stacked: one is the detached sessions you `/bg` out by hand, looked after by the daemon of 21.4; the other is the subagents the main session dispatches in-session, which default to the background and call you back on completion with a task-notification. The latter layer is nailed down by snapshot source, the former only by strings; they belong to the same background-capability landscape, but whether the subagents are supervised by that daemon — whether they count as the fleet's "troops" — is a boundary the current evidence can't bridge, and this chapter doesn't write it as established fact.

## 21.7 Wrapping up its own work: commit, push, open a draft PR

A background task that can detach from the terminal and run unattended for hours needs one last link to close the loop: having finished the work, it has to land the result for you rather than leave a pile of changes hanging for you to collect by hand when you're back. The strings have a passage on exactly this; in paraphrase here, with the exact key phrases quoted in the appendix, its behavioral gist is: when you've changed code, commit it, push the branch, then open a draft PR (`gh pr create --draft`) without stopping to ask; don't end the job in an uncommitted state with something like "say the word and I'll open the PR."

What's notable is that it nails the red lines into the same passage: Never push to main/master, force-push, or merge. This is consistent with Auto Mode's Git rules in [Chapter 18](18-auto-mode.md): pushing to your own feature branch goes through, while pushing to the default branch and rewriting history are stopped. It also runs with this box's auto_commit_deploy convention — commit and open-PR belong to "tidy the work up for you," while push-to-trunk and merge belong to "make the ship-it call for you"; the former can be automatic, the latter must be left to a person. The strings then spell out a fallback: if you never isolated a worktree (EnterWorktree failed, or your cwd was already the user's own checkout), the wrap-up logic switches to a more conservative path. This whole passage is behavioral description with no matching source — the tier where you can read verbatim in the binary strings that "it's told to do this," but can't see "how it's implemented in code."

## 21.8 What it looks like in the telemetry

What this hooks into is the discrete-event telemetry from [Chapter 16](16-observability.md). The nearly eighty `tengu_bg_*` events are themselves a feature map — count each prefix's term density in the strings and the weight distribution across the fleet's parts falls out. To be clear up front: this is a static string-term count in the binary, not a runtime event-firing frequency. The static topology is roughly: `background` dominates with a thousand-plus occurrences, `attach` five hundred-plus, `daemon` three hundred-plus, saying "detach + take over + daemon lifecycle" is the trunk; further down `detach`, `spare`, `task-notification` at the one-to-two-hundred scale, `supervisor`, `roster` in the tens, matching the more specialized subsystems above; sparsest are the likes of `/bg` and `draft PR` in the single to low-double digits — entry and wrap-up actions referenced in only a few places, so appearing rarely in the strings is expected.

The use of this term-density table isn't just that it looks nice. It's a piece of side evidence: `background`/`attach`/`daemon` far outweighing `spare`/`roster` in the strings matches the subsystem division of labor inferred in 21.4 — the entry point and daemon life-and-death are the heavily-referenced trunk, the pre-warm pool and roster the more specialized, less-referenced maintenance subsystems. But don't read it as runtime frequency: this is static topology, not call counts from telemetry. Chapter 16's "replay a run with discrete events" lands here as: you can follow the `tengu_bg_*` sequence a background session actually emits and string together its whole life from dispatch, spawn, attach, to final detach or reap — provided you can get hold of that run's full `tengu_bg_*` event sequence.

## 21.9 An evolution you can reconcile across two ends

This capability, too, can be reconciled across both ends, and more cleanly than the earlier chapters — because its evidence was split across two ends to begin with.

The snapshot end (the leaked snapshot, labeled v2.1.88): the in-session background substrate is already complete. The big files under `components/tasks/` (`BackgroundTasksDialog`, `BackgroundTaskStatus`, `TaskListV2`), `tasks/LocalAgentTask/`, and the two-hundred-plus-KB `AgentTool` — with `run_in_background` and worktree isolation — are all in there, readable line by line. That is, the "subagents default to the background, come back on completion via task-notification" layer had taken shape by snapshot time.

The current end (2.1.202): the daemon supervisor that grows a single-session background task into "detach-from-terminal + multi-session fleet supervision" is entirely new growth. `/bg`, `claude agents`/`attach`/`logs`, the on-demand daemon, those nearly eighty `tengu_bg_*` events — none can be found in the snapshot (`grep -rlF 'tengu_bg_'` hits zero files), surviving only as strings in the 2.1.202 binary. Line the two up and you see what happened over these six months: how the background subagents get dispatched within a session was there at snapshot time; supervising these background sessions as a fleet that needs health checks, crash respawn, and orphan adoption is a layer laid down after the snapshot. Another line of "didn't leak with the snapshot doesn't mean it can't be analyzed" — except this time, what leaked and what didn't fall right along the seam between "in-session" and "across-session."

## 21.10 Appendix: how we know this (the reverse-engineering method, reproducible)

This chapter's evidence comes from four self-reproducible routes, only this time the four are especially uneven in weight.

First, read the snapshot source. The in-session background substrate is all in the v2.1.88 snapshot: `AgentTool`'s `run_in_background`, the background-task UI under `components/tasks/`, `LocalAgentTask` — two-hundred-plus KB of components plus a family of hooks, locatable to files. It gives the mechanism of the "how the in-session background subagents get dispatched, how they're collected via task-notification" layer.

Second, look at the CLI's own command surface. `claude agents` is a proper Commander subcommand; `claude --help` / `claude agents` shows the agents / attach / logs actions and their descriptions, and the kill switch `CLAUDE_CODE_DISABLE_AGENT_VIEW` is queryable in settings. This route is cleaner than digging in the binary, but it can only confirm the command surface and the switch exist, not reach the supervisor underneath.

Third, extract strings from the binary. The bulk of this chapter is here: those nearly eighty `tengu_bg_*` event names, `/bg`'s detach copy, "Sessions keep running if you close the terminal," the on-demand daemon's idle-exit and yielding rule, the wrap-up commit/push/draft-PR passage — all read verbatim from the 2.1.202 binary with `strings` plus `grep -aF`. In the snapshot `grep -rlF 'tengu_bg_'` hits zero files, so for the whole block of daemon fleet supervision, strings are the only handle.

Fourth, capture plaintext traffic. Stand up a plaintext reverse proxy (recipe in [Chapter 17](17-autonomy-goal-loop.md), 17.5) and you can see the request dispatching a background subagent, and the `<task-notification>` that re-enters the main loop after it finishes. But this route has a natural ceiling for this chapter worth stating: how the daemon supervises its workers and sessions, how the PTY is relayed, how the roster is recorded all go over local inter-process communication and PTY, not HTTPS — they never touch the network, and the reverse proxy can't catch them. This is exactly why the third route (strings) carries the load here: for the behavioral truth of fleet supervision, there's no corroboration but strings.

The honest boundary, laid out in one line. The in-session background-subagent layer is nailed down by source (source, strings, plus the tool descriptions injected into this session — three-way corroboration); `/bg`/`claude agents`/the on-demand daemon/the kill switch/the wrap-up instructions/the nearly eighty event names are nailed down by 2.1.202 strings; but under what conditions each event fires, and how dispatch, respawn, and adopt chain into a state machine, is inference-tier from the event names, not source; as for the daemon supervisor's process-model details, the PTY-RV transport protocol internals, and the roster file's exact format — those are an unreachable blind spot, neither in the snapshot nor over the network. This seam the chapter has not tried to paper over, from start to finish.

---

## Appendix: key strings and tool descriptions for the background fleet (with elisions)

Below are the originals this chapter leans on. The UX copy, the on-demand daemon and kill switch, the three variants of subagents-background-by-default, and the wrap-up instructions are all read verbatim from the 2.1.202 binary; the `tengu_bg_*` event family gives a grouped map of event names (names verbatim, the grouping is our inference); longer stretches are elided with `…`. The daemon supervisor's internal state-machine logic has no matching source to quote, so only the event names it leaves in the strings are listed here.

### `/bg` and `claude agents` (verbatim, 2.1.202 strings)

```text
/bg detaches this session to run in the background, and `claude agents` shows
every backgrounded session in one table with a status color — glance to see which
ones need you, space to reply, enter to attach.

(onboarding hint) /bg this session, then run `claude agents` in a new terminal

(survives terminal close) Sessions keep running if you close the terminal.

(resume conflict) … as a background agent. Open `claude agents` to attach to it, or
stop it there first to resume here.

(Commander subcommand, verbatim) .command("agents").description("Manage background agents")
```

Author-summarized command surface (not verbatim; the raw strings pass the subcommand argument as a `${e}` template, rendered here as `<id>` for readability):

```text
claude agents        list every background session
claude attach <id>   take one over in the current terminal
claude logs <id>     page through its recent output
```

### The on-demand daemon and the kill switch (verbatim, 2.1.202 strings)

```text
(idle-exit) nothing holding this daemon open — will idle-exit shortly …
the next `claude agents` or `claude --bg` will start a new one

(yielding rule) an on-demand daemon never displaces a running one …
only a transient daemon can be displaced

(settings-schema kill switch) Disable agent view (claude agents, --bg,
/background, the on-demand daemon). Typically set in managed settings.
Equivalent to CLAUDE_CODE_DISABLE_AGENT_VIEW=1.
```

### Subagents default to the background (verbatim, three variants, source + strings)

```text
Subagents run in the background by default; you'll be notified when one completes.
Pass `run_in_background: false` for a synchronous run when you need the result
before continuing.

Agents run in the background by default; you will be notified when one completes.
Set to false to run this agent synchronously when you need its result before
continuing.

Agents run in the background by default. When an agent runs in the background, you
will be automatically notified when it completes — do NOT sleep, poll, or
proactively check on its progress.
```

### Wrap-up on completion: commit / push / draft PR (behavioral-description excerpt, with head/tail elision, not strictly verbatim; 2.1.202 strings)

```text
… the task: when you've made code changes, commit them, push the branch, and open
a draft PR (`gh pr create --draft`) without stopping to ask — don't end the job
with uncommitted work or 'say the word and I'll open the PR'. Never push to
main/master, force-push, or merge. If you're working in the user's own checkout
instead — you never isolated, EnterWorktree failed, or your cwd was already a
worktree …
```

### The daemon fleet supervisor's event family (event names verbatim, grouping is inference; 2.1.202 strings, 0 snapshot files)

```text
[entry / dispatch]   bg_agent_action  bg_dispatch  _dispatch_rescued  _stale_drop
                     _sigkill_escalate  _low_mem  _fallback  _rejected
                     _watcher_failed  bg_classify  _classifier_config

[daemon lifecycle]   _daemon_spawn_failed  _daemon_install  _daemon_zombie_restart
                     _zombie_false_positive  _daemon_cold_start_ask  _ask_answer
                     _daemon_service_stale_exec  _service_poll_fallthrough
                     _daemon_wmi_fallback (Windows)  _daemon_macos_aqua_wrap (macOS)
                     _daemon_binary_takeover  _daemon_bg_disabled_skip

[spare / worker pool] _spare_claim_fail  _spare_spawn  _spare_claim  _spare_enable
                     _worker_spawn  _exit  _vanished  _stalled
                     _prewarm_per_sweep  _low_mem_mb  _retire_pinned_low_mem

[respawn/adopt/orphan] _retired  _respawn  _respawn_unconfirmed_bail  _stale
                     _exhausted  _no_transcript  _adopt  _adopt_token_lost_respawn
                     _upgrade_respawn  _unverified  _sock_unlinked  _orphan_reap
                     _roster_orphan_adopted  _roster_parse_failed

[attach / detach]    _attach  _attach_outcome  _first_frame  _legacy_autorespawn
                     _upgrade  _stall_respawn  _stall_ms  _stall_gave_up  _attach_kick

[transport / PTY]    _pty_unavailable  _pty_auth_mismatch  _ptyhost_crash
                     _proto_mismatch  _rv_reply_rejected  _rv_connect_exhausted
                     _rv_auth_mismatch  _bridge_flush_truncated  _skew_nudge
                     _binary_takeover
```

> Provenance: `/bg`, `claude agents`/`attach`/`logs`, the on-demand daemon, the kill switch, the wrap-up instructions, and all the `tengu_bg_*` event names come from plaintext-string extraction of the 2.1.202 client binary; the subagents-background-by-default tool descriptions are additionally corroborated in two places — the v2.1.88 snapshot source and the tool set injected into this session. Event names are verbatim, the grouping is our inference, and `…` marks elisions. The daemon supervisor's internal state-machine logic (each event's firing conditions, the wiring of dispatch/respawn/adopt) has no matching source and is inference-tier; the PTY-RV transport protocol internals and the roster file format hit 0 files under `grep -rlF 'tengu_bg_'` in the snapshot and are an unreachable blind spot.
