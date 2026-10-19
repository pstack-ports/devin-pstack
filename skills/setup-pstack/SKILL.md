---
name: setup-pstack
description: Configure which models pstack uses per role and at what reasoning budget. Detects your available models and writes an always-on rule plus model-pinning subagent profiles that override the skill defaults. Use for /pstack:setup-pstack, "configure pstack models", "pstack budget", or changing pstack's model choices.
---

# Setup pstack

Write `~/.devin/rules/pstack-models.md`, an always-on rule that sets pstack's model per role, and the model-pinning subagent profiles under `~/.config/devin/agents/` that those roles spawn through.

## Steps

### 1. Detect available models

Run `devin models list` and read the slugs it prints. That is the dependable source. Devin subagents take their model from the `model:` field of a custom subagent profile, so every real slug you write must come from this list. If you cannot detect any, ask the user to paste the slugs they have access to. Never write a real slug you have not confirmed is available. The aliases `inherit-parent` and `auto` are always valid even though they are not detected slugs.

### 2. Load current state

The default role-to-model mapping is the rule shape shown in step 5 below. If `~/.devin/rules/pstack-models.md` already exists, read it and treat its `# budget` line and its role values as the current choices. Otherwise start from those defaults. A line whose role is not in step 5, such as `how critics`, is from a retired role. Drop it.

### 3. Budget, map, and confirm

**(a) Ask for a budget.** Prefer ask_user_question over free text. Offer these four options with these exact labels, and name the current budget when the rule records one. With no rule, say that `large` matches the skill defaults.

- `unlimited — max reasoning`
- `large — xhigh reasoning`
- `medium — high reasoning`
- `small — medium reasoning`

**(b) Apply it.** Build the working table from the skill defaults, and on a re-run keep any role you changed by family, list, or alias (`inherit-parent`, `auto`). `unlimited`, `large`, `medium`, and `small` set the effort token of every real slug, panel entries included, to `max`, `xhigh`, `high`, or `medium`. The effort token is the last token, or the one before a trailing speed suffix (`fast`, `lightning`, or `priority` depending on family), on the ladder `max` > `xhigh` > `high` > `medium` > `low`. If the result is not a detected slug, use the same family's detected slug with the highest effort at or below the target, else mark the role as needing a choice. `inherit-parent` and `auto` do not change. So `unlimited` turns `claude-opus-5-5-xhigh` into `claude-opus-5-5-max`. SWE-2 slugs top out at `max` too, so under `unlimited` `swe-2-high` becomes `swe-2-max`. `large` keeps both defaults. `small` turns them into `claude-opus-5-5-medium` and `swe-2-medium`.

**(c) Show the roles and confirm.** Show every role with its model, marking any real slug not in the detected set as needing a choice. Also list each line step 2 dropped. Ask whether to accept as-is or change specific roles, offering the detected models plus `inherit-parent` and `auto` (both mean: this role runs on the parent chat model, which is how Adaptive users stay on Adaptive) as the options. Prefer ask_user_question over free text. For panel roles (arena runners, architect runners, interrogate reviewers) the value is a list, and one subagent runs per entry, alias entries included, so the list length sets the count. `arena cross-judge pool` is also a list, but Arena selects one value from it whose model family differs from the parent's when possible. `swarm workers` is the default model for every worker unless a race or comparison assigns another model per arm.

### 4. Validate

Every real slug written must be in the detected set. `inherit-parent` and `auto` always pass. If a chosen real slug is not available, stop and ask again.

### 5. Write the rule

Write `~/.devin/rules/pstack-models.md` with `trigger: always_on`, a `# budget` line with the chosen label and its target effort, and one line per role, using the same labels poteto-mode uses. Overwrite the whole file so re-runs stay idempotent. Shape:

```
---
description: pstack per-role model choices (overrides skill defaults)
trigger: always_on
---
# pstack model configuration. One line per role. Delete a line to fall back to the skill default.
# `inherit-parent` or `auto` as a value: the role runs on the parent chat model (spawn `subagent_general` or `subagent_explore`). Alias entries in a panel list still count toward its fan-out.
# budget: large (xhigh)
feature, refactoring: swe-2-high
bug-fix: swe-2-high
perf-issue: swe-2-high
hillclimb: swe-2-high
judgment and prose: claude-opus-5-5-xhigh
hardest tasks: claude-opus-5-5-xhigh
how explorer: swe-2-high
how explainer: claude-opus-5-5-xhigh
why investigators: swe-2-high
why synthesizer: claude-opus-5-5-xhigh
reflect tooling: swe-2-high
reflect judgment, divergent, synthesizer: claude-opus-5-5-xhigh
arena runners: claude-opus-5-5-xhigh, swe-2-high
arena cross-judge pool: claude-opus-5-5-xhigh, swe-2-high
swarm workers: swe-2-high
architect runners: claude-opus-5-5-xhigh, swe-2-high
interrogate reviewers: claude-opus-5-5-xhigh, swe-2-high
```

### 6. Write the model profiles

For each distinct real slug `M` in the rule, write two custom subagent profiles. Skills spawn them as `run_subagent` profiles `pstack-M` (write-capable) and `pstack-M-ro` (read-only). Skip `inherit-parent` and `auto` values; those map to the built-in `subagent_general` and `subagent_explore` profiles and need no file.

`~/.config/devin/agents/pstack-M.md`:

```
---
name: pstack-M
description: pstack delegate pinned to model M, spawned by pstack skills for write work.
model: M
max-nesting: 3
---
You are a pstack delegate. Read the `poteto-mode` skill's `SKILL.md` in full before doing any work, including its inline Principles index. Use the absolute path your parent gave you. With no path, search `~/.config/devin/` and the project's `.devin/` for a file whose path ends in `skills/poteto-mode/SKILL.md`. If none exists, stop and tell your parent. Work the brief your parent passed and report evidence, not claims.
```

`~/.config/devin/agents/pstack-M-ro.md`:

```
---
name: pstack-M-ro
description: Read-only pstack reviewer pinned to model M, spawned by pstack skills for research and review.
model: M
allowed-tools:
  - read
  - grep
  - glob
  - find_file_by_name
  - exec
  - web_search
  - webfetch
  - mcp_list_servers
  - mcp_list_tools
  - mcp_call_tool
  - mcp_read_resource
---
You are a read-only pstack subagent. Work the brief your parent passed. Report findings with file-and-line evidence. Do not edit files or run commands that mutate state.
```

List any existing `~/.config/devin/agents/pstack-*.md` files whose slug is no longer in the rule and offer to remove them.

### 7. Confirm

Tell the user the rule and profiles were written and that they apply to new sessions. Re-running this skill updates both.

### 8. Offer a verification skill (optional)

Check whether the project has a way to drive the real app for proof (a `verify-*` skill, or an existing harness). If not, offer once: "want a project-local verification skill, so agents can drive the app the way a user does and prove changes work? I can generate one with /pstack:create-verification-skill." On yes, invoke `/pstack:create-verification-skill` (resolves wherever pstack is installed: workspace, user, or plugin). On no, move on without pushing.

### 9. Offer a formal verification skill (optional)

Check whether the project already has a formal verification skill (a project skill that runs a model checker, a proof, or another verifier against the code). If not, offer once: "want a formal verification skill, so agents can prove a key property of your code holds in every case in scope, not only the cases tests reach? I can generate one with /pstack:create-formal-verification-skill." On yes, invoke `/pstack:create-formal-verification-skill` (resolves wherever pstack is installed: workspace, user, or plugin). On no, move on without pushing.
