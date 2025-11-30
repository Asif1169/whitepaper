# Security Framework

## Comprehensive Security Architecture

MonBridge implements multiple overlapping security layers to protect users and protocol integrity.

---

## 1. Reentrancy Prevention

### The Reentrancy Vulnerability
```
Vulnerable Pattern:
1. Contract sends tokens to external address
2. External address (attacker) calls back into contract
3. Before token transfer complete, attacker exploits state
4. Attacker transfers funds multiple times

Result: Funds stolen
```

### MonBridge Protection: State Lock
```solidity
bool private _locked

modifier nonReentrant() {
    require(!_locked, "Reentrant call")
    _locked = true
    // ... execute logic
    _locked = false
}
```

**How It Works**:
1. Before any external call, set lock to true
2. If attacker tries recursive call, lock prevents re-entry
3. External call completes
4. Lock released

**Protection Level**: High - Prevents all reentrancy attacks

---

## 2. Access Control Framework

### Role-Based Permissions

**Owner Authority**:
- Add/remove routers
- Configure slippage parameters
- Configure liquidity requirements
- Emergency pause/resume
- Withdraw accumulated fees

**User Capabilities**:
- Call swap functions (public)
- Query pricing (public)
- Check venue status (public)
- View fees (public)

### Access Control Enforcement
```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "Not owner")
    _
}

// Only owner can call:
function addRouter(address router) external onlyOwner { }
function setSlippageConfig(...) external onlyOwner { }
```

**Protection Level**: High - Sensitive operations protected

---

## 3. Oracle-Based Price Protection (TWAP)

### Flash Loan Attack Vector

**Attack Scenario**:
```
1. Flash loan 1M DAI at start of block
2. Dump DAI on Uniswap (artificially move price)
3. Trade through MonBridge at manipulated price
4. Repay flash loan at end of block
5. Attacker profits from temporary price movement
```

### TWAP Defense Mechanism

**Time-Weighted Average Price**:
```
Price Average = Sum of (Price_i * Time_i) / Total Time

Example:
1:00 PM - Price: $1.00
1:15 PM - Price: $1.02
1:30 PM - Price: $0.98

30-min TWAP = (1.00*15 + 1.02*15 + 0.98*15) / 45 = $1.00
Actual price: $1.05 (abnormal movement)

TWAP vs Actual = 5% deviation
If threshold is 2%, swap rejected
```

### Configuration
```solidity
struct TWAPConfig {
    uint32 twapInterval = 1800        // 30 minutes
    uint16 maxPriceDeviation = 500    // 5% max deviation
    bool enableTWAPCheck = true       // Active
}
```

### Protection Effect
```
Normal Swap:
Price within 5% of TWAP → Allowed

Flash Loan Attack:
Artificial price spike → Detected as > 5% deviation → Rejected
```

**Protection Level**: High - Prevents flash loan exploitation

---

## 4. Liquidity Validation

### Minimum Liquidity Requirements

**Purpose**: Prevent trading through illiquid pools that cause extreme slippage.

```
Requirement: $1,000,000 minimum USD liquidity

Pool A: $50M liquidity → APPROVED
Pool B: $500K liquidity → REJECTED
```

**Benefits**:
- Ensures quality venues
- Reduces slippage
- Prevents exploitation through illiquid venues

### Venue Quality Assessment

**Metrics**:
- Success/failure rates
- Historical slippage
- Total volume routed
- Liquidity depth

**Disable Venues**:
- Automatic suspension if exceeding failure threshold
- Prevents cascading failures
- Alternative venues still available

---

## 5. Smart Contract Risk Mitigation

### Arithmetic Safety

**Overflow/Underflow Protection**:
```
Vulnerable: x + y could exceed max uint256
Protected: Use safe math library for all calculations
```

**Multi-Precision Calculations**:
- Prevent precision loss
- Handle varying token decimals
- Accurate fee calculations

### State Management

**Check-Effect-Interaction Pattern**:
```
✓ Check conditions (validation)
✓ Update state (effects)
✓ Call external contracts (interactions)

NOT:
✗ Call external contracts first (opens reentrancy)
✗ Update state after (exposes invalid intermediate state)
```

### Immutable Deployment

Once deployed:
- Code cannot be modified
- Logic cannot be changed
- Guarantees users' rules don't change
- Prevents post-deployment compromises

---

## 6. Token-Specific Security

### Fee-on-Transfer Token Handling

**Risk**: Standard ERC20 assumptions don't hold
```
Normal: Transfer 100 tokens → Receive 100 tokens
Fee Token: Transfer 100 tokens → Receive 98 tokens (2% fee)

Unprotected code:
Send 100 tokens
Expect 100 tokens
Receive 98 tokens
Calculation fails → Swap fails or loses funds
```

**MonBridge Protection**:
- Detect fee-on-transfer mechanism
- Calculate net amounts correctly
- Route through compatible venues
- Account for fees in output calculations

### Token Blacklisting

**Malicious Token Detection**:
```
Blacklist prevents:
├─ Honeypot tokens (cannot withdraw)
├─ Rug pull tokens (unlimited minting)
├─ Backdoored tokens (admin controls)
└─ Scam tokens (stealing private keys)
```

**Implementation**:
```solidity
mapping(address => bool) public tokenBlacklist

modifier validTokens(address[] calldata path) {
    for (uint i = 0; i < path.length; i++) {
        require(!tokenBlacklist[path[i]], "Blacklisted")
    }
    _
}
```

### Decimal Precision

**Risk**: Different tokens have different decimals
```
Token A: 6 decimals (USDC)
Token B: 18 decimals (DAI)

Without handling:
100 USDC = 100 * 10^6 = 100,000,000
100 DAI = 100 * 10^18 = 100,000,000,000,000,000

Comparison: 100,000,000 vs 100,000,000,000,000,000 (Wrong!)
```

**MonBridge Normalization**:
- Query token decimals
- Convert to common base for comparison
- Accurate price calculations across decimals

---

## 7. Transaction-Level Safety

### Slippage Protection

**User Specifies**: "I accept maximum 1% slippage"

**Execution**:
1. Get quote: Token B amount = 990 (1% less than expected 1000)
2. Set minimum output: 990 Token B
3. Execute swap
4. Verify received >= 990
5. If not, transaction reverts (all changes undone)

**Protection**: User receives quoted amount or entire transaction fails

### Deadline Protection

**Risk**: Transaction stuck in mempool, executed hours later at old price
```
User submits swap at 2:00 PM
Network congestion
Transaction executes at 3:30 PM
Price changed significantly
User receives worse price than expected
```

**MonBridge Solution**:
```solidity
require(block.timestamp <= deadline, "Expired")
```

If transaction not included before deadline, it reverts.

### Amount Validation

**Before Execution**:
- Verify user has sufficient input tokens
- Verify MonBridge has router allowance
- Check sufficient liquidity at venue

**After Execution**:
- Verify output tokens received
- Verify amount meets minimum threshold
- Verify correct recipient

---

## 8. Venue-Level Protections

### Router Health Tracking

```
Router Info Tracked:
├─ Active status (enabled/disabled)
├─ Last successful swap (timestamp)
├─ Failure count (consecutive failures)
└─ Total volume (cumulative routed)

Response Strategy:
- 0-5 failures: Monitor closely
- 5-10 failures: Increase monitoring
- >10 failures: Disable venue
- Manual re-enable by owner after investigation
```

### Automatic Failover

```
Primary Router Fails
    ↓
Increment failure counter
    ↓
If threshold exceeded → Disable
    ↓
Try Secondary Router
    ↓
If successful → Continue
    ↓
If all fail → Transaction reverts
```

### Venue Selection Logic

Routes traffic to:
1. **Active routers only** (disabled routers skipped)
2. **Best pricing** (among active routers)
3. **High health score** (prefer reliable venues)
4. **Sufficient liquidity** (meets minimum requirements)

---

## 9. Circuit Breaker (Emergency Controls)

### Pause Mechanism

```solidity
bool public paused

modifier whenNotPaused() {
    require(!paused, "Paused")
    _
}

function pause() external onlyOwner {
    paused = true  // Halts all swaps
}

function resume() external onlyOwner {
    paused = false  // Resumes operations
}
```

**Use Cases**:
- Security vulnerability discovered
- Unusual market conditions detected
- Exploitation attempt identified
- Network consensus issues

**Effect**:
- New swaps blocked
- Existing state preserved
- Can be resumed safely

---

## 10. Event Logging & Audit Trail

### Comprehensive Event Recording

```solidity
event SwapExecuted(
    address indexed user,
    address indexed router,
    address tokenIn,
    address tokenOut,
    uint amountIn,
    uint amountOut,
    uint fee,
    uint actualSlippage,
    SwapType swapType
)
```

**Recorded Information**:
- Who executed (user address)
- Which router was used
- Input/output tokens
- Amounts involved
- Fee collected
- Slippage experienced
- Type of swap

**Auditability**:
- Every swap visible on-chain
- Cannot be modified or hidden
- Permanent record
- Queryable by anyone

---

## Security Trade-offs

### What MonBridge Optimizes For

✅ **Smart Contract Security**: On-chain logic is immutable and transparent  
✅ **Flash Loan Protection**: TWAP oracle prevents manipulation  
✅ **Reentrancy Safety**: State lock prevents recursive attacks  
✅ **Access Control**: Owner-gated sensitive operations  
✅ **Token Safety**: Blacklisting and fee-on-transfer support  

### Inherent Limitations

⚠️ **Venue Risk**: Depends on quality of integrated DEX venues  
⚠️ **User Error**: Cannot prevent user from setting wrong slippage tolerance  

### Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|-----------|
| **Bad Token** | Medium | Blacklist + fee detection |
| **Bad Venue** | Medium | Health monitoring + failover |
| **User Error** | Low | Clear pricing display |

---

## Ongoing Security Practices

### Code Auditing

- Continuous monitoring for vulnerabilities
- Transparent disclosure of findings



### Upgrade Path
- Smart contract proxy patterns (if needed)
- Careful testing before deployment
- Gradual rollout for major changes

---

**Next**: See [Trade Execution Models](06-trade-execution.md) for how execution happens within this security framework.
