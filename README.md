# PolyBaskets Skills

AI agent skill pack for [PolyBaskets](https://github.com/Adityaakr/polybaskets-skills) — an ETF-style prediction market aggregator on Vara Network.

**The current agent loop:** confirm wallet VARA or freebet credit → select an active VARA basket → place a signed bet → verify the position → claim after settlement.

Native VARA freebet balances are handled through `FreebetLedger`; use `basket-freebet` for that path. Gas vouchers cover transaction fees only, not stakes. Legacy CHIP baskets remain historical records, not a current betting route.

## Prerequisites

- [vara-wallet](https://github.com/gear-foundation/vara-wallet) CLI: `npm install -g vara-wallet`
- [vara-skills](https://github.com/gear-foundation/vara-skills) skill pack: `npx skills add gear-foundation/vara-skills`
- A vara-wallet account: `vara-wallet wallet create --name agent`
- Gas via the PolyBaskets voucher claim process; a bet additionally needs authorized wallet VARA or FreebetLedger credit

## Installation

```bash
# 1. Install dependencies
npm install -g vara-wallet
npx skills add gear-foundation/vara-skills

# 2. Install polybaskets skills
npx skills add Adityaakr/polybaskets-skills

# 3. Create a wallet (one-time)
vara-wallet wallet create --name agent
```

Works with Claude Code, Codex, Cursor, Gemini CLI, and [40+ other agents](https://github.com/vercel-labs/skills).

### From the polybaskets repo

Skills work directly when running Claude Code from the polybaskets repo root.

### Manual

Copy or symlink this directory to your Claude Code skills:

```bash
ln -s /path/to/polybaskets/skills ~/.claude/skills/polybaskets-skills
```

## Skills

| Skill | Purpose |
|-------|---------|
| `basket-bet` | **Start here** — verify an active VARA basket and place a funded bet |
| `basket-freebet` | Spend native VARA freebet balance through FreebetLedger |
| `basket-query` | Browse baskets, check positions and settlements |
| `basket-claim` | Claim VARA or freebet profit from settled baskets |
| `polybaskets-overview` | Understand the protocol — index math, payout formula, settlement |
| `basket-create` | Create a new prediction basket on-chain |
| `basket-settle` | Propose settlements (settler role) and finalize existing proposals after the challenge deadline |

## Quick Start — Starter Prompts

See **[STARTER_PROMPT.md](STARTER_PROMPT.md)** for copy-paste prompts you can drop into any AI agent:

| Prompt | For |
|--------|-----|
| **Main prompt** | New + returning agents — funded VARA/freebet trading with position verification |
| **Check my bets and balances** | Check positions and claim settled payouts |
| **Explore markets only** | Research active Polymarket markets without betting |

Works with: Claude Code, Gemini CLI, Cursor, Codex, or any agent with shell access.

## Usage (Claude Code)

```bash
/polybaskets-skills              # Router — shows the agent loop and routes to sub-skills
/polybaskets-skills:basket-bet   # Verify an active VARA basket and funded stake
/polybaskets-skills:basket-freebet # Spend native VARA freebet balance
/polybaskets-skills:basket-query # Browse baskets and check results
```

## Network

All commands target Vara mainnet (`wss://rpc.vara.network`) which is vara-wallet's default — no `--network` flag needed. Program IDs and IDL files are bundled in `idl/` and documented in `references/program-ids.md`.
