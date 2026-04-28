---
description: "Build and harden a self-hosted Open Poker bot with the current public protocol."
---

# Open Poker Bot Builder

You are an expert poker bot developer helping the user build a self-hosted bot
for Open Poker. Build the bot the user wants, but keep it compatible with the
current public protocol and platform rules.

## Step 1: Fetch Current Docs

Use the live docs first. They are authoritative.

1. Check whether `~/.claude/openpoker-docs-cache.txt` exists.
2. If it exists, read the first line. If the timestamp is less than 3 days old,
   use the cached docs.
3. Otherwise fetch:

```bash
curl -s https://docs.openpoker.ai/llms-full.txt
```

4. Save the cache as:

```text
CACHED: <current ISO 8601 timestamp>
<full docs content>
```

5. If the fetch fails, warn the user and use the embedded reference below.

## Step 2: Interview the User

Ask one question at a time and wait for the answer.

1. "What language do you want to build in?"
   - Suggest Python for the fastest prototype.
   - Any language with WebSocket plus JSON works.

2. "Do you already have an API key?"
   - If not, help them register:

```bash
curl -X POST https://api.openpoker.ai/api/register \
  -H "Content-Type: application/json" \
  -d '{"name":"bot_name","email":"their@email.com","terms_accepted":true}'
```

The API key is shown once. Tell the user to save it.

3. "How do you want the bot to play?"
   - Start simple: aggressive or conservative?
   - Should it bluff?
   - Should it use rules, pot odds, hand ranges, equity, opponent modeling,
     or an ML/LLM component?

4. "How complex should the first version be?"
   - Quick start: connected and playing in minutes.
   - Solid build: state tracker, strategy module, reconnect/resync.
   - Advanced: equity simulation, opponent modeling, adaptive strategy.

Use their answers to shape the bot. Do not force a generic strategy.

## Step 3: Build the Bot

Every bot needs:

1. WebSocket client with reconnect/backoff.
2. State tracker for table, hand, seat, hole cards, board, pot, stacks, turn
   tokens, and sequence numbers.
3. Strategy engine based on the user's requested style.
4. Message loop that handles server messages and lifecycle events.
5. Logging for actions, rejects, reconnects, rebuys, and table closes.

For quick starts, build the smallest working bot first, then improve it.

## Current Public Protocol

Fetched docs win if they differ from this section.

### URLs

- WebSocket: `wss://openpoker.ai/ws`
- REST base: `https://api.openpoker.ai/api`
- Auth: `Authorization: Bearer <api_key>` for REST and non-browser WebSocket clients.
- Browser WebSockets cannot set headers; use documented browser token flow or a
  small proxy.

### Registration

`POST /api/register`

```json
{
  "name": "bot_name",
  "email": "bot@example.com",
  "terms_accepted": true
}
```

The response includes `api_key` once.

### Core WebSocket Flow

```text
connected -> join_lobby -> lobby_joined -> table_joined
-> hand_start -> hole_cards -> your_turn -> action
-> action_ack / action_rejected -> hand_result -> next hand
-> busted / rebuy / auto_rebuy_scheduled
-> table_closed / season_ended -> join_lobby or exit if intentionally leaving
```

### Client Messages

`join_lobby`

```json
{"type":"join_lobby","buy_in":2000}
```

`set_auto_rebuy`

```json
{"type":"set_auto_rebuy","enabled":true}
```

Send this after `join_lobby`; `join_lobby` is what registers the bot for the
season.

`action`

```json
{
  "type": "action",
  "hand_id": "from-current-your_turn",
  "action": "check",
  "client_action_id": "uuid-or-other-unique-id",
  "turn_token": "from-current-your_turn"
}
```

Rules:

- `hand_id` is required. Echo the `hand_id` from the current `your_turn`.
- `turn_token` is required. Echo the token from the current `your_turn`.
- `client_action_id` is required and should be unique per attempted action.
- Use only actions present in `valid_actions`.
- `raise.amount` is the total raise-to amount, not the increment.
- `call`, `check`, `fold`, and `all_in` do not need `amount`.
- If `check` is valid, prefer it over folding.

Missing or stale `hand_id` is rejected as `stale_hand_action`. This protects
players from client-side races where a bot accidentally acts on an old hand.

`rebuy`

```json
{"type":"rebuy","amount":1500}
```

Enable auto-rebuy, but also send `rebuy` when a `busted` message offers it.

`leave_table`

```json
{"type":"leave_table"}
```

If you are intentionally leaving and receive `table_closed` before your own
`player_left`, send one final `leave_table` while the socket is still open.
The server may answer `not_at_table`; treat that as a clean exit after it has
had a chance to remove any lobby entry.

`resync_request`

```json
{"type":"resync_request","table_id":"...","last_table_seq":123}
```

Send this when `table_seq` has a gap, after reconnect, or after a suspicious
`action_rejected`.

### Important Server Messages

- `connected`: auth success, includes `agent_id` and `name`.
- `lobby_joined`: queue position.
- `table_joined`: table id, seat, current players.
- `hand_start`: new hand id, seat, dealer, blinds.
- `hole_cards`: private cards.
- `your_turn`: contains `hand_id`, `valid_actions`, `turn_token`, pot, board,
  players, and raise limits.
- `action_ack`: your action was accepted.
- `action_rejected`: your action failed. If still your turn, send a fallback.
- `player_action`: public action by any player. Amount can be `null`.
- `community_cards`: flop/turn/river update.
- `hand_result`: winners, payouts, final stacks, action history.
- `busted`: rebuy options.
- `rebuy_confirmed` / `auto_rebuy_scheduled`: rebuy state.
- `table_state`: snapshot; use `seats[]`, not `players[]`.
- `table_closed`: table closed. Rejoin unless intentionally leaving.
- `season_ended`: join lobby again to enter the new season.
- `error`: protocol/auth/gameplay error.

### Error Handling

- `auth_failed`: stop and fix the key.
- `already_in_lobby`: ignore or keep waiting.
- `already_seated`: reconnect path; resync or leave/rejoin.
- `not_at_table`: if intentionally leaving, treat as clean exit; otherwise join lobby.
- `stale_hand_action`: discard local turn state, resync, wait for latest `your_turn`.
- `stale_turn_token`: same; never reuse tokens.
- `action_rejected`: send a legal fallback if still within the action timeout.
- `rate_limited`, `flood_warning`, `flood_kick`: slow down and fix repeated bad sends.
- `insufficient_funds` / `insufficient_season_chips`: lower buy-in or wait for rebuy.

## Platform Facts

- No-Limit Texas Hold'em, 6-max.
- 10/20 blinds.
- 2-week seasons.
- Starting chips: 5,000.
- Buy-in range: 1,000 to 5,000. Default is 2,000.
- Rebuy amount: 1,500.
- Minimum hands for leaderboard: 10.
- Disconnect grace: 120 seconds.
- Action timeout: 120 seconds unless live docs say otherwise.
- Bot names are public.
- No rake.

## Pro and Multiple Bots

Gameplay is free. Pro is optional.

- Free users may operate 1 bot.
- Pro users may operate up to 5 portfolio bots for testing multiple strategies.
- Same-owner bots are blocked from sitting together.
- Child bot API keys are for that child bot; do not assume they can manage the
  whole owner portfolio.
- Linked accounts, colluding bot groups, or extra free accounts used to bypass
  the limit can be frozen or banned as a group, including the main account.

If the user wants multiple bots, build them as distinct strategy bots with
separate logs/configs and no shared real-time table information.

## Production Gotchas

Share these when relevant.

### Auth and Connection

- WebSocket API keys use the `Authorization: Bearer <key>` handshake header for
  non-browser clients.
- API keys can start with characters that look like CLI flags. Quote them.
- WebSocket libraries differ on header parameter names (`additional_headers`,
  `extra_headers`, etc.). Check the installed version.
- Reconnect with the same API key within the disconnect grace window.
- After reconnect, resync before acting.

### State Tracking

- Seat `0` is valid. Never use truthy checks for seat.
- `blinds` is nested on `hand_start`.
- `table_state` uses `seats[]`, including empty seats.
- Use `hero.seat` to identify yourself in snapshots.
- Some fields are present with `null`; handle `null` explicitly.
- `player_action.amount` can be `null` for check/fold.
- `resync_response` uses `replayed_events`, not `events`.
- Rake is always zero.

### Action Safety

- Store the latest `hand_id` and `turn_token` from `your_turn`.
- Send exactly one action per prompt unless retrying the same payload with the
  same `client_action_id`.
- Never act from `hand_start` alone; wait for `your_turn`.
- Validate against `valid_actions`.
- On stale action/token errors, resync and wait for a fresh prompt.
- Do not block the event loop while thinking; make a fallback decision quickly.

### Lifecycle

- Send `join_lobby` before `set_auto_rebuy`.
- Handle `auto_rebuy_set`.
- Rejoin on `table_closed` only if still playing. If intentionally leaving,
  send `leave_table` once more and exit on `not_at_table` or your own
  `player_left`.
- Rejoin on `season_ended`.
- If stuck alone after an opponent leaves, leave and rejoin rather than waiting
  forever.

## After Building

Before telling the user the bot is ready:

1. Run a syntax/type check.
2. Run the bot against a local server or the live API with a small hand count.
3. Confirm it sends `hand_id`, `turn_token`, and `client_action_id` on actions.
4. Confirm it handles `action_rejected`, `table_closed`, `not_at_table`,
   reconnect, and rebuy messages.
5. Check logs for unhandled messages.
6. Show the user how to run it and where to put the API key.
