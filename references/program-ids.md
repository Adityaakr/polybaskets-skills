# PolyBaskets Program IDs and Network Config

## Shell Variables

Copy-paste this block at the start of any PolyBaskets interaction session:

```bash
# Program IDs (Vara mainnet, Season 3)
# NOTE: BASKET_MARKET was migrated to the id below. The legacy id
# 0xea8373e8b4441ef6e95325c1044d23ebf615b43fdef60a48623836a15ca7a25a is DEAD —
# baskets created there are rejected by the bet-quote-service ("not active").
BASKET_MARKET="0xa749ccd80d71637b450789e12e3d94524e9ae17877d1b59f5ddda784f89a2cba"
BET_TOKEN="0x186f6cda18fea13d9fc5969eec5a379220d6726f64c1d5f4b346e89271f917bc"
BET_LANE="0x35848dea0ab64f283497deaff93b12fe4d17649624b2cd5149f253ef372b29dc"
FREEBET_LEDGER="0x2bb74834402fb7da9144d2ab91c1570e97237ad0ead1f7feb392162c3e3ad64e"
DAILY_CONTEST="0x1f320a71665f990701daf3862aa4cbb98943859a726c2b497bd053e120149a77"

# Voucher + quote backends
VOUCHER_URL="https://voucher-backend-production-5a1b.up.railway.app/voucher"
BET_QUOTE_URL="https://bet-quote-service-production.up.railway.app"

# IDL paths (relative to skill pack root)
_PB="${POLYBASKETS_SKILLS_DIR:-skills}"
IDL="$_PB/idl/polymarket-mirror.idl"
BET_TOKEN_IDL="$_PB/idl/bet_token_client.idl"
BET_LANE_IDL="$_PB/idl/bet_lane_client.idl"
FREEBET_LEDGER_IDL="$_PB/idl/freebet-ledger.idl"
```

If running from the polybaskets repo root, IDL files are also at:
- `program/polymarket-mirror.idl`
- `bet-token/client/bet_token_client.idl`
- `bet-lane/client/bet_lane_client.idl`
- `freebet-ledger/freebet-ledger.idl`

## Program Roles

| Program | Purpose |
|---------|---------|
| BasketMarket | Core contract: baskets, settlement state, agent names, and native VARA lane |
| BetToken | CHIP fungible token with **hourly** claim (500 base, +10 per UTC-day streak, cap 600 on day 11) |
| BetLane | Primary betting lane using CHIP tokens |
| FreebetLedger | Native VARA freebet balance ledger; spends into `Vara` baskets and receives returned principal |

## Network

Vara mainnet (`wss://rpc.vara.network`) is vara-wallet's default. No `--network` flag or env var needed.

```bash
# Just works — mainnet by default
vara-wallet call $BASKET_MARKET BasketMarket/GetBasketCount --args '[]' --idl $IDL
```

## Gas — Voucher System (hourly-tranche model)

Agents get gas through the PolyBaskets voucher backend. No VARA purchase needed.

**Season 3 model:** one voucher per agent, funded **+500 VARA per hourly tranche**. A single batched `POST /voucher` registers the required program set and adds a tranche. Each top-up extends `validUpTo` by 24h (sliding window — voucher expires only after ≥24h of silence, then the hourly cron revokes and remainder returns to the issuer). Hard per-wallet rate limit: 1 funded POST per hour; 2nd POST within the window returns 429 with `Retry-After`.

```bash
# GET voucher state (free, never rate-limited)
curl -s "$VOUCHER_URL/$MY_ADDR"
# Returns:
# {
#   "voucherId": "0x...",                   // the voucher ID to pass as --voucher
#   "programs": ["0x...", ...],              // programs currently whitelisted on this voucher
#   "validUpTo": "2026-04-23T12:00:00Z",
#   "varaBalance": "1757000000000000",       // on-chain balance in planck; null if balanceKnown=false
#   "balanceKnown": true,                    // false when the backend couldn't reach the chain
#   "lastRenewedAt": "2026-04-22T11:00:00Z", // when the last tranche was issued / topped up
#   "nextTopUpEligibleAt": "2026-04-22T12:00:00Z", // clamped to now when canTopUpNow=true
#   "canTopUpNow": true                      // true if >=1h has passed since lastRenewedAt
# }

# POST to fund / top up / register programs (batched — single call registers all listed programs)
# ⚠ `programs` is an ARRAY of contract program IDs, NOT your wallet address
curl -s -X POST "$VOUCHER_URL" -H 'Content-Type: application/json' \
  -d '{"account":"'"$MY_ADDR"'","programs":["'"$BASKET_MARKET"'","'"$BET_TOKEN"'","'"$BET_LANE"'"]}'
# On HTTP 200 → { "voucherId": "0x..." }
#   Three cases end up here:
#     (a) New voucher issued with 500 VARA + requested programs (first request ever),
#     (b) Top-up: +500 VARA added, duration extended 24h, missing programs appended,
#     (c) Free append: the server was unable to top up (1h window OR IP ceiling
#         exhausted) but the request listed programs not on the voucher; those
#         programs were appended free of charge, no VARA delta.
# On HTTP 429 → { "statusCode":429, "error":"Too Many Requests",
#                 "message":"Per-wallet rate limit: 1 voucher request per hour",
#                 "nextEligibleAt":"...", "retryAfterSec":1234 }
#   Response also includes `Retry-After: <seconds>` HTTP header.
#   Only fires when the voucher ALREADY has every requested program AND
#   you're inside the 1h window (or the IP cap is exhausted with no
#   missing programs to append for free). Reuse the existing voucherId
#   from the prior GET — the voucher is still valid.
```

**Rules:**
- **Voucher top-up rule:** GET first. POST only when there is no voucher, one of the required programs is missing, or `balanceKnown=true` AND `varaBalance < 10000000000000` (10 VARA) AND `canTopUpNow=true`. Reuse the existing voucher while the known balance is at least 10 VARA, even if `canTopUpNow=true`.
- **Program set:** CHIP sessions need `[$BASKET_MARKET, $BET_TOKEN, $BET_LANE]`. Native freebet sessions also need `$FREEBET_LEDGER` because the user sends `FreebetLedger/SpendFreebet`.
- **Drained-voucher STOP:** when `balanceKnown=true` AND `varaBalance < 10000000000000` (10 VARA) AND `canTopUpNow=false`, you're inside the 1h window with no budget — STOP and wait until `nextTopUpEligibleAt`.
- **RPC outage fallback:** if `balanceKnown=false`, do NOT treat a zero balance as "drained" — the backend just couldn't reach the chain. Reuse the current voucher if one exists, and do not top up solely from `canTopUpNow`.
- **Controller throttle:** 6 POSTs per IP per hour (NestJS @Throttle). Headroom for retries on transient failures — the business rate limit is the per-wallet DB check.
- **Per-IP abuse gate:** 40 tranches per IP per UTC day (`PER_IP_TRANCHES_PER_DAY`; set to 0 in ops config to disable). On hit, the 429 response includes `Retry-After` set to seconds until next UTC midnight — but if your request lists missing programs, those still get appended free of charge (200, no tranche charged).

## Actor ID Format

Sails `actor_id` args require **hex format** — SS58 addresses are rejected:

```bash
# Get your hex address
MY_ADDR=$(vara-wallet balance | jq -r .address)
# → 0xe00801c1a5b8aef60d3a...
```

## Token Units

Both VARA and CHIP use 12 decimals. Method args for `u256`/`u128` amounts expect **raw units**:
- 100 CHIP = `"100000000000000"` (100 * 10^12)
- 1 CHIP = `"1000000000000"` (10^12)
- `--value` flag (for VARA lane) auto-converts from VARA by default

## vara-wallet Response Format

All vara-wallet output is JSON:

```bash
# Queries return:
{"result": <value>}                          # plain types (u64, bool, str)
{"result": {"ok": {...}}}                    # Result<T, E> success
{"result": {"err": "ErrorVariant"}}          # Result<T, E> error

# Mutations return:
{"txHash": "0x...", "blockHash": "0x...", "blockNumber": 123, "messageId": "0x...", "result": <value>}
```

Extract values with jq:
```bash
# Query result
vara-wallet ... | jq -r '.result'
vara-wallet ... | jq '.result.ok'

# Mutation result
vara-wallet ... | jq -r '.result'
```
