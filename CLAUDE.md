# CLAUDE.md — lcls-twincat-common-components

Repo-specific notes. **Shared rules, coding standards, and publishing workflow live in the
central `~/.claude/CLAUDE.md`** (auto-loaded by Claude Code) — read it first; everything there
applies here.

## Session start rule

**At the start of every new session, read this file (and the central `~/.claude/CLAUDE.md`) in
full before doing any work, and follow the rules in them throughout the session.**

## Commit rules

**Never amend an existing commit (`git commit --amend`, fixup/squash, or any history rewrite)
without asking first** — it may already be pushed. Default to a new follow-up commit.

**Never add a `Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>` trailer (or
any Copilot co-author trailer) to commit messages in this repo.**

## This repo

- Library of reusable LCLS TwinCAT **common components** (device FBs, position-state helpers,
  sensors) plus their TcUnit test suites.
- Source layout: device FBs under `Library/Devices/<DEVICE>/`, shared POUs under
  `Library/POUs/`, tests under `Library/Tests/`.

## Repo-specific notes

_(Add anything unique to this repo here — fork name, build specifics, device-family
conventions. Generic TwinCAT ST standards stay in the central file.)_
