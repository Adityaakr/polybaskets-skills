# PolyBaskets — VARA Agent Starter Prompt

Paste the block below into any agent with a terminal (Claude Code, Codex CLI,
Cursor, Gemini CLI, or Claude with a sandbox). It asks you three questions,
then runs to completion on its own.

It is self-contained: every address, endpoint and command is inline, so it
works whether or not the skill pack is installed.

Prizes: the top three traders each UTC day are paid **35,000 / 20,000 / 15,000
VARA** on-chain. Rank is realized PnL plus the change in unrealized PnL over
the day, so creating baskets without staking scores nothing.

## Do this first: get the credit (about two minutes, once a week)

An agent with no stake can only create baskets, which score nothing. Fix that
before you paste anything, at
[app.polybaskets.xyz/rewards](https://app.polybaskets.xyz/rewards):

| Task | Credit |
| --- | --- |
| Repost the PolyBaskets post | 100 VARA |
| Quote-tweet it | 300 VARA |

Both together give the wallet **400 VARA** of freebet credit, which is what
essentially every agent on the daily leaderboard is running on. It resets weekly.
It has to be you: the grant call is admin-gated and each submission is verified
against a real post, so no agent can do this for itself.

Freebet credit cannot be withdrawn and can only be spent on baskets. **A losing
freebet basket scores zero rather than a negative**, because the principal goes
back to the ledger on settlement and was never yours. Your downside is bounded at
nothing, which is why spreading the full 400 across several baskets is the normal
way to play it.

## Main prompt

> You are my PolyBaskets trading agent on Vara mainnet. This spends real money,
> so ask me the three questions below, wait for my answers, then run the rest
> without further prompting. Never invent a basket id, transaction hash or
> market id: if you did not run the command, say so.
>
> **Ask me first (one message, then wait)**
>
> 1. **Theme.** What should the baskets express? For example "Bitcoin strength
>    this week", "the Fed holds", or "surprise me" and you choose high-volume
>    markets.
> 2. **How many baskets** this session? 1 to 5, default 2.
> 3. **Wallet VARA, only if you want it.** By default this session stakes
>    freebet credit if the wallet has any, and stakes nothing otherwise. That
>    needs no permission from me, because freebet credit is not withdrawable and
>    can only ever be spent on baskets. If you also want real VARA from the
>    wallet at risk, say the amount now, for example "up to 20 VARA". Say nothing
>    and no wallet VARA is touched.
>
> Do not ask me about balances, funding or key storage. Step 2 determines all of
> that by itself and tells you what it found.
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
> **Step 2 — Work out what this session can do**
>
> Do not ask me any of this and do not judge it by eye. Run the block; it sets
> `STAKE_MODE` and every later step obeys it.
>
> ```bash
> # 1. What is actually available.
> FREEBET_RAW=$(vara-wallet call $FREEBET_LEDGER FreebetLedger/BalanceOf \
>   --args "[\"$MY_ADDR\"]" --idl $FREEBET_IDL \
>   | jq -r 'if (.result|type)=="object" then (.result.value // .result.ok // 0) else (.result // 0) end | tostring')
> WALLET_RAW=$(vara-wallet balance --account agent | jq -r '.balanceRaw // "0"')
> case "$FREEBET_RAW" in ''|*[!0-9]*) FREEBET_RAW=0 ;; esac
> case "$WALLET_RAW"  in ''|*[!0-9]*) WALLET_RAW=0  ;; esac
>
> # 2. Will this key still exist later? BasketMarket/Claim resolves the owner
> #    from msg::source() and takes no owner argument, so a position is
> #    claimable only by this exact wallet. A marker left by an earlier run is
> #    proof the filesystem persists; with no marker yet, container signals
> #    decide and the answer is assumed unsafe.
> MARKER="$HOME/.vara-wallet/.pb-persist"
> BOOT_ID=$( (cat /proc/sys/kernel/random/boot_id 2>/dev/null \
>   || sysctl -n kern.boottime 2>/dev/null || echo unknown) | tr -d ' \t' )
> KEY_PERSISTED=false
> if [ -f "$MARKER" ] && ! grep -qxF "$BOOT_ID" "$MARKER" 2>/dev/null; then
>   KEY_PERSISTED=true      # written under a different boot, so it survived one
> elif [ ! -f /.dockerenv ] \
>   && ! grep -qaE 'docker|containerd|kubepods|lxc' /proc/1/cgroup 2>/dev/null; then
>   KEY_PERSISTED=true      # an ordinary host filesystem
> fi
> mkdir -p "$HOME/.vara-wallet" && printf '%s\n' "$BOOT_ID" >> "$MARKER" 2>/dev/null || true
>
> # 3. Pick the mode. Freebet credit is not withdrawable and its principal
> #    returns to the ledger on claim, so an unclaimable freebet position costs
> #    only forgone profit and is worth taking. Wallet VARA is real principal,
> #    so it needs both my explicit authorization and a key that survives.
> AUTHORIZED_WALLET_VARA=0        # whole VARA, only if I named a figure in answer 3
> MIN_STAKE_RAW=1000000000000     # 1 VARA, the smallest bet worth placing
> AUTH_RAW=$(( AUTHORIZED_WALLET_VARA * 1000000000000 ))
>
> STAKE_MODE=create_only
> WHY="no freebet credit and no wallet VARA authorized"
> if [ "$FREEBET_RAW" -ge "$MIN_STAKE_RAW" ]; then
>   STAKE_MODE=freebet
>   WHY=""
> elif [ "$AUTH_RAW" -ge "$MIN_STAKE_RAW" ] && [ "$WALLET_RAW" -lt "$AUTH_RAW" ]; then
>   WHY="wallet holds $WALLET_RAW planck, short of the $AUTH_RAW you authorized"
> elif [ "$AUTH_RAW" -ge "$MIN_STAKE_RAW" ] && [ "$KEY_PERSISTED" != true ]; then
>   WHY="this looks like a disposable container, so real VARA staked here could never be claimed back"
> elif [ "$AUTH_RAW" -ge "$MIN_STAKE_RAW" ]; then
>   STAKE_MODE=wallet
>   WHY=""
> fi
>
> # 4. Size each bet from what is there, never more. Whole VARA only: the quote
> #    carries planck and --value carries whole VARA, and the two must describe
> #    the same stake, so a fractional amount cannot be expressed.
> STAKE_VARA=0
> if [ "$STAKE_MODE" != create_only ]; then
>   NUM_BASKETS=${NUM_BASKETS:-2}
>   if [ "$STAKE_MODE" = freebet ]; then
>     # Stake the whole balance on one basket, not a slice of it. On claim the
>     # contract returns min(gross, shares) to the ledger, so a winning position
>     # gives the entire principal back and the same credit funds the next bet.
>     # Dividing it only shrinks every bet for no benefit.
>     PER_RAW=$FREEBET_RAW
>   else
>     # Real principal: never exceed what I authorized, and cap one bet at 10 VARA.
>     PER_RAW=$(( AUTH_RAW / NUM_BASKETS ))
>     CAP_RAW=10000000000000
>     [ "$PER_RAW" -gt "$CAP_RAW" ] && PER_RAW=$CAP_RAW
>   fi
>   STAKE_VARA=$(( PER_RAW / 1000000000000 ))
>   if [ "$STAKE_VARA" -lt 1 ]; then
>     STAKE_MODE=create_only
>     WHY="the available stake works out to under 1 VARA per basket once split $NUM_BASKETS ways"
>   fi
> fi
> STAKE_RAW=$(( STAKE_VARA * 1000000000000 ))
>
> echo "freebet: $FREEBET_RAW planck | wallet: $WALLET_RAW planck | key persists: $KEY_PERSISTED"
> echo "STAKE_MODE=$STAKE_MODE stake per bet: $STAKE_RAW planck (${STAKE_VARA} VARA) ${WHY:+| reason: $WHY}"
> ```
>
> Three outcomes, and you act on whichever one printed:
>
> `STAKE_MODE=freebet` or `wallet`: run every step. Stake `STAKE_RAW` per bet,
> which is already sized to what exists. When the mode is `freebet` and the key
> does not persist, place the bets anyway and say once in the report that any
> profit will be unclaimable from this wallet, because the credit itself is not
> yours to lose.
>
> `STAKE_MODE=create_only`: **do not settle for this yet.** It is the last
> resort, not a normal outcome, because unstaked baskets score nothing. When the
> reason is missing credit, stop and tell me this, in one short message:
>
> > This wallet has no freebet credit, so nothing can be staked. Get 400 VARA in
> > about two minutes at https://app.polybaskets.xyz/rewards: repost the
> > PolyBaskets post for 100 and quote-tweet it for 300, once each per week. It
> > has to be you, because the grant is admin-gated and verified against a real
> > post. My address is `$MY_ADDR`. Tell me when it is done and I will re-run
> > Step 2 and trade.
>
> Then wait. If I say it is funded, re-run Step 2 and continue as normal. Only if
> I explicitly tell you to go ahead without a stake do you run the create-only
> session: Steps 3 to 5, skip Steps 6 and 8, and report it as create-only rather
> than as trading. Never retry the grant yourself, never look for another funding
> route, and never fall back to the retired CHIP lane.
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
> What decides the score, so choose with it in mind. At settlement a position
> pays `shares * payout_per_share / index_at_creation_bps`, and the profit booked
> is `payout - shares`. `payout_per_share` is 10000 when the basket resolves your
> way, so the multiple is `10000 / entry index`: enter at 5000 and a win roughly
> doubles, enter at 8000 and it returns about a quarter. A basket whose legs are
> already near-certain has a high entry index and very little room to move, which
> is why safe-looking baskets score poorly.
>
> Two consequences worth holding in mind while you pick. On the freebet path a
> loss books zero rather than a negative, so the downside of the credit is
> bounded at losing the credit. And how soon a basket can settle is set entirely
> by its legs, because settlement needs every leg resolved plus the settler's
> challenge window: legs that end sooner let you claim and stake again inside the
> session, while long-dated legs mean one bet per grant. Which of those you want
> is my call, so tell me the trade-off you are making rather than assuming.
>
> **Step 5 — Create each basket (repeat Steps 5 and 6 per basket)**
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
> Run Step 5 then Step 6 for one basket, confirm the position, and only then
> start the next. Repeat until you have done the number of baskets I asked for.
> One transaction at a time from this account, never in parallel, and never
> create all the baskets first and bet afterwards: a basket can stop being
> tradable between the two.
>
> **Step 6 — Stake**
>
> ```bash
> [ "$STAKE_MODE" = create_only ] && { echo "no stake this session: $WHY"; SKIP_STAKE=1; }
> ```
>
> If that printed a skip, go straight to Step 9. Do not stake a smaller amount
> instead, and do not stake from a different address.
>
> A quote is valid for 30 seconds, so measure gas against a throwaway quote
> first, then fetch a fresh quote and send immediately with the cached limit.
>
> `STAKE_VARA` and `STAKE_RAW` are already set by Step 2 and sized to what the
> wallet actually holds. Do not reassign them here and do not substitute a round
> number: a stake the balance cannot cover fails on-chain.
>
> ```bash
> echo "staking $STAKE_VARA VARA ($STAKE_RAW planck) per basket, mode $STAKE_MODE"
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
> **Step 6b — Recycle the credit (freebet only)**
>
> This is how one 400 VARA grant funds a whole session. On claim the contract
> splits the position: `min(gross, shares)` goes back to your ledger balance and
> anything above the stake is paid to your wallet as real VARA. A basket that
> resolved your way therefore returns the full stake and leaves the credit ready
> to use again; a loss consumes it in proportion.
>
> So after each bet, wait for that basket to settle, claim it, confirm the credit
> came back, and stake again:
>
> ```bash
> vara-wallet call $BASKET_MARKET BasketMarket/GetSettlement --args "[$BASKET_ID]" --idl $IDL
> # once status is Finalized:
> vara-wallet --account agent call $BASKET_MARKET BasketMarket/Claim \
>   --args "[$BASKET_ID]" --voucher $VOUCHER_ID --idl $IDL
> vara-wallet call $FREEBET_LEDGER FreebetLedger/BalanceOf --args "[\"$MY_ADDR\"]" --idl $FREEBET_IDL
> ```
>
> If the balance is back, re-run Step 2's sizing and go again from Step 4. If it
> came back at zero the basket went against you and the credit is spent: report
> that and stop, do not look for another funding route.
>
> How long that takes is set by the basket's legs. Settlement needs every leg
> resolved plus the settler's challenge window, so a basket whose markets end
> months out cannot be recycled inside a session and you get one bet per grant.
> Tell me plainly which situation I am in rather than waiting indefinitely.
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
> Nothing to claim when `STAKE_MODE=create_only`, so skip this step in that case.
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
> Mode:                     [STAKE_MODE, with the reason when create_only]
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
