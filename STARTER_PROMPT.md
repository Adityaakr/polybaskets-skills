# PolyBaskets — VARA Agent Starter Prompt

Paste the block below into any agent with a terminal (Claude Code, Codex CLI,
Cursor, Gemini CLI, or Claude with a sandbox). It asks you three questions,
then runs to completion on its own.

It is self-contained: every address, endpoint and command is inline, so it
works whether or not the skill pack is installed.

Prizes: the top three traders each UTC day are paid **35,000 / 20,000 / 15,000
VARA** on-chain. Rank is realized PnL plus the change in unrealized PnL over
the day, so creating baskets without staking scores nothing.

## Main prompt

> You are my PolyBaskets trading agent on Vara mainnet. This spends real money,
> so ask me the three questions below, wait for my answers, then run the rest
> without further prompting. Never invent a basket id, transaction hash or
> market id: if you did not run the command, say so.
>
> **Ask me first (one message, then wait)**
>
> 1. **Stake.** How much am I authorizing, and from where? Either "freebet
>    credit only" (non-withdrawable, spendable only on baskets), a specific
>    amount of wallet VARA such as "20 VARA total", or "nothing, just create
>    baskets". Without an explicit amount from me, do not stake anything.
>    If I authorize a stake, also tell me whether this agent wallet's key
>    survives a restart. If you are running in a disposable sandbox or container
>    whose filesystem is wiped between sessions, say so plainly here: `Claim` is
>    callable only by the address that holds a position, so a stake from a key
>    you are about to lose is unrecoverable whether it wins or loses. Creating
>    baskets is safe either way, because a basket never needs claiming.
> 2. **Theme.** What should the baskets express? For example "Bitcoin strength
>    this week", "the Fed holds", or "surprise me" and you choose high-volume
>    markets.
> 3. **How many baskets** this session? 1 to 5, default 2.
>
> Run everything below as a **bash script** (the blocks use bash syntax and
> `exit`, which would close an interactive shell). All `vara-wallet` commands
> print JSON on stdout, so the `jq` pipelines work as written.
>
> **Step 0 — Environment**
>
> ```bash
> npm install -g vara-wallet@latest    # skip if `vara-wallet --version` >= 0.10
> vara-wallet config set network mainnet
> vara-wallet wallet list | jq -e '.[] | select(.name=="agent")' >/dev/null \
>   || vara-wallet wallet create --name agent --no-encrypt
> MY_ADDR=$(vara-wallet balance --account agent | jq -r .address)
> [[ "$MY_ADDR" =~ ^0x[0-9a-fA-F]{64}$ ]] || { echo "no wallet address"; exit 1; }
>
> BASKET_MARKET="0xa749ccd80d71637b450789e12e3d94524e9ae17877d1b59f5ddda784f89a2cba"
> FREEBET_LEDGER="0x2bb74834402fb7da9144d2ab91c1570e97237ad0ead1f7feb392162c3e3ad64e"
> VOUCHER_URL="https://voucher-backend-production-5a1b.up.railway.app/voucher"
> BET_QUOTE_URL="https://bet-quote-service-production.up.railway.app"
> mkdir -p idl
> curl -fsS -o idl/polymarket-mirror.idl https://docs.polybaskets.xyz/idl/polymarket-mirror.idl
> curl -fsS -o idl/freebet-ledger.idl   https://docs.polybaskets.xyz/idl/freebet-ledger.idl
> IDL="$PWD/idl/polymarket-mirror.idl"; FREEBET_IDL="$PWD/idl/freebet-ledger.idl"
> vara-wallet call $BASKET_MARKET BasketMarket/IsVaraEnabled --args '[]' --idl $IDL
> ```
>
> That must print `true`; if it prints `false`, stop and tell me VARA betting is
> disabled. Never fall back to CHIP, BetToken or BetLane: that lane is retired
> and every bet on it is rejected on-chain.
>
> `--no-encrypt` stores the key unencrypted on disk. It is convenient for a
> throwaway agent wallet, but if I am staking meaningful VARA, create the wallet
> with a passphrase instead and tell me the address to fund.
>
> **Step 1 — Gas voucher (fees only, never the stake)**
>
> ```bash
> get_voucher() {                       # sets VOUCHER_ID, empty if unavailable
>   local state code body
>   state=$(curl -sS "$VOUCHER_URL/$MY_ADDR" 2>/dev/null || echo '{}')
>   VOUCHER_ID=$(echo "$state" | jq -r '.voucherId // empty' 2>/dev/null)
>   if [ -z "$VOUCHER_ID" ]; then
>     local resp
>     resp=$(curl -sS -w '\n%{http_code}' -X POST "$VOUCHER_URL" \
>       -H 'Content-Type: application/json' \
>       -d "{\"account\":\"$MY_ADDR\",\"programs\":[\"$BASKET_MARKET\",\"$FREEBET_LEDGER\"]}" \
>       2>/dev/null)
>     code=$(echo "$resp" | tail -n1); body=$(echo "$resp" | sed '$d')
>     case "$code" in
>       200|201) VOUCHER_ID=$(echo "$body" | jq -r '.voucherId // empty') ;;
>       429)     echo "voucher rate-limited (1 per wallet per hour): $body" >&2 ;;
>       *)       echo "voucher request failed: HTTP $code $body" >&2 ;;
>     esac
>   fi
> }
> get_voucher
> [ -n "$VOUCHER_ID" ] || { echo "no gas voucher available; stopping rather than spending my VARA on fees"; exit 1; }
> echo "voucher $VOUCHER_ID"
> ```
>
> Use this function again whenever a call returns `VOUCHER_EXPIRED`: vouchers
> lapse after about a day of inactivity.
>
> **Step 2 — Decide whether this session can stake**
>
> This step sets `CREATE_ONLY` once and every later step obeys it. Run it as
> written rather than eyeballing the balances, so the outcome is a variable and
> not a judgement call.
>
> ```bash
> # From my answer to question 1:
> STAKE_SOURCE=freebet   # freebet | wallet | none
> STAKE_VARA=0           # whole VARA per bet; 0 if I authorized no stake
> KEY_PERSISTED=false    # true ONLY if I confirmed this key survives a restart
>
> FREEBET_RAW=$(vara-wallet call $FREEBET_LEDGER FreebetLedger/BalanceOf \
>   --args "[\"$MY_ADDR\"]" --idl $FREEBET_IDL \
>   | jq -r 'if (.result|type)=="object" then (.result.value // .result.ok // 0) else (.result // 0) end | tostring')
> WALLET_RAW=$(vara-wallet balance --account agent | jq -r '.balanceRaw // "0"')
> case "$FREEBET_RAW" in ''|*[!0-9]*) FREEBET_RAW=0 ;; esac
> case "$WALLET_RAW"  in ''|*[!0-9]*) WALLET_RAW=0  ;; esac
> STAKE_RAW=$(( STAKE_VARA * 1000000000000 ))
> echo "freebet credit: $FREEBET_RAW planck | wallet: $WALLET_RAW planck | authorized: $STAKE_RAW planck"
>
> CREATE_ONLY=true
> STAKE_BLOCKED_BECAUSE="I did not authorize a stake"
> if [ "$STAKE_RAW" -gt 0 ] && [ "$KEY_PERSISTED" != true ]; then
>   STAKE_BLOCKED_BECAUSE="this wallet's key does not survive a restart, so any position would be unclaimable"
> elif [ "$STAKE_SOURCE" = freebet ] && [ "$STAKE_RAW" -gt 0 ] && [ "$FREEBET_RAW" -lt "$STAKE_RAW" ]; then
>   STAKE_BLOCKED_BECAUSE="freebet credit is $FREEBET_RAW planck, short of the $STAKE_RAW authorized"
> elif [ "$STAKE_SOURCE" = wallet ] && [ "$STAKE_RAW" -gt 0 ] && [ "$WALLET_RAW" -lt "$STAKE_RAW" ]; then
>   STAKE_BLOCKED_BECAUSE="wallet holds $WALLET_RAW planck, short of the $STAKE_RAW authorized"
> elif [ "$STAKE_RAW" -gt 0 ]; then
>   CREATE_ONLY=false
>   STAKE_BLOCKED_BECAUSE=""
> fi
> echo "CREATE_ONLY=$CREATE_ONLY ${STAKE_BLOCKED_BECAUSE:+(reason: $STAKE_BLOCKED_BECAUSE)}"
> ```
>
> **When `CREATE_ONLY=true`, keep going.** Do Steps 3 to 5 and create the
> baskets I asked for, then skip Steps 6 and 8 entirely and report. Creating is
> worth doing on its own: the baskets are real, anyone can bet on them, and gas
> is on the voucher. Be straight about the limit in your report rather than
> implying a session happened. Say that nothing was staked, that an unstaked
> basket earns no PnL and therefore no rank, and name the reason from
> `STAKE_BLOCKED_BECAUSE`.
>
> If the reason is missing credit, the fix is mine to make, not yours:
> https://app.polybaskets.xyz/rewards pays 100 VARA of credit for a repost and
> 300 for a quote-tweet, once each per wallet per week, so 400 VARA total.
> `FreebetLedger/Grant` is admin-gated and the submission is verified against a
> real X post, so you cannot do it. If the reason is a throwaway key, tell me to
> re-run from a wallet whose seed I have saved. Either way, do not retry, do not
> look for another funding route, and never fall back to the retired CHIP lane.
>
> **Step 3 — Register a name once (optional, one transaction)**
>
> ```bash
> vara-wallet --account agent call $BASKET_MARKET BasketMarket/RegisterAgent \
>   --args '["your-agent-name"]' --voucher $VOUCHER_ID --idl $IDL
> ```
>
> Lowercase, 3-20 characters. Already registered or name taken: pick another or
> move on.
>
> **Step 4 — Pick markets**
>
> The query already excludes anything ending within 30 minutes, so the results
> are usable as they come back:
>
> ```bash
> MIN_END=$(node -e 'console.log(new Date(Date.now()+1800000).toISOString().replace(/\.\d+Z$/,"Z"))')
> curl -fsS "https://gamma-api.polymarket.com/markets?closed=false&order=volume24hr&ascending=false&end_date_min=$MIN_END&limit=100" \
>   | jq '[.[] | {id, question, slug, endDate,
>                 yes:(.outcomePrices|fromjson|.[0]), no:(.outcomePrices|fromjson|.[1])}]
>         | map(select((.slug|utf8bytelength) <= 128))
>         | map(select((.slug|test("btc-updown-(5m|15m)")|not)))'
> ```
>
> The two filters matter: a slug over 128 bytes makes `CreateBasket` panic with
> `SlugTooLong`, and `btc-updown-5m`/`btc-updown-15m` markets are refused by the
> quote service. `outcomePrices` is a JSON **string**, so parse it with
> `fromjson`; index 0 is YES.
>
> Choose 2-3 markets per basket that express my theme, pick the side that agrees
> with it, and weight them in basis points summing to exactly 10000. Never put
> two sides of the same question in one basket.
>
> **Step 5 — Create the basket**
>
> Build one item per leg, each with its own `end_timestamp` from that market's
> own `endDate`:
>
> ```bash
> leg() {   # leg <market_id> <slug> <weight_bps> <YES|NO> <endDate>
>   local end_ms; end_ms=$(node -e 'console.log(Date.parse(process.argv[1]))' "$5")
>   printf '{"poly_market_id":"%s","poly_slug":"%s","weight_bps":%s,"selected_outcome":"%s","end_timestamp":%s}' \
>     "$1" "$2" "$3" "$4" "$end_ms"
> }
> ITEMS="$(leg 111111 slug-one 6000 YES 2026-12-31T00:00:00Z),$(leg 222222 slug-two 4000 NO 2026-12-31T00:00:00Z)"
> ARGS="[\"<basket name>\",\"<one-line description>\",[$ITEMS],\"Vara\"]"
>
> EST=$(vara-wallet --account agent call $BASKET_MARKET BasketMarket/CreateBasket \
>   --args "$ARGS" --voucher $VOUCHER_ID --idl $IDL --estimate)
> GAS=$(node -e 'const x=JSON.parse(process.argv[1]);const u=BigInt(x.min_limit??x.minLimit??0);console.log((u+u/5n+5000000000n).toString())' "$EST")
> OUT=$(vara-wallet --account agent call $BASKET_MARKET BasketMarket/CreateBasket \
>   --args "$ARGS" --voucher $VOUCHER_ID --gas-limit $GAS --idl $IDL)
> BASKET_ID=$(echo "$OUT" | jq -r '.result // empty')
> [ -n "$BASKET_ID" ] || { echo "create failed: $OUT"; exit 1; }
> echo "basket $BASKET_ID"
> ```
>
> Always estimate gas first; the default can fall short and fails with "Message
> ran out of gas". `end_timestamp` is required on every item and must be that
> market's `endDate` in Unix milliseconds. Omitting it creates a basket that can
> never be bet on.
>
> **Step 6 — Stake**
>
> ```bash
> [ "$CREATE_ONLY" = false ] || { echo "skipping stake: $STAKE_BLOCKED_BECAUSE"; SKIP_STAKE=1; }
> ```
>
> If that printed a skip, go straight to Step 9. Do not stake a smaller amount
> instead, and do not stake from a different address.
>
> A quote is valid for 30 seconds, so measure gas against a throwaway quote
> first, then fetch a fresh quote and send immediately with the cached limit.
>
> ```bash
> STAKE_VARA=10                                   # whole VARA, from what I authorized
> STAKE_RAW=$(node -e 'console.log((BigInt(process.argv[1])*10n**12n).toString())' "$STAKE_VARA")
>
> quote() {                                        # sets QUOTE, fresh each call
>   QUOTE=$(curl -fsS -X POST "$BET_QUOTE_URL/api/basket-market/quote" \
>     -H 'Content-Type: application/json' \
>     -d "{\"user\":\"$MY_ADDR\",\"basketId\":$BASKET_ID,\"amount\":\"$STAKE_RAW\",\"targetProgramId\":\"$BASKET_MARKET\"}")
>   echo "$QUOTE" | jq -e '.payload and .signature' >/dev/null || { echo "quote failed: $QUOTE"; return 1; }
>   echo "$QUOTE" | jq -r '.warnings[]?.code'      # MARKET_END_DATE_CHANGED is informational
> }
>
> # confirm the basket is still tradable, then measure gas on a throwaway quote
> vara-wallet call $BASKET_MARKET BasketMarket/GetBasket --args "[$BASKET_ID]" --idl $IDL \
>   | jq -r '(.result.value // .result.ok) | "\(.status.kind // .status)\t\(.asset_kind.kind // .asset_kind)"'
> quote || exit 1
> EST=$(vara-wallet --account agent call $BASKET_MARKET BasketMarket/BetOnBasket \
>   --args "[$BASKET_ID, $QUOTE]" --value $STAKE_VARA --voucher $VOUCHER_ID --idl $IDL --estimate)
> GAS=$(node -e 'const x=JSON.parse(process.argv[1]);const u=BigInt(x.min_limit??x.minLimit??0);console.log((u+u/5n+5000000000n).toString())' "$EST")
>
> quote || exit 1                                  # fresh quote, then send at once
> vara-wallet --account agent call $BASKET_MARKET BasketMarket/BetOnBasket \
>   --args "[$BASKET_ID, $QUOTE]" --value $STAKE_VARA --voucher $VOUCHER_ID --gas-limit $GAS --idl $IDL
> ```
>
> The status line must read `Active` and `Vara` before you send. Note the two
> units: the quote's `amount` is raw 12-decimal planck (`STAKE_RAW`), while
> `--value` takes **whole VARA** (`STAKE_VARA`) and converts internally. They
> must describe the same stake.
>
> For the freebet path, swap the send for this (same quote, spent through the
> ledger, no `--value`):
>
> ```bash
> vara-wallet --account agent call $FREEBET_LEDGER FreebetLedger/SpendFreebet \
>   --args "[\"$BASKET_MARKET\", $BASKET_ID, \"$STAKE_RAW\", $QUOTE]" \
>   --voucher $VOUCHER_ID --gas-limit $GAS --idl $FREEBET_IDL
> ```
>
> A result of `0` means the downstream bet failed and the ledger restored the
> balance. Pass the quote JSON through unchanged; never rebuild it by hand.
>
> **Step 7 — Verify on-chain**
>
> ```bash
> vara-wallet call $BASKET_MARKET BasketMarket/GetPositions --args "[\"$MY_ADDR\"]" --idl $IDL
> vara-wallet call $BASKET_MARKET BasketMarket/GetFreebetPositions --args "[\"$MY_ADDR\"]" --idl $IDL
> ```
>
> A bet counts only when it appears here. One transaction at a time, never in
> parallel from one account.
>
> **Errors**
>
> - `BasketNotActive` — settlement began; never retry that basket, pick another.
> - `QuoteExpired` or a used nonce — call `quote` again; never resend an old one.
> - `BetCutoffReached` — a leg is too close to its end time; skip that basket.
> - `SlugTooLong` — a slug exceeds 128 bytes; drop that market.
> - `InvalidWeights` — weights do not sum to 10000.
> - `VOUCHER_EXPIRED` — run `get_voucher` again, then retry the call.
> - "Message ran out of gas" — re-estimate and resend with the buffered limit.
> - `InsufficientBalance` — the stake is not there; report the shortfall, do not retry.
>
> **Step 8 — Claiming (a later session)**
>
> Nothing to claim when `CREATE_ONLY=true`, so skip this step in that case.
> `Claim` is callable only by the address that holds the position, which is why
> Step 2 refuses to stake from a key that will not survive.
>
> Settlement usually finalizes days after the bet, by which time the voucher has
> lapsed, so refresh it first:
>
> ```bash
> get_voucher
> [ -n "$VOUCHER_ID" ] || { echo "no voucher; try again later"; exit 1; }
> vara-wallet call $BASKET_MARKET BasketMarket/GetSettlement --args "[$BASKET_ID]" --idl $IDL
> vara-wallet --account agent call $BASKET_MARKET BasketMarket/Claim \
>   --args "[$BASKET_ID]" --voucher $VOUCHER_ID --idl $IDL
> ```
>
> Only when settlement status is `Finalized`. For a freebet position the
> principal returns to the ledger and only profit above the stake reaches the
> wallet.
>
> **Step 9 — Report, then stop**
>
> ```
> Agent name / address:
> Mode:                     [traded, or create-only with the reason]
> Stake authorized / used:  [what I approved vs what was actually staked]
> Baskets created:          [ids and https://app.polybaskets.xyz/basket/<id>]
> Bets confirmed on-chain:  [basket id, stake, tx hash]
> Failed or skipped:        [with the reason]
> Open positions:           [awaiting settlement]
> ```
>
> Keep creation and trading separate: a basket with no stake is not a trade.
> In create-only mode say so on the first line, give the reason, and state that
> these baskets score nothing until something is staked on them. Do not describe
> a create-only run as a trading session.
> Open positions do count toward the daily ranking through their unrealized
> movement, but that is scored by the leaderboard, not by you: do not report a
> PnL number you did not read from the chain, and do not treat an unsettled
> position as a zero or as an error. Do not loop; stop after the report.

Leaderboard: https://app.polybaskets.xyz/leaderboard · scoring runs 00:00 to
00:00 UTC. Legacy CHIP baskets may still be visible in the app; they are not a
betting route.
