# OFAuth agent skills

Agent skills that teach coding agents (Claude Code, Cursor, Codex and others) how to build on OFAuth.

| Skill | What it covers |
| --- | --- |
| [`ofauth`](skills/ofauth/SKILL.md) | Adding OFAuth to an app: OAuth sign-in, linked accounts, Access API calls, webhooks, Realtime and Vault+ |

## Install

Copy `skills/ofauth/` into your agent's skills folder, for example `.claude/skills/ofauth/` for Claude Code. Every file the skill uses is listed in [`skills/ofauth/files.txt`](skills/ofauth/files.txt).

Docs: https://docs.ofauth.com

## Versions

Each skill's version is in its `SKILL.md` front matter. Releases are tagged `<skill>-v<version>`, for example `ofauth-v1.1.0`.

OFAuth is an independent service and is not affiliated with OnlyFans.
