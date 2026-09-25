---
name: polybaskets-overview
description: Use when the agent or user needs to understand what PolyBaskets is, how baskets work, the index calculation, the payout model, or the settlement lifecycle. Do not use when the task is to execute an on-chain action.
---

# PolyBaskets Overview

## What Is PolyBaskets

PolyBaskets is an ETF-style prediction market aggregator on Vara Network. It bundles multiple Polymarket outcomes into a single weighted basket — a portfolio in one trade.

## The Agent Loop

```
Search active VARA baskets  →  Check status and funding  →  Get signed quote  →  Bet  →  Wait  →  Claim payout
```

1. **Choose a basket** — read `BasketMarket/GetBasket` and require `status: Active`; the indexer may lag.
2. **Match the asset kind** — only `Vara` baskets are part of the current betting flow. They take wallet VARA or FreebetLedger credit; the voucher only pays gas.
3. **Get a fresh signed quote** — the current contracts no longer accept a manually supplied entry index.
4. **Place one bet** — use the funded VARA or freebet lane, then check the resulting position. `BasketNotActive` means settlement began; choose another basket.
5. **Wait and claim** — PnL is realized after basket settlement.

You can also skip steps 2-4 and bet on an existing basket created by another user.

## Native VARA Freebet

Native VARA freebet is a separate non-withdrawable balance stored in `FreebetLedger`. It is not wallet-owned spendable VARA.

The flow is:

```
FreebetLedger balance  →  SpendFreebet  →  Vara basket freebet position  →  Claim profit
```

Rules:
- Only `asset_kind: "Vara"` baskets are eligible.
- `BasketMarket/IsVaraEnabled` must be true.
- Agents call `FreebetLedger/SpendFreebet`, not `BasketMarket/BetOnBasketFromFreebetLedger` directly.
- Freebet principal returns to `FreebetLedger` on claim; only profit above the stake is paid to the wallet.

See `../basket-freebet/SKILL.md` for the executable agent flow.

## Core Concepts

### Basket

A named collection of Polymarket outcomes with percentage weights (must sum to 100%). The current contract default requires at least 2 items, but admins can change `min_items_per_basket`; the hard cap is 32 items. Each item specifies:
- A Polymarket market (by numeric ID and slug)
- A selected outcome (YES or NO)
- A weight in basis points (e.g. 40% = 4000 bps, all must sum to 10000 bps = 100%)

### Basket Index

The index is a weighted probability score:

```
index = sum( weight_bps[i] / 10000 * probability[i] )
```

Ranges from 0.0 to 1.0. When a user bets, the current index is recorded on their `Position` as `index_at_creation_bps` (u16, 1-10000). The basket itself does not store an index — it is computed from live Polymarket prices.

See `../references/index-math.md` for formulas and worked examples.

### Position

A user's bet on a basket. Records:
- `shares` — amount of VARA wagered
- `index_at_creation_bps` — the entry index. If the same user bets on the same basket more than once, the contract stores a share-weighted average entry index.
- `claimed` — whether payout has been collected

### Payout

After settlement:

```
payout = shares * (settlement_index / entry_index)
```

If settlement index > entry index: profit. If lower: loss.

### Settlement Lifecycle

```
Active  →  SettlementPending  →  Settled
           (12-min challenge)     (users can claim)
```

1. **Active** — basket accepts bets
2. **SettlementPending** — settler proposes resolution with each item's final outcome from Polymarket. A challenge window begins; read the actual duration from `BasketMarket/GetConfig.liveness_ms`.
3. **Settled** — after the challenge window, anyone may call `FinalizeSettlement`. Users can then claim payouts.

## Programs

| Program | Role |
|---------|------|
| **BasketMarket** | Core contract: baskets, VARA bets, settlements, claims |
| **FreebetLedger** | Native VARA freebet balance ledger; spends into Vara baskets and receives returned principal |

## Current Asset Kind

Create only `asset_kind: "Vara"` baskets. Legacy `Bet` baskets may still appear in historical views, but they are not supported for new bets. Users bet with wallet VARA via BasketMarket or freebet credit via FreebetLedger. The wallet or ledger must fund the stake; the voucher only funds gas.

## Where to Go Next

**Choose the matching flow:**
1. Browse baskets and verify chain status and asset kind: `../basket-query/SKILL.md`.
2. For an `Active`/`Vara` basket, use wallet VARA through `../basket-bet/SKILL.md` or freebet credit through `../basket-freebet/SKILL.md`.
3. Claim a finalized position through `../basket-claim/SKILL.md`.

**Native VARA freebet flow:**
- Spend non-withdrawable VARA freebet balance: `../basket-freebet/SKILL.md`

**Settler role only:**
- Settle a basket: `../basket-settle/SKILL.md`
