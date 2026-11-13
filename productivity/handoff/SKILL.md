---
name: handoff
description: Compact the current conversation into a handoff document for another agent to pick up.
argument-hint: "What will the next session be used for?"
disable-model-invocation: true
license: MIT
metadata:
  source: https://github.com/mattpocock/skills/tree/5863806dd4e15c63174bee4e7515a4256792ecfa/skills/productivity/handoff
  copyright: Copyright (c) 2026 Matt Pocock
  derived-from: handoff, modified
---

Write a handoff document summarising the current conversation so a fresh agent can continue the work. The reader has none of this conversation and will trust the document without re-checking it, so write for someone starting cold.

If the user passed arguments, treat them as a description of what the next session will focus on and tailor the doc accordingly.

Cover these, in this order:

1. **Goal**: what the work is for, and what done looks like.
2. **State**: what is finished and what is in flight. Mark each claim **verified** (and say how: a test run, a command, a file you read) or **assumed**, because the reader will build on an assumption as if it were a fact. Say where the work lives: repository, branch, the commit it is on, and whether there are uncommitted changes, since the next agent may be on another machine where those do not exist.
3. **Decisions**: what was decided and why, and what was tried and dropped, so the next agent neither reopens a settled question nor repeats a dead end.
4. **Next**: the next concrete step, and anything that is blocked and on whom.
5. **Suggested skills**: the skills the next agent should call the Skill tool for, if any apply.

Do not duplicate content already captured in other artifacts (specs, plans, ADRs, issues, commits, diffs). Reference them by path or URL instead.

Redact any sensitive information, such as API keys, passwords, or personally identifiable information.

Save it as `handoff-<slug>.md` in the temporary directory of the user's OS (`$TMPDIR`, else `/tmp`; `%TEMP%` on Windows), not the current workspace, unless the user names another place. Then tell the user three things: the full path, that a temporary directory can be emptied on reboot or between sessions, and the line to give the next agent: `Read <path> and continue from it.`
