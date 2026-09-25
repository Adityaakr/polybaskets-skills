---
name: polybaskets-skills
description: Route PolyBaskets agents through the current VARA-only basket, wallet-stake, and freebet workflows on Vara mainnet. Legacy BetToken/BetLane baskets are historical and not a current betting route.
---

# PolyBaskets Agent Router

PolyBaskets combines Polymarket outcomes into on-chain baskets. The current agent flow uses `asset_kind: "Vara"` only. The live BasketMarket program is `0xa749ccd80d71637b450789e12e3d94524e9ae17877d1b59f5ddda784f89a2cba`. Never assume that an older betting lane targets this program. Do not use `BetLane/PlaceBet` or create `asset_kind: "Bet"` baskets as a fallback.

Before any write action, read the task-specific skill completely:

| Task | Skill |
|---|---|
| Browse baskets, positions, and settlement state | `basket-query/SKILL.md` |
| Create a new active VARA basket | `basket-create/SKILL.md` |
| Place a wallet-funded VARA stake | `basket-bet/SKILL.md` |
| Spend or claim a non-withdrawable VARA freebet | `basket-freebet/SKILL.md` |
| Claim a finalized VARA payout | `basket-claim/SKILL.md` |
| Understand index, settlement, and PnL | `polybaskets-overview/SKILL.md` |

Use `STARTER_PROMPT.md` for a bounded agent session. Mainnet only. The agent must have an authorized funding source for the **stake**: spendable wallet VARA or FreebetLedger credit. A gas voucher pays fees only. Check `BasketMarket/IsVaraEnabled` and stop if false. If there is no stake balance or no user-approved wallet VARA budget, report the limitation; do not create baskets simply to appear active on the leaderboard.

For every bet: select an on-chain `Active`/`Vara` basket, check its markets remain outside the bet cutoff, request a fresh BasketMarket signed quote, estimate and send sequentially, and verify the position. `BasketNotActive` means that basket can no longer take bets; select another instead of retrying. Basket creation is not a bet and does not produce PnL. PnL is realized only after settlement finalizes.

Current `vara-wallet` versions may return a successful `GetBasket` query as `{ "result": { "kind": "Ok", "value": { ... } } }`; older versions may use `result.ok`. Normalize with `(.result.value // .result.ok)` in jq and read enum fields such as `status.kind` and `asset_kind.kind`.

## Campaign ranking

Daily top three use realized PnL plus the change in unrealized PnL during the UTC
window, frozen at 00:00 UTC. Open positions are valued with the payout formula at
observed market indices. Counts of baskets, bets, approvals, or claims never increase
rank. Only strictly positive total PnL qualifies for a prize; a day with no
positive score has no winners. Equal scores use wallet address ascending. Freebet PnL counts only the user's
profit, excluding ledger principal. Do not present old Season 2 activity scores as
campaign ranks. Missing market snapshots mean unavailable PnL, not an automatic zero.
