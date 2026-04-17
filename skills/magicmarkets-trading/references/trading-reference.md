# Trading API — Full Reference

Base URL: `https://api.magicmarkets.com`

All requests authenticated via `X-Api-Key: <token>` header (unless noted).

## Table of Contents

1. [The three bet types (back, lay, parlay)](#the-three-bet-types)
2. [Token management](#token-management)
3. [Betslips](#betslips)
4. [Orders](#orders)
5. [Order updates](#order-updates)
6. [Order fields](#order-fields)
7. [Bet fields](#bet-fields)
8. [Parlay legs](#parlay-legs)
9. [Event info](#event-info)
10. [Offer history](#offer-history)
11. [Blocked endpoints](#blocked-endpoints)

---

## The three bet types

| `betslip_type` / `order_type` | Meaning |
|-------------------------------|---------|
| `normal` | **Back** bet — win if the outcome happens |
| `lay` | **Lay** bet — win if the outcome does NOT happen |
| `parlay` | **Accumulator** — 2-10 legs combined; all must win |

A normal bet at 2.00 for 10 USDT pays 20 USDT on win (profit +10).

A lay bet at 2.00 for 10 USDT wins 10 USDT if the outcome does not occur;
loses 10 × (2.00 − 1) = 10 USDT if it does.

Parlay odds multiply across legs (approximately): three legs at 2.00 each
combine to ~8.00.

---

## Token management

Requires **browser session auth** (`session` + `magic-metadata-jwt` headers),
not the API key. Available from the account dashboard.

### POST `/web/token/` — Create a token

```json
{
  "name": "my-trading-bot",
  "expires_at": "2027-03-01T00:00:00Z"
}
```

**Response (201):**

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

`token` is plaintext, shown only once. Max 16 active tokens per account.

### GET `/web/token/` — List tokens
Token values masked as `"***"`.

### PATCH `/web/token/{id}/` — Update
Both `name` and `expires_at` optional.

### DELETE `/web/token/{id}/` — Soft-delete
Sets `cleared = true`.

---

## Betslips

### POST `/v1/betslips/` — Create a betslip

**Request body (BetslipCreateRequest):**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `betslip_type` | string | `"normal"` | `"normal"`, `"lay"`, or `"parlay"` |
| `sport` | string | null | Required for normal/lay, null for parlay |
| `event_id` | string | null | Required for normal/lay, null for parlay |
| `bet_type` | string | null | Required for normal/lay, null for parlay |
| `legs` | array | null | Required for parlay (2-10 BetslipLeg), null otherwise |
| `equivalent_bets` | boolean | `true` | Include equivalent bets across bookies |
| `exclude_danger` | boolean | `false` | Skip bookies with bets in danger status |
| `user_data` | string | null | Optional reference (max 512 chars) |

**BetslipLeg fields** (for parlay):
- `sport` (required)
- `event_id` (required)
- `bet_type` (required)

**Query parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `display_net_prices` | boolean | `false` | Show net prices (after commission) |
| `obfuscate_accounts` | boolean | `false` | Hide bookie usernames (always effective) |

**Response (BetslipResponse):**

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

**⚠️ PriceListEntry is nested!** Each entry wraps the price info in `effective`:

```json
{"effective": {"price": <number>, "min": <["USDT",n]|null>, "max": <["USDT",n]|null>}}
```

Access as `entry["effective"]["price"]`, not `entry["price"]`.

- `price` — decimal odds, rounded up to nearest Betfair Classic ladder step
- `min` — minimum stake at this level, `["USDT", amount]` or null
- `max` — maximum stake at this level (excluding sportradar), `["USDT", amount]` or null

`total` — sum of max stakes across all levels.

### GET `/v1/betslips/` — List open betslips

Returns IDs only for the authenticated customer. Same query params as POST.

### GET `/v1/betslips/{betslip_id}/` — Get one

Returns full BetslipResponse. Same query params.

---

## Orders

### POST `/v1/orders/` — Place an order

**Request body (OrderCreateRequest):**

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `betslip_id` | string | **yes** | — | Betslip to place order on |
| `price` | number | **yes** | — | Desired price (decimal odds) |
| `stake` | `[ccy, amt]` | **yes** | — | `[currency, amount]` — any currency accepted on input |
| `duration` | number | **yes** | — | Order duration in minutes |
| `exchange_mode` | string | no | `"make_and_take"` | `"make_and_take"`, `"make"`, `"take"` |
| `keep_open_ir` | boolean | no | `false` | Keep order open in-running |
| `accept_partial_fill` | boolean | no | `true` | Accept partial fills |
| `accept_better_price` | boolean | no | `true` | Accept prices better than requested |
| `force_want_price` | boolean | no | `false` | Force exact price |
| `min_taker_want_stake` | `[ccy, amt]` | no | null | Min taker stake |
| `current_score` | string | no | null | Current score for in-play orders |
| `user_data` | string | no | null | Optional customer reference |
| `request_uuid` | string | no | null | Idempotency key — safe to retry |

### GET `/v1/orders/` — List orders

Returns orders for the authenticated customer. All bookie-identifying fields
stripped; all stakes in USDT.

**Query parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `page` | integer | Page number (default 1) |
| `page_size` | integer | Default 25, max 1000 |
| `status` | string[] | `open`, `pending`, `done`, `failed` |
| `sport` | string[] | Filter by sport code |
| `event_id` | string[] | Filter by event ID |
| `order_type` | string[] | `normal`, `lay`, `parlay` |
| `date_from` | string | ISO datetime lower bound |
| `date_to` | string | ISO datetime upper bound |
| `search` | string | Free text search |

**Response (OrderListEnvelope):** `{"status": "ok", "data": [OrderResponse, ...]}`

### GET `/v1/orders/{order_id}/` — Get one

Returns a single OrderResponse.

---

## Order updates

### GET `/v1/orders/updates/`

Returns recently updated orders within a time range.

| Param | Required | Description |
|-------|----------|-------------|
| `updated_at_from` | **yes** | ISO datetime (must be >= 60s in the past) |
| `updated_at_to` | **yes** | ISO datetime (must be >= 60s in the past) |
| `placer` | no | Filter by customer username |

---

## Order fields

Full OrderResponse schema:

| Field | Type | Description |
|-------|------|-------------|
| `order_id` | integer | Unique ID |
| `order_type` | string | `normal`, `lay`, `parlay` |
| `bet_type` | string | Canonical bet type string |
| `bet_type_description` | string | Human-readable |
| `sport` | string | Sport code |
| `placer` | string | Customer username |
| `want_price` | number | Requested price |
| `want_stake` | `["USDT", n]`/null | Requested stake |
| `ccy_rate` | number | FX rate vs GBP at order time |
| `placement_time` | ISO string | When placed |
| `expiry_time` | ISO string | When it expires |
| `closed` | boolean | Whether closed |
| `close_reason` | string/null | `order_filled`, `timed_out`, `cancelled`, etc. |
| `event_info` | EventInfo | Event metadata (see below) |
| `bets` | FormattedBetResponse[] | Individual fills |
| `user_data` | string/null | Customer reference |
| `status` | string | `open`, `pending`, `done`, `failed` |
| `keep_open_ir` | boolean | In-running flag (default false) |
| `exchange_mode` | string/null | `make_and_take`, `make`, `take` |
| `price` | number/null | Matched price |
| `stake` | `["USDT", n]`/null | Aggregate matched stake |
| `profit_loss` | `["USDT", n]`/null | P&L |
| `bet_bar_values` | object/null | Internal display values |
| `legs` | ParlayLeg[]/null | Per-leg details for parlays |

---

## Bet fields

Each order's `bets` array contains FormattedBetResponse objects:

| Field | Type | Description |
|-------|------|-------------|
| `bet_id` | integer | Unique bet ID |
| `order_id` | integer | Parent order |
| `order_ccy_rate` | number | FX at order time |
| `status` | `BetStatus`/string | Status object or code string |
| `sport` | string | Sport code |
| `event_id` | string | Event ID |
| `bet_type` | string | Bet type |
| `ccy_rate` | number | FX vs GBP |
| `want_price` | number | Requested price |
| `got_price` | number/null | Filled price |
| `want_stake` | `["USDT", n]`/null | Requested stake |
| `got_stake` | `["USDT", n]`/null | Filled stake |
| `profit_loss` | `["USDT", n]`/null | P&L |
| `reconciled` | string/null | Reconciliation state (string, not boolean) |
| `exchange_role` | string/null | `maker`, `taker`, or null |
| `legs` | ParlayLeg[]/null | For parlay bets |

### BetStatus object

```json
{
  "code": "success",
  "response_pmm": { }
}
```

`code` values include `success`, `failed`, `pending`. `response_pmm` is PMM
response data with stakes already converted to USDT, or null.

---

## Parlay legs

When `order_type: "parlay"`, the order's `legs` array contains:

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

| Field | Type | Description |
|-------|------|-------------|
| `id` | integer | Leg ID |
| `sport` | string | Sport code |
| `event_id` | string | Event ID |
| `bet_type` | string | Bet type |
| `bet_type_description` | string | Human-readable |
| `price` | number/null | Achieved price for this leg |
| `outcome` | string/null | `won`, `lost`, `void`, `push`, or null |

---

## Event info

Included in every order response. Shape depends on `event_type`.

### Normal events

Football, ice hockey, tennis, etc.

```json
{
  "event_type": "normal",
  "event_id": "2026-04-14,29060,29064",
  "event_name": "Philadelphia Flyers vs. Montreal Canadiens",
  "home_id": 29060,
  "home_team": "Philadelphia Flyers",
  "away_id": 29064,
  "away_team": "Montreal Canadiens",
  "competition_id": 1294,
  "competition_name": "USA NHL",
  "competition_country": "US",
  "start_time": "2026-04-14T23:00:00+00:00",
  "date": "2026-04-14",
  "result": {
    "tp1_home": 2, "tp1_away": 0,
    "tp2_home": 3, "tp2_away": 2,
    "tp3_home": 4, "tp3_away": 2,
    "tall_home": 4, "tall_away": 2,
    "pen_home": null, "pen_away": null
  }
}
```

### Multirunner events

Outright/futures markets.

```json
{
  "event_type": "multirunner",
  "event_id": "2025-08-28,multirunner,100268425",
  "event_name": "UEFA - Champions League 2025/26",
  "teams": [{"team_id": 178, "name": "Real Madrid"}],
  "competition_id": 28,
  "competition_name": "UEFA Champions League",
  "competition_country": "EU",
  "start_time": "2025-08-28T00:00:00+00:00",
  "end_time": "2026-05-30T23:00:00+00:00",
  "date": "2025-08-28",
  "result": {
    "runner_results": [{"team_id": 178, "position": 1}],
    "non_runner_count": 0
  }
}
```

### Parlay events

Top-level `event_id` is null. `leg_event_infos` array holds one EventInfo per
leg (each can be normal or multirunner).

### RunnerResult position values

| Value | Meaning |
|-------|---------|
| 0 | Unknown |
| 1 | First |
| 2 | Second |
| -1 | Void |
| -2 | Non-runner |
| -3 | Eliminated |

### Result fields by sport

| Sport | Fields |
|-------|--------|
| Football | `ht_home`, `ht_away`, `ft_home`, `ft_away` |
| Ice hockey | `tp1_*`, `tp2_*`, `tp3_*`, `tall_*`, `pen_*` |
| Multirunner | `runner_results` array, `non_runner_count` |

---

## Offer history

### GET `/web/offerhist/{sport}/{event_id}/{bet_type}/`

Returns historical price data. Currently proxies as-is from the upstream;
future plan is to aggregate per-bookie prices into a best-price series.

---

## Blocked endpoints

These endpoints exist in the underlying platform but are **intentionally
blocked** in magic-api, returning 404. The Trading API abstracts bookie
identities away:

| Endpoint | Purpose (blocked) |
|----------|-------------------|
| `GET /v1/matrix/bookies/` | Bookie list |
| `POST /v1/matrix/accounts/` | Account matrix |
| `GET /v1/customers/{username}/bookie_accounts/` | Per-customer bookie accounts |
| `GET /v1/customers/{username}/bookie_balances/` | Per-customer bookie balances |
| `GET /v1/groups/{group_id}/bookie_accounts/` | Per-group bookie accounts |

If a developer asks for these, explain they're blocked by design.
