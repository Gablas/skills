---
name: handover
description: Write a self-contained handover prompt to /tmp so a fresh agent (Codex, Cursor, another Claude session, a subagent) can continue the work with no prior context. Use for any handoff — continuing an in-progress task, parallelizing work, getting a second opinion, or escalating a stuck investigation to another model. Triggers: "hand this off", "write a prompt for Codex/Cursor/<agent>", "brief another agent", "make a handover", "let another session pick this up", "escalate to another model". The skill produces a single markdown file and prints its path — it does not launch the other agent.
---

# Writing a handover prompt

## When to use

Any time the user wants another agent to pick up the work. The receiving agent might:
- **Continue a task** you started (most common — half-implemented feature, partial refactor, research in progress)
- **Do work in parallel** on a separate track
- **Cross-check / second opinion** on something you built
- **Try a stuck problem** with different heuristics

Do not use for generic note-taking or end-of-session summaries.

## Guiding principle

The reader has **zero context**. No access to this conversation, no memory of what you tried, no idea which files matter. Every fact they need must be on the page. Brief like you would a smart colleague who just walked into the room — explain the goal, the current state, and what you want them to do next. Terse command-style briefs produce shallow, generic work from the receiving agent; include enough context that they can make judgment calls instead of blindly following steps.

## Structure: core + situational

Every handover has the same **core spine**. Add **situational sections** on top based on what kind of handoff it is.

### Core (always include)

1. **Task** — one sentence. What should the other agent achieve?
2. **Repo layout** — absolute path to the working dir and the files that matter. Package manager, tool-version quirks (Node version, pnpm vs npm), env setup commands. Branch/worktree if it matters.
3. **Current state** — what's done, what's in progress, what's untouched. Include uncommitted changes: `git status -sb`, a short `git diff --stat` of anything notable, any background jobs still running with their IDs + output-file paths.
4. **Next steps** — numbered, actionable. Step 1 should be runnable without thinking.
5. **Success criteria** — how the agent (and you) know it's done. Tests passing, a PR opened, a specific output in a file, user-visible behavior.

### Add for task-continuation handoffs

- **Design decisions made so far** — the non-obvious choices with *why*. Future agents waste hours re-litigating settled decisions when this is missing.
- **Open questions / things to decide** — call out what you deliberately left open.

### Add for debugging / stuck-problem handoffs

- **Symptom** — concrete observed behavior, error messages verbatim, timings if timing matters.
- **Evidence** — a table or list of experiments/data points gathered. Include successes and failures; the contrast is usually the signal.
- **Ruled out** — hypotheses disproven, each with a one-line reason. Prevents rerunning dead ends.
- **Hypotheses to test** — ordered by likelihood, each with a concrete check.
- **Reproduce quickly** — exact commands. Include shortcuts that skip expensive steps ("submit the existing artifact directly, don't rebuild").

### Always include

- **Anti-goals / don't do** — regressions from earlier fixes, dead ends, or decisions the user has ruled out. Critical on every handover type, not just debugging. This is the single highest-leverage section.

## Quality bar

- **Paths must be absolute.** `/Users/.../recorder/forge.config.js:45`, not "the forge config".
- **Commands must be copy-pasteable.** Include env-var setup. If a command needs `set -a && source .env && set +a` first, write that out.
- **Cite concrete values.** Submission IDs, PR numbers, error codes, timestamps, SHAs, file:line. "Apple returned an error" is useless; "submission 27a2868b returned Invalid / statusCode 4000 on path .../GStreamer/Versions/1.0/GStreamer: 'The signature of the binary is invalid'" is leverage.
- **No external refs the reader can't reach.** Don't say "see the conversation"; the conversation isn't there. Quote it inline.
- **Don't dump code.** Reference files by path:line, paste only the minimum snippet needed to understand a decision.
- **Flag uncommitted work.** If the state the next agent needs lives in the working tree (not in git), say so explicitly and list the files. Silently-uncommitted state is a common handover failure.

## File location + naming

Write to `/tmp/handover-<YYYYMMDD-HHMM>-<slug>.md`. The slug is 2–4 kebab-case words summarizing the task ("notarization-stuck", "migrate-eslint-config", "finish-auth-hook"). Local time, not UTC:

```bash
date +%Y%m%d-%H%M
```

Never overwrite an existing file — if the path exists, append `-v2`, `-v3`, etc.

## Gathering context before writing

Verify anything you're about to assert:
- `git status -sb` + `git log --oneline -10` for repo state
- Read the relevant config files rather than relying on memory
- Pull logs you'll reference so you can quote exact lines
- If there are background tasks (Bash `run_in_background: true`), include task IDs + output-file paths so the next agent can inspect them

## Ending the skill

After writing the file:
1. Print only the absolute path
2. In 1–2 sentences, describe what's in it
3. Do not offer to launch the other agent unless the user asked

## Anti-patterns

- Treating every handover as a bug report. Most handovers are task continuations, not post-mortems. Skip "Evidence / Ruled out / Hypotheses" when the work isn't stuck.
- Vague next-steps ("continue the migration") — give the first concrete command or file to edit.
- Copying the entire chat transcript — noise drowns signal.
- Omitting anti-goals — the most common cause of a receiving agent regressing earlier fixes.
- Writing to the working repo instead of `/tmp` — handovers are scratch, not source.
- Forgetting to mention uncommitted working-tree state.
