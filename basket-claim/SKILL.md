---
name: basket-claim
description: Use when the agent needs to claim payout from a settled basket via vara-wallet. Do not use before settlement is finalized.
---

# Basket Claim

Claim payout from a settled PolyBaskets basket via `vara-wallet`.

## Setup

**MAINNET ONLY.** Run `vara-wallet config set network mainnet` before anything else. NEVER switch to testnet — there are no contracts there.

```bash
vara-wallet config set network mainnet
BASKET_MARKET="0xa749ccd80d71637b450789e12e3d94524e9ae17877d1b59f5ddda784f89a2cba"
BET_LANE="0x35848dea0ab64f283497deaff93b12fe4d17649624b2cd5149f253ef372b29dc"
FREEBET_LEDGER="0x2bb74834402fb7da9144d2ab91c1570e97237ad0ead1f7feb392162c3e3ad64e"
_PB="${POLYBASKETS_SKILLS_DIR:-skills}"
IDL="$_PB/idl/polymarket-mirror.idl"
BET_LANE_IDL="$_PB/idl/bet_lane_client.idl"
FREEBET_LEDGER_IDL="$_PB/idl/freebet-ledger.idl"
```

## Pre-Check Workflow

### 1. Verify settlement is finalized

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetSettlement \
  --args '[<basket_id>]' --idl $IDL
```

Check the result:
- `status: "Finalized"` — ready to claim
- `status: "Proposed"` — challenge window not yet passed, or nobody has finalized yet. Anyone can call `FinalizeSettlement` after `challenge_deadline`; before that, wait.
- Error `SettlementNotFound` — not yet settled

```bash
# Parse settlement status
vara-wallet call $BASKET_MARKET BasketMarket/GetSettlement \
  --args '[<basket_id>]' --idl $IDL | jq '.result.ok.status'
```

Get your hex address first (SS58 won't work for actor_id args):
```bash
MY_ADDR=$(vara-wallet balance | jq -r .address)
```

### 2. Verify position exists and is unclaimed

```bash
# VARA lane
vara-wallet call $BASKET_MARKET BasketMarket/GetPositions \
  --args '["'$MY_ADDR'"]' --idl $IDL | jq '.result[] | select(.basket_id == <basket_id>)'

# native VARA freebet lane
vara-wallet call $BASKET_MARKET BasketMarket/GetFreebetPositions \
  --args '["'$MY_ADDR'"]' --idl $IDL | jq '.result[] | select(.basket_id == <basket_id>)'

# BET lane
vara-wallet call $BET_LANE BetLane/GetPosition \
  --args '["'$MY_ADDR'", <basket_id>]' --idl $BET_LANE_IDL
```

Check `claimed: false`.

## Claim (VARA Lane)

For baskets with `asset_kind: "Vara"`:

```bash
vara-wallet --account agent call $BASKET_MARKET BasketMarket/Claim --voucher $VOUCHER_ID \
  --args '[<basket_id>]' --idl $IDL
```

Returns `u128` — payout amount in minimal VARA units (divide by 10^12 for VARA).

For native VARA freebet positions, use the same `BasketMarket/Claim` method. The contract returns the freebet principal to `FreebetLedger` and sends only profit above the original stake to the user's wallet.

### Example

```bash
# Claim from basket 0
PAYOUT=$(vara-wallet --account agent call $BASKET_MARKET BasketMarket/Claim --voucher $VOUCHER_ID \
  --args '[0]' --idl $IDL)
echo "Payout: $PAYOUT"
```

## Claim (BET Token Lane)

For baskets with `asset_kind: "Bet"`:

```bash
vara-wallet --account agent call $BET_LANE BetLane/Claim --voucher $VOUCHER_ID \
  --args '[<basket_id>]' --idl $BET_LANE_IDL
```

Returns `u256` — payout amount in BET token units.

### Example

```bash
# Claim from basket 1 via BET lane
vara-wallet --account agent call $BET_LANE BetLane/Claim --voucher $VOUCHER_ID \
  --args '[1]' --idl $BET_LANE_IDL
```

## Payout Calculation

```
payout = shares * (settlement_index / entry_index)
```

The `payout_per_share` is pre-computed in the Settlement struct during proposal. You can preview your expected payout before claiming:

```bash
# Get settlement payout_per_share
SETTLEMENT=$(vara-wallet call $BASKET_MARKET BasketMarket/GetSettlement \
  --args '[<basket_id>]' --idl $IDL)
echo $SETTLEMENT | jq '.result.ok.payout_per_share'
```

See `../references/index-math.md` for detailed formula and examples.

## Verify After Claim

```bash
# Check position is now claimed
vara-wallet call $BASKET_MARKET BasketMarket/GetPositions \
  --args '["'$MY_ADDR'"]' --idl $IDL | jq '.result[] | select(.basket_id == <basket_id>) | .claimed'

# Check native freebet position is now claimed, when applicable
vara-wallet call $BASKET_MARKET BasketMarket/GetFreebetPositions \
  --args '["'$MY_ADDR'"]' --idl $IDL | jq '.result[] | select(.basket_id == <basket_id>) | .claimed'

# Check freebet principal was returned to ledger, when applicable
vara-wallet call $FREEBET_LEDGER FreebetLedger/BalanceOf \
  --args '["'$MY_ADDR'"]' --idl $FREEBET_LEDGER_IDL

# Check VARA balance increased
vara-wallet balance
```

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `SettlementNotFinalized` | Settlement not yet finalized | Wait for finalization |
| `AlreadyClaimed` | Already claimed this basket | No action needed |
| `NothingToClaim` | No position in this basket | Verify position exists |
| `SettlementNotFound` | No settlement proposed | Wait for settler to propose |
| `TransferFailed` | VARA transfer failed | Check contract balance, retry |
