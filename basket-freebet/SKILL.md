---
name: basket-freebet
description: Use when the agent needs to place or claim native VARA freebet bets through FreebetLedger. This is NOT the CHIP/BetLane flow: use it for FreebetLedger BalanceOf, SpendFreebet, GetFreebetPositions, Vara baskets, and the principal-return/profit-only claim model.
---

# Basket Freebet

Place native VARA freebet bets through `FreebetLedger/SpendFreebet`.

This path is separate from the Season 3 CHIP lane:
- **CHIP lane**: `BetToken/Claim` -> `BetToken/Approve` -> `BetLane/PlaceBet`.
- **Native VARA freebet lane**: existing `FreebetLedger` balance -> `FreebetLedger/SpendFreebet` -> `BasketMarket` records a freebet position.

Freebet principal is non-withdrawable. On claim, the principal returns to `FreebetLedger`; only profit above the stake is paid to the user's wallet.

## Setup

**MAINNET ONLY.** Run `vara-wallet config set network mainnet` before anything else. NEVER switch to testnet.

```bash
vara-wallet config set network mainnet

BASKET_MARKET="0xa749ccd80d71637b450789e12e3d94524e9ae17877d1b59f5ddda784f89a2cba"
BET_TOKEN="0x186f6cda18fea13d9fc5969eec5a379220d6726f64c1d5f4b346e89271f917bc"
BET_LANE="0x35848dea0ab64f283497deaff93b12fe4d17649624b2cd5149f253ef372b29dc"
FREEBET_LEDGER="0x2bb74834402fb7da9144d2ab91c1570e97237ad0ead1f7feb392162c3e3ad64e"
VOUCHER_URL="https://voucher-backend-production-5a1b.up.railway.app/voucher"
_PB="${POLYBASKETS_SKILLS_DIR:-skills}"
IDL="$_PB/idl/polymarket-mirror.idl"
FREEBET_LEDGER_IDL="$_PB/idl/freebet-ledger.idl"

MY_ADDR=$(vara-wallet balance --account agent | jq -r .address)
if [ -z "$MY_ADDR" ] || [ "$MY_ADDR" = "null" ]; then
  echo "Failed to resolve agent wallet address; aborting before voucher request."
  exit 1
fi
```

`MY_ADDR` must be the hex actor id (`0x...`), not SS58.

## Gas Voucher

For freebet sessions, make sure the voucher covers `FREEBET_LEDGER` too. Use one batched POST with all likely session programs so later calls do not fail because the voucher whitelist is incomplete.

```bash
VOUCHER_STATE_URL="$VOUCHER_URL/$MY_ADDR"
VOUCHER_STATE=$(curl -s "$VOUCHER_STATE_URL")
VOUCHER_ID=$(echo "$VOUCHER_STATE" | jq -r .voucherId)
CAN_TOP_UP=$(echo "$VOUCHER_STATE" | jq -r .canTopUpNow)
VARA_BALANCE=$(echo "$VOUCHER_STATE" | jq -r .varaBalance)
BALANCE_KNOWN=$(echo "$VOUCHER_STATE" | jq -r .balanceKnown)
NEXT_ELIGIBLE=$(echo "$VOUCHER_STATE" | jq -r .nextTopUpEligibleAt)
HAS_FREEBET_LEDGER=$(echo "$VOUCHER_STATE" | jq -r --arg p "$(echo "$FREEBET_LEDGER" | tr '[:upper:]' '[:lower:]')" '(.programs // []) | map(ascii_downcase) | index($p) != null')
HAS_BASKET_MARKET=$(echo "$VOUCHER_STATE" | jq -r --arg p "$(echo "$BASKET_MARKET" | tr '[:upper:]' '[:lower:]')" '(.programs // []) | map(ascii_downcase) | index($p) != null')
LOW_VOUCHER_BALANCE="10000000000000" # 10 VARA in planck
NEED_TOP_UP=false
if [ "$BALANCE_KNOWN" = "true" ] && [ "$VARA_BALANCE" -lt "$LOW_VOUCHER_BALANCE" ]; then
  NEED_TOP_UP=true
fi

if [ "$VOUCHER_ID" = "null" ] || [ "$HAS_FREEBET_LEDGER" != "true" ] || [ "$HAS_BASKET_MARKET" != "true" ] || { [ "$NEED_TOP_UP" = "true" ] && [ "$CAN_TOP_UP" = "true" ]; }; then
  RESP=$(curl -s -w "\n%{http_code}" -X POST "$VOUCHER_URL" \
    -H 'Content-Type: application/json' \
    -d '{"account":"'"$MY_ADDR"'","programs":["'"$BASKET_MARKET"'","'"$BET_TOKEN"'","'"$BET_LANE"'","'"$FREEBET_LEDGER"'"]}')
  HTTP_CODE=$(echo "$RESP" | tail -n1)
  BODY=$(echo "$RESP" | sed '$d')
  case "$HTTP_CODE" in
    200) VOUCHER_ID=$(echo "$BODY" | jq -r .voucherId) ;;
    429) echo "Voucher rate-limited. Reusing existing voucherId from GET." ;;
    *) echo "Voucher POST failed: HTTP $HTTP_CODE - $BODY" && exit 1 ;;
  esac
fi
echo "Voucher: $VOUCHER_ID"
```

STOP if `BALANCE_KNOWN=true`, voucher balance is below 10 VARA, and `CAN_TOP_UP=false`. Never spend wallet-owned VARA for gas unless the user explicitly approves it in the current session.

## Pre-Checks

### 1. Confirm FreebetLedger is configured

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetFreebetLedger \
  --args '[]' --idl $IDL

vara-wallet call $FREEBET_LEDGER FreebetLedger/IsBetProgramAuthorized \
  --args '["'$BASKET_MARKET'"]' --idl $FREEBET_LEDGER_IDL
```

Expected:
- `GetFreebetLedger` equals `$FREEBET_LEDGER`.
- `IsBetProgramAuthorized` returns `true`.

If either check fails, do not place a freebet. Report the configuration mismatch.

### 2. Check native freebet balance

```bash
FREEBET_BALANCE=$(vara-wallet call $FREEBET_LEDGER FreebetLedger/BalanceOf \
  --args '["'$MY_ADDR'"]' --idl $FREEBET_LEDGER_IDL | jq -r '.result')
echo "Freebet balance raw: $FREEBET_BALANCE"
```

Native VARA uses 12 decimals. `100000000000000` = 100 VARA freebet.

Agents cannot self-grant freebet. `FreebetLedger/Grant` is admin-only and must attach native VARA value; normal agents only read `BalanceOf` and call `SpendFreebet`.

### 3. Pick an eligible basket

Native freebet bets only work on active `asset_kind: "Vara"` baskets and only while VARA support is enabled.

```bash
vara-wallet call $BASKET_MARKET BasketMarket/IsVaraEnabled \
  --args '[]' --idl $IDL

vara-wallet call $BASKET_MARKET BasketMarket/GetBasket \
  --args '[<BASKET_ID>]' --idl $IDL | jq '.result.ok | {id, name, status, asset_kind, items}'
```

Required:
- `IsVaraEnabled` is `true`.
- basket `status` is `"Active"`.
- basket `asset_kind` is `"Vara"`.

If the current deployment is CHIP-only, do not try `SpendFreebet`; use `basket-bet/SKILL.md` instead.

## Compute Entry Index

`FreebetLedger/SpendFreebet` takes `index_at_creation_bps` directly. Do not use the BetLane quote service for this path; it validates `asset_kind: "Bet"` and signs quotes for `BetLane`, not for native freebet.

Compute the current basket index from live Polymarket Gamma prices immediately before sending:

```bash
BASKET_JSON=$(vara-wallet call $BASKET_MARKET BasketMarket/GetBasket \
  --args '[<BASKET_ID>]' --idl $IDL | jq -c '.result.ok')

INDEX_BPS=$(node -e '
const basket = JSON.parse(process.argv[1]);
async function main() {
  let weighted = 0;
  for (const item of basket.items) {
    const res = await fetch(`https://gamma-api.polymarket.com/markets/${item.poly_market_id}`);
    if (!res.ok) throw new Error(`Gamma fetch failed for ${item.poly_market_id}: ${res.status}`);
    const market = await res.json();
    const prices = JSON.parse(market.outcomePrices);
    const selected = String(item.selected_outcome ?? item.selectedOutcome).toUpperCase();
    const probability = Number(selected === "YES" ? prices[0] : prices[1]);
    if (!Number.isFinite(probability) || probability <= 0 || probability > 1) {
      throw new Error(`Invalid ${selected} probability for ${item.poly_market_id}`);
    }
    weighted += Number(item.weight_bps ?? item.weightBps) * probability;
  }
  const bps = Math.max(1, Math.min(10000, Math.round(weighted)));
  console.log(bps);
}
main().catch((err) => { console.error(err.message); process.exit(1); });
' "$BASKET_JSON")

echo "index_at_creation_bps=$INDEX_BPS"
```

The index is a basis-point probability from 1 to 10000. Send the transaction right after computing it; stale prices create a worse entry.

## Spend Freebet

Use raw planck units for the amount:
- 5 VARA = `"5000000000000"`
- 10 VARA = `"10000000000000"`
- 100 VARA = `"100000000000000"`

There is no CHIP approval step. The ledger debits the caller's freebet balance and forwards the amount as native value to `BasketMarket/BetOnBasketFromFreebetLedger`.

```bash
BASKET_ID=<BASKET_ID>
FREEBET_AMOUNT="10000000000000" # 10 VARA freebet

EST=$(vara-wallet --account agent call $FREEBET_LEDGER FreebetLedger/SpendFreebet \
  --args '["'$BASKET_MARKET'", '$BASKET_ID', "'$FREEBET_AMOUNT'", '$INDEX_BPS']' \
  --voucher $VOUCHER_ID --idl $FREEBET_LEDGER_IDL --estimate) && \
GAS_LIMIT=$(node -e 'const x=JSON.parse(process.argv[1]); const used=BigInt(x.min_limit??x.minLimit??x.gas_for_reply??x.gasForReply??0); const withBuffer=used + used/5n + 5000000000n; console.log(withBuffer.toString())' "$EST") && \
vara-wallet --account agent call $FREEBET_LEDGER FreebetLedger/SpendFreebet \
  --args '["'$BASKET_MARKET'", '$BASKET_ID', "'$FREEBET_AMOUNT'", '$INDEX_BPS']' \
  --voucher $VOUCHER_ID --gas-limit $GAS_LIMIT --idl $FREEBET_LEDGER_IDL
```

If the mutation returns `0`, treat it as downstream bet failure and verify state. The ledger restores the freebet balance on downstream failure.

## Verify Position

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetFreebetPositions \
  --args '["'$MY_ADDR'"]' --idl $IDL | jq '.result[] | select(.basket_id == '$BASKET_ID')'

vara-wallet call $FREEBET_LEDGER FreebetLedger/BalanceOf \
  --args '["'$MY_ADDR'"]' --idl $FREEBET_LEDGER_IDL
```

If no freebet position exists after a failed or ambiguous send, recompute `INDEX_BPS`, check `BalanceOf`, and retry once with a higher gas buffer. Do not blind-loop `SpendFreebet`; the ledger rejects concurrent `(user, basket, program)` spends with `OperationInProgress`.

## Claim Freebet Basket Profit

Freebet positions are claimed through `BasketMarket/Claim`, the same method as native VARA positions.

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetSettlement \
  --args "[$BASKET_ID]" --idl $IDL | jq '.result.ok.status'

vara-wallet --account agent call $BASKET_MARKET BasketMarket/Claim \
  --args "[$BASKET_ID]" --voucher $VOUCHER_ID --idl $IDL
```

Claim behavior:
- gross payout is computed from the freebet position.
- up to the original stake is returned to `FreebetLedger`.
- only profit above the original stake is sent to the wallet.
- the freebet position is marked `claimed`.

After claim, verify both:

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetFreebetPositions \
  --args '["'$MY_ADDR'"]' --idl $IDL | jq '.result[] | select(.basket_id == '$BASKET_ID')'

vara-wallet call $FREEBET_LEDGER FreebetLedger/BalanceOf \
  --args '["'$MY_ADDR'"]' --idl $FREEBET_LEDGER_IDL
```

## Full Agent Flow

1. Set mainnet, variables, `MY_ADDR`, and voucher with `FREEBET_LEDGER` included.
2. Claim any finalized native freebet profits first using `BasketMarket/Claim`.
3. Read `FreebetLedger/BalanceOf`; stop if balance is zero or below intended size.
4. Scan/create only `asset_kind: "Vara"` baskets, and confirm `IsVaraEnabled=true`.
5. Form a market thesis from Gamma market descriptions and resolution criteria.
6. Compute `INDEX_BPS` immediately before each bet.
7. Call `FreebetLedger/SpendFreebet` sequentially with explicit `--gas-limit`.
8. Verify `GetFreebetPositions` and ledger balance after every bet.
9. Stop and report freebet balance before/after, baskets bet, amount spent, claims recovered, skipped baskets, and failed operations.

Suggested sizing:
- High conviction: 20-50 VARA freebet.
- Medium conviction: 10 VARA freebet.
- Low conviction: 5 VARA freebet or skip.
- Never spend more than the available `FreebetLedger/BalanceOf`.

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `InsufficientBalance` | Freebet balance is lower than amount | Lower amount or earn/grant more freebet |
| `BetProgramNotAuthorized` | BasketMarket not authorized in ledger | Stop and report ops/config issue |
| `OperationInProgress` | Pending spend for same user/program/basket | Wait, query freebet position and balance, retry once only if unchanged |
| `DownstreamBetFailed` / result `0` | BasketMarket rejected the bet | Check basket status, asset kind, VARA enabled, and index |
| `InvalidFreebetAmount` | Amount is zero or no value reached BasketMarket | Use non-zero raw amount |
| `BasketAssetMismatch` | Basket is `Bet`, not `Vara` | Use a `Vara` basket or CHIP lane instead |
| `VaraDisabled` | Native VARA lane disabled | Stop; freebet spending cannot work on this deployment |
| `InvalidIndexAtCreation` | Index is outside 1..10000 | Recompute and clamp from current Gamma prices |
| `FreebetLedgerNotConfigured` | BasketMarket has no ledger configured or caller is not ledger | Stop and report config issue |
| `FreebetLedgerReturnFailed` | Claim could not return principal to ledger | Stop and report; do not retry blindly |
