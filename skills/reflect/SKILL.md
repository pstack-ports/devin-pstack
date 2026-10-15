---
name: reflect
description: Spawn three parallel review subagents over the active transcript, surface learnings, and route each to a concrete edit on an existing skill. Use when the user says reflect.
triggers: [user]
---

# Reflect

Mine the current conversation for durable learnings, then route them into skill edits.

Invoke when the user says "reflect" or "/pstack:reflect". Skip when the conversation is trivial, off-topic, or already covered by an existing skill the parent followed correctly. One-offs are not learnings.

## 1. Locate the active transcript

Before fanning out, find this conversation's transcript in Devin's session store. `devin list` names the sessions for the current directory; their data lives under `~/Library/Application Support/Devin/` on macOS (`%APPDATA%\devin` on Windows). Do not read another project's sessions — that crosses workspace boundaries and reads private chats from unrelated projects.

```bash
ls -t "$HOME/Library/Application Support/Devin"/User/**/*.jsonl "$HOME/Library/Application Support/Devin"/User/acp-messages/*.db 2>/dev/null | head -10
```

Order candidates by real modification time and take the one whose opening user prompt matches this conversation. If none matches or the store is unreadable, pass a tight digest of the session instead.

## 2. Spawn three reviewers in parallel

One message, three `run_subagent` calls. Reviewers need MCP access to look up the tickets, chat threads, and observability traces the transcript references, so do not use `subagent_explore` — it has no MCP tools.

Each reviewer and the synthesizer name a role line in the `pstack-models.md` rule and a default. Map the value to the spawn's `profile`: a real slug `M` becomes `pstack-M-ro`; `auto`, `inherit-parent`, or a missing rule or line uses `subagent_general` with a read-only brief. Create a missing `pstack-M-ro` profile per `/pstack:setup-pstack` rather than swapping models. Pick each subagent's reasoning level and fast or standard speed for the job, per the Subagents section of the **poteto-mode** skill.

| Lens | Role line | Default `model` | Prompt template |
|---|---|---|---|
| Judgment | `reflect judgment, divergent, synthesizer` | `claude-opus-5-5-xhigh` | `references/judgment-reviewer.md` |
| Tooling | `reflect tooling` | `swe-2-high` | `references/tooling-reviewer.md` |
| Divergent | `reflect judgment, divergent, synthesizer` | `claude-opus-5-5-xhigh` | `references/divergent-reviewer.md` |

Pass each template verbatim, substituting the transcript path or digest where marked. Reviewers return findings in the `run_subagent` response body.

## 3. Synthesize

One `run_subagent` call, `profile` from the `reflect judgment, divergent, synthesizer` line (default `claude-opus-5-5-xhigh`, so `pstack-claude-opus-5-5-xhigh-ro`), mapped per the rule above. It spot-verifies citations through MCP, which `subagent_explore` cannot do. Pass `references/synthesizer.md` verbatim, with each reviewer's full output inlined where marked. It returns an Accepted / Rejected / Backlog list.

## 4. Structural enforcement check

Move any Accepted item that a lint rule, script, metadata flag, or runtime check would enforce more reliably to Backlog. See the **encode-lessons-in-structure** principle skill.

## 5. Apply

Present the synthesizer's full Accepted / Rejected / Backlog output and wait for explicit approval before applying any Accepted edit. The user picks the subset and may redirect routings. Skill changes affect every future agent in the org. Do not auto-apply.

File each Backlog item to your team's devex or backlog tracker without waiting. Only the Accepted list waits for approval.

Follow each approved row's Routing exactly:

- Trivial existing-skill edit (a one-line bullet, a tightened sentence, a stale fact corrected): the parent does it directly.
- Substantive existing-skill edit (a new section, a new pattern table, more than ~10 lines): run the Authoring or modifying a skill playbook (`skills/poteto-mode/playbooks/authoring-a-skill.md`) and its draft / test / iterate loop.
- `tune description: <skill path>` (the skill exists but didn't trigger when it should have): run the same playbook's description-optimization pass.
- `new skill: <kebab-name>`: author the SKILL.md per that playbook. Do not invent the shape ad hoc.

If your environment ships a SKILL.md validator, run it on every touched skill before declaring done.

## 6. Summarize for the user

Short list, no preamble:

- Edits applied: `<skill path>`. What changed, one line each.
- New skills created: `<skill path>`. One line each (rare).
- Backlog filed to the devex tracker: `<issue title>` (`<tags>`). One line each.
- Dropped: one line per rejected finding + reason from the synthesizer.
