---
name: basket-query
description: Use when the agent needs to read basket state, user positions, settlement status, config, or basket count from the on-chain contracts. All queries are free (no gas, no account needed). Do not use for state-changing operations.
---

# Basket Query

All queries are read-only and free — no `--account` needed.

## Setup

**MAINNET ONLY.** Run `vara-wallet config set network mainnet` before anything else. NEVER switch to testnet — there are no contracts there.

```bash
# Set network and variables (see ../references/program-ids.md)
vara-wallet config set network mainnet
BASKET_MARKET="0xa749ccd80d71637b450789e12e3d94524e9ae17877d1b59f5ddda784f89a2cba"
FREEBET_LEDGER="0x2bb74834402fb7da9144d2ab91c1570e97237ad0ead1f7feb392162c3e3ad64e"
_PB="${POLYBASKETS_SKILLS_DIR:-skills}"
IDL="$_PB/idl/polymarket-mirror.idl"
FREEBET_LEDGER_IDL="$_PB/idl/freebet-ledger.idl"
```

## Get Your Hex Address

Sails `actor_id` args require hex format — SS58 addresses won't work:

```bash
MY_ADDR=$(vara-wallet balance | jq -r .address)
echo $MY_ADDR  # 0xe008...
```

## BasketMarket Queries

### Get basket count

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetBasketCount --args '[]' --idl $IDL
```

Returns `u64` — total baskets created. Basket IDs are 0-indexed.

### Get a basket

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetBasket --args '[0]' --idl $IDL
```

With `vara-wallet` 0.20+, successful responses are nested under `.result.value`; older versions may use `.result.ok`. Parse both with jq:

```bash
# Enum fields such as status and asset_kind may be objects with a .kind key.
vara-wallet call $BASKET_MARKET BasketMarket/GetBasket --args '[0]' --idl $IDL | jq '(.result.value // .result.ok) | {id, status:(.status.kind // .status), asset_kind:(.asset_kind.kind // .asset_kind)}'
```

Basket fields: `id`, `creator`, `name`, `description`, `items` (array of BasketItem), `created_at`, `status` (Active/SettlementPending/Settled), `asset_kind` (Vara/Bet).

### Get user positions

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetPositions \
  --args '["'$MY_ADDR'"]' --idl $IDL
```

Returns `vec Position`. Each position has: `basket_id`, `user`, `shares`, `claimed`, `index_at_creation_bps`.

### Get native freebet positions

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetFreebetPositions \
  --args '["'$MY_ADDR'"]' --idl $IDL
```

Returns native VARA freebet positions recorded by `FreebetLedger/SpendFreebet`. Each position has the same shape as native VARA positions: `basket_id`, `user`, `shares`, `claimed`, `index_at_creation_bps`.

To check whether BasketMarket is wired to the expected ledger:

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetFreebetLedger \
  --args '[]' --idl $IDL
```

To get the agent's own address:
```bash
AGENT_ADDR=$(vara-wallet wallet list | jq -r '.[0].address')
```

### Get settlement

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetSettlement --args '[0]' --idl $IDL
```

Returns `Result<Settlement, BasketMarketError>`. Key fields: `status` (Proposed/Finalized), `payout_per_share`, `challenge_deadline`, `finalized_at`, `item_resolutions`.

### Check config

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetConfig --args '[]' --idl $IDL
```

Returns `BasketMarketConfig`: `admin_role`, `settler_role`, `liveness_ms`, `vara_enabled`, `min_items_per_basket`.

### Check VARA enabled

```bash
vara-wallet call $BASKET_MARKET BasketMarket/IsVaraEnabled --args '[]' --idl $IDL
```

Returns `bool`.

## FreebetLedger Queries

### Check native VARA freebet balance

```bash
vara-wallet call $FREEBET_LEDGER FreebetLedger/BalanceOf \
  --args '["'$MY_ADDR'"]' --idl $FREEBET_LEDGER_IDL
```

Returns `u128` raw VARA units. 1 VARA = `1000000000000`.

### Check grant by id

```bash
vara-wallet call $FREEBET_LEDGER FreebetLedger/GetGrant \
  --args '["<grant_id>"]' --idl $FREEBET_LEDGER_IDL
```

### Check authorized bet program

```bash
vara-wallet call $FREEBET_LEDGER FreebetLedger/IsBetProgramAuthorized \
  --args '["'$BASKET_MARKET'"]' --idl $FREEBET_LEDGER_IDL
```

If this returns `false`, agents must not attempt `SpendFreebet`.
