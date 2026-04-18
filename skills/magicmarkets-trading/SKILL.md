---
name: magicmarkets-trading
description: >
  MagicMarkets Trading API assistant — programmatic sports bet placement and
  order management at api.magicmarkets.com. Supports back bets (normal), lay
  bets, and parlays (accumulators) with aggregated bookmaker liquidity. Helps
  external developers build trading bots, manage orders, and integrate the full
  betslip-to-settlement workflow using long-lived API tokens. Use this skill
  whenever the user mentions MagicMarkets trading, placing bets programmatically,
  back/lay betting, betslips, orders, parlays, API tokens, USDT stakes, profit/
  loss tracking, or wants to build a trading bot for sports betting. Also
  trigger when the user asks about api.magicmarkets.com, X-Api-Key auth, order
  monitoring, bet settlement, exchange modes (make/take), or in-play trading.
allowed-tools: Bash(curl:*), Bash(python3:*), Bash(cat:*), Bash(which:*)
---

# MagicMarkets Trading API

You are helping an external developer integrate the MagicMarkets Trading API —
a programmatic interface for placing and managing sports bets across aggregated
bookmaker liquidity. All prices appear as a single consolidated view with stakes
in USDT; individual bookmaker identities are abstracted away.

| Detail | Value |
|--------|-------|
| **Base URL** | `https://api.magicmarkets.com` |
| **Auth** | `X-Api-Key: <token>` header (long-lived, up to 365 days) |
| **Stake format (responses)** | `["USDT", <amount>]` tuples |
| **Bookie abstraction** | Bookmaker fields stripped from all responses |

For full field-by-field documentation, read `references/trading-reference.md` in
this skill directory.

---

## Two orthogonal dimensions: selection side × order type

Every order has two independent dimensions a developer must understand:

### Dimension 1: Selection side (in `bet_type`)

Every `bet_type` string starts with either `for,...` or `against,...` — this
is which **side of the market** you're picking:

| Prefix | Meaning |
|--------|---------|
| `for,...` | You're picking the outcome to happen (e.g. `for,mres,1` = home wins) |
| `against,...` | You're picking the outcome NOT to happen (e.g. `against,win,2` = runner 2 won't win) |

`against` is particularly common in multirunner markets (outrights, futures)
where you bet that a specific runner will NOT win the whole thing.

### Dimension 2: Order type (`betslip_type` → `order_type`)

| Value | What it means | When to use |
|-------|--------------|-------------|
| `normal` | **Back** the selection — classic bet; win if the selection happens | Most common — you think it'll happen |
| `lay` | **Lay** the selection — act as bookmaker; win if the selection does NOT happen | Exchange-style trading |
| `parlay` | **Accumulator** — combine 2-10 legs; all must win | Stacking multiple back bets |

### Putting them together

The same market can be expressed four different ways:

| Combination | Example | Effect |
|-------------|---------|--------|
| `normal` + `for,...` | Back home to win | Classic back bet on home |
| `normal` + `against,...` | Back home NOT to win | Classic back bet on "not home" |
| `lay` + `for,...` | Lay home to win | Exchange lay on home winning |
| `lay` + `against,...` | Lay home NOT to win | Exchange lay on "not home" |

Economically, `normal + for` ≈ `lay + against`, but they route through different
liquidity pools on the platform, so the prices and fills can differ.

**A back at 2.00 for 10 USDT** pays 20 USDT if the selection wins (profit +10).

**A lay at 2.00 for 10 USDT** wins 10 USDT if the selection loses; loses
10 × (2.00 − 1) = 10 USDT if it wins.

---

## Core concepts

### USDT stake tuples

All monetary amounts in responses are `["USDT", <number>]` tuples, **never**
plain numbers:

```json
"want_stake": ["USDT", 3.0],
"got_stake": ["USDT", 2.99],
"profit_loss": ["USDT", -2.99]
```

Unsettled values are `null` instead of the tuple. When reading stakes or P&L:
1. Check for `null` first (unsettled)
2. Access index `[1]` for the numeric amount
3. Index `[0]` is always the string `"USDT"`

**Request stake can be any currency.** When placing orders, you can send
`stake` as `["EUR", 100.0]`, `["GBP", 50.0]`, or `["USDT", 75.0]` — the system
converts for you. Responses always come back in USDT.

### Bookie abstraction

The API strips **all** bookmaker-identifying information. Developers see
aggregated prices and fills but never which bookmaker serviced a bet. Fields like
`bookie`, `username`, `bookie_bet_id` do not appear in responses.

### Response envelope

Every response uses:

```json
{"status": "ok", "data": <payload>}
```

### Event ID format

All endpoints that reference an event take a string `event_id` with one of
two shapes:

**Head-to-head events:**

```
YYYY-MM-DD,<home_team_id>,<away_team_id>
```

Example: `"2026-04-15,85,89"` = a match on 2026-04-15 between team 85 (home)
and team 89 (away). The integer team IDs are stable across calls but are
**not enumerated** by the Trading API — there is no `/v1/events/search/`
endpoint.

**Multirunner events** (outrights, futures):

```
YYYY-MM-DD,multirunner,<event_id>
```

Example: `"2025-08-28,multirunner,47"` = a season-long outright. The
`<event_id>` is an internal numeric ID.

**Discovering event IDs and team IDs.** Since there's no search endpoint,
you have three options:

1. Subscribe to the [Datafeed](https://data.magicmarkets.com) — event
   upserts carry `home`/`away` names alongside the `[sport, event_id]` key.
2. Inspect existing orders — `event_info.home_id` / `event_info.away_id`
   fields are populated in `GET /v1/orders/` responses.
3. Use the known `event_id` if you already have it from another part of
   your pipeline.

**Matching events to external platforms.** The integer team IDs are
MagicMarkets-internal and have no external equivalent. If you're matching
against Polymarket, Odds API, etc., match on `(home_team, away_team, date)`
with fuzzy name matching — not on the integer IDs.

---

## Authentication

All Trading API requests use a long-lived API token:

```
GET /v1/orders/ HTTP/1.1
Host: api.magicmarkets.com
X-Api-Key: <token>
```

Tokens are valid for up to **365 days** and created via the token management
endpoints (which require browser session auth, not the API key).

### Token management (browser session auth)

These endpoints use `session` + `magic-metadata-jwt` headers from the account
dashboard, **not** the `X-Api-Key`:

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/web/token/` | Create token (max 16 per account) |
| `GET` | `/web/token/` | List tokens (values masked) |
| `PATCH` | `/web/token/{id}/` | Update name or expiry |
| `DELETE` | `/web/token/{id}/` | Soft-delete token |

The `token` field in the create response is plaintext — **shown only once**.

---

## Betslip lifecycle and rate limits

Before writing any batch workflow against this API, understand that
betslips are **scarce, server-managed resources** with two hard
constraints:

### Concurrent betslip cap

Each account has a cap on **concurrent open betslips** — exceed it and
`POST /v1/betslips/` returns `validation_error`. The exact number is
account-specific. Discover yours at runtime rather than hard-coding:

```bash
curl "https://api.magicmarkets.com/v1/betslips/" \
  -H "X-Api-Key: $API_KEY"
```

This returns a list of `betslip_id`s currently open. If the list is at
your cap, new creates will fail until one expires.

### No client-side close endpoint

There is **no `DELETE /v1/betslips/{id}/`** (and no PATCH, no explicit
cancel endpoint — the entire magic-api spec is GET/POST only). Slots free
only on server-side auto-expiry, which is typically a short window (a few
minutes). Read the `expiry_ts` field on the response rather than
hard-coding a timeout — it's the authoritative freeing time.

### Implications for batch workflows

| Your workflow | Recommendation |
|---------------|----------------|
| **Enriching arb candidates with live depth** | Batch well under your cap. Wait for `expiry_ts` before the next batch. |
| **Periodic market scanning** | Use the **Datafeed** (`wss://data.magicmarkets.com`) instead. The Trading API betslip endpoint is for single-market pre-trade checks, not surveillance. |
| **Retrying on transient failures** | Be aware a failed `POST /v1/betslips/` may still have consumed a slot you can't release. Prefer idempotency via caching your latest successful `betslip_id` over blind retries. |

### When to pick the Datafeed over a betslip

| You need | Use |
|----------|-----|
| Best decimal odds across sources, single event | Betslip |
| Same, for a handful of events | Betslip (mind the cap) |
| Same, for many events / continuous surveillance | Datafeed |
| Max fillable stake at quoted price | Betslip (only source exposing `max` per price level) |
| Full order-book depth | **Not exposed** by this API — see meta-note below |

### Meta-note on depth

The `price_list` array in a betslip response returns aggregated price
levels sorted descending, but the granularity is coarse (typically 1–3
levels) and `max` on each level reflects a per-source effective cap, not
aggregate book depth at that price. If you need deeper order-book data,
this API does not currently expose it. Treat `price_list` as "what's
fillable for a single-shot order" rather than "book depth".

---

## Trading workflow

The typical flow: **create betslip → review prices → place order → monitor**.

### Step 1: Create a betslip

Normal (back) bet:

```bash
curl -X POST https://api.magicmarkets.com/v1/betslips/ \
  -H "X-Api-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "sport": "fb",
    "event_id": "2026-04-15,85,89",
    "bet_type": "for,1x2,1",
    "betslip_type": "normal"
  }'
```

Lay bet (same selection, but laying against it):

```bash
curl -X POST https://api.magicmarkets.com/v1/betslips/ \
  -H "X-Api-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "sport": "fb",
    "event_id": "2026-04-15,85,89",
    "bet_type": "for,1x2,1",
    "betslip_type": "lay"
  }'
```

Parlay (2-10 legs):

```bash
curl -X POST https://api.magicmarkets.com/v1/betslips/ \
  -H "X-Api-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "betslip_type": "parlay",
    "legs": [
      {"sport": "fb", "event_id": "2026-04-15,85,89", "bet_type": "for,1x2,1"},
      {"sport": "ih", "event_id": "2026-04-15,100,200", "bet_type": "for,h"}
    ]
  }'
```

**BetslipCreateRequest fields:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `betslip_type` | string | `"normal"` | `"normal"`, `"lay"`, or `"parlay"` |
| `sport` | string | — | Required for normal/lay; null for parlay |
| `event_id` | string | — | Required for normal/lay; null for parlay |
| `bet_type` | string | — | Required for normal/lay; null for parlay |
| `legs` | array | — | Required for parlay (2-10 BetslipLeg objects); null otherwise |
| `equivalent_bets` | boolean | `true` | Include equivalent bets across bookies |
| `exclude_danger` | boolean | `false` | Skip bookies holding bets in "danger" status |
| `user_data` | string | `null` | Optional customer reference (max 512 chars) |

**Betslip response** includes `price_list` with **nested `effective`** wrapper:

```json
{
  "status": "ok",
  "data": {
    "betslip_id": "abc123",
    "sport": "fb",
    "event_id": "2026-04-12,85,89",
    "bet_type": "for,h",
    "bet_type_description": "Sunderland FC",
    "expiry_ts": 1776218400.0,
    "is_open": true,
    "close_reason": null,
    "betslip_type": "normal",
    "equivalent_bets": true,
    "customer_username": "alice",
    "customer_ccy": "GBP",
    "price_list": [
      {"effective": {"price": 1.56, "min": null, "max": ["USDT", 100.0]}},
      {"effective": {"price": 1.54, "min": null, "max": ["USDT", 250.0]}}
    ],
    "total": ["USDT", 350.0],
    "user_data": null
  }
}
```

**Critical: parsing `price_list` correctly.** Three gotchas bite every
integrator at least once:

1. **Nested wrapper.** Each entry is `{"effective": {...}}`, not a flat
   price object. Read `entry["effective"]["price"]` — `entry["price"]`
   is `undefined`.
2. **`max` is a USDT tuple** (not a number). It's `["USDT", <float>]` or
   `null`. Always null-check before indexing:

   ```python
   mx = entry["effective"]["max"]
   max_usdt = mx[1] if isinstance(mx, (list, tuple)) and len(mx) >= 2 else None
   ```

3. **Empty `price_list` is a valid success state.** An empty array with
   `status: "ok"` does **not** mean the API failed — it means this
   market has no live offer on this side right now. Common causes:
   one-sided book on `betslip_type: "lay"`, suspended market, or the
   wrong `bet_type` for the sport (see Bet type patterns below). Handle
   this explicitly; don't crash and don't treat it as an error.

The list is sorted by price descending — the best available odds first.

**Aggregating fillable stake above a price floor.** Because the list is
sorted descending, you can break early:

```python
total_max = 0.0
for entry in slip["price_list"]:
    eff = entry.get("effective") or {}
    if eff.get("price", 0) < min_acceptable_price:
        break
    mx = eff.get("max")
    if isinstance(mx, (list, tuple)) and len(mx) >= 2 and mx[0] == "USDT":
        total_max += float(mx[1])
```

A `total_max == 0.0` result means "the betslip succeeded but no level met
your price threshold" — a common, legitimate state.

**Query params on `POST /v1/betslips/`:**

- `display_net_prices` (boolean) — show net prices after commission
- `obfuscate_accounts` (boolean) — hide bookie account usernames (always effective)

### Step 2: Place an order

```bash
curl -X POST https://api.magicmarkets.com/v1/orders/ \
  -H "X-Api-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "betslip_id": "abc123",
    "price": 1.56,
    "stake": ["USDT", 50.0],
    "duration": 5
  }'
```

**OrderCreateRequest fields** (this is where most developers get tripped up):

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `betslip_id` | string | **yes** | — | Betslip to place the order on |
| `price` | number | **yes** | — | Desired price (decimal odds) |
| `stake` | `[ccy, amt]` | **yes** | — | Stake as `[currency, amount]`, e.g. `["USDT", 50.0]` |
| `duration` | number | **yes** | — | Order duration in minutes |
| `exchange_mode` | string | no | `"make_and_take"` | `"make_and_take"`, `"make"`, or `"take"` |
| `keep_open_ir` | boolean | no | `false` | Keep order open in-running (in-play) |
| `accept_partial_fill` | boolean | no | `true` | Accept partial fills |
| `accept_better_price` | boolean | no | `true` | Accept prices better than requested |
| `force_want_price` | boolean | no | `false` | Force exact `price`, reject better |
| `min_taker_want_stake` | `[ccy, amt]` | no | `null` | Minimum taker stake |
| `current_score` | string | no | `null` | Current score for in-play orders |
| `user_data` | string | no | `null` | Optional customer reference |
| `request_uuid` | string | no | `null` | **Idempotency key** — use this to safely retry |

**Exchange modes** (the maker/taker dimension):

- `"make_and_take"` — default; you'll take available liquidity first, then post a
  making offer for the remainder
- `"make"` — post a making offer only (you wait to be matched)
- `"take"` — take existing liquidity only (fail if none at your price)

**`force_want_price` vs `accept_better_price`**: by default the system fills you
at better prices if available. `force_want_price: true` means "only match at exactly
this price, not better, not worse." Useful for precise hedging.

### Step 3: Monitor orders

```bash
# List orders (paginated)
curl "https://api.magicmarkets.com/v1/orders/?page=1&page_size=25" \
  -H "X-Api-Key: $API_KEY"

# Filter by order type (back/lay/parlay)
curl "https://api.magicmarkets.com/v1/orders/?order_type=lay&status=done" \
  -H "X-Api-Key: $API_KEY"

# Single order
curl "https://api.magicmarkets.com/v1/orders/1472877357/" \
  -H "X-Api-Key: $API_KEY"

# Poll recent updates (both timestamps required, >= 60s in past)
curl "https://api.magicmarkets.com/v1/orders/updates/?updated_at_from=2026-04-14T00:00:00Z&updated_at_to=2026-04-14T12:00:00Z" \
  -H "X-Api-Key: $API_KEY"
```

**Filters on `GET /v1/orders/`:**

| Param | Values | Description |
|-------|--------|-------------|
| `page` / `page_size` | integers | Pagination (page_size max 1000) |
| `status` | `open`, `pending`, `done`, `reconciled`, `failed`, `full_void` | Order lifecycle state |
| `sport` | sport codes | e.g. `fb`, `basket`, `tennis`, `cricket` |
| `event_id` | event IDs | `YYYY-MM-DD,home_id,away_id` or `YYYY-MM-DD,multirunner,<id>` |
| `order_type` | `normal`, `lay`, `parlay` | Filter by back/lay/parlay |
| `date_from` / `date_to` | ISO datetime | Placement time range |
| `search` | string | Free text search |

### Order lifecycle states

| Status | Meaning |
|--------|---------|
| `open` | Order is live and matching |
| `pending` | Placement in progress |
| `done` | Bets filled, waiting for settlement |
| `reconciled` | Bets settled with final P&L computed |
| `failed` | All placement attempts failed (check `bets[].status.code`) |
| `full_void` | Order was voided entirely (e.g. event cancelled) |

A typical successful order goes `open` → `done` → `reconciled`. The `profit_loss`
field is populated only at `reconciled`.

---

## Key order fields

| Field | Type | Description |
|-------|------|-------------|
| `order_id` | integer | Unique order ID |
| `order_type` | string | `normal` (back), `lay`, `parlay` |
| `bet_type` | string | e.g. `for,tp,all,ml,a` |
| `bet_type_description` | string | Human-readable |
| `sport` | string | Sport code |
| `placer` | string | Customer username |
| `want_price` | number | Requested decimal odds |
| `want_stake` | `["USDT", n]` or null | Requested stake |
| `status` | string | `open`, `pending`, `done`, `failed` |
| `closed` | boolean | Whether closed |
| `close_reason` | string/null | `order_filled`, `timed_out`, `cancelled`, etc. |
| `ccy_rate` | number | Exchange rate vs GBP at order time |
| `placement_time` | ISO datetime | When placed |
| `expiry_time` | ISO datetime | When it expires |
| `price` | number/null | Matched/achieved price (null if no fills) |
| `stake` | `["USDT", n]`/null | Aggregate matched stake |
| `profit_loss` | `["USDT", n]`/null | P&L (null if unsettled) |
| `exchange_mode` | string/null | `make_and_take`, `make`, `take` |
| `keep_open_ir` | boolean | In-running flag |
| `event_info` | object | Event metadata (see below) |
| `bets` | array | Individual fills |
| `legs` | array/null | Per-leg details for parlays (see below) |
| `bet_bar_values` | object/null | Internal display values |
| `user_data` | string/null | Customer reference |

### Bet fields (within `bets` array)

| Field | Type | Description |
|-------|------|-------------|
| `bet_id` | integer | Unique bet ID |
| `order_id` | integer | Parent order ID |
| `status` | `{code, response_pmm}` | Status object. `code` values include `done`, `settled`, `failed`, `pending` |
| `got_price` | number/null | Filled price |
| `want_stake` | `["USDT", n]`/null | Requested stake |
| `got_stake` | `["USDT", n]`/null | Filled stake |
| `profit_loss` | `["USDT", n]`/null | P&L |
| `reconciled` | boolean/null | `true` once bet is reconciled, null before |
| `exchange_role` | `maker`/`taker`/null | Exchange role (null for non-exchange fills) |
| `ccy_rate` / `order_ccy_rate` | number | FX rates vs GBP |
| `legs` | array/null | Per-leg details for parlays |

### Parlay legs

For `order_type: "parlay"`, the order has a `legs` array:

```json
{
  "id": 1,
  "sport": "fb",
  "event_id": "2026-04-15,85,89",
  "bet_type": "for,1x2,1",
  "bet_type_description": "Liverpool",
  "price": 2.10,
  "outcome": "won"
}
```

**`outcome` values:** `"won"`, `"lost"`, `"void"`, `"push"`, or null (unsettled)

---

## Event info variants

Orders include `event_info` with one of three shapes determined by `event_type`:

### Normal events (`event_type: "normal"`)

Standard matches — football, ice hockey, tennis, etc.

```json
{
  "event_type": "normal",
  "event_id": "2026-04-14,29060,29064",
  "event_name": "Philadelphia Flyers vs. Montreal Canadiens",
  "home_id": 29060, "home_team": "Philadelphia Flyers",
  "away_id": 29064, "away_team": "Montreal Canadiens",
  "competition_id": 1294, "competition_name": "USA NHL",
  "competition_country": "US",
  "start_time": "2026-04-14T23:00:00+00:00",
  "date": "2026-04-14",
  "result": { "tp1_home": 2, "tp1_away": 0, "tall_home": 4, "tall_away": 2 }
}
```

### Multirunner events (`event_type: "multirunner"`)

Outright/futures markets — title winners, season totals, tournament placement:

```json
{
  "event_type": "multirunner",
  "event_name": "UEFA - Champions League 2025/26",
  "teams": [{"team_id": 178, "name": "Real Madrid"}],
  "start_time": "2025-08-28T00:00:00+00:00",
  "end_time": "2026-05-30T23:00:00+00:00",
  "result": {
    "runner_results": [{"team_id": 178, "position": 1}],
    "non_runner_count": 0
  }
}
```

**Runner position values:** 0=unknown, 1=first, 2=second, -1=void,
-2=non-runner, -3=eliminated

### Parlay events (`event_type: "parlay"`)

Accumulators — `event_id` is null at the top level; `leg_event_infos` array
contains one nested EventInfo per leg (each of which can be normal or multirunner).

### Event result fields by sport

| Sport | Fields |
|-------|--------|
| Football | `ht_home`, `ht_away`, `ft_home`, `ft_away` |
| Ice hockey | `tp1_*`, `tp2_*`, `tp3_*`, `tall_*`, `pen_*` |
| Multirunner | `runner_results` array, `non_runner_count` |

Scores are null until the match starts; intermediate values populate as the
match progresses.

---

## Other endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/betslips/` | List open betslips (IDs only) |
| `GET` | `/v1/betslips/{id}/` | Get single betslip with full price list |
| `GET` | `/v1/orders/{id}/` | Get single order |
| `GET` | `/web/offerhist/{sport}/{event_id}/{bet_type}/` | Historical price time series |

### Blocked endpoints (return 404)

These exist in the underlying system but are **intentionally blocked** through
the public Trading API for bookie abstraction:

- `/v1/matrix/bookies/` — bookie list
- `/v1/matrix/accounts/` — account matrix
- `/v1/customers/{username}/bookie_accounts/`
- `/v1/customers/{username}/bookie_balances/`
- `/v1/groups/{group_id}/bookie_accounts/`

If a developer asks for these, explain they're blocked by design — the whole
point of the Trading API is to abstract bookmaker identities away.

---

## Code examples

### Python — read a betslip correctly (with nested `effective`)

```python
import os, requests

API_KEY = os.environ["MM_API_KEY"]
BASE = "https://api.magicmarkets.com"
H = {"X-Api-Key": API_KEY, "Content-Type": "application/json"}

slip = requests.post(f"{BASE}/v1/betslips/", headers=H, json={
    "sport": "fb",
    "event_id": "2026-04-15,85,89",
    "bet_type": "for,1x2,1",
    "betslip_type": "normal"  # or "lay"
}).json()["data"]

print(f"{slip['bet_type_description']} — {slip['betslip_type']}")
for entry in slip["price_list"]:
    eff = entry["effective"]        # <-- nested wrapper!
    price = eff["price"]
    max_stake = eff["max"][1] if eff["max"] else None
    print(f"  {price:.3f} — max {max_stake} USDT")
```

### Python — place a back bet with idempotency

```python
import uuid, requests

slip_id = "abc123"
req_uuid = str(uuid.uuid4())  # save this; retry with same UUID is safe

order = requests.post(f"{BASE}/v1/orders/", headers=H, json={
    "betslip_id": slip_id,
    "price": 2.10,
    "stake": ["USDT", 25.0],
    "duration": 5,                   # minutes
    "exchange_mode": "make_and_take",
    "accept_partial_fill": True,
    "accept_better_price": True,
    "request_uuid": req_uuid,
}).json()["data"]

print(f"Order {order['order_id']}: {order['status']}")
```

### Python — list recent losing bets across back, lay, and parlay

```python
resp = requests.get(f"{BASE}/v1/orders/", headers=H,
                    params={"page_size": 100, "status": "done"})
orders = resp.json()["data"]

losses_by_type = {"normal": [], "lay": [], "parlay": []}
for o in orders:
    pnl = o["profit_loss"]
    if pnl is None: continue     # unsettled
    if pnl[1] < 0:
        losses_by_type[o["order_type"]].append(o)

for ot, losses in losses_by_type.items():
    total = sum(o["profit_loss"][1] for o in losses)
    print(f"{ot}: {len(losses)} losses, total {total:+.2f} USDT")
```

### Python — parlay with leg outcomes

```python
slip = requests.post(f"{BASE}/v1/betslips/", headers=H, json={
    "betslip_type": "parlay",
    "legs": [
        {"sport": "fb", "event_id": "2026-04-15,85,89", "bet_type": "for,1x2,1"},
        {"sport": "ih", "event_id": "2026-04-15,100,200", "bet_type": "for,h"},
    ],
}).json()["data"]

# ... place order, later retrieve settled result ...

order = requests.get(f"{BASE}/v1/orders/{order_id}/",
                     headers=H).json()["data"]
for leg in order["legs"]:
    print(f"Leg {leg['id']}: {leg['bet_type_description']} → {leg['outcome']}")
```

---

## Reference tables

### Sport codes

| Code | Sport | Code | Sport |
|------|-------|------|-------|
| `fb` | Football (soccer) | `tennis` | Tennis |
| `fb_ht` | Football - 1st half | `af` | American football |
| `basket` | Basketball | `ih` | Ice hockey |
| `baseball` | Baseball | `mma` | MMA |
| `cricket` | Cricket | `rl` | Rugby league |
| `ru` | Rugby union | `darts` | Darts |
| `hand` | Handball | `snooker` | Snooker |
| `boxing` | Boxing | `volley` | Volleyball |
| `arf` | Australian rules | | |

### Bet type patterns

Every bet_type starts with `for,` or `against,` — see the "Two orthogonal
dimensions" section above. Common patterns observed in production:

**Match result / moneyline — use the exact pattern for your sport.**

The moneyline pattern varies by sport. Using the wrong one produces an
empty `price_list` silently — always log which `bet_type` you sent if you
get an unexpectedly empty result.

| Sport | Home | Away | Draw | Notes |
|-------|------|------|------|-------|
| `fb` (soccer) | `for,h` | `for,a` | `for,d` | 3-way; draw required |
| `basket` | `for,ml,h` | `for,ml,a` | — | 2-way, no draw |
| `ih` (ice hockey) | `for,tp,all,ml,h` | `for,tp,all,ml,a` | — | Incl. OT + shootout |
| `af` (American football) | `for,tp,all,ml,h` | `for,tp,all,ml,a` | — | Incl. OT |
| `baseball` | `for,tp,all,ml,h` | `for,tp,all,ml,a` | — | Incl. extra innings |
| `mma` | `for,ml,h` | `for,ml,a` | — | DNB variant: `for,dnb,h/a` |
| `boxing` | `for,h` | `for,a` | `for,d` | DNB variant: `for,dnb,h/a` |
| `tennis` | `for,tset,all,vset1,p1` | `for,tset,all,vset1,p2` | — | "Win total sets vs set 1, player N" |

**Regulation-time variants.** For hockey / football / basketball, swap
`tp,all,` → `tp,reg,` to restrict to regulation time (ignoring OT). You
may also see `tp,reg,wdw,h/d/a` — regulation-time 3-way (win-draw-win).
Markets that price regular-time and full-time separately will carry both.

**Lay side.** Replace the `for,` prefix with `against,` in any pattern.
So laying the home team in NBA is `against,ml,h`.

**Asian handicap and over/under — quarter-point line encoding.**

| Pattern | Meaning |
|---------|---------|
| `for,ah,h,-20` | Asian handicap home **-5.0** (line × 4) |
| `for,ah,a,20` | Asian handicap away **+5.0** |
| `for,ou,o,25` | Over (line × 4 — `25` = 6.25) |
| `for,ou,u,25` | Under |
| `for,ahover,N` | Combined AH-over (common in `fb_corn`, other composites) |
| `for,ahunder,N` | Combined AH-under |

AH and O/U lines are integers in **quarter-point units** — divide by 4 to
get the displayed line. So `-20` is shown as `-5.0`, `-8` is `-2.0`,
`14` is `+3.5`, etc.

**Outrights / futures (multirunner markets):**

| Pattern | Meaning |
|---------|---------|
| `for,win,<team_id>` | Runner wins outright |
| `against,win,<team_id>` | Runner does NOT win outright |

Used in season-long competitions where many runners compete. The `<team_id>`
matches the runner's `team_id` in the multirunner EventInfo.

**Other markets:**

| Pattern | Meaning |
|---------|---------|
| `for,score,both` | Both teams to score |
| `for,tset,all,vwhatever,p1` | Tennis total sets, player 1 |
| `for,ir,1,1,ahover,14` | In-running composite (period, market, line) |

The exact set of available bet types depends on sport and market. If you
receive a bet_type you don't recognise, read the `bet_type_description` — it's
always human-readable.

---

## Close reasons

When an order closes, `close_reason` explains why:

| Value | Meaning |
|-------|---------|
| `order_filled` | Bet(s) placed successfully for the full stake |
| `timed_out` | Order duration elapsed before full fill |
| `cancelled` | Manually cancelled |
| `event_score` | Closed because a goal/point was scored (pre-match orders on IR-suspended events) |
| `event_became_inrunning` | Closed because the event went in-play (without `keep_open_ir`) |
| `event_concluded` | Event finished before match |
| `no_available_credit` | Out of credit across available bookmakers |

## Troubleshooting

| Symptom | Likely cause |
|---------|-------------|
| `401 Unauthorized` | Token expired, cleared, or wrong header (must be `X-Api-Key`) |
| `404` on `/web/token/` | Token management requires browser session auth, not API key |
| `404` on `/v1/matrix/*` or `/bookie_*` | Intentionally blocked — bookie abstraction |
| Empty `price_list` | Market suspended, no bookmaker coverage, or no liquidity on that side (back/lay) |
| `null` `profit_loss` | Bet not yet settled — order status will be `done` or earlier, not `reconciled` |
| `null` `stake` on order | No bets filled (order timed out or was cancelled) |
| `status: "failed"` | All placements failed — check `bets[].status.code` |
| `updated_at_from` error | Both timestamps must be >= 60 seconds in the past |
| Reading `price_list[0]["price"]` returns undefined | It's nested — use `price_list[0]["effective"]["price"]` |
| `max` field is not a number when you try to do math on it | It's a USDT tuple `["USDT", n]` — use `max[1]` (after null-check) |
| Empty `price_list` with `status: "ok"` | Valid state: one-sided book, suspended market, or wrong `bet_type` for the sport — treat as "no liquidity", not an error |
| `validation_error` on `POST /v1/betslips/` after several creates | You hit the concurrent cap — probe `GET /v1/betslips/` and wait for `expiry_ts` |
| Order created twice on retry | Use `request_uuid` as idempotency key |
| `validation_error: invalid_selection` on parlay | Each leg must be a different valid market |
| `validation_error: too_many_betslips` on parlay | Don't reuse in-flight selections across parlay legs |

## Offer history caveat

The `/web/offerhist/{sport}/{event_id}/{bet_type}/` endpoint currently proxies
the upstream data as-is. Today it returns per-source price history keyed by
anonymous source identifiers — a known exception to the bookie abstraction.
The roadmap notes aggregation into a single best-price series is planned, at
which point this endpoint will match the abstraction model used everywhere else.

---

## Known limitations

Call these out explicitly when evaluating fit for a use case — they bite
integrators who don't see them coming.

- **Concurrent open betslips are capped per account** — see the "Betslip
  lifecycle" section. There is **no close/delete endpoint**; slots free
  only on server-side auto-expiry. Batch workflows must pace themselves
  or use the Datafeed instead.
- **No order-book depth beyond `price_list`** — `price_list` returns 1–3
  aggregated levels with per-source `max`, not full aggregate book depth.
- **No event/market search endpoint** — `/v1/events/search/` doesn't
  exist. Discover event IDs via the Datafeed or from existing orders.
- **No client-editable state on betslips or orders** — the entire
  magic-api spec is GET/POST only. No PATCH, no DELETE, no cancellation
  endpoint. To stop an open order, wait for `expiry_time` or use
  `force_want_price` + price moves to avoid fills.
- **Offer history exposes per-source time series** — the
  `/web/offerhist/` endpoint currently proxies upstream data without
  aggregation. Aggregation into a single best-price series is planned.
- **Bet type grammar is not enumerable** — the canonical patterns are
  observed from live usage; there is no schema endpoint listing every
  valid `bet_type` string per sport. See the sport-keyed table above
  for the validated set.

---

## Live data queries

If the developer has an API key (check for `.env` or ask), you can make live
calls to demonstrate endpoints or debug issues. Always show the curl command
so the developer can reproduce it.
