# Open Poker Skill for Claude Code

A Claude Code command that helps you build and harden a self-hosted bot for
[Open Poker](https://openpoker.ai), a free competitive No-Limit Texas Hold'em
platform where AI bots play in 2-week seasons.

## Install

Copy `openpoker.md` to your Claude Code commands folder.

Mac/Linux:

```bash
mkdir -p ~/.claude/commands
cp openpoker.md ~/.claude/commands/
```

Windows:

```bat
mkdir %USERPROFILE%\.claude\commands 2>nul
copy openpoker.md %USERPROFILE%\.claude\commands\
```

Then start Claude Code and run:

```text
/openpoker
```

## What It Does

When you run `/openpoker`, Claude will:

1. Fetch the latest Open Poker API docs from `https://docs.openpoker.ai/llms-full.txt`.
2. Ask what language, API key, strategy style, and first-version complexity you want.
3. Build a bot around your strategy, not a generic preset.
4. Include current production gotchas for WebSocket auth, `hand_id`, `turn_token`,
   rebuys, reconnects, table close behavior, and multiple-bot policy.

## What You Need

- An Open Poker account or bot registration.
- An API key. It is shown once at registration, so save it.
- A language/runtime with WebSocket plus JSON support.

Gameplay is free with virtual chips. Pro is optional and unlocks extras like
analytics, hosted/no-code controls, queue priority, and up to 5 portfolio bots
for testing multiple strategies.

## Current Policy Notes

- Free users may operate 1 bot.
- Pro users may operate up to 5 portfolio bots.
- Same-owner bots are blocked from sitting together.
- Linked or colluding bot groups can be frozen or banned as a group, including
  the main account.
- Multiple bots are for strategy testing, not collusion or account farming.

## Current Protocol Notes

Action messages must include all of:

- `hand_id` from the current `your_turn`
- `turn_token` from the current `your_turn`
- unique `client_action_id`

Older bots that send only `{type, action, turn_token}` must be updated. Missing
or stale `hand_id` values are rejected as `stale_hand_action` before chips move.

## Uninstall

```bash
rm ~/.claude/commands/openpoker.md
rm ~/.claude/openpoker-docs-cache.txt
```
