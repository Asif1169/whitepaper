# Core Features

## 1. Multi-Protocol Liquidity Aggregation

### Overview
MonBridge queries and aggregates liquidity from multiple DEX protocols simultaneously, providing unified access to fragmented liquidity sources.

### Supported Protocols
- Constant Product Market Makers (CPMM)
- Concentrated Liquidity Models
- Multiple DEX Router Standards
- Various Fee Tiers and Pool Configurations

### Key Benefits
- **Consolidated Access**: Query multiple venues from single transaction
- **Best Pricing**: Automatic routing to venue with optimal pricing
- **Liquidity Aggregation**: Access combined liquidity across protocols
- **Automatic Fallback**: If one venue unavailable, routes through alternatives
- **Venue Redundancy**: Never dependent on single liquidity source

### Implementation
- Direct on-chain queries to DEX factory and router contracts
- Real-time reserve and pricing data
- Continuous venue monitoring and health tracking

---

## 2. Intelligent Trade Routing

### Multi-Hop Path Analysis
MonBridge evaluates complex multi-hop swap sequences:

**Direct Path**:
```
Token A → DEX → Token B (1 hop)
```

**Multi-Hop Path**:
```
Token A → DEX1 → Token C → DEX2 → Token B (2+ hops)
```

**Benefit**: Accesses deeper liquidity through intermediate tokens, reducing slippage for large trades.

### Route Optimization Algorithm
- Evaluates all possible paths
- Calculates price impact for each route
- Selects route minimizing total cost (slippage + fees)
- Factors in venue reliability and health

### Automatic Split Trading
For large orders, MonBridge can split execution across multiple venues:

```
Total Order: 10,000 USDC → Token X

Strategy 1: Single Venue
10,000 USDC → DEX1 → 5% slippage = 9,500 Token X

Strategy 2: Split Across 2 Venues  
5,000 USDC → DEX1 → 2% slippage = 4,900 Token X
5,000 USDC → DEX2 → 2% slippage = 4,900 Token X
Total = 9,800 Token X (better outcome)
```

### Dynamic Route Selection
- Impact-triggered splitting: Activates when single venue impact exceeds threshold
- Configurable split limits: Maximum number of splits per order
- Automatic optimization: Determines optimal split distribution

---

## 3. Advanced Slippage Management

### Slippage Configuration Framework

**Three-Tier System**:
1. **Base Slippage**: Minimum acceptable slippage (e.g., 50 BPS = 0.5%)
2. **Impact Multiplier**: Adjustment based on calculated price impact
3. **Maximum Slippage**: Hard ceiling for safety (e.g., 500 BPS = 5%)

### User-Defined Thresholds
Traders specify maximum acceptable slippage:
```
Swap USDC for DAI
Max Slippage: 100 BPS (1%)

If actual slippage would exceed 100 BPS → Transaction reverts
User protection against unexpected price movement
```

### Impact-Responsive Adjustment
- Contract calculates expected price impact
- Adjusts acceptable slippage based on impact
- Larger trades automatically get higher slippage allowance
- Prevents reverting on legitimate large orders

### Real-Time Price Validation
Before execution:
1. Get user's minimum output amount
2. Query DEX for actual output
3. Validate actual meets or exceeds minimum
4. Transaction only proceeds if validated

---

## 4. Liquidity Quality Assessment

### Minimum Liquidity Requirements
- Enforces USD-denominated liquidity thresholds
- Prevents trading through illiquid pools
- Configurable per venue or globally
- Reduces slippage by using quality venues

### Venue Health Tracking

**Metrics Tracked**:
- Success rate of executed swaps
- Failure count for each venue
- Total volume routed through venue
- Last successful swap timestamp
- Current active status

**Example**:
```
Router A: 999 successful swaps, 1 failure = 99.9% success
Router B: 900 successful swaps, 100 failures = 90% success
Router C: 10 successful swaps, 10 failures = 50% success

Routing prioritizes Router A, uses B as fallback, C disabled
```

### Automatic Venue Suspension
- Configurable failure threshold (e.g., 10 failures)
- Venues automatically disabled when threshold exceeded
- Prevents cascading failures through bad venues
- Manual re-enabling by protocol owner

### Dynamic Activation/Deactivation
- Real-time venue status monitoring
- Automatic routing around disabled venues
- Manual override capability
- Transparent venue status queryable

---

## 5. Security & Protective Mechanisms

### Reentrancy Protection
```
check → effect → interaction pattern
State locked during execution
Prevents recursive malicious calls
Protects user funds and contract state
```

### Access Control
- Owner authorization for sensitive operations
- Role-based permissions
- Cannot be overridden by users
- Prevents unauthorized configuration changes

### Circuit Breakers
- Emergency pause mechanism
- Halts new swaps during security events
- Preserves existing state
- Owner-activated only

### Oracle-Based Price Protection
**TWAP (Time-Weighted Average Price)**:
- Averages prices over time intervals (e.g., 30 minutes)
- Detects flash loan manipulation attempts
- Prevents use of artificially moved prices
- Configurable deviation thresholds

**Flash Loan Prevention**:
```
Flash Loan Attack Scenario:
1. Attacker borrows large token amount
2. Artificially moves price
3. Creates favorable swap conditions
4. Attempts swap through MonBridge

TWAP Protection:
- Historical price averaging detects unnatural movement
- Swap rejected if price deviates from TWAP too much
- Attack fails, attacker loses loan
```

### Fee-on-Transfer Token Support
- Detects tokens implementing transfer fee mechanics
- Handles non-standard ERC20 implementations
- Calculates net amounts accurately
- Prevents failures on exotic tokens

### Token Blacklisting
- Maintains list of suspicious/malicious tokens
- Prevents swaps involving blacklisted tokens
- Protects users from scam tokens
- Configurable by protocol governance

---

## 6. Fee-on-Transfer Token Compatibility

### Token Type Detection
```
Standard ERC20 Token:
Send 100 tokens → Recipient receives 100 tokens

Fee-on-Transfer Token:
Send 100 tokens → 2% fee → Recipient receives 98 tokens
```

### Handling Mechanisms
- Automatic fee detection
- Fee percentage recording
- Special routing for compatibility
- Accurate amount calculation despite fees

### Use Case
```
User wants: 10,000 SHIB → USDC
SHIB is fee-on-transfer (2% fee)

Without MonBridge:
Send 10,000 SHIB → Contract applies 2% fee → Only 9,800 SHIB received
Swap fails or results in wrong amounts

With MonBridge:
Detects fee-on-transfer mechanism
Routes through compatible venue
Accounts for 2% fee in amounts
Swap succeeds with correct output
```

---

## 7. Transparent Economic Model

### Protocol Fee Structure
```
User Swap: 1,000 USDC → DAI
Expected Output: 1,000 DAI (assuming 1:1 price)

Fee Divisor: 1000 (means 0.1% fee)
Protocol Fee: 1,000 USDC * 0.1% = 1 USDC

Result:
- MonBridge Routes: 999 USDC through DEX
- User Receives: ~999 DAI (after DEX fees)
- MonBridge Accumulates: 1 USDC to treasury
```

### Multi-Denomination Fee Accumulation

**ETH Swaps**:
```
User swaps ETH → Token
Fee collected in ETH
Stored in protocol contract: feeAccumulatedETH
```

**Token Swaps**:
```
User swaps Token A → Token B
Fee collected in Token A
Stored in mapping: feeAccumulated[tokenA]
```

### Fee Transparency
- All fees visible before swap execution
- User knows exact amount to be charged
- No hidden or surprise fees
- Fees auditable on-chain

---

## 8. Router Health Monitoring

### Continuous Performance Tracking
```
Each Router Tracked:
├─ isActive: Currently enabled
├─ lastSuccessfulSwap: Timestamp of last success
├─ failureCount: Consecutive failures
└─ totalVolume: Cumulative volume routed
```

### Automatic Response
```
During Swap Execution:
1. Attempt through primary router
2. If fails:
   - Increment failure counter
   - Log failure timestamp
   - If failures exceed threshold → Disable router
3. Try next available router
4. Continue until successful or all routers exhausted
```

### Health Status Dashboard (On-Chain Queryable)
```
Router A: ACTIVE (999 swaps, 1 failure, 1M volume)
Router B: ACTIVE (900 swaps, 0 failures, 950K volume)
Router C: DISABLED (Exceeded failure threshold)
Router D: ACTIVE (500 swaps, 2 failures, 480K volume)
```

---

## 9. Emergency Controls

### Pause Mechanism
- Owner can pause contract instantly
- Prevents new swaps during security events
- Does not affect existing state
- Can be resumed after threat resolved

### Manual Router Management
- Owner can manually add/remove routers
- Enables rapid response to venue issues
- Override automatic systems if needed
- Emergency bypass capability

---

## Feature Comparison: MonBridge vs. Alternatives

| Feature | Centralized Agg. | MonBridge |
|---------|-----------------|-----------|
| **Liquidity Aggregation** | Yes | Yes |
| **Multi-Hop Routing** | Yes | Yes |
| **Split Trading** | Sometimes | Yes |
| **Slippage Control** | Yes | Advanced |
| **Server Dependency** | Critical | None |
| **API Reliability** | Variable | N/A |
| **Censorship Resistant** | No | Yes |
| **24/7 Availability** | Not guaranteed | Guaranteed |
| **Code Transparency** | No | Full |
| **Flash Loan Protection** | Sometimes | Yes (TWAP) |
| **Fee Structure** | Hidden | Transparent |
| **Venue Redundancy** | Limited | Automatic |

---

**Next**: See [Trade Execution Models](06-trade-execution.md) for detailed explanation of how trades execute through MonBridge.
