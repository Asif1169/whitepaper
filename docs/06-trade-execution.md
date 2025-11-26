# Trade Execution Models

## Trade Execution Overview

MonBridge supports multiple execution models optimized for different trade sizes and market conditions.

---

## 1. Single-Venue Execution

### When Used
- Small to medium trades
- Direct path available (Token A → Token B exists at venue)
- Single venue offers best pricing
- Lowest complexity routing

### Execution Flow

```
Step 1: User Submits Request
User calls MonBridge: "Swap 1000 USDC for DAI"
                      Max slippage: 100 BPS

Step 2: Price Discovery
Contract queries DEX venue:
- Reserve USDC: 50M
- Reserve DAI: 50M
- Spot price: 1:1
- Quote: 1000 USDC → ~990 DAI (99 BPS slippage)

Step 3: Validation
- Slippage 99 BPS < Max 100 BPS ✓
- Liquidity sufficient ✓
- Venue healthy ✓

Step 4: Execution
Contract calls: DEX.swap(1000 USDC, 990 DAI min, user)
DEX executes swap atomically

Step 5: Settlement
- 1000 USDC transferred from user to DEX
- ~990 DAI transferred from DEX to user
- Protocol fee (1 USDC) accumulated

Step 6: Confirmation
User receives 990 DAI
Transaction logged on-chain
Event emitted with details
```

### Gas Efficiency
- Minimal contract calls
- Direct routing path
- Lowest transaction cost
- Preferred for small trades

### Example Transactions

**Transaction A - Small Trade**:
```
Input: 100 USDC
Output: 99 DAI
Slippage: 1%
Gas: ~150K
Cost: $3-10
```

**Transaction B - Inefficient for Large Trade**:
```
Input: 1,000,000 USDC
Output: 900,000 DAI (10% slippage)
Slippage: 10%
Cost: Heavy - Not recommended
```

---

## 2. Multi-Hop Execution

### When Used
- Deeper liquidity needed through intermediate token
- Direct pair has insufficient liquidity
- Intermediate path reduces total slippage
- Token A not directly paired with Token B

### Execution Flow

```
User Goal: Swap 500 USDC for SHIB

Scenario 1: Direct Path (poor liquidity)
USDC → SHIB (Direct)
500 USDC → 50,000 SHIB (50% slippage) ❌

Scenario 2: Multi-Hop Path (better liquidity)
USDC → ETH → SHIB
500 USDC → 0.25 ETH → 62,500 SHIB (25% slippage) ✓

MonBridge chooses: Scenario 2
```

### Technical Execution

```
Step 1: Multi-Hop Planning
Contract analyzes all possible paths:
- USDC → SHIB (Direct)
- USDC → ETH → SHIB
- USDC → DAI → SHIB
- USDC → WBTC → SHIB
etc.

Step 2: Impact Calculation
For each path, calculate:
- Expected output
- Total slippage
- Number of hops
- Combined fees

Step 3: Path Selection
Choose path with:
- Best output amount
- Acceptable slippage
- Healthy venues

Step 4: Atomic Execution
Execute path atomically:

Hop 1: USDC → ETH
- Contract receives input: 500 USDC
- Contract calls Venue1: swap(500 USDC for ETH)
- Contract receives: 0.25 ETH

Hop 2: ETH → SHIB
- Contract calls Venue2: swap(0.25 ETH for SHIB)
- Contract receives: 62,500 SHIB

Step 5: Settlement
- User provides 500 USDC
- User receives 62,500 SHIB
- All hops complete atomically (all-or-nothing)
- Protocol fee accumulated
```

### Examples

**Multi-Hop Example 1: Rare Token Pair**:
```
Goal: Swap PEPE for DOGE (no direct liquidity)

Option: PEPE → ETH → USDC → DOGE
- Hop 1: PEPE to ETH at venue A
- Hop 2: ETH to USDC at venue B
- Hop 3: USDC to DOGE at venue C

Result: Successful swap through available liquidity
```

**Multi-Hop Example 2: Liquidity Depth**:
```
Direct Swap: USDC to USDT (common pair)
100,000 USDC → 99,500 USDT (50 BPS slippage)

But if direct pair has low liquidity:
100,000 USDC → 50,000 DAI → 99,500 USDT (similar slippage)
Better outcome through intermediate hop
```

### Gas Considerations
- Additional hop = additional contract calls
- More gas expensive than single-hop
- Only used when output benefit exceeds gas cost

---

## 3. Split Trading (Order Fragmentation)

### When Used
- Very large trades exceeding single-venue liquidity capacity
- Price impact would be excessive with single route
- Multiple venues available with adequate liquidity
- Automatic benefit calculation shows improvement

### Split Triggering Logic

```
User submits: 5,000,000 USDC → DAI
Max slippage: 200 BPS

Analysis:
- Direct swap at single venue: 5,000,000 USDC → 4,500,000 DAI (10% impact)
- Expected slippage: 1000 BPS (exceeds max 200 BPS) ❌
- Split trade analysis initiated

Option 1: Direct (FAILS threshold)
Slippage: 1000 BPS
Result: ❌ Rejected

Option 2: Split 50/50 Across 2 Venues
Venue A: 2,500,000 USDC → 2,475,000 DAI (1% impact)
Venue B: 2,500,000 USDC → 2,475,000 DAI (1% impact)
Total: 5,000,000 USDC → 4,950,000 DAI (Slippage: 100 BPS) ✓

Result: Split execution selected
```

### Execution Process

```
Step 1: Split Determination
Contract calculates:
- Number of optimal splits (2-4)
- Distribution percentages
- Expected output from each split

Step 2: Segment Execution

Segment 1: 2,500,000 USDC
- Route through Venue A
- Receive ~2,475,000 DAI

Segment 2: 2,500,000 USDC
- Route through Venue B
- Receive ~2,475,000 DAI

Step 3: Output Aggregation
- Total received: ~4,950,000 DAI
- Combine all outputs
- Verify meets minimum threshold

Step 4: User Settlement
- User sends: 5,000,000 USDC
- User receives: 4,950,000 DAI
- Fee accumulated: ~5,000 USDC (0.1%)
- Final user take-home: 4,950,000 DAI
```

### Configuration

```solidity
struct SplitConfig {
    bool enableAutoSplit = true          // Split enabled
    uint16 minSplitImpactBPS = 100      // Split if impact > 1%
    uint8 maxSplits = 4                 // Max 4 segments
}
```

### Examples

**Large Trade Example 1**:
```
Order: 10,000,000 USDC → ETH
Direct execution: 3,000 ETH (excessive slippage: 20%)

Split Execution:
- Split into 4 segments of 2,500,000 USDC each
- Venue 1: 2,500,000 USDC → 760 ETH (4% slippage)
- Venue 2: 2,500,000 USDC → 760 ETH (4% slippage)
- Venue 3: 2,500,000 USDC → 755 ETH (5% slippage)
- Venue 4: 2,500,000 USDC → 750 ETH (5% slippage)
Total: 10,000,000 USDC → 3,025 ETH (4.8% avg slippage)

Result: Split improved output by 25 ETH
```

**Large Trade Example 2**:
```
Order: 50,000,000 USDC → DAI
Optimal: Split into 3 venues

Venue A (Best Liquidity): 20,000,000 USDC → 19,800,000 DAI
Venue B (Good Liquidity): 20,000,000 USDC → 19,900,000 DAI
Venue C (Adequate): 10,000,000 USDC → 9,950,000 DAI

Total: 50,000,000 USDC → 49,650,000 DAI (0.7% avg slippage)
```

### Split Limitations
- Maximum 4 segments to limit complexity
- Each segment must meet minimum liquidity
- All segments must execute successfully
- Revert if any segment fails

---

## 4. Swap Type Variations

### ETH to Token

**Use Case**: User has native blockchain token, wants ERC20 token

```
User has: 10 ETH
User wants: 50,000 USDC
User calls: swapExactETHForTokens(amountOutMin, path, recipient)

Execution:
1. User sends 10 ETH to MonBridge
2. MonBridge wraps ETH to WETH
3. MonBridge routes WETH through selected venue(s)
4. MonBridge receives USDC
5. MonBridge sends USDC to user
6. Transaction complete
```

### Token to ETH

**Use Case**: User has ERC20 token, wants native blockchain token

```
User has: 50,000 USDC
User wants: 10 ETH
User calls: swapExactTokensForETH(amountIn, amountOutMin, path, recipient)

Execution:
1. User approves MonBridge to spend 50,000 USDC
2. User sends call to swap function
3. MonBridge receives 50,000 USDC from user
4. MonBridge routes USDC through venue(s)
5. MonBridge receives WETH
6. MonBridge unwraps WETH to ETH
7. MonBridge sends ETH to user
8. Transaction complete
```

### Token to Token

**Use Case**: Swap one ERC20 for another

```
User has: 100,000 LINK
User wants: 50,000 USDC
User calls: swapExactTokensForTokens(amountIn, amountOutMin, path, recipient)

Execution:
1. User approves MonBridge for LINK
2. MonBridge receives LINK from user
3. MonBridge routes through selected path(s)
4. MonBridge receives USDC
5. MonBridge sends USDC to user
6. Transaction complete
```

---

## 5. Fee-on-Transfer Token Execution

### Specialized Handling

**Standard ERC20 Flow**:
```
Send 100 tokens
Receive 100 tokens
Calculate on: 100 tokens
```

**Fee-on-Transfer Token Flow**:
```
Send 100 tokens
2% fee deducted
Receive 98 tokens ← Must account for this!
```

### Execution Example

```
User: Swap 10,000 SHIB for USDC
SHIB has 2% transfer fee

Standard (Broken):
Send 10,000 SHIB
Expect 10,000 SHIB to arrive at DEX
Actually receive 9,800 SHIB (2% fee)
Calculation mismatch → Error

MonBridge Handling:
1. Detect SHIB is fee-on-transfer
2. Adjust expected amount: 10,000 * 98% = 9,800 SHIB
3. Route 10,000 SHIB to venue
4. Venue receives 9,800 SHIB
5. Get quote based on 9,800 SHIB
6. Execute knowing actual amount
7. Receive corresponding USDC
8. Send to user
Success ✓
```

---

## 6. Real-Time Execution Flow (Detailed)

### Complete Execution Timeline

```
T+0ms: User Submits Request
- User calls: swapExactTokensForTokens(
    100 USDC,
    95 DAI min,
    [USDC, DAI],
    user,
    deadline
  )
- User's wallet signs transaction

T+100ms: Transaction Enters Mempool
- Transaction visible to network
- Waiting for block inclusion

T+2000ms: Block Included
- Network nodes begin execution

T+2100ms: MonBridge Contract Receives Call
- Validates inputs
- Checks user has approved USDC
- Checks amount > 0

T+2150ms: Price Discovery
- Query Venue 1 (Uniswap V2): 100 USDC → 99.5 DAI
- Query Venue 2 (Uniswap V3): 100 USDC → 99.7 DAI
- Query Venue 3 (Other DEX): 100 USDC → 99.2 DAI

T+2200ms: Route Selection
- Evaluate quotes
- Selected: Venue 2 (99.7 DAI best)
- Check health: Venue 2 active ✓
- Check liquidity: $100M available ✓

T+2250ms: Validation
- Slippage check: 99.7 vs 95 min → Acceptable ✓
- TWAP check: Price vs 30-min average → Within bounds ✓
- User allowance: >= 100 USDC → Sufficient ✓

T+2300ms: Token Transfer (Input)
- Contract calls USDC.transferFrom(user, Venue2, 100)
- Venue 2 receives 100 USDC

T+2350ms: Swap Execution
- Contract calls Venue2.swap(100 USDC for ≥95 DAI)
- Venue 2 executes swap
- Venue 2 sends output

T+2400ms: Receive Output
- Contract receives 99.7 DAI from Venue 2
- Verify amount >= 95 ✓

T+2450ms: Fee Accumulation
- Fee: 100 * 0.1% = 0.1 USDC (collected from input)
- feeAccumulated[USDC] += 0.1 USDC

T+2500ms: User Settlement
- Contract transfers 99.7 DAI to user

T+2550ms: Event Emission
- Log SwapExecuted event:
  User: user address
  Router: Venue 2 address
  TokenIn: USDC
  TokenOut: DAI
  AmountIn: 100
  AmountOut: 99.7
  Fee: 0.1
  Slippage: 30 BPS
  Type: TokenToToken

T+2600ms: Transaction Complete
- All state changes finalized
- User has received 99.7 DAI
- Protocol has accumulated 0.1 USDC
- On-chain record created
```

---

## 7. Atomic vs Non-Atomic Execution

### Atomic Execution (All MonBridge Transactions)

**Definition**: Either complete fully or fail completely; no partial execution

```
Example: Swap 1000 USDC for DAI

All-or-Nothing:
✓ User sends 1000 USDC → Receive ~1000 DAI
✓ Or entire transaction reverts → User keeps 1000 USDC, pays gas only

NO partial execution:
✗ User sends 1000 USDC → Receives only 500 DAI (NOT allowed)
✗ This never happens in MonBridge
```

### Failure Scenarios

**Scenario 1: Insufficient Output**
```
Trade: 1000 USDC → ≥999 DAI minimum
Quote: 990 DAI available

Action: Transaction reverts
Result:
- No USDC transferred
- No DAI received
- User keeps 1000 USDC
- Only gas fee paid (as compensation for failed attempt)
```

**Scenario 2: Venue Failure**
```
Primary venue becomes unavailable mid-transaction

Action: Contract attempts secondary venue
If success: Swap completes with alternative venue
If failure: Transaction reverts, user unaffected
```

**Scenario 3: Slippage Exceeded**
```
Trade: 1000 USDC → ≥950 DAI (5% slippage max)
Quote: 930 DAI (7% slippage)

Action: Transaction reverts
Result: User unaffected, no funds transferred
```

### Benefits of Atomicity
- User never loses funds (only gets or doesn't get swap)
- No "stuck" transactions with partial execution
- Clear success/failure outcomes
- Safe for automated trading

---

## 8. Performance & Gas Optimization

### Single-Hop vs Multi-Hop Gas Cost ( MONAD)

```
Single-Hop Trade Gas: ~150,000 gas (~$0.005-0.01)
Multi-Hop Trade Gas: ~250,000 gas ($0.01-0.02)
Split Trade (2 venues) Gas: ~350,000 gas ($0.02-0.05)
Split Trade (4 venues) Gas: ~650,000 gas ($0.05-0.1)

Gas varies by:
- Network congestion
- Token complexity
- Transfer fee detection
- Number of validations required
```

### Cost-Benefit Analysis

```
Example: 10,000,000 USDC → DAI

Single-Hop:
- Gas: 150K = $5
- Slippage: 10% = $1,000,000 (POOR)
- Total cost: $1,000,005

Split Trade (4 venues):
- Gas: 650K = $20
- Slippage: 1% = $100,000 (GOOD)
- Total cost: $100,020

Benefit: $900,000 saved despite higher gas
```

---

**Next**: See [Economic Model](07-economics.md) for fee structure and protocol economics.
