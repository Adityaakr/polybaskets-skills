---
name: basket-claim
description: Use when the agent needs to claim a finalized native VARA or freebet payout from BasketMarket via vara-wallet. Do not use before settlement finalizes.
---

# Claim a VARA Basket

Use Vara mainnet and the BasketMarket IDL. Legacy `Bet` baskets are not part of the current claim workflow.

```bash
vara-wallet config set network mainnet
BASKET_MARKET="0xa749ccd80d71637b450789e12e3d94524e9ae17877d1b59f5ddda784f89a2cba"
FREEBET_LEDGER="0x2bb74834402fb7da9144d2ab91c1570e97237ad0ead1f7feb392162c3e3ad64e"
_PB="${POLYBASKETS_SKILLS_DIR:-skills}"
IDL="$_PB/idl/polymarket-mirror.idl"
FREEBET_LEDGER_IDL="$_PB/idl/freebet-ledger.idl"
MY_ADDR=$(vara-wallet balance --account agent | jq -r .address)
BASKET_ID=<BASKET_ID>
```

First check the on-chain basket is `Vara` and the settlement is `Finalized`. `vara-wallet` 0.20+ returns successful `Result` payloads under `.result.value` (older versions may use `.result.ok`); enum fields may be objects with a `.kind` key.

```bash
vara-wallet call "$BASKET_MARKET" BasketMarket/GetBasket \
  --args "[$BASKET_ID]" --idl "$IDL" \
  | jq '(.result.value // .result.ok) | {status:(.status.kind // .status), asset_kind:(.asset_kind.kind // .asset_kind)}'
vara-wallet call "$BASKET_MARKET" BasketMarket/GetSettlement \
  --args "[$BASKET_ID]" --idl "$IDL" \
  | jq '(.result.value // .result.ok) | {status:(.status.kind // .status), payout_per_share}'
```

Check `BasketMarket/GetPositions` and `BasketMarket/GetFreebetPositions` for this user and ID. If neither contains an unclaimed position, stop; basket creation alone gives no payout.

```bash
vara-wallet call "$BASKET_MARKET" BasketMarket/GetPositions \
  --args '["'"$MY_ADDR"'"]' --idl "$IDL" \
  | jq --arg id "$BASKET_ID" '.result[] | select((.basket_id|tostring) == $id)'
vara-wallet call "$BASKET_MARKET" BasketMarket/GetFreebetPositions \
  --args '["'"$MY_ADDR"'"]' --idl "$IDL" \
  | jq --arg id "$BASKET_ID" '.result[] | select((.basket_id|tostring) == $id)'
```

When the settlement is finalized and the position is unclaimed, call `BasketMarket/Claim` with a valid gas voucher covering BasketMarket. For a freebet position, principal returns to FreebetLedger and only profit above the stake reaches the wallet.

```bash
vara-wallet --account agent call "$BASKET_MARKET" BasketMarket/Claim \
  --args "[$BASKET_ID]" --voucher "$VOUCHER_ID" --idl "$IDL"
```

Afterward, re-read the position and, for freebets, `FreebetLedger/BalanceOf`. Do not retry an ambiguous claim before checking chain state. `SettlementNotFinalized` means wait; `AlreadyClaimed` and `NothingToClaim` mean no further claim should be sent.
