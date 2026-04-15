# Trading API — Full Reference

Base URL: `https://api.magicmarkets.com`

All requests authenticated via `X-Api-Key: <token>` header (unless noted).

## Table of Contents

1. [Token management](#token-management) (lines ~10-50)
2. [Betslips](#betslips) (lines ~55-110)
3. [Orders](#orders) (lines ~115-220)
4. [Order updates](#order-updates) (lines ~225-245)
5. [Order fields](#order-fields) (lines ~250-310)
6. [Bet fields](#bet-fields) (lines ~315-355)
7. [Event info](#event-info) (lines ~360-440)
8. [Offer history](#offer-history) (lines ~445-455)
9. [Exchange rates](#exchange-rates) (lines ~460-490)

---

## Token management

Requires **browser session auth** (`Session` + `magic-metadata-jwt` headers),
not the API key. Available from the account dashboard.

### POST `/web/token/` — Create a token

```json
{
  "name": "my-trading-bot",
  "expires_at": "2027-03-01T00:00:00Z"   // optional, defaults to 365 days
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

The `token` field is only shown once at creation. Max 16 active tokens per account.

### GET `/web/token/` — List tokens

Token values masked as `"***"`.

### PATCH `/web/token/{id}/` — Update a token

Both `name` and `expires_at` are optional.

### DELETE `/web/token/{id}/` — Delete a token

Soft-deletes (`cleared = true`).

---

## Betslips

Betslips discover available prices and liquidity for a market selection.

### POST `/v1/betslips/` — Create a betslip

**Query parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `display_net_prices` | boolean | false | Show net prices (after commission) |
| `obfuscate_accounts` | boolean | false | Hide bookie account usernames |

**Response:**

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

**`price_list`** — sorted by price descending:

| Field | Type | Description |
|-------|------|-------------|
| `price` | number | Decimal odds, rounded to Betfair Classic ladder |
| `min` | `["USDT", n]` or null | Minimum stake at this price |
| `max` | `["USDT", n]` or null | Maximum stake at this price |

**`total`** — sum of max stakes across all price levels.

### GET `/v1/betslips/` — List betslips

Returns all open betslips.

### GET `/v1/betslips/{betslip_id}/` — Get a betslip

Returns a single betslip.

---

## Orders

### POST `/v1/orders/` — Place an order

Places a new order on a betslip.

### GET `/v1/orders/` — List orders

**Query parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `page` | integer | Page number (default 1) |
| `page_size` | integer | Results per page (default 25, max 1000) |
| `status` | string[] | Filter: `open`, `pending`, `failed`, `reconciled` |
| `sport` | string[] | Filter by sport code |
| `event_id` | string[] | Filter by event ID |
| `order_type` | string[] | Filter: `normal`, `lay`, `parlay` |
| `date_from` | string | ISO datetime lower bound |
| `date_to` | string | ISO datetime upper bound |
| `search` | string | Search text |

**Response:**

```json
{
  "status": "ok",
  "data": [
    {
      "order_id": 1472877357,
      "order_type": "normal",
      "bet_type": "for,tp,all,ml,a",
      "bet_type_description": "Montreal Canadiens Moneyline (Inc. Overtime)",
      "sport": "ih",
      "want_price": 1.65,
      "want_stake": ["USDT", 3.0],
      "status": "reconciled",
      "closed": true,
      "close_reason": "order_filled",
      "price": 1.65,
      "stake": ["USDT", 2.99],
      "profit_loss": ["USDT", -2.99],
      "exchange_mode": "make_and_take",
      "event_info": { "..." },
      "bets": [ "..." ]
    }
  ]
}
```

### GET `/v1/orders/{order_id}/` — Get an order

Returns a single order.

---

## Order updates

### GET `/v1/orders/updates/`

Returns orders updated within a time range. Both params are required.

| Param | Required | Description |
|-------|----------|-------------|
| `updated_at_from` | yes | ISO datetime (must be >= 60s in the past) |
| `updated_at_to` | yes | ISO datetime (must be >= 60s in the past) |
| `placer` | no | Filter by customer username |

---

## Order fields

| Field | Type | Description |
|-------|------|-------------|
| `order_id` | integer | Unique order ID |
| `order_type` | string | `normal`, `lay`, `parlay` |
| `bet_type` | string | Canonical bet type (e.g. `for,tp,all,ml,a`) |
| `bet_type_description` | string | Human-readable description |
| `sport` | string | Sport code |
| `placer` | string | Customer username |
| `want_price` | number | Requested price (decimal odds) |
| `want_stake` | `["USDT", n]` | Requested stake |
| `ccy_rate` | number | Exchange rate (GBP base) at order time |
| `placement_time` | datetime | When the order was placed |
| `expiry_time` | datetime | When the order expires |
| `closed` | boolean | Whether the order is closed |
| `close_reason` | string or null | `order_filled`, `timed_out`, `cancelled`, etc. |
| `status` | string | `open`, `reconciled`, `failed`, `pending` |
| `keep_open_ir` | boolean | Keep order open in-play |
| `exchange_mode` | string | `make_and_take`, `make`, `take` |
| `price` | number or null | Matched price (null if no fills) |
| `stake` | `["USDT", n]` or null | Matched stake |
| `profit_loss` | `["USDT", n]` or null | P&L (null if unsettled) |
| `user_data` | string or null | Optional customer reference |
| `event_info` | object | Event metadata |
| `bets` | array | Individual bet fills |

---

## Bet fields

Each order contains a `bets` array of individual fills:

| Field | Type | Description |
|-------|------|-------------|
| `bet_id` | integer | Unique bet ID |
| `order_id` | integer | Parent order ID |
| `order_ccy_rate` | number | Exchange rate at order time |
| `status` | object | `{"code": "settled"}` or `{"code": "failed"}` |
| `sport` | string | Sport code |
| `event_id` | string | Event ID |
| `bet_type` | string | Bet type |
| `ccy_rate` | number | Exchange rate |
| `want_price` | number | Requested price |
| `got_price` | number or null | Filled price |
| `want_stake` | `["USDT", n]` | Requested stake |
| `got_stake` | `["USDT", n]` | Filled stake |
| `profit_loss` | `["USDT", n]` or null | P&L |
| `reconciled` | boolean | Whether reconciled |
| `exchange_role` | string or null | `maker`, `taker`, or null |

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
  "result": null
}
```

### Parlay events

Accumulators with multiple legs.

The `event_info` has `event_type: "parlay"` with a `leg_event_infos` array. Each
element is a normal or multirunner event info.

### Result fields by sport

| Sport | Fields |
|-------|--------|
| Football | `ht_home`, `ht_away`, `ft_home`, `ft_away` |
| Ice hockey | `tp1_*`, `tp2_*`, `tp3_*`, `tall_*`, `pen_*` |
| Multirunner | `runner_results` array, `non_runner_count` |

---

## Offer history

### GET `/web/offerhist/{sport}/{event_id}/{bet_type}/`

Returns historical price data for a specific market, aggregated into a
best-price time series.

---

## Exchange rates

### GET `/v1/xrates/`

Returns exchange rates relative to the account base currency.

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
