# Architecture & Design

## System Overview

MonBridge implements a fully decentralized architecture where all critical functions execute on-chain through smart contracts. This eliminates dependency on centralized infrastructure.

### Core Architectural Layers

```
┌─────────────────────────────────────────────┐
│      User/App Interface Layer               │
│   (Web3 Wallet, DApp Frontend)              │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│  MonBridge Smart Contract Protocol          │
│  (Fully On-Chain, Decentralized)            │
│                                              │
│  ├── Price Discovery Engine                 │
│  ├── Route Optimization Algorithm           │
│  ├── Liquidity Aggregation Layer            │
│  ├── Risk Management Framework              │
│  └── Execution & Settlement Logic           │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│  DEX Protocol Integrations                  │
│  (On-Chain DEX Smart Contracts)             │
│                                              │
│  ├── Multi-Protocol Routers                 │
│  ├── Liquidity Pair Contracts               │
│  ├── Factory Contracts                      │
│  └── Venue Health Monitoring                │
└─────────────────────────────────────────────┘
```

**Key Point**: Every component above DEX protocols exists as smart contracts. Zero external API calls. Zero centralized servers. Pure on-chain execution.

## Architecture Components

### 1. Smart Contract Protocol Layer

All MonBridge logic lives in immutable smart contracts:

**Protocol Contract Functions**:
- Price quote aggregation from multiple venues
- Optimal route calculation
- Slippage computation
- Trade execution and settlement
- Fee accounting and accumulation
- Router health tracking
- Emergency controls

**Execution Model**:
- User calls MonBridge contract with swap parameters
- Contract queries pricing from integrated DEX venues
- Contract calculates optimal route
- Contract executes swap through selected venue
- Contract settles tokens directly to user
- User receives confirmation on-chain

### 2. Price Discovery Engine

**Decentralized Operation**:
- Queries on-chain pricing from multiple DEX venues simultaneously
- No reliance on external price APIs or centralized oracles
- Direct smart contract calls to DEX factory/router contracts
- Real-time reserve queries from liquidity pair contracts

**Process**:
1. User submits swap request (Token A → Token B, Amount X)
2. MonBridge contract queries all integrated DEX venues
3. Each venue returns available liquidity and price quotes
4. Contract compares quotes locally
5. Contract identifies optimal route(s)
6. Contract executes via selected venue(s)

**Resilience**:
- If one venue is unavailable, others are queried
- Contract adapts routing around failed venues
- No single venue outage affects overall functionality

### 3. Route Optimization Engine

**On-Chain Calculations**:
- Computes optimal single-hop vs. multi-hop paths
- Evaluates split execution across multiple venues
- Calculates price impact for different route options
- Selects route minimizing total slippage + fees

**Supported Route Types**:

**Single-Hop Direct**:
```
Token A → (Venue 1) → Token B
```

**Multi-Hop Path**:
```
Token A → (Venue 1) → Token C → (Venue 2) → Token B
```

**Split Execution**:
```
Token A → (50% Venue 1) → Token B
       → (50% Venue 2) → Token B
       Result: Combined Output
```

### 4. Liquidity Aggregation

**Multi-Venue Integration**:
- Maintains active registry of integrated DEX venues
- Tracks liquidity depth at each venue
- Monitors reserve states on-chain
- Aggregates available liquidity across protocols

**Venue Management**:
- Dynamic router registration
- Health score tracking per venue
- Automatic suspension of failing venues
- Factory integration for liquidity discovery

### 5. Execution & Settlement

**Atomic Execution**:
- All swap operations are atomic—either complete entirely or revert
- Token transfers settled directly on-chain
- No intermediate custody or bridging

**Settlement Process**:
1. User approves MonBridge contract to transfer input tokens
2. User calls swap function with parameters
3. Contract receives input tokens from user
4. Contract executes routing through selected venue(s)
5. Contract receives output tokens from venue
6. Contract transfers output tokens to user
7. Contract accumulates protocol fees
8. Transaction confirmed on-chain atomically

### 6. Risk Management Framework

**On-Chain Safeguards**:

**Reentrancy Protection**:
- State lock mechanism prevents recursive calls
- Check-Effect-Interaction pattern enforced
- No external calls until state fully updated

**Slippage Controls**:
- User specifies maximum acceptable slippage
- Contract validates actual slippage against threshold
- Transaction reverts if slippage exceeds limit

**Price Validation**:
- TWAP oracle checks for flash loan manipulation
- Prices must remain within historical deviation bounds
- Prevents abuse through artificial price movement

**Venue Health**:
- Tracks failure rates per DEX venue
- Automatically disables venues exceeding thresholds
- Redirects traffic to healthy venues

**Token Validation**:
- Blacklist mechanism for suspicious tokens
- Transfer fee detection and handling
- Decimal precision normalization

## Data Structures & State Management

### Router Registry

```solidity
struct RouterInfo {
    bool isActive              // Currently enabled for routing
    uint256 lastSuccessfulSwap // Last successful transaction
    uint256 failureCount       // Consecutive failures
    uint256 totalVolume        // Cumulative volume routed
}

mapping(address => RouterInfo) routerInfo
```

Maintained entirely on-chain, queryable by anyone.

### Slippage Configuration

```solidity
struct SlippageConfig {
    uint16 baseSlippageBPS      // Minimum slippage allowance
    uint16 impactMultiplier     // Adjustment for price impact
    uint16 maxSlippageBPS       // Hard ceiling
}
```

Configurable by protocol governance, affects all swaps.

### Liquidity Validation

```solidity
struct LiquidityConfig {
    uint256 minLiquidityUSD     // Minimum USD liquidity required
    bool requireLiquidityCheck  // Whether to enforce checks
}
```

Ensures only quality venues receive routing traffic.

### TWAP Oracle Configuration

```solidity
struct TWAPConfig {
    uint32 twapInterval         // Time window for average
    uint16 maxPriceDeviationBPS // Deviation from historical norm
    bool enableTWAPCheck        // Whether oracle is active
}
```

Protects against flash loan exploitation.

## Query Path for Swaps

**Complete On-Chain Query Sequence**:

```
User Request
    ↓
MonBridge Contract Receives Request
    ↓
Query Venue 1 Liquidity & Price
Query Venue 2 Liquidity & Price
Query Venue 3 Liquidity & Price
Query Venue N Liquidity & Price
    ↓
Contract Evaluates All Quotes Locally
    ↓
Contract Calculates Optimal Route
    ↓
Contract Validates Against Risk Parameters
    ├─ Slippage within limits?
    ├─ Liquidity sufficient?
    ├─ Price oracle validation passed?
    └─ Venue health acceptable?
    ↓
Route Approved
    ↓
Contract Executes Through Selected Venue
    ↓
Contract Receives Output Tokens
    ↓
Contract Transfers to User
Contract Accumulates Fees
    ↓
Transaction Complete On-Chain
```

**Key Property**: No step above involves external APIs or centralized servers. Every operation is verifiable on-chain.

## Fee Accumulation & Protocol Economics

**On-Chain Fee Tracking**:

```solidity
uint public feeAccumulatedETH           // Native token fees
mapping(address => uint) feeAccumulated  // ERC20 token fees
```

- Fees collected during each swap
- Accumulated in protocol contract
- Withdrawable by authorized operator
- Transparent and auditable

## Emergency Controls

**Circuit Breaker Mechanism**:
- Owner can pause contract during emergencies
- Prevents new swaps during security events
- Existing state remains intact
- Can be resumed after threat addressed

**Manual Router Management**:
- Owner can manually disable failing routers
- Owner can manually add new routers
- Enables rapid response to venue issues

## Failure Modes & Recovery

| Failure Scenario | Decentralized Solution |
|-----------------|----------------------|
| **Venue Outage** | Automatically routes through alternative venues |
| **Network Congestion** | Transaction queued normally; always executes |
| **Server Failure** | No impact—no servers involved |
| **API Downtime** | No impact—no APIs used |
| **Infrastructure Issue** | No centralized infrastructure to fail |
| **Price Manipulation** | TWAP oracle detects and prevents |

**Fundamental Advantage**: Most failure modes that affect centralized aggregators simply do not apply to fully on-chain protocols.

## Comparison: Centralized vs. Decentralized Architecture

### Centralized Aggregator Stack

```
User → API Server (quote) → Processing Server → DEX
       ↓
       If servers down → Complete failure
```

### MonBridge Decentralized Stack

```
User → Smart Contract (query DEX directly) → DEX
       ↓
       If one DEX down → Route to alternative
       If network congestion → Just takes longer
       If central operator fails → Protocol still works
```

The critical difference: MonBridge removes the failure-prone middle layer entirely.

---

**Next**: See [Decentralization Model](03-decentralization-model.md) for detailed analysis of how MonBridge achieves true decentralization.
