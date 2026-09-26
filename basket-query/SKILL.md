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
DAILY_CONTEST="0x1f320a71665f990701daf3862aa4cbb98943859a726c2b497bd053e120149a77"
DAILY_CONTEST_IDL="$_PB/idl/daily-contest.idl"
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

## DailyContest Queries

The contest program settles each UTC day a few minutes after 00:00 UTC, pays the
top three, and stores the result. These reads are the record of what it paid. Use
them, not the app's winners panel, when reporting who won: that panel has shown
wrong wallets with a null reward while the contract paid the correct ones.

A day id is whole UTC days since the Unix epoch: `$(( $(date -u +%s) / 86400 ))`
is today, and today is never settled yet.

### Who won a day, and was it me

```bash
DAY=${DAY:-$(( $(date -u +%s) / 86400 - 1 ))}   # yesterday in UTC
vara-wallet call $DAILY_CONTEST DailyContest/GetDay --args "[$DAY]" --idl $DAILY_CONTEST_IDL \
  | jq -r --arg me "$MY_ADDR" '
      def vara: tostring | if length > 12 then .[:-12] else "0" end;
      .result | if .kind == "Ok" then
        (.value.winners | to_entries[] |
          "#\(.key + 1)  \(.value.account)  profit \(.value.realized_profit | vara) VARA  paid \(.value.reward | vara) VARA" +
          (if .value.account == $me then "  <- you" else "" end))
      else "not settled yet: \(.value.kind)" end'
```

`Err` with `DayNotFound` means the day has not been settled. A settled day with
no positive score returns `status: NoWinner` and an empty winner list. Amounts are
raw 12-decimal units; the jq above trims them to whole VARA. Do not convert with
`tonumber`, which loses precision above 2^53.

### Prize amounts and what is left to pay

```bash
vara-wallet call $DAILY_CONTEST DailyContest/GetConfig --args '[]' --idl $DAILY_CONTEST_IDL | jq '.result.prize_payouts'
vara-wallet call $DAILY_CONTEST DailyContest/GetRewardPool --args '[]' --idl $DAILY_CONTEST_IDL
```
