---
name: swarm
description: "Fan out N parallel workers, drain them, and return one report. Use for /pstack:swarm, 'swarm this', or parallel coverage, races, gauntlets, and exploration."
triggers: [user]
---

# Swarm

## Start

Open a todolist with one entry per phase before launching anything.

1. Frame
2. Fan out
3. Aggregate
4. Report

## Phase A: Frame

1. State the done predicate and the artifact or report the swarm must return.
2. Choose the shape. Partition into slices, race N workers on identical briefs, or mix both. For a race or mixed shape, declare `first pass`, `rank all`, or `best-of` before spawning.
4. Pick the worker model from the `swarm workers` line in `~/.devin/rules/pstack-models.md`. If the rule or that line is missing, use `swe-2-high`. Map the value to the workers' `profile`: a real slug `M` becomes `pstack-M`; `auto` or `inherit-parent` uses `subagent_general`, so the workers run on the parent model. Create a missing `pstack-M` profile per `/pstack:setup-pstack`. An entry `devin models list` doesn't report, or any `fusion-*` entry, counts as a rejected slug; never create a profile for it. If a pinned slug is rejected at spawn, use the default and say so. If it rejects the default, use the closest valid slug of the same family `devin models list` reports. For a model race, name each arm's model up front. Pick each subagent's reasoning level and fast or standard speed for the job, per the Subagents section of the **poteto-mode** skill.
5. Give each worker its own writable output when it writes. When workers verify or measure commits, each brief names the exact SHAs. A measurement brief also names the method (sample count, what one sample is, order). The worker records both in its result.

## Phase B: Fan out

Spawn all N workers in one `run_subagent` message with the step 4 `profile` and `is_background: true`. Workers that need an isolated VM — dependency installs, a running app, a browser — can instead run as cloud Devin sessions via `devin --cloud -p "<brief>"`, tracked through `devin list` and their session URLs. Keep a worker local only when it needs access to something on the user's computer. Cloud sessions load plugin skills but not plugin subagent profiles, so a cloud worker cannot spawn `pstack-*` or `pstack:poteto-agent` delegates — keep cloud workers leaf-level and run anything that fans out locally.

When a worker must start from a non-default pushed branch, name the branch in its brief so the cloud session or worktree checks it out first.

Every brief stands alone. Include the goal, scope, exact slice or race arm, how to verify, and what to report. Reports use `PASS`, `ISSUES`, or `BLOCKED` with evidence. A worker that can prove a defect reports `ISSUES` and lists every issue it can prove, not only the first.

If a worker drops out, proceed with N-1 and note it.

## Phase C: Aggregate

Read the terminal results. Drop a result that does not record the SHAs and method its brief names, and respawn that worker once. After a second miss, record a gap. A gap does not count as a pass. For coverage, every required slice needs a result. For a race, apply the selection rule declared up front. Use first pass, rank all, or best-of. Do not paste raw worker dumps.

## Phase D: Report

Return one consolidated in-chat report with the table, issue one-liners, gaps or dropouts, and the race rule when used.
