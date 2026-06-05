# Token Analytics Reference

All commands use Foundry `cast`. Set `RPC_URL` and `TOKEN` before running:

```bash
RPC_URL=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .rpcUrl' assets/networks.json)
TOKEN=<token_contract_address>
```

---

## Token Metadata

Retrieve the name, symbol, and decimals of any ERC-20 token.

```bash
# Token name
cast call $TOKEN "name()(string)" --rpc-url $RPC_URL

# Token symbol
cast call $TOKEN "symbol()(string)" --rpc-url $RPC_URL

# Decimals
cast call $TOKEN "decimals()(uint8)" --rpc-url $RPC_URL
```

**Agent display format:**
```
Token:    <name> (<symbol>)
Decimals: <decimals>
Contract: <TOKEN>
Network:  <displayName>
```

---

## Total Supply

```bash
DECIMALS=$(cast call $TOKEN "decimals()(uint8)" --rpc-url $RPC_URL)
RAW=$(cast call $TOKEN "totalSupply()(uint256)" --rpc-url $RPC_URL)

# Human-readable (divide by 10^decimals)
cast --to-unit $RAW ether   # works for 18-decimal tokens
# For non-18 decimals, use:
python3 -c "print($RAW / 10**$DECIMALS)"
```

**Agent display format:**
```
Total Supply: <amount> <symbol>
Raw:          <RAW>
```

---

## Circulating Supply

Circulating supply = Total Supply − burned − treasury/reserve holdings.

```bash
BURN_ADDR=$(jq -r '.wellKnownAddresses.burnAddress' assets/networks.json)
ZERO_ADDR=$(jq -r '.wellKnownAddresses.zeroAddress' assets/networks.json)
TREASURY=<treasury_address_if_known>

TOTAL=$(cast call $TOKEN "totalSupply()(uint256)" --rpc-url $RPC_URL)
BURNED=$(cast call $TOKEN "balanceOf(address)(uint256)" $BURN_ADDR --rpc-url $RPC_URL)
ZERO_BAL=$(cast call $TOKEN "balanceOf(address)(uint256)" $ZERO_ADDR --rpc-url $RPC_URL)

# Circulating = TOTAL - BURNED - ZERO_BAL (subtract treasury if address known)
python3 -c "
total=$TOTAL; burned=$BURNED; zero=$ZERO_BAL
circ = total - burned - zero
decimals=$DECIMALS
print(f'Circulating Supply: {circ/10**decimals:,.4f}')
print(f'Burned:             {burned/10**decimals:,.4f}')
"
```

> If the user does not provide a treasury address, omit that deduction and
> note the caveat in the output.

---

## Burned Supply

```bash
BURN_ADDR=$(jq -r '.wellKnownAddresses.burnAddress' assets/networks.json)
ZERO_ADDR=$(jq -r '.wellKnownAddresses.zeroAddress' assets/networks.json)

BURNED_DEAD=$(cast call $TOKEN "balanceOf(address)(uint256)" $BURN_ADDR --rpc-url $RPC_URL)
BURNED_ZERO=$(cast call $TOKEN "balanceOf(address)(uint256)" $ZERO_ADDR --rpc-url $RPC_URL)

python3 -c "
d=$BURNED_DEAD; z=$BURNED_ZERO; dec=$DECIMALS
print(f'Burned (0xdead): {d/10**dec:,.4f}')
print(f'Burned (0x0000): {z/10**dec:,.4f}')
print(f'Total Burned:    {(d+z)/10**dec:,.4f}')
"
```

---

## Token Balance

Query any wallet's token balance.

```bash
WALLET=<wallet_address>
RAW=$(cast call $TOKEN "balanceOf(address)(uint256)" $WALLET --rpc-url $RPC_URL)
python3 -c "print(f'Balance: {$RAW / 10**$DECIMALS:,.4f}')"
```

---

## Token Price

Price discovery via a Uniswap V2-style DEX pair on Pharos.
The agent must know or find the pair contract address.

```bash
PAIR=<dex_pair_contract_address>

# Get reserves: returns (reserve0, reserve1, blockTimestampLast)
RESERVES=$(cast call $PAIR "getReserves()(uint112,uint112,uint32)" --rpc-url $RPC_URL)
RESERVE0=$(echo $RESERVES | awk '{print $1}')
RESERVE1=$(echo $RESERVES | awk '{print $2}')

# Determine which reserve is the token and which is the quote asset (e.g. USDC)
TOKEN0=$(cast call $PAIR "token0()(address)" --rpc-url $RPC_URL)

python3 -c "
r0=$RESERVE0; r1=$RESERVE1
# Adjust for decimals of each token (token typically 18, USDC 6)
# If TOKEN is token0: price = r1/r0 * (10**token_decimals / 10**quote_decimals)
token_dec=18; quote_dec=6
if '$TOKEN0'.lower() == '$TOKEN'.lower():
    price = (r1 / 10**quote_dec) / (r0 / 10**token_dec)
else:
    price = (r0 / 10**quote_dec) / (r1 / 10**token_dec)
print(f'Price: \${price:.6f} per token')
"
```

> If no pair address is known, inform the user that on-chain price data
> requires a DEX liquidity pair and ask them to provide the pair address.

---

## Market Cap

Requires price (from DEX pair) and circulating supply.

```bash
python3 -c "
price=<price_from_above>
circ=<circulating_supply_from_above>
mcap = price * circ
print(f'Market Cap (est.): \${mcap:,.2f}')
print(f'Fully Diluted MC:  \${price * <total_supply>:,.2f}')
"
```

---

## Transfer Volume

Count Transfer events and aggregate token volume over a block range.

```bash
# ERC-20 Transfer topic
TRANSFER_TOPIC="0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef"

# Fetch logs (last ~50,000 blocks as default range)
LATEST=$(cast block-number --rpc-url $RPC_URL)
FROM_BLOCK=$(python3 -c "print($LATEST - 50000)")

cast logs \
  --from-block $FROM_BLOCK \
  --to-block $LATEST \
  --address $TOKEN \
  $TRANSFER_TOPIC \
  --rpc-url $RPC_URL \
  --json > /tmp/transfers.json

# Count transfers and sum volume
python3 -c "
import json, sys
logs = json.load(open('/tmp/transfers.json'))
total_volume = 0
for log in logs:
    # data field is the transfer amount (uint256, hex)
    amt = int(log['data'], 16)
    total_volume += amt
dec = $DECIMALS
print(f'Transfer Events:  {len(logs):,}')
print(f'Total Volume:     {total_volume/10**dec:,.4f} tokens')
print(f'Block Range:      {$FROM_BLOCK} → {$LATEST}')
"
```

> For very active tokens, reduce the block range to avoid RPC timeouts.
> Use `--from-block` / `--to-block` to paginate as needed.

---

## Top Holders

Derive top holders by aggregating Transfer event logs.

```bash
# Reuse /tmp/transfers.json from Transfer Volume step (or re-fetch)

python3 -c "
import json
from collections import defaultdict

logs = json.load(open('/tmp/transfers.json'))
balances = defaultdict(int)

for log in logs:
    topics = log['topics']
    # topic[1] = from address (padded), topic[2] = to address (padded)
    from_addr = '0x' + topics[1][-40:]
    to_addr   = '0x' + topics[2][-40:]
    amount    = int(log['data'], 16)
    balances[from_addr] -= amount
    balances[to_addr]   += amount

# Filter out zero/negative (incomplete history artefacts)
positive = {k: v for k, v in balances.items() if v > 0}
top = sorted(positive.items(), key=lambda x: x[1], reverse=True)[:10]

dec = $DECIMALS
print('Top 10 Holders (from sampled transfer logs):')
print(f'{\"Rank\":<5} {\"Address\":<44} {\"Balance\":>20}')
print('-' * 72)
for i, (addr, bal) in enumerate(top, 1):
    print(f'{i:<5} {addr:<44} {bal/10**dec:>20,.4f}')

print()
print('Note: Based on sampled block range — not a full historical snapshot.')
"
```

---

## Allowance

Query how much a spender is approved to use on behalf of an owner.

```bash
OWNER=<owner_address>
SPENDER=<spender_address>

RAW=$(cast call $TOKEN "allowance(address,address)(uint256)" $OWNER $SPENDER --rpc-url $RPC_URL)
python3 -c "print(f'Allowance: {$RAW / 10**$DECIMALS:,.4f} tokens')"
```

---

## Full Token Report (Combined)

When the user asks for a full token overview, run all read-only queries
and present a consolidated report:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  TOKEN ANALYTICS REPORT — Pharos Chain
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Token:              <name> (<symbol>)
Contract:           <address>
Network:            <displayName>
Decimals:           <decimals>

── Supply ───────────────────────────────
Total Supply:       <total>
Burned:             <burned>
Circulating:        <circulating>

── Price & Market Cap ───────────────────
Price (DEX):        $<price>   (or "No liquidity pair found")
Market Cap (est.):  $<mcap>

── Activity (last 50k blocks) ───────────
Transfer Events:    <count>
Volume:             <volume> <symbol>

── Top Holders ──────────────────────────
<top_holders_table>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
