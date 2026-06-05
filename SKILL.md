---
name: pharos-token-analytics
description: >
  Use this skill for any token analytics task on the Pharos blockchain.
  Covers ERC-20 token price discovery, holder counts, supply stats (total,
  circulating, burned), top-holder distribution, transfer volume, and
  token metadata queries — all via Foundry cast commands and Pharos RPC.
  Invoke whenever the user asks about a token's price, holders, supply,
  market cap, burn rate, or any on-chain token statistics on Pharos Chain
  / Pharos Network (including atlantic-testnet and mainnet).
version: 0.1.0
requires:
  anyBins:
    - cast
    - jq
---

# Pharos Token Analytics Skill

On-chain token intelligence for the Pharos blockchain. Query ERC-20 token
price, holder stats, supply data, transfer volume, and top holders using
Foundry (`cast`) CLI commands against Pharos RPC endpoints.

---

## Prerequisites

1. **Install Foundry** (MANDATORY before any other action):

   ```bash
   # Check first
   which cast

   # If not found, install:
   curl -L https://foundry.paradigm.xyz | bash
   source ~/.zshenv && foundryup

   # Verify
   cast --version
   ```

2. **Install jq** (required for JSON parsing):

   ```bash
   which jq || (apt-get install -y jq 2>/dev/null || brew install jq)
   ```

3. **No private key required** — all operations in this skill are read-only.

---

## Network Configuration

Read RPC endpoints from `assets/networks.json`.

- **Default network**: `atlantic-testnet` (used when user does not specify).
- **Switch to mainnet**: use `mainnet` entry when user specifies.

```bash
RPC_URL=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .rpcUrl' assets/networks.json)
```

---

## Capability Index

| User Need | Capability | Detailed Instructions |
|---|---|---|
| Token name / symbol / decimals | `cast call` → ERC-20 metadata methods | → `references/analytics.md#token-metadata` |
| Total supply | `cast call totalSupply()` | → `references/analytics.md#total-supply` |
| Circulating supply (excl. burn/treasury) | `cast call` on known reserve addresses | → `references/analytics.md#circulating-supply` |
| Burned token amount | `cast call balanceOf(dead address)` | → `references/analytics.md#burned-supply` |
| Token balance of any address | `cast call balanceOf(address)` | → `references/analytics.md#token-balance` |
| On-chain price (via DEX pair) | `cast call getReserves()` on Uniswap-style pair | → `references/analytics.md#token-price` |
| Market cap estimate | price × circulating supply | → `references/analytics.md#market-cap` |
| Transfer event count (volume proxy) | `cast logs` on Transfer topic | → `references/analytics.md#transfer-volume` |
| Top holders snapshot | `cast logs` Transfer events → aggregate | → `references/analytics.md#top-holders` |
| Token allowance | `cast call allowance(owner,spender)` | → `references/analytics.md#allowance` |

---

## General Error Handling

| Error | CLI Signature | Handling |
|---|---|---|
| Not an ERC-20 contract | Empty / revert on `name()` | Inform user; address may not be an ERC-20 |
| No DEX pair found | Zero reserves | Inform user; token may not have on-chain liquidity |
| RPC timeout | `connection refused` / timeout | Retry once; suggest checking network config |
| Invalid address | `invalid address` | Prompt correct format: `0x` + 40 hex chars |
| No events found | Empty log array | Token may have no transfer activity in range |

---

## Security Reminders

- All operations are **read-only** — no private key is needed or used.
- Always confirm the target network with the user before querying.
- When displaying holder data, note that on-chain logs may be incomplete
  for tokens with very high transfer volumes (pagination required).
