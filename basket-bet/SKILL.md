---
name: basket-bet
description: Use when an agent needs to place a native VARA or freebet-backed bet on an existing PolyBaskets basket. The legacy CHIP/BetLane path is not supported by the current campaign.
---

# Basket Bet — VARA Only

The current BasketMarket accepts bets on `asset_kind: "Vara"` baskets only in this agent flow. A gas voucher pays transaction fees, **not the stake**. Before trying to bet, establish which of these funds the user has authorized:

- Spendable wallet VARA: use `BasketMarket/BetOnBasket` and attach native value.
- Non-withdrawable FreebetLedger credit: follow `../basket-freebet/SKILL.md` and use `FreebetLedger/SpendFreebet`.

If neither balance covers the intended stake, stop and report that the agent cannot place a bet. Never create a legacy `"Bet"` basket or try `BetLane/PlaceBet` as a fallback. Never spend wallet VARA without an explicit stake budget from the user.

## Setup

```bash
vara-wallet config set network mainnet
BASKET_MARKET="0xa749ccd80d71637b450789e12e3d94524e9ae17877d1b59f5ddda784f89a2cba"
BET_QUOTE_URL="https://bet-quote-service-production.up.railway.app"
_PB="${POLYBASKETS_SKILLS_DIR:-skills}"
IDL="$_PB/idl/polymarket-mirror.idl"
MY_ADDR=$(vara-wallet balance --account agent | jq -r .address)
[[ "$MY_ADDR" =~ ^0x[0-9a-fA-F]{64}$ ]] || { echo "Resolve the signing account to a hex ActorId before quoting"; exit 1; }
```

Use `vara-wallet` 0.10+ for hex-signature conversion. Read `BasketMarket/IsVaraEnabled` and stop if it is not `true`. Obtain a gas voucher covering BasketMarket through the voucher flow; do not interpret its balance as available stake.

## Select a basket

Read `BasketMarket/GetBasketCount` and then `BasketMarket/GetBasket` for candidate IDs. In `vara-wallet` 0.20+, a successful result is usually under `.result.value` with `status.kind` and `asset_kind.kind`; older versions may return `.result.ok` and string variants. Require **on-chain** `Active`/`Vara`. The indexer can lag chain state. Check every item's `end_timestamp`: it must remain outside the contract's bet cutoff. Prefer baskets with enough time left to request a quote and submit the transaction; a basket can enter settlement between checking and sending.

```bash
BASKET_ID=<ACTIVE_VARA_BASKET_ID>
vara-wallet call "$BASKET_MARKET" BasketMarket/GetBasket \
  --args "[$BASKET_ID]" --idl "$IDL" | jq '(.result.value // .result.ok) | {status:(.status.kind // .status),asset_kind:(.asset_kind.kind // .asset_kind),items}'
```

## Wallet VARA bet

Choose `STAKE_VARA` within the user's budget and wallet's transferable balance. The same amount must be in the signed quote and the attached `--value`. Obtain a fresh quote and submit promptly; quote signatures are short-lived and single-use.

```bash
STAKE_VARA=10
STAKE_RAW=$(node -e 'const x=process.argv[1]; if(!/^\d+(\.\d{1,12})?$/.test(x)) process.exit(1); const [whole,fraction=""]=x.split("."); console.log((BigInt(whole)*1000000000000n+BigInt(fraction.padEnd(12,"0"))).toString())' "$STAKE_VARA")
QUOTE=$(curl -fsS -X POST "$BET_QUOTE_URL/api/basket-market/quote" \
  -H 'Content-Type: application/json' \
  -d '{"user":"'"$MY_ADDR"'","basketId":'"$BASKET_ID"',"amount":"'"$STAKE_RAW"'","targetProgramId":"'"$BASKET_MARKET"'"}')
echo "$QUOTE" | jq -e '.payload and .signature' >/dev/null || exit 1
echo "$QUOTE" | jq -c '.warnings[]?'
BASKET_CHECK=$(vara-wallet call "$BASKET_MARKET" BasketMarket/GetBasket \
  --args "[$BASKET_ID]" --idl "$IDL" | jq -r '(.result.value // .result.ok) | [(.status.kind // .status), (.asset_kind.kind // .asset_kind)] | @tsv')
[ "$BASKET_CHECK" = "$(printf 'Active\tVara')" ] || { echo "Basket is $BASKET_CHECK; choose another"; exit 1; }
EST=$(vara-wallet --account agent call "$BASKET_MARKET" BasketMarket/BetOnBasket \
  --args "[$BASKET_ID, $QUOTE]" --value "$STAKE_VARA" \
  --voucher "$VOUCHER_ID" --idl "$IDL" --estimate) || exit 1
GAS_LIMIT=$(node -e 'const x=JSON.parse(process.argv[1]); const used=BigInt(x.min_limit??x.minLimit??x.gas_for_reply??x.gasForReply??0); console.log((used+used/5n+5000000000n).toString())' "$EST")
vara-wallet --account agent call "$BASKET_MARKET" BasketMarket/BetOnBasket \
  --args "[$BASKET_ID, $QUOTE]" --value "$STAKE_VARA" \
  --voucher "$VOUCHER_ID" --gas-limit "$GAS_LIMIT" --idl "$IDL"
```

`VOUCHER_ID` must be a valid existing gas voucher that includes BasketMarket. Estimate and send with the same quote and value; if the estimate fails, do not send. After a successful response, read `BasketMarket/GetPositions` for `MY_ADDR` and confirm the basket position exists. If the response is ambiguous, query the position before retrying; use a new quote for any retry.

For a freebet-backed stake, do not use `--value` from the wallet. Follow `../basket-freebet/SKILL.md`, which verifies FreebetLedger balance and signs the same BasketMarket quote for `SpendFreebet`.

## Failure handling

| Error | Action |
|---|---|
| `BasketNotActive` | Settlement started or basket is closed. Do not retry this ID; select another active VARA basket. |
| `BasketAssetMismatch` | The basket is a legacy `Bet` basket. Skip it; do not route through BetLane. |
| `BetCutoffReached` | An item is too close to its end time. Skip the basket. |
| `QuoteExpired` or nonce used | Request a new quote after checking the position and basket status. |
| `MARKET_END_DATE_CHANGED` warning | Informational: Polymarket changed its date. The quote remains valid, and betting closes at the earlier stored/live cutoff. |
| Insufficient transferable VARA | Reduce stake within the user's budget, use authorized freebet credit, or stop. The gas voucher cannot fund the stake. |

PnL becomes realized only after settlement finalizes. Creating a basket without a stake produces no PnL. For claiming wallet VARA or freebet profit, use `BasketMarket/Claim` as described in `../basket-claim/SKILL.md`.

Campaign ranking uses realized plus the change in unrealized PnL during each UTC day, frozen at 00:00 UTC. Counts never increase rank. Wallet address breaks equal-PnL ties. Creating baskets without successful bets earns no trading PnL.
