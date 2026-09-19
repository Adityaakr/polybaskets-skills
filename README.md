# PolyBaskets Skills

AI agent skill pack for [PolyBaskets](https://github.com/Adityaakr/polybaskets-skills) — an ETF-style prediction market aggregator on Vara Network.

**The default agent loop:** claim free CHIP tokens hourly → bet on prediction baskets → collect payouts when markets resolve → repeat.

Native VARA freebet balances are handled separately through `FreebetLedger`; use `basket-freebet` for that path.

## Prerequisites

- [vara-wallet](https://github.com/gear-foundation/vara-wallet) CLI: `npm install -g vara-wallet`
- [vara-skills](https://github.com/gear-foundation/vara-skills) skill pack: `npx skills add gear-foundation/vara-skills`
- A vara-wallet account: `vara-wallet wallet create --name agent`
- Gas via the PolyBaskets voucher claim process (no VARA purchase needed)

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
| `basket-bet` | **Start here** — claim CHIP, pick a basket, place bets |
| `basket-freebet` | Spend native VARA freebet balance through FreebetLedger |
| `basket-query` | Browse baskets, check positions and settlements |
| `basket-claim` | Claim payout from settled baskets |
| `polybaskets-overview` | Understand the protocol — index math, payout formula, settlement |
| `basket-create` | Create a new prediction basket on-chain |
| `basket-settle` | Propose settlements (settler role) and finalize existing proposals after the challenge deadline |

## Quick Start — Starter Prompts

See **[STARTER_PROMPT.md](STARTER_PROMPT.md)** for copy-paste prompts you can drop into any AI agent:

| Prompt | For |
|--------|-----|
| **Main Prompt — Full Session** | New + returning agents — full Season 3 trading session with hourly CHIP claim, conviction-sized bets, bounded ~60-90 TX |
| **Check my bets and balances** | Check positions and claim settled payouts |
| **Hourly routine (returning user)** | Runs the session loop with hourly CHIP + drained-voucher STOP rule |
| **Explore markets only** | Research active Polymarket markets without betting |
| **Claim all payouts** | Claim all Finalized basket payouts |
| **Max volume session** | Fully autonomous — executes the Main Prompt without asking questions |

Works with: Claude Code, Gemini CLI, Cursor, Codex, or any agent with shell access.

## Usage (Claude Code)

```bash
/polybaskets-skills              # Router — shows the agent loop and routes to sub-skills
/polybaskets-skills:basket-bet   # Start the loop — claim CHIP and bet
/polybaskets-skills:basket-freebet # Spend native VARA freebet balance
/polybaskets-skills:basket-query # Browse baskets and check results
```

## Network

All commands target Vara mainnet (`wss://rpc.vara.network`) which is vara-wallet's default — no `--network` flag needed. Program IDs and IDL files are bundled in `idl/` and documented in `references/program-ids.md`.
