---
name: magicmarkets-trading
description: >
  MagicMarkets Trading API assistant — programmatic sports bet placement and
  order management at api.magicmarkets.com. Helps external developers build
  trading bots, manage orders, and integrate the trading workflow using long-lived
  API tokens. Use this skill whenever the user mentions MagicMarkets trading,
  placing bets programmatically, betslips, orders, API tokens, USDT stakes,
  profit/loss tracking, or wants to build a trading bot for sports betting. Also
  trigger when the user asks about api.magicmarkets.com, X-Api-Key authentication,
  order monitoring, bet settlement, or exchange modes (make/take).
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
| **Stake format** | `["USDT", <amount>]` tuples everywhere |
| **Bookie abstraction** | Bookmaker fields stripped from all responses |

For full field-by-field documentation, read `references/trading-reference.md` in
this skill directory.

---

## Core concepts

### USDT stake tuples

All monetary amounts are `["USDT", <number>]` tuples, **never** plain numbers:

```json
"want_stake": ["USDT", 3.0],
"got_stake": ["USDT", 2.99],
"profit_loss": ["USDT", -2.99]
```

Unsettled values are `null` instead of the tuple. This is the most important data
format detail — getting it wrong breaks all parsing. When writing code that reads
stakes or P&L, always:
1. Check for `null` first (unsettled)
2. Access index `[1]` for the numeric amount
3. Index `[0]` is always the string `"USDT"`

### Bookie abstraction

The API strips **all** bookmaker-identifying information. Developers see
aggregated prices and fills but never which bookmaker serviced a bet. Fields like
`bookie`, `username`, `bookie_bet_id` do not appear in responses.

### Response envelope

Every response uses:

```json
{"status": "ok", "data": <payload>}
```

Where `<payload>` is an object (single resource) or array (list endpoints).

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

These endpoints use `Session` + `magic-metadata-jwt` headers from the account
dashboard, **not** the `X-Api-Key`:

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/web/token/` | Create token (max 16 per account) |
| `GET` | `/web/token/` | List tokens (values masked) |
| `PATCH` | `/web/token/{id}/` | Update name or expiry |
| `DELETE` | `/web/token/{id}/` | Soft-delete token |

**Create token request:**

```json
{
  "name": "my-trading-bot",
  "expires_at": "2027-03-01T00:00:00Z"
}
```

**Response (201):** The `token` field is the plaintext value — **shown only once**.

```json
{
  "status": "ok",
  "data": {
    "id": "f7476e90-ae37-4291-93fe-9be90f4cc9a1",
    "token": "o-cRxug6ipHTRKqOwllb1l_TChoO11UdTSBfgb8wHNc",
    "name": "my-trading-bot",
    "created_at": "2026-04-13T08:02:03.880741+00:00",
    "expires_at": "2027-04-13T08:02:03.859820+00:00",
    "cleared": false
  }
}
```

---

## Trading workflow

The typical flow: **create betslip → review prices → place order → monitor**.

### Step 1: Create a betslip

```bash
curl -X POST https://api.magicmarkets.com/v1/betslips/ \
  -H "X-Api-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{...}'
```

The response includes a `price_list` sorted by price descending:

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
    "customer_ccy": "GBP",
    "price_list": [
      {"price": 1.56, "min": null, "max": ["USDT", 100.0]},
      {"price": 1.54, "min": null, "max": ["USDT", 250.0]}
    ],
    "total": ["USDT", 350.0],
    "user_data": null
  }
}
```

Each price level: `price` (decimal odds), `min`/`max` (USDT tuple or null).
`total` sums all max stakes.

**Query params:** `display_net_prices` (boolean), `obfuscate_accounts` (boolean)

### Step 2: Place an order

```bash
curl -X POST https://api.magicmarkets.com/v1/orders/ \
  -H "X-Api-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "betslip_id": "abc123",
    "want_price": 1.56,
    "want_stake": ["USDT", 50.0]
  }'
```

### Step 3: Monitor orders

```bash
# List orders (paginated)
curl "https://api.magicmarkets.com/v1/orders/?page=1&page_size=25" \
  -H "X-Api-Key: $API_KEY"

# Single order
curl "https://api.magicmarkets.com/v1/orders/1472877357/" \
  -H "X-Api-Key: $API_KEY"

# Poll recent updates (both timestamps required, >= 60s in past)
curl "https://api.magicmarkets.com/v1/orders/updates/?updated_at_from=2026-04-14T00:00:00Z&updated_at_to=2026-04-14T12:00:00Z" \
  -H "X-Api-Key: $API_KEY"
```

---

## Orders endpoint

### GET `/v1/orders/` — List orders

| Param | Type | Description |
|-------|------|-------------|
| `page` | integer | Page number (default 1) |
| `page_size` | integer | Results per page (default 25, max 1000) |
| `status` | string[] | `open`, `pending`, `failed`, `reconciled` |
| `sport` | string[] | Filter by sport code |
| `event_id` | string[] | Filter by event ID |
| `order_type` | string[] | `normal`, `lay`, `parlay` |
| `date_from` | string | ISO datetime lower bound |
| `date_to` | string | ISO datetime upper bound |
| `search` | string | Search text |

### GET `/v1/orders/updates/` — Poll updates

| Param | Required | Description |
|-------|----------|-------------|
| `updated_at_from` | yes | ISO datetime (>= 60s in past) |
| `updated_at_to` | yes | ISO datetime (>= 60s in past) |
| `placer` | no | Filter by username |

---

## Key order fields

| Field | Type | Description |
|-------|------|-------------|
| `order_id` | integer | Unique order ID |
| `order_type` | string | `normal`, `lay`, `parlay` |
| `bet_type` | string | e.g. `for,tp,all,ml,a` |
| `bet_type_description` | string | Human-readable |
| `sport` | string | Sport code |
| `want_price` | number | Requested decimal odds |
| `want_stake` | `["USDT", n]` | Requested stake |
| `status` | string | `open`, `reconciled`, `failed`, `pending` |
| `closed` | boolean | Whether closed |
| `close_reason` | string/null | `order_filled`, `timed_out`, `cancelled` |
| `price` | number/null | Matched price (null if no fills) |
| `stake` | `["USDT", n]`/null | Matched stake |
| `profit_loss` | `["USDT", n]`/null | P&L (null if unsettled) |
| `exchange_mode` | string | `make_and_take`, `make`, `take` |
| `event_info` | object | Event metadata (see below) |
| `bets` | array | Individual fills |

### Bet fields (within `bets` array)

| Field | Type | Description |
|-------|------|-------------|
| `bet_id` | integer | Unique bet ID |
| `status` | object | `{"code": "settled"}` or `{"code": "failed"}` |
| `got_price` | number/null | Filled price |
| `want_stake` | `["USDT", n]` | Requested stake |
| `got_stake` | `["USDT", n]` | Filled stake |
| `profit_loss` | `["USDT", n]`/null | P&L |
| `exchange_role` | string/null | `maker`, `taker`, or null |

---

## Event info variants

Orders include `event_info` with one of three shapes:

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
  "start_time": "2026-04-14T23:00:00+00:00",
  "date": "2026-04-14",
  "result": { "tp1_home": 2, "tp1_away": 0, "tall_home": 4, "tall_away": 2 }
}
```

### Multirunner events (`event_type: "multirunner"`)

Outright/futures markets:

```json
{
  "event_type": "multirunner",
  "event_name": "UEFA - Champions League 2025/26",
  "teams": [{"team_id": 178, "name": "Real Madrid"}],
  "start_time": "2025-08-28T00:00:00+00:00",
  "end_time": "2026-05-30T23:00:00+00:00"
}
```

### Parlay events (`event_type: "parlay"`)

Accumulators — `leg_event_infos` array with nested normal/multirunner per leg.

### Result fields by sport

| Sport | Fields |
|-------|--------|
| Football | `ht_home`, `ht_away`, `ft_home`, `ft_away` |
| Ice hockey | `tp1_*`, `tp2_*`, `tp3_*`, `tall_*`, `pen_*` |
| Multirunner | `runner_results` array, `non_runner_count` |

---

## Other endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/v1/betslips/` | List open betslips |
| `GET` | `/v1/betslips/{id}/` | Get single betslip |
| `GET` | `/v1/orders/{id}/` | Get single order |
| `GET` | `/web/offerhist/{sport}/{event_id}/{bet_type}/` | Historical prices |
| `GET` | `/v1/xrates/` | Exchange rates (GBP base) |

**Exchange rates response:**

```json
{
  "status": "ok",
  "data": [
    {"ccy": "USDT", "rate": 1.343171},
    {"ccy": "USD", "rate": 1.343171},
    {"ccy": "EUR", "rate": 1.148302},
    {"ccy": "GBP", "rate": 1.0}
  ]
}
```

---

## Code examples

### Python — list recent losing bets

```python
import os, requests

API_KEY = os.environ["MM_API_KEY"]
BASE = "https://api.magicmarkets.com"
HEADERS = {"X-Api-Key": API_KEY}

resp = requests.get(f"{BASE}/v1/orders/", headers=HEADERS,
                    params={"page_size": 100, "status": "reconciled"})
orders = resp.json()["data"]

for o in orders:
    pnl = o["profit_loss"]       # ["USDT", -2.99] or null
    if pnl is None:
        continue                 # unsettled
    if pnl[1] < 0:
        stake = o["stake"]       # ["USDT", 2.99]
        print(f"Order {o['order_id']}: {o['bet_type_description']} "
              f"| stake {stake[1]:.2f} | P&L {pnl[1]:+.2f} USDT")
```

### Python — full trading flow

```python
import requests

API_KEY = "your-token"
BASE = "https://api.magicmarkets.com"
H = {"X-Api-Key": API_KEY, "Content-Type": "application/json"}

# 1. Create betslip
slip = requests.post(f"{BASE}/v1/betslips/", headers=H, json={
    "sport": "fb",
    "event_id": "2026-04-15,85,89",
    "bet_type": "for,1x2,1"
}).json()["data"]

print(f"Betslip {slip['betslip_id']}: {slip['bet_type_description']}")
for level in slip["price_list"]:
    print(f"  {level['price']} — max {level['max'][1]} USDT")

# 2. Place order at best price
best = slip["price_list"][0]
order = requests.post(f"{BASE}/v1/orders/", headers=H, json={
    "betslip_id": slip["betslip_id"],
    "want_price": best["price"],
    "want_stake": ["USDT", 25.0]
}).json()["data"]

print(f"Order {order['order_id']} placed, status: {order['status']}")

# 3. Check later
result = requests.get(f"{BASE}/v1/orders/{order['order_id']}/",
                       headers=H).json()["data"]
if result["profit_loss"]:
    print(f"P&L: {result['profit_loss'][1]:+.2f} USDT")
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

---

## Troubleshooting

| Symptom | Likely cause |
|---------|-------------|
| `401 Unauthorized` | Token expired, cleared, or wrong header (must be `X-Api-Key`) |
| `404` on `/web/token/` | Token management requires browser session auth, not API key |
| Empty `price_list` | Market suspended or no bookmaker coverage |
| `null` `profit_loss` | Bet not yet settled |
| `null` `stake` on order | No bets filled (order may have timed out) |
| `status: "failed"` | All placements failed — check `bets[].status` |
| `updated_at_from` error | Both timestamps must be >= 60 seconds in the past |

---

## Live data queries

If the developer has an API key (check for `.env` or ask), you can make live
calls to demonstrate endpoints or debug issues. Always show the curl command
so the developer can reproduce it.
