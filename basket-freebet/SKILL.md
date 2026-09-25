---
name: basket-freebet
description: >-
  Use when the agent needs to place or claim native VARA freebet bets through FreebetLedger: BalanceOf, SpendFreebet, GetFreebetPositions, Vara baskets, and principal-return/profit-only claims.
---

# Basket Freebet

Place native VARA freebet bets through `FreebetLedger/SpendFreebet`.

This is the current non-withdrawable VARA credit path. A gas voucher pays fees but does not increase the FreebetLedger balance.
- **Native VARA freebet lane**: existing `FreebetLedger` balance -> `FreebetLedger/SpendFreebet` -> `BasketMarket` records a freebet position.

Freebet principal is non-withdrawable. On claim, the principal returns to `FreebetLedger`; only profit above the stake is paid to the user's wallet.

## Setup

**MAINNET ONLY.** Run `vara-wallet config set network mainnet` before anything else. NEVER switch to testnet.

```bash
vara-wallet config set network mainnet

BASKET_MARKET="0xa749ccd80d71637b450789e12e3d94524e9ae17877d1b59f5ddda784f89a2cba"
FREEBET_LEDGER="0x2bb74834402fb7da9144d2ab91c1570e97237ad0ead1f7feb392162c3e3ad64e"
VOUCHER_URL="https://voucher-backend-production-5a1b.up.railway.app/voucher"
BET_QUOTE_URL="https://bet-quote-service-production.up.railway.app"
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
    -d '{"account":"'"$MY_ADDR"'","programs":["'"$BASKET_MARKET"'","'"$FREEBET_LEDGER"'"]}')
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
  --args '["'$MY_ADDR'"]' --idl $FREEBET_LEDGER_IDL | jq -r 'if (.result|type)=="object" then (.result.value // .result.ok // 0) else (.result // 0) end')
echo "Freebet balance raw: $FREEBET_BALANCE"
```

Native VARA uses 12 decimals. `100000000000000` = 100 VARA freebet.

Agents cannot self-grant freebet. `FreebetLedger/Grant` is admin-only and must attach native VARA value; normal agents only read `BalanceOf` and call `SpendFreebet`.

**Claiming needs this exact wallet.** `BasketMarket/Claim` resolves the owner
from the caller and takes no owner argument, so only the address holding a
position can claim it. On the freebet path the principal returns to the ledger on
claim and only profit reaches the wallet, so losing the key forfeits profit on
credit that was never withdrawable. That is a real cost but not the operator's
principal, so a freebet bet from a short-lived wallet is still worth placing.
Real wallet VARA is different: never stake it from a key that will not survive,
because the principal goes with it.

**When the balance is zero.** There is no funding call to fall back to, so stop
and hand this back to the operator rather than searching for one. Credit comes
from the weekly tasks at https://app.polybaskets.xyz/rewards: a repost pays 100
VARA of credit and a quote-tweet pays 300, each claimable once per wallet and
once per X account per week, so 400 VARA per week in total. Both are verified
against a real X post, so the operator has to do them. Referrals pay a further
200 to 400 VARA when an invited friend reaches 50 and then 500 transactions.
Report the zero balance, name that page, and wait. Do not create unstaked
baskets to appear active and do not attempt the retired CHIP lane.

### 3. Pick an eligible basket

Native freebet bets only work on active `asset_kind: "Vara"` baskets and only while VARA support is enabled.

```bash
vara-wallet call $BASKET_MARKET BasketMarket/IsVaraEnabled \
  --args '[]' --idl $IDL

vara-wallet call $BASKET_MARKET BasketMarket/GetBasket \
  --args '[<BASKET_ID>]' --idl $IDL | jq '(.result.value // .result.ok) | {id, name, status:(.status.kind // .status), asset_kind:(.asset_kind.kind // .asset_kind), items}'
```

Required:
- `IsVaraEnabled` is `true`.
- basket `status` is `"Active"`.
- basket `asset_kind` is `"Vara"`.

If `BasketMarket/IsVaraEnabled` is false, stop; freebet spending is unavailable.

## Signed VARA quote

The deployed `FreebetLedger/SpendFreebet` accepts a `SignedVaraBetQuote`, not a manually computed index. Request it from `/api/basket-market/quote` for the **BasketMarket** target, then spend it before the short quote deadline.

## Spend Freebet

Use raw planck units for the amount:
- 5 VARA = `"5000000000000"`
- 10 VARA = `"10000000000000"`
- 100 VARA = `"100000000000000"`

The ledger debits the caller's freebet balance and forwards the amount as native value to `BasketMarket/BetOnBasketFromFreebetLedger`.

```bash
BASKET_ID=<BASKET_ID>
FREEBET_AMOUNT="10000000000000" # 10 VARA freebet
QUOTE=$(curl -s -X POST "$BET_QUOTE_URL/api/basket-market/quote" \
  -H 'Content-Type: application/json' \
  -d '{"user":"'"$MY_ADDR"'","basketId":'"$BASKET_ID"',"amount":"'"$FREEBET_AMOUNT"'","targetProgramId":"'"$BASKET_MARKET"'"}')
echo "$QUOTE" | jq -e '.payload and .signature' >/dev/null 2>&1 || { echo "Quote failed: $QUOTE"; exit 1; }
BASKET_CHECK=$(vara-wallet call "$BASKET_MARKET" BasketMarket/GetBasket \
  --args "[$BASKET_ID]" --idl "$IDL" | jq -r '(.result.value // .result.ok) | [(.status.kind // .status), (.asset_kind.kind // .asset_kind)] | @tsv')
[ "$BASKET_CHECK" = "$(printf 'Active\tVara')" ] || { echo "Basket is now $BASKET_CHECK; choose another"; exit 1; }
EST=$(vara-wallet --account agent call $FREEBET_LEDGER FreebetLedger/SpendFreebet \
  --args "[\"$BASKET_MARKET\", $BASKET_ID, \"$FREEBET_AMOUNT\", $QUOTE]" \
  --voucher $VOUCHER_ID --idl $FREEBET_LEDGER_IDL --estimate) && \
GAS_LIMIT=$(node -e 'const x=JSON.parse(process.argv[1]); const used=BigInt(x.min_limit??x.minLimit??x.gas_for_reply??x.gasForReply??0); const withBuffer=used + used/5n + 5000000000n; console.log(withBuffer.toString())' "$EST") && \
vara-wallet --account agent call $FREEBET_LEDGER FreebetLedger/SpendFreebet \
  --args "[\"$BASKET_MARKET\", $BASKET_ID, \"$FREEBET_AMOUNT\", $QUOTE]" \
  --voucher $VOUCHER_ID --gas-limit $GAS_LIMIT --idl $FREEBET_LEDGER_IDL
```

If the mutation returns `0`, treat it as downstream bet failure and verify state. The ledger restores the freebet balance on downstream failure.

## Verify Position

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetFreebetPositions \
  --args '["'$MY_ADDR'"]' --idl $IDL | jq '(if (.result|type)=="object" then (.result.value // .result.ok // []) else (.result // []) end)[] | select(.basket_id == '$BASKET_ID')'

vara-wallet call $FREEBET_LEDGER FreebetLedger/BalanceOf \
  --args '["'$MY_ADDR'"]' --idl $FREEBET_LEDGER_IDL
```

If no freebet position exists after a failed or ambiguous send, check `BalanceOf` and the on-chain basket status. Request a **new quote** before one retry; the old quote may have expired. Never retry `BasketNotActive` on the same basket. Do not blind-loop `SpendFreebet`; the ledger rejects concurrent `(user, basket, program)` spends with `OperationInProgress`.

## Claim Freebet Basket Profit

Freebet positions are claimed through `BasketMarket/Claim`, the same method as native VARA positions.

```bash
vara-wallet call $BASKET_MARKET BasketMarket/GetSettlement \
  --args "[$BASKET_ID]" --idl $IDL | jq '(.result.value // .result.ok) | .status.kind // .status'

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
  --args '["'$MY_ADDR'"]' --idl $IDL | jq '(if (.result|type)=="object" then (.result.value // .result.ok // []) else (.result // []) end)[] | select(.basket_id == '$BASKET_ID')'

vara-wallet call $FREEBET_LEDGER FreebetLedger/BalanceOf \
  --args '["'$MY_ADDR'"]' --idl $FREEBET_LEDGER_IDL
```

## Full Agent Flow

1. Set mainnet, variables, `MY_ADDR`, and voucher with `FREEBET_LEDGER` included.
2. Claim any finalized native freebet profits first using `BasketMarket/Claim`.
3. Read `FreebetLedger/BalanceOf`; stop if balance is zero or below intended size.
4. Scan/create only `asset_kind: "Vara"` baskets, and confirm `IsVaraEnabled=true`.
5. Form a market thesis from Gamma market descriptions and resolution criteria.
6. Request a fresh signed BasketMarket quote immediately before each bet.
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
| `InsufficientBalance` | Freebet balance is lower than amount | Lower the amount, or stop and tell the operator to earn credit at app.polybaskets.xyz/rewards |
| `BetProgramNotAuthorized` | BasketMarket not authorized in ledger | Stop and report ops/config issue |
| `OperationInProgress` | Pending spend for same user/program/basket | Wait, query freebet position and balance, retry once only if unchanged |
| `DownstreamBetFailed` / result `0` | BasketMarket rejected the bet | Check basket status, asset kind, VARA enabled, and index |
| `InvalidFreebetAmount` | Amount is zero or no value reached BasketMarket | Use non-zero raw amount |
| `BasketAssetMismatch` | Basket is legacy `Bet`, not `Vara` | Choose a `Vara` basket |
| `VaraDisabled` | Native VARA lane disabled | Stop; freebet spending cannot work on this deployment |
| `InvalidIndexAtCreation` | Signed quote has an invalid quoted index | Request a fresh quote; never invent the index locally |
| `FreebetLedgerNotConfigured` | BasketMarket has no ledger configured or caller is not ledger | Stop and report config issue |
| `FreebetLedgerReturnFailed` | Claim could not return principal to ledger | Stop and report; do not retry blindly |
