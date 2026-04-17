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

## The three bet types: back, lay, parlay

MagicMarkets supports three fundamentally different kinds of bets, distinguished
by the `betslip_type` field (which becomes `order_type` once you place the order):

| `betslip_type` | What it means | When to use |
|---------------|--------------|-------------|
| `normal` | **Back bet** — you win if the outcome happens | You think Liverpool will win |
| `lay` | **Lay bet** — you act as the bookmaker; you win if the outcome does NOT happen | You think Liverpool will NOT win |
| `parlay` | **Accumulator** — combine 2-10 legs; all must win for payout | You want to stack multiple back bets |

**Back vs lay** is the core exchange-trading distinction:

- A **back** bet at odds 2.00 with stake 10 USDT pays 20 USDT if it wins (profit 10)
- A **lay** bet at odds 2.00 with stake 10 USDT means you accept 10 USDT if the outcome
  does not occur, but pay out 10 USDT (10 × (2.00 − 1)) if it does

The Trading API surfaces both back and lay liquidity for the same selection — when
you create a `normal` betslip you see the best prices to back; when you create a
`lay` betslip you see the best prices to lay.

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

**Important**: each price level is `{"effective": {...}}` — read
`entry["effective"]["price"]` not `entry["price"]`. Sorted by price descending.

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
| `status` | `open`, `pending`, `done`, `failed` | Order lifecycle state |
| `sport` | sport codes | e.g. `fb`, `basket`, `tennis` |
| `event_id` | event IDs | `YYYY-MM-DD,home_id,away_id` |
| `order_type` | `normal`, `lay`, `parlay` | Filter by back/lay/parlay |
| `date_from` / `date_to` | ISO datetime | Placement time range |
| `search` | string | Free text search |

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
| `status` | `{code, response_pmm}` | Status object with code (`success`, `failed`, `pending`) |
| `got_price` | number/null | Filled price |
| `want_stake` | `["USDT", n]` | Requested stake |
| `got_stake` | `["USDT", n]`/null | Filled stake |
| `profit_loss` | `["USDT", n]`/null | P&L |
| `reconciled` | string/null | Reconciliation state (string, not boolean) |
| `exchange_role` | `maker`/`taker`/null | Exchange role |
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
| `fb` | Football | `tennis` | Tennis |
| `fb_ht` | Football - 1st half | `af` | American football |
| `basket` | Basketball | `ih` | Ice hockey |
| `baseball` | Baseball | `mma` | MMA |
| `ru` | Rugby union | `rl` | Rugby league |
| `hand` | Handball | `darts` | Darts |
| `boxing` | Boxing | `snooker` | Snooker |
| `arf` | Australian rules | `volley` | Volleyball |

### Bet type patterns

| Pattern | Meaning |
|---------|---------|
| `for,1x2,1` | Home win |
| `for,1x2,x` | Draw |
| `for,1x2,2` | Away win |
| `for,ah,h,-1` | Asian handicap, home -1 |
| `for,ah,a,1` | Asian handicap, away +1 |
| `for,ou,o,2.5` | Over 2.5 |
| `for,ou,u,2.5` | Under 2.5 |
| `for,h` | Moneyline / head-to-head |
| `for,tp,all,ml,a` | Moneyline (inc. OT), away |
| `for,mres,1` | Match result, home |

---

## Troubleshooting

| Symptom | Likely cause |
|---------|-------------|
| `401 Unauthorized` | Token expired, cleared, or wrong header (must be `X-Api-Key`) |
| `404` on `/web/token/` | Token management requires browser session auth, not API key |
| `404` on `/v1/matrix/*` or `/bookie_*` | Intentionally blocked — bookie abstraction |
| Empty `price_list` | Market suspended or no bookmaker coverage |
| `null` `profit_loss` | Bet not yet settled |
| `null` `stake` on order | No bets filled (order timed out or was cancelled) |
| `status: "failed"` | All placements failed — check `bets[].status.code` |
| `updated_at_from` error | Both timestamps must be >= 60 seconds in the past |
| Reading `price_list[0]["price"]` returns undefined | It's nested — use `price_list[0]["effective"]["price"]` |
| Order created twice on retry | Use `request_uuid` as idempotency key |

---

## Live data queries

If the developer has an API key (check for `.env` or ask), you can make live
calls to demonstrate endpoints or debug issues. Always show the curl command
so the developer can reproduce it.
