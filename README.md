# Pharos Token Analytics

**A Pharos Agent Centre Skill** for on-chain token analytics on the [Pharos blockchain](https://pharos.xyz).

Query ERC-20 token price, holder distribution, supply stats, transfer volume, and more — entirely via Foundry `cast` read-only commands. No private key required.

---

## Capabilities

| Query | Description |
|---|---|
| Token Metadata | Name, symbol, decimals |
| Total Supply | Raw on-chain totalSupply() |
| Circulating Supply | Total minus burned & treasury |
| Burned Supply | Balances at 0xdead and 0x0000 |
| Token Balance | Any wallet's ERC-20 balance |
| DEX Price | Spot price from Uniswap V2-style pair |
| Market Cap | Price × circulating supply |
| Transfer Volume | Event count & aggregate volume over block range |
| Top Holders | Derived from Transfer log aggregation |
| Allowance | Approved spend between owner/spender |
| Full Report | All of the above in one formatted output |

---

## Installation

```bash
# Via npx (Claude Code / compatible agents)
npx skills add https://github.com/<your-username>/pharos-token-analytics

# Or manually — copy SKILL.md into your agent's skills directory:
# Claude Code:  ~/.claude/skills/
# OpenClaw:     ~/.openclaw/skills/
# Codex:        ~/.codex/skills/
```

---

## Prerequisites

- [Foundry](https://getfoundry.sh/) (`cast` + `forge`)
- `jq` (JSON CLI tool)
- No wallet / private key needed — all operations are read-only

---

## Usage Examples

```
"What is the total supply of the USDC token on Pharos testnet?"

"Show me the top 10 holders of token 0xABC...123 on Pharos."

"How many tokens have been burned for 0xDEF...456?"

"Give me a full analytics report for token 0x72df...5fe7 on atlantic-testnet."

"What's the current price of token 0xABC...123 via its DEX pair 0xDEF...456?"
```

---

## File Structure

```
pharos-token-analytics/
├── SKILL.md                  # Agent skill definition (frontmatter + instructions)
├── assets/
│   └── networks.json         # Pharos RPC endpoints, chain IDs, known tokens
└── references/
    └── analytics.md          # Full cast command templates for every capability
```

---

## Networks Supported

| Network | Chain ID | RPC |
|---|---|---|
| Atlantic Testnet | 688688 | https://testnet.dplabs-internal.com |
| Mainnet | 2288 | https://rpc.pharos.xyz |

---

## License

MIT-0 — free to use, modify, and redistribute. No attribution required.
