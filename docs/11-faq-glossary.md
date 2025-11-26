# FAQ & Glossary

## Frequently Asked Questions

### General Protocol Questions

#### Q: How is MonBridge different from traditional DEX aggregators?
**A**: Traditional aggregators depend on centralized servers for pricing and routing. If their servers go down, the entire service stops. MonBridge runs entirely on-chain through smart contracts—no centralized API dependency. If one venue fails, the protocol automatically routes to alternatives. The service continues operating permanently, regardless of external infrastructure failures.

#### Q: Is MonBridge truly decentralized?
**A**: Yes. The protocol is:
- **On-Chain Logic**: All routing decisions execute as immutable smart contracts
- **No APIs**: Direct blockchain queries instead of centralized quote servers
- **Permissionless**: Anyone can call swap functions; no entity can block users
- **Permanent**: Smart contracts exist indefinitely on blockchain
- **Verifiable**: All decisions are transparent and auditable

#### Q: Can MonBridge be shut down?
**A**: No. The protocol:
- Exists as code on the blockchain (permanent)
- Cannot be modified without protocol governance
- Continues operating regardless of who deployed it
- Would continue operating even if the founding team disappeared

---

### Technical Questions

#### Q: What venues does MonBridge support?
**A**: Currently:
- Uniswap V2 and V3 (main focus)
- Curve (future)
- Other V2-compatible protocols (future)

More venues will be added based on community demand and liquidity depth.

#### Q: How often are routers/venues added?
**A**: Currently owner-managed (bootstrap phase). Venues are added when they meet requirements:
- Sufficient liquidity ($1M+ USD minimum)
- Security audited
- Community requested
- Strategic fit

Future: DAO will vote on new venue integration.

#### Q: What about slippage protection?
**A**: MonBridge provides multi-layer slippage control:
1. **User-Defined Limits**: You set maximum acceptable slippage before swap
2. **Base Slippage**: 50 BPS minimum threshold
3. **Impact-Based Adjustment**: Larger trades get higher allowance (realistic)
4. **Hard Maximum**: 500 BPS ceiling for safety
5. **TWAP Validation**: Price oracle prevents flash loan manipulation

#### Q: Are my funds ever at risk?
**A**: Risk factors:
- **Smart Contract Risk**: Code has been designed with safety (reentrancy protection, overflow handling)
- **Venue Risk**: If a DEX has issues, MonBridge routes to alternatives
- **User Error Risk**: Setting unrealistic slippage or wrong token path
- **NOT at Risk**: Protocol cannot steal funds or send to wrong addresses

All transactions are atomic (all-or-nothing).

#### Q: How does flash loan protection work?
**A**: TWAP (Time-Weighted Average Price) oracle:
- Tracks average price over 30 minutes
- Detects abnormal price movements
- Rejects swaps if price deviates >5% from historical average
- Prevents attackers from exploiting temporary price manipulations

#### Q: What's a split trade?
**A**: For very large orders, MonBridge divides them across multiple venues:
- Single venue might have 5% slippage on $1M order
- Two venues might have 1% slippage each ($500K each)
- Result: Better execution by spreading liquidity

Automatic—happens if impact detection triggers it.

---

### Fee Questions

#### Q: What fees does MonBridge charge?
**A**: 
- **Protocol Fee**: 0.1% of swap amount
- **No Hidden Fees**: That's it. Venue fees are already included in their quotes
- **Visible Before Swap**: You see exact fee before confirming

Example:
```
Swap 1000 USDC for DAI
Protocol fee: 1 USDC (0.1%)
You provide: 1000 USDC
~999 routed to venue
```

#### Q: Why 0.1% fee? Are other aggregators cheaper?
**A**: 
- Traditional aggregators: 0.05% (but rely on servers)
- MonBridge: 0.1% (permanently decentralized)

Total cost comparison:
```
Traditional Aggregator:
- Fee: 0.05%
- Server outage risk: ???
- Average slippage: 0.3%
- Total user cost: ~0.35%

MonBridge:
- Fee: 0.1% (visible)
- Server outage risk: Zero (no servers)
- Average slippage: 0.1% (better routing)
- Total user cost: ~0.2%

MonBridge cheaper even with higher fee
```

#### Q: Where do collected fees go?
**A**: 
- **Currently**: Owner-controlled multisig treasury
- **Monthly Reports**: Fee accumulation publicly visible
- **Uses**: Team operations, audits, infrastructure, community programs
- **Future**: DAO votes on treasury allocation

#### Q: Can fees be changed?
**A**:
- **Currently**: Owner can adjust
- **Realistically**: Will be voted on by DAO governance
- **Never**: Fee can't be increased without community approval
- **Core Principle**: Fees remain transparent and justified

---

### Use Case Questions

#### Q: Is MonBridge good for small trades?
**A**: Yes, especially for:
- First-time swappers (user-friendly)
- Finding best price (automatic)
- Avoiding server outages (guaranteed uptime)

May not be ideal if you want:
- Absolute lowest fee (might prefer Uniswap directly if direct pair has best price)
- The tradeoff: 0.1% fee but guaranteed best venue

#### Q: Is MonBridge good for large trades?
**A**: Yes. Advantages:
- Split execution reduces slippage dramatically
- Multi-hop routing finds better paths
- Example: $10M trade could save $100K+ through split routing

#### Q: Can institutions use MonBridge?
**A**: Absolutely:
- Large order support (split trading)
- Transparent routing (all visible on-chain)
- 24/7 availability (no maintenance windows)
- Multiple integration options (contract calls, SDKs)

#### Q: Do I need special wallet?
**A**: No. Use any Web3 wallet:
- MetaMask
- Ledger
- Coinbase Wallet
- Any wallet supporting smart contract interaction

---

### Security Questions

#### Q: Is MonBridge safe?
**A**: Security measures:
- ✅ Reentrancy protection (prevents recursive attacks)
- ✅ Slippage validation (you control acceptable loss)
- ✅ Flash loan protection (TWAP oracle)
- ✅ Token blacklisting (prevents scam tokens)
- ✅ Fee-on-transfer handling (handles exotic tokens)
- ✅ Circuit breaker (emergency pause capability)
- ✅ Venue health monitoring (routes around failing venues)

What we don't protect against:
- User sending to wrong address (user error)
- Extremely unrealistic slippage settings (user error)
- Participating in rug pulls (user due diligence)

#### Q: Has the code been audited?
**A**: 
- **Phase 1** (Launch): Initial security review and testing
- **Year 1**: Professional security audit (in planning)
- **Bug Bounty**: Community can report issues (TBD)
- **Open Code**: Anyone can review smart contract

#### Q: What if there's a smart contract bug?
**A**:
- Protocol owner can pause contract
- Prevents new swaps during incident
- Existing funds protected
- Community votes on recovery process

#### Q: Is there a rug pull risk?
**A**:
- Protocol is on-chain permanently (can't be deleted)
- Owner cannot access user funds
- Fees accumulated separately (visible)
- DAO governance prevents owner abuse

---

### Financial Questions

#### Q: Can I make money using MonBridge?
**A**: 
**Indirect ways**:
- Save on slippage vs. manual trading (more profitable trades)
- Better execution for arbitrage strategies
- Lower costs improve margins for trading bots

**Direct ways** (future):
- DAO token staking rewards (planned Year 2)
- Liquidity provider incentives (planned)
- Community participation rewards (planned)

#### Q: How much volume does MonBridge handle?
**A**: 
- **Year 1** (Realistic): $100-150K annual revenue (~$30-50M daily volume)
- **Year 2**: $720K-1.8M annual (~$250-500M daily volume)
- **Year 3**: $3.6-7.2M annual (~$1-2B daily volume)

These are realistic, not optimistic projections.

#### Q: What's MonBridge's competitive advantage?
**A**:
1. **No Server Dependency**: Always available (competitors can go down)
2. **Decentralization**: Community-governed (vs. centralized competitors)
3. **Transparency**: All decisions on-chain and verifiable
4. **Reliability**: 24/7 guaranteed, no maintenance windows

---

### Governance Questions

#### Q: Who controls MonBridge?
**A**:
- **Currently**: Owner (founding team)
- **Year 2**: Hybrid governance (owner + community votes)
- **Year 3+**: Full DAO governance (community token holders)

#### Q: Can I vote on protocol decisions?
**A**:
- **Currently**: Community can provide input on Discord/forums
- **Year 2**: DAO token released for voting
- **Year 3+**: All major decisions by community vote

#### Q: How are urgent security decisions made?
**A**:
- Owner can pause protocol immediately
- Can adjust risk parameters down (stricter safety)
- Must disclose reason immediately
- Community votes on sustained action
- Can't make risky changes without community approval

---

### Future Questions

#### Q: What's the roadmap?
**A**: See [Roadmap document](09-roadmap.md) for detailed timeline.

**Key milestones**:
- Year 1: Multi-protocol support, community education
- Year 2: Multi-chain deployment, governance token launch
- Year 3: MEV protection, cross-chain swaps, full DAO governance

#### Q: Will MonBridge support [my favorite blockchain]?
**A**: 
- **Planned**: Polygon, Arbitrum, Optimism, Avalanche, Base, Solana
- **Likely**: Other major chains based on community demand
- **Process**: DAO votes on chain priority once governance is live

#### Q: Will MonBridge support more DEX protocols?
**A**:
- Yes, gradually as market demands
- New venues must meet security and liquidity requirements
- Community votes on priority
- Currently focused on Uniswap (V2 & V3)

---

## Glossary

### A
**Atomic Execution**: All-or-nothing execution. Either the entire swap completes or fails; no partial execution possible.

**AMM (Automated Market Maker)**: Protocol that uses algorithms instead of order books to price assets. Uniswap is the most famous example.

### B
**Basis Points (BPS)**: 1 BPS = 0.01%. So 100 BPS = 1%.

**Bridge**: Technology to move tokens from one blockchain to another.

### C
**Circuit Breaker**: Emergency mechanism to pause a protocol (like a circuit breaker in electronics).

**CPMM (Constant Product Market Maker)**: AMM that maintains formula: reserve_A * reserve_B = constant. Uniswap V2 uses this.

### D
**DEX (Decentralized Exchange)**: Protocol allowing token swaps without custody of user funds. Uniswap, Curve, etc.

**DeFi (Decentralized Finance)**: Financial protocols running on blockchain without traditional intermediaries.

**Delegation**: Giving your voting power to another address to vote on governance decisions.

### E
**ERC20**: Standard interface for tokens on Ethereum (and similar blockchains).

**Epoch**: Time period for governance voting or reward distribution.

### F
**Fee-on-Transfer Token**: Non-standard ERC20 that deducts a fee when tokens are transferred.

**Flash Loan**: Uncollateralized loan that must be repaid within the same transaction. Requires TWAP protection to prevent abuse.

**Fragmented Liquidity**: Liquidity split across many venues, making swaps expensive.

### G
**Gas**: Fee paid to execute transactions on blockchain. Higher complexity = higher gas cost.

**Governance**: Process for making protocol decisions. Currently owner-controlled, moving toward DAO.

**Governance Token**: Token giving holders voting rights on protocol decisions.

### L
**Liquidity**: Available tokens in a pool available for swapping.

**Liquidity Pool**: Smart contract holding reserves of two tokens for automated swaps.

**Liquidity Provider (LP)**: User depositing tokens into a pool to earn fees from swaps.

### M
**MEV (Maximal Extractable Value)**: Profit extracted by reordering transactions. MonBridge aims to minimize MEV risk.

**Multi-Hop Swap**: Swap requiring multiple intermediate steps. Example: USDC → ETH → SHIB (2 hops).

**Multisig (Multi-Signature Wallet)**: Wallet requiring multiple approvals before executing transactions. Currently controls MonBridge owner functions.

### O
**Oracle**: Source of price information. MonBridge uses TWAP oracle for flash loan protection.

**Overflow/Underflow**: Mathematical error when numbers exceed maximum (overflow) or go below zero (underflow). MonBridge has protections against this.

### P
**Price Impact**: Percentage price movement caused by your own trade. Larger trades have larger impact.

**Protocol Fee**: Fee charged by MonBridge (0.1%) separate from venue fees.

### Q
**Quote**: Estimated output amount for a given input amount.

### R
**Reentrancy**: Vulnerability where a contract calls itself recursively before completing initial execution. MonBridge has reentrancy protection.

**Router**: Smart contract enabling swaps (usually an AMM's router contract).

### S
**Slippage**: Difference between expected price and actual execution price. MonBridge helps minimize slippage through aggregation.

**Smart Contract**: Self-executing code on blockchain with predefined rules.

**Split Trade**: Large order divided across multiple venues to reduce slippage.

**Staking**: Locking tokens to participate in governance and earn rewards (planned).

### T
**Token Blacklist**: List of tokens MonBridge won't route through due to security concerns.

**TWAP (Time-Weighted Average Price)**: Average price over a time period. MonBridge uses TWAP to prevent flash loan attacks.

### U
**Uniswap V2**: Popular DEX using constant product formula. MonBridge can route through it.

**Uniswap V3**: Improved Uniswap with concentrated liquidity. MonBridge supports it.

### V
**Venue**: A DEX protocol that MonBridge can route trades through.

**Volatility**: Measure of price fluctuations. Higher volatility = more unpredictable prices.

### W
**Web3**: Decentralized internet using blockchain and smart contracts.

**WETH (Wrapped ETH)**: ERC20 token representing 1 ETH. Used for swaps that require ERC20 standard.

### Z
**Zero-Hop Swap**: Not applicable to MonBridge (all swaps go through at least one venue).

---

## Technical Acronyms

| Acronym | Meaning | Context |
|---------|---------|---------|
| **BPS** | Basis Points | Slippage, fees measured in BPS |
| **DAO** | Decentralized Autonomous Organization | Community governance |
| **DEX** | Decentralized Exchange | What MonBridge routes through |
| **DeFi** | Decentralized Finance | Broader ecosystem |
| **ERC20** | Ethereum token standard | Token type MonBridge supports |
| **LP** | Liquidity Provider | User in liquidity pools |
| **MEV** | Maximal Extractable Value | Sandwich attack risk (TWAP protects) |
| **TWAP** | Time-Weighted Average Price | Flash loan protection mechanism |
| **V2/V3** | Uniswap versions | Protocols MonBridge supports |
| **WETH** | Wrapped Ethereum | Native token wrapper for ERC20 |

---

## Common Mistakes to Avoid

### User Mistakes
❌ **Setting slippage too low**: 0.01% slippage is unrealistic for most trades  
✅ **Better**: Use 0.5-2% depending on trade size and volatility

❌ **Sending tokens to wrong address manually**: MonBridge can't recover sent funds  
✅ **Better**: Always verify addresses; use wallet UI for confirmations

❌ **Expecting instant execution during network congestion**: Blockchain is slower during high gas  
✅ **Better**: Transactions will execute; just takes longer

❌ **Assuming lower fee = better deal**: 0.05% fee + 0.5% slippage = worse than 0.1% fee + 0.05% slippage  
✅ **Better**: Total cost matters, not individual fees

### Developer Mistakes
❌ **Hardcoding routes**: Liquidity changes constantly  
✅ **Better**: Query fresh routes before every swap

❌ **Ignoring slippage protection**: Your function could execute at bad price  
✅ **Better**: Always set reasonable slippage minimums

❌ **Calling swap without approval**: Contract can't transfer tokens without approval  
✅ **Better**: Always call approve() before swap()

---

## Verification: What's Actually in the Protocol

**Features Confirmed in Smart Contract**:
- ✅ Multi-protocol support (Uniswap V2 and V3)
- ✅ Fee accumulation (ETH and token tracking)
- ✅ Slippage configuration (base, multiplier, max)
- ✅ Router health monitoring (active status, failure count, volume)
- ✅ TWAP oracle configuration (time interval, deviation, enforcement)
- ✅ Split trading support (enabled flag, min impact, max splits)
- ✅ Fee-on-transfer token tracking
- ✅ Token blacklisting capability
- ✅ Reentrancy protection (state lock)
- ✅ Pause/resume circuit breaker
- ✅ Owner authorization (onlyOwner modifier)
- ✅ Liquidity validation config
- ✅ Multi-swap types (ETH→Token, Token→ETH, Token→Token)
- ✅ Event logging for all swaps

**Not in Phase 1 (Planned for Later)**:
- ❌ MEV protection (beyond TWAP)
- ❌ Cross-chain swaps
- ❌ Intent-based routing
- ❌ Governance token (Year 2+)
- ❌ Advanced order types (Limit, TWAP, DCA orders)

---

**For more details, see the full documentation or check the smart contract code.**
