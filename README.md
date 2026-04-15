# MagicMarkets Trading Plugin for Claude Code

A Claude Code plugin that helps you integrate the **MagicMarkets Trading API** — a programmatic interface for placing and managing sports bets across aggregated bookmaker liquidity.

## What it does

When you ask Claude Code about placing bets, managing orders, or building a trading bot, this plugin provides:

- **Accurate API guidance** — correct endpoints, auth headers, USDT tuple format, and response structures
- **Working code snippets** — Python examples for the full trading workflow (betslip → order → monitoring)
- **Live API calls** — if you have an API key configured, Claude can query your orders and demonstrate endpoints
- **Reference lookups** — order fields, bet fields, event info variants, sport codes, bet type patterns

## Install

```bash
claude plugin install magicmarkets-trading
```

## Quick start

After installing, just ask Claude Code naturally:

- *"Write a Python trading bot for MagicMarkets"*
- *"How do I place a bet on the MagicMarkets API?"*
- *"List my recent losing bets with their P&L"*
- *"What does the order response look like?"*
- *"Explain the USDT stake format"*

## API overview

| Detail | Value |
|--------|-------|
| **Base URL** | `https://api.magicmarkets.com` |
| **Auth** | `X-Api-Key` header (long-lived token, up to 365 days) |
| **Stake format** | `["USDT", <amount>]` tuples |
| **Workflow** | Create betslip → review prices → place order → monitor |
| **Order types** | Normal, lay, parlay |
| **Sports** | Football, basketball, tennis, ice hockey, baseball, and 16 more |

## Configuration

If you want Claude to make live API calls on your behalf, add your API key to a `.env` file in your project:

```
MM_API_KEY=your-api-token
```

## Key concepts

**USDT tuples** — All monetary amounts are `["USDT", <number>]` arrays, never plain numbers. Unsettled values are `null`.

**Bookie abstraction** — All bookmaker-identifying fields are stripped. You see aggregated prices and fills but never which bookmaker serviced a bet.

**Response envelope** — Every response is `{"status": "ok", "data": <payload>}`.

## Documentation

- [Full API docs](https://api.magicmarkets.com)
- Plugin reference files are in `skills/magicmarkets-trading/references/`
