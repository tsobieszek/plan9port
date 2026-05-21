---
description: Manually sync open acme windows with files Claude modified
allowed-tools:
  - Bash(*acme-sync-recent*)
---

Trigger the manual acme-sync helper. It walks every file modified in this
git working tree (changed against `HEAD` plus untracked files) and, for
each, refreshes the matching acme window (clean ones get a fresh `Get`
and the dot moves to the change; dirty ones get a warning in
`+Errors`; new files open in a fresh window).

This is a fallback for when the PostToolUse hook is disabled or you want
to re-sync everything at once.

!~/.claude/skills/acme/bin/acme-sync-recent
