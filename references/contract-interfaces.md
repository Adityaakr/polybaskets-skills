# PolyBaskets Contract Interfaces

Complete annotated interface for the PolyBaskets programs. IDL files bundled in `../idl/`.

## BasketMarket (`polymarket-mirror.idl`)

### Types

```
BasketItem {
  poly_market_id: str      # Numeric Polymarket market ID string
  poly_slug: str           # Polymarket market slug
  weight_bps: u16          # Weight in basis points (0-10000)
  selected_outcome: Outcome # YES or NO
  end_timestamp: u64       # REQUIRED. Polymarket endDate in Unix ms.
                           # Omitting it (or sending 0) creates a basket that can
                           # never be bet on: every quote fails the cutoff check.
}

Outcome = YES | NO

BasketAssetKind = Vara | Bet

Basket {
  id: u64                  # Unique basket ID
  creator: actor_id        # Creator's address
  name: str                # Max 128 chars
  description: str         # Max 512 chars
  items: vec BasketItem    # min_items_per_basket..32 items, weights must sum to 10000
  created_at: u64          # Block timestamp
  status: BasketStatus     # Active | SettlementPending | Settled
  asset_kind: BasketAssetKind
}

Position {
  basket_id: u64
  user: actor_id
  shares: u128             # VARA amount bet (in minimal units)
  claimed: bool
  index_at_creation_bps: u16  # Entry index (1-10000)
}

Settlement {
  basket_id: u64
  proposer: actor_id
  item_resolutions: vec ItemResolution
  payout_per_share: u128   # Pre-computed payout ratio
  payload: str             # Settlement metadata/proof
  proposed_at: u64
  challenge_deadline: u64  # Must pass before finalization
  finalized_at: opt u64
  status: SettlementStatus # Proposed | Finalized
}

ItemResolution {
  item_index: u8           # Index into basket.items (0-based)
  resolved: Outcome        # Final outcome
  poly_slug: str           # Must match basket item's slug
  poly_condition_id: opt str
  poly_price_yes: u16      # Final YES price in bps
  poly_price_no: u16       # Final NO price in bps
}

BasketMarketConfig {
  admin_role: actor_id
  settler_role: actor_id
  liveness_ms: u64
  vara_enabled: bool
  min_items_per_basket: u32
}
```

### State-Changing Methods (require `--account`)

| Method | Args | Returns | Notes |
|--------|------|---------|-------|
| `CreateBasket` | `name, description, items, asset_kind` | `u64` (basket_id) | Weights must sum to 10000 |
| `BetOnBasket` | `basket_id, signed_quote` | `u128` (shares) | Requires `--value` in VARA and a fresh BasketMarket quote |
| `BetOnBasketFromFreebetLedger` | `user, basket_id, signed_quote` | `u128` (shares) | Ledger-only downstream method; normal agents call `FreebetLedger/SpendFreebet` instead |
| `Claim` | `basket_id` | `u128` (payout) | Only after settlement finalized |
| `ProposeSettlement` | `basket_id, item_resolutions, payload` | `null` | Settler role only |
| `FinalizeSettlement` | `basket_id` | `null` | Permissionless after challenge window |
| `SetConfig` | `config` | `null` | Admin role only |
| `SetFreebetLedger` | `ledger_id` | `null` | Admin role only |
| `SetVaraEnabled` | `enabled` | `null` | Admin role only |

### Query Methods (free, no `--account`)

| Method | Args | Returns |
|--------|------|---------|
| `GetBasket` | `basket_id` | `Result<Basket, Error>` |
| `GetBasketCount` | none | `u64` |
| `GetConfig` | none | `BasketMarketConfig` |
| `GetFreebetLedger` | none | `actor_id` |
| `GetFreebetPositions` | `user` (actor_id) | `vec Position` |
| `GetPositions` | `user` (actor_id) | `vec Position` |
| `GetSettlement` | `basket_id` | `Result<Settlement, Error>` |
| `IsVaraEnabled` | none | `bool` |

### Events

- `BasketCreated { basket_id, creator, asset_kind }`
- `VaraBetPlaced { basket_id, user, amount, user_total, index_at_creation_bps }`
- `FreebetBetPlaced { basket_id, user, amount, user_total, index_at_creation_bps }`
- `SettlementProposed { basket_id, asset_kind, proposer, payout_per_share, challenge_deadline }`
- `SettlementFinalized { basket_id, asset_kind, finalized_at, payout_per_share }`
- `Claimed { basket_id, user, amount }`
- `VaraSupportUpdated { enabled }`
- `ConfigUpdated { config }`

---

## BetToken (legacy reference only)

CHIP is retired for betting. Kept for reading historical data; do not claim or
spend CHIP for current bets. (`bet_token_client.idl`)

Fungible token (VFT) with hourly claim windows and daily streak bonuses.

### Key Methods

| Method | Args | Returns | Notes |
|--------|------|---------|-------|
| `Claim` | none | `ClaimState` | Hourly token claim with daily streak bonus |
| `Transfer` | `to, value` | `bool` | Standard VFT transfer |
| `Approve` | `spender, value` | `bool` | Approve spending allowance |
| `TransferFrom` | `from, to, value` | `bool` | Transfer from approved allowance |
| `AdminMint` | `to, value` | `null` | Admin only |

### Key Queries (BetToken service)

| Method | Args | Returns |
|--------|------|---------|
| `BalanceOf` | `account` | `u256` |
| `GetClaimPreview` | `user` | `ClaimPreview { amount, streak_days, next_claim_at, can_claim_now }` |
| `GetClaimState` | `user` | `ClaimState { last_claim_at, streak_days, total_claimed, claim_count }` |
| `GetClaimConfig` | none | `ClaimConfig { base_claim_amount, max_claim_amount, streak_step, streak_cap_days, claim_period, day_start_offset_ms, claim_paused }` |
| `IsClaimPaused` | none | `bool` |
| `IsSpenderAllowed` | `spender` | `bool` |
| `Allowance` | `owner, spender` | `u256` |
| `TotalSupply` | none | `u256` |

### Metadata Service Queries

`Name`, `Symbol`, and `Decimals` are on the **Metadata** service, not BetToken:

| Method | Args | Returns | vara-wallet service prefix |
|--------|------|---------|---------------------------|
| `Name` | none | `str` | `Metadata/Name` |
| `Symbol` | none | `str` | `Metadata/Symbol` |
| `Decimals` | none | `u8` | `Metadata/Decimals` |

---

## BetLane (`bet_lane_client.idl`) — legacy reference only

Historical betting lane using CHIP/BetToken. It is disabled for the current
campaign and must not be selected by agents; current bets use BasketMarket VARA
or FreebetLedger.

### Key Methods

| Method | Args | Returns | Notes |
|--------|------|---------|-------|
| `PlaceBet` | `basket_id, amount, signed_quote` | `u256` (shares) | Requires BetToken approval first; `signed_quote` contains `payload.quoted_index_bps` |
| `Claim` | `basket_id` | `u256` (payout) | After settlement finalized |

### Key Queries

| Method | Args | Returns |
|--------|------|---------|
| `GetPosition` | `user, basket_id` | `Position { shares: u256, claimed, index_at_creation_bps }` |
| `GetPositions` | `user, offset, limit` | `Result<vec UserPositionView, Error>` |
| `GetConfig` | none | `BetLaneConfig { min_bet, max_bet, payouts_allowed_while_paused, quote_signer }` |
| `IsPaused` | none | `bool` |
| `BasketProgramId` | none | `actor_id` |
| `BetTokenId` | none | `actor_id` |
| `GetDependencies` | none | `BetLaneDependencies { basket_program_id, bet_token_id }` |

Note: BetLane `Position.shares` is `u256` (BET token units), unlike BasketMarket `Position.shares` which is `u128` (VARA minimal units).

---

## FreebetLedger (`freebet-ledger.idl`)

Native VARA freebet balance ledger. It stores non-withdrawable VARA credits, lets users spend those credits into authorized native `Vara` baskets, and receives returned principal when the basket is claimed.

### Types

```
FreebetGrant {
  id: str
  recipient: actor_id
  amount: u128
  reason: str
  granted_at: u64
}
```

### Key Methods

| Method | Args | Returns | Notes |
|--------|------|---------|-------|
| `SpendFreebet` | `bet_program_id, basket_id, amount, signed_quote` | `u128` | Agent path. Debits caller balance and forwards `amount` plus signed quote into BasketMarket |
| `Grant` | `to, grant_id, reason` | `u128` | Admin only. Must attach native VARA value; idempotent by `grant_id` |
| `ReturnFreebet` | `user, basket_id` | `u128` | Authorized bet program only. BasketMarket calls this during claim to return principal |
| `AuthorizeBetProgram` | `program_id` | `null` | Admin only |
| `RevokeBetProgram` | `program_id` | `null` | Admin only |

### Key Queries

| Method | Args | Returns |
|--------|------|---------|
| `BalanceOf` | `user` | `u128` |
| `GetGrant` | `grant_id` | `Option<FreebetGrant>` |
| `IsBetProgramAuthorized` | `program_id` | `bool` |
| `GetPendingSpendCount` | none | `u64` |
| `Admin` | none | `actor_id` |

### Agent Rules

- Only spend freebet on baskets where `asset_kind == "Vara"` and `BasketMarket/IsVaraEnabled == true`.
- Do not call `BasketMarket/BetOnBasketFromFreebetLedger` directly; the contract accepts only the configured ledger as caller.
- Request a fresh quote from `/api/basket-market/quote`, targeting BasketMarket, immediately before `SpendFreebet`. The BetLane quote endpoint is for CHIP only.
- On `BasketMarket/Claim`, freebet principal returns to FreebetLedger and only profit above stake is sent to the wallet.
