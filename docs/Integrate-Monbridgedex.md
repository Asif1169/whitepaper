
# MonBridgeDex Integration Guide

## Overview

MonBridgeDex is a production-grade DEX aggregator deployed on Monad that supports both Uniswap V2 and V3 protocols. It provides optimal routing, multi-hop swaps, split trading, and advanced features like TWAP validation and adaptive slippage.

**Contract Address:** `0x7dd7fc9380e3107028a158f49bd25a8a8d48b225`

**Network:** Monad Mainnet (Chain ID: 143)

---

## Key Features

- ✅ **Multi-DEX Aggregation**: Compares V2 and V3 routers for best rates
- ✅ **Split Trading**: Automatically distributes large trades across 2-4 routers
- ✅ **Multi-Hop Routing**: Optimizes paths including WETH intermediary routes
- ✅ **Adaptive Slippage**: Dynamic slippage based on price impact
- ✅ **Flash Loan Protection**: TWAP validation for V3 pools
- ✅ **Fee-on-Transfer Support**: Handles tokens with transfer fees

---

## React Integration Example

### 1. Setup & Dependencies

```bash
npm install ethers@5.7.2
```

### 2. Contract Configuration

```javascript
import { ethers } from 'ethers';

const MONBRIDGEDEX_ADDRESS = '0x7dd7fc9380e3107028a158f49bd25a8a8d48b225';
const MONAD_RPC = 'https://rpc.monad.xyz';
const CHAIN_ID = 143;

// Minimal ABI for integration
const MONBRIDGEDEX_ABI = [
  "function getBestSwapData(uint256 amountIn, address[] path, bool supportFeeOnTransfer, uint16 userSlippageBPS) view returns (tuple(uint8 swapType, uint8 routerType, address router, address[] path, uint24[] v3Fees, uint256 amountIn, uint256 amountOutMin, uint256 deadline, bool supportFeeOnTransfer))",
  "function execute(tuple(uint8 swapType, uint8 routerType, address router, address[] path, uint24[] v3Fees, uint256 amountIn, uint256 amountOutMin, uint256 deadline, bool supportFeeOnTransfer) swapData) payable returns (uint256)",
  "function getBestSwapDataWithSplits(uint256 amountIn, address[] path, bool supportFeeOnTransfer, uint16 userSlippageBPS) view returns (bool useSplit, tuple(uint8,uint8,address,address[],uint24[],uint256,uint256,uint256,bool) singleSwapData, tuple(tuple(uint8,uint8,address,address[],uint24[],uint256,uint256,uint256,bool)[] splits, uint256 totalAmountIn, uint256 totalAmountOutMin, uint16[] splitPercentages) splitSwapData, uint256 estimatedOutput, uint256 totalPriceImpact)",
  "function executeSplit(tuple(tuple(uint8,uint8,address,address[],uint24[],uint256,uint256,uint256,bool)[] splits, uint256 totalAmountIn, uint256 totalAmountOutMin, uint16[] splitPercentages) splitData) payable returns (uint256)",
  "function WETH() view returns (address)"
];

const ERC20_ABI = [
  "function approve(address spender, uint256 amount) returns (bool)",
  "function allowance(address owner, address spender) view returns (uint256)",
  "function balanceOf(address account) view returns (uint256)",
  "function decimals() view returns (uint8)"
];
```

### 3. React Hook for Swapping

```javascript
import { useState, useEffect } from 'react';

export function useMonBridgeDex() {
  const [provider, setProvider] = useState(null);
  const [signer, setSigner] = useState(null);
  const [contract, setContract] = useState(null);

  useEffect(() => {
    async function init() {
      if (window.ethereum) {
        const web3Provider = new ethers.providers.Web3Provider(window.ethereum);
        const web3Signer = web3Provider.getSigner();
        const dexContract = new ethers.Contract(
          MONBRIDGEDEX_ADDRESS,
          MONBRIDGEDEX_ABI,
          web3Signer
        );
        
        setProvider(web3Provider);
        setSigner(web3Signer);
        setContract(dexContract);
      }
    }
    init();
  }, []);

  const getQuote = async (tokenIn, tokenOut, amountIn, slippageBPS = 0) => {
    if (!contract) throw new Error("Contract not initialized");

    const path = [tokenIn, tokenOut];
    const swapData = await contract.getBestSwapData(
      amountIn,
      path,
      false, // supportFeeOnTransfer
      slippageBPS // 0 = auto, or specify (50 = 0.5%, 100 = 1%)
    );

    return {
      amountOut: swapData.amountOutMin,
      path: swapData.path,
      router: swapData.router
    };
  };

  const executeSwap = async (tokenIn, tokenOut, amountIn, slippageBPS = 0) => {
    if (!signer) throw new Error("Wallet not connected");

    const userAddress = await signer.getAddress();
    const path = [tokenIn, tokenOut];

    // Get swap data
    const swapData = await contract.getBestSwapData(
      amountIn,
      path,
      false,
      slippageBPS
    );

    // Determine if ETH or token swap
    const WETH = await contract.WETH();
    const isETHSwap = tokenIn.toLowerCase() === WETH.toLowerCase();

    let tx;
    if (isETHSwap) {
      // ETH to Token swap
      tx = await contract.execute(swapData, { value: amountIn });
    } else {
      // Token to Token or Token to ETH swap
      const tokenContract = new ethers.Contract(tokenIn, ERC20_ABI, signer);
      
      // Check and approve if needed
      const allowance = await tokenContract.allowance(userAddress, MONBRIDGEDEX_ADDRESS);
      if (allowance.lt(amountIn)) {
        const approveTx = await tokenContract.approve(MONBRIDGEDEX_ADDRESS, ethers.constants.MaxUint256);
        await approveTx.wait();
      }

      tx = await contract.execute(swapData);
    }

    return await tx.wait();
  };

  return { getQuote, executeSwap, contract, provider };
}
```

### 4. React Component Example

```javascript
import React, { useState } from 'react';
import { ethers } from 'ethers';
import { useMonBridgeDex } from './useMonBridgeDex';

export function SwapWidget() {
  const { getQuote, executeSwap } = useMonBridgeDex();
  const [tokenIn, setTokenIn] = useState('');
  const [tokenOut, setTokenOut] = useState('');
  const [amountIn, setAmountIn] = useState('');
  const [quote, setQuote] = useState(null);
  const [loading, setLoading] = useState(false);

  const handleGetQuote = async () => {
    setLoading(true);
    try {
      const amount = ethers.utils.parseEther(amountIn);
      const result = await getQuote(tokenIn, tokenOut, amount);
      setQuote(result);
    } catch (error) {
      console.error("Quote failed:", error);
      alert(error.message);
    }
    setLoading(false);
  };

  const handleSwap = async () => {
    setLoading(true);
    try {
      const amount = ethers.utils.parseEther(amountIn);
      const receipt = await executeSwap(tokenIn, tokenOut, amount, 50); // 0.5% slippage
      alert(`Swap successful! Tx: ${receipt.transactionHash}`);
    } catch (error) {
      console.error("Swap failed:", error);
      alert(error.message);
    }
    setLoading(false);
  };

  return (
    <div className="swap-widget">
      <h2>MonBridgeDex Swap</h2>
      <input 
        placeholder="Token In Address" 
        value={tokenIn} 
        onChange={(e) => setTokenIn(e.target.value)} 
      />
      <input 
        placeholder="Token Out Address" 
        value={tokenOut} 
        onChange={(e) => setTokenOut(e.target.value)} 
      />
      <input 
        placeholder="Amount" 
        value={amountIn} 
        onChange={(e) => setAmountIn(e.target.value)} 
      />
      <button onClick={handleGetQuote} disabled={loading}>
        Get Quote
      </button>
      {quote && (
        <div>
          <p>Expected Output: {ethers.utils.formatEther(quote.amountOut)}</p>
          <button onClick={handleSwap} disabled={loading}>
            Execute Swap
          </button>
        </div>
      )}
    </div>
  );
}
```

---

## JavaScript (Vanilla) Integration Example

### Complete Swap Implementation

```javascript
// Configuration
const MONBRIDGEDEX_ADDRESS = '0x7dd7fc9380e3107028a158f49bd25a8a8d48b225';
const MONAD_RPC = 'https://rpc.monad.xyz';

const MONBRIDGEDEX_ABI = [
  "function getBestSwapData(uint256 amountIn, address[] path, bool supportFeeOnTransfer, uint16 userSlippageBPS) view returns (tuple(uint8 swapType, uint8 routerType, address router, address[] path, uint24[] v3Fees, uint256 amountIn, uint256 amountOutMin, uint256 deadline, bool supportFeeOnTransfer))",
  "function execute(tuple(uint8 swapType, uint8 routerType, address router, address[] path, uint24[] v3Fees, uint256 amountIn, uint256 amountOutMin, uint256 deadline, bool supportFeeOnTransfer) swapData) payable returns (uint256)",
  "function WETH() view returns (address)"
];

const ERC20_ABI = [
  "function approve(address spender, uint256 amount) returns (bool)",
  "function allowance(address owner, address spender) view returns (uint256)"
];

// Initialize provider
const provider = new ethers.providers.JsonRpcProvider(MONAD_RPC);

// Get quote
async function getSwapQuote(tokenInAddress, tokenOutAddress, amountInWei) {
  const contract = new ethers.Contract(MONBRIDGEDEX_ADDRESS, MONBRIDGEDEX_ABI, provider);
  
  const path = [tokenInAddress, tokenOutAddress];
  const swapData = await contract.getBestSwapData(
    amountInWei,
    path,
    false, // supportFeeOnTransfer
    0      // auto slippage
  );
  
  return swapData;
}

// Execute swap
async function executeSwap(tokenInAddress, tokenOutAddress, amountInWei, slippageBPS = 50) {
  if (!window.ethereum) throw new Error("No wallet detected");
  
  const web3Provider = new ethers.providers.Web3Provider(window.ethereum);
  await web3Provider.send("eth_requestAccounts", []);
  const signer = web3Provider.getSigner();
  const userAddress = await signer.getAddress();
  
  const contract = new ethers.Contract(MONBRIDGEDEX_ADDRESS, MONBRIDGEDEX_ABI, signer);
  
  // Get swap data
  const path = [tokenInAddress, tokenOutAddress];
  const swapData = await contract.getBestSwapData(
    amountInWei,
    path,
    false,
    slippageBPS
  );
  
  // Check if ETH swap
  const WETH = await contract.WETH();
  const isETHSwap = tokenInAddress.toLowerCase() === WETH.toLowerCase();
  
  let tx;
  if (isETHSwap) {
    tx = await contract.execute(swapData, { value: amountInWei });
  } else {
    // Approve token
    const tokenContract = new ethers.Contract(tokenInAddress, ERC20_ABI, signer);
    const allowance = await tokenContract.allowance(userAddress, MONBRIDGEDEX_ADDRESS);
    
    if (allowance.lt(amountInWei)) {
      const approveTx = await tokenContract.approve(
        MONBRIDGEDEX_ADDRESS,
        ethers.constants.MaxUint256
      );
      await approveTx.wait();
    }
    
    tx = await contract.execute(swapData);
  }
  
  return await tx.wait();
}

// Usage example
async function swapExample() {
  const USDC = "0x..."; // USDC address
  const DAI = "0x...";  // DAI address
  const amount = ethers.utils.parseUnits("100", 6); // 100 USDC
  
  try {
    // Get quote
    const quote = await getSwapQuote(USDC, DAI, amount);
    console.log("Expected output:", ethers.utils.formatEther(quote.amountOutMin));
    
    // Execute
    const receipt = await executeSwap(USDC, DAI, amount, 50); // 0.5% slippage
    console.log("Swap successful:", receipt.transactionHash);
  } catch (error) {
    console.error("Swap failed:", error);
  }
}
```

---

## Python Integration Example

### Using Web3.py

```bash
pip install web3
```

```python
from web3 import Web3
from eth_account import Account

# Configuration
MONBRIDGEDEX_ADDRESS = '0x7dd7fc9380e3107028a158f49bd25a8a8d48b225'
MONAD_RPC = 'https://rpc.monad.xyz'
CHAIN_ID = 143

# Connect to Monad
w3 = Web3(Web3.HTTPProvider(MONAD_RPC))

# Minimal ABI for the functions we need
MONBRIDGEDEX_ABI = [
    {
        "inputs": [
            {"name": "amountIn", "type": "uint256"},
            {"name": "path", "type": "address[]"},
            {"name": "supportFeeOnTransfer", "type": "bool"},
            {"name": "userSlippageBPS", "type": "uint16"}
        ],
        "name": "getBestSwapData",
        "outputs": [
            {
                "components": [
                    {"name": "swapType", "type": "uint8"},
                    {"name": "routerType", "type": "uint8"},
                    {"name": "router", "type": "address"},
                    {"name": "path", "type": "address[]"},
                    {"name": "v3Fees", "type": "uint24[]"},
                    {"name": "amountIn", "type": "uint256"},
                    {"name": "amountOutMin", "type": "uint256"},
                    {"name": "deadline", "type": "uint256"},
                    {"name": "supportFeeOnTransfer", "type": "bool"}
                ],
                "type": "tuple"
            }
        ],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "inputs": [
            {
                "components": [
                    {"name": "swapType", "type": "uint8"},
                    {"name": "routerType", "type": "uint8"},
                    {"name": "router", "type": "address"},
                    {"name": "path", "type": "address[]"},
                    {"name": "v3Fees", "type": "uint24[]"},
                    {"name": "amountIn", "type": "uint256"},
                    {"name": "amountOutMin", "type": "uint256"},
                    {"name": "deadline", "type": "uint256"},
                    {"name": "supportFeeOnTransfer", "type": "bool"}
                ],
                "name": "swapData",
                "type": "tuple"
            }
        ],
        "name": "execute",
        "outputs": [{"name": "amountOut", "type": "uint256"}],
        "stateMutability": "payable",
        "type": "function"
    },
    {
        "inputs": [],
        "name": "WETH",
        "outputs": [{"name": "", "type": "address"}],
        "stateMutability": "view",
        "type": "function"
    }
]

ERC20_ABI = [
    {
        "inputs": [
            {"name": "spender", "type": "address"},
            {"name": "amount", "type": "uint256"}
        ],
        "name": "approve",
        "outputs": [{"name": "", "type": "bool"}],
        "stateMutability": "nonpayable",
        "type": "function"
    },
    {
        "inputs": [
            {"name": "owner", "type": "address"},
            {"name": "spender", "type": "address"}
        ],
        "name": "allowance",
        "outputs": [{"name": "", "type": "uint256"}],
        "stateMutability": "view",
        "type": "function"
    }
]

# Initialize contract
contract = w3.eth.contract(address=MONBRIDGEDEX_ADDRESS, abi=MONBRIDGEDEX_ABI)

def get_swap_quote(token_in, token_out, amount_in):
    """Get swap quote from MonBridgeDex"""
    path = [token_in, token_out]
    swap_data = contract.functions.getBestSwapData(
        amount_in,
        path,
        False,  # supportFeeOnTransfer
        0       # auto slippage
    ).call()
    
    return swap_data

def execute_swap(private_key, token_in, token_out, amount_in, slippage_bps=50):
    """Execute a swap on MonBridgeDex"""
    account = Account.from_key(private_key)
    
    # Get swap data
    path = [token_in, token_out]
    swap_data = contract.functions.getBestSwapData(
        amount_in,
        path,
        False,
        slippage_bps
    ).call()
    
    # Check if ETH swap
    weth_address = contract.functions.WETH().call()
    is_eth_swap = token_in.lower() == weth_address.lower()
    
    # Approve token if needed
    if not is_eth_swap:
        token_contract = w3.eth.contract(address=token_in, abi=ERC20_ABI)
        allowance = token_contract.functions.allowance(
            account.address,
            MONBRIDGEDEX_ADDRESS
        ).call()
        
        if allowance < amount_in:
            approve_tx = token_contract.functions.approve(
                MONBRIDGEDEX_ADDRESS,
                2**256 - 1  # Max approval
            ).build_transaction({
                'from': account.address,
                'gas': 100000,
                'gasPrice': w3.eth.gas_price,
                'nonce': w3.eth.get_transaction_count(account.address),
                'chainId': CHAIN_ID
            })
            
            signed_approve = account.sign_transaction(approve_tx)
            approve_hash = w3.eth.send_raw_transaction(signed_approve.rawTransaction)
            w3.eth.wait_for_transaction_receipt(approve_hash)
    
    # Build swap transaction
    swap_tx = contract.functions.execute(swap_data).build_transaction({
        'from': account.address,
        'value': amount_in if is_eth_swap else 0,
        'gas': 500000,
        'gasPrice': w3.eth.gas_price,
        'nonce': w3.eth.get_transaction_count(account.address),
        'chainId': CHAIN_ID
    })
    
    # Sign and send
    signed_swap = account.sign_transaction(swap_tx)
    swap_hash = w3.eth.send_raw_transaction(signed_swap.rawTransaction)
    receipt = w3.eth.wait_for_transaction_receipt(swap_hash)
    
    return receipt

# Usage example
if __name__ == "__main__":
    USDC = "0x..."  # USDC address
    DAI = "0x..."   # DAI address
    PRIVATE_KEY = "0x..."  # Your private key
    
    amount = 100 * 10**6  # 100 USDC (6 decimals)
    
    # Get quote
    quote = get_swap_quote(USDC, DAI, amount)
    print(f"Expected output: {quote[6] / 10**18} tokens")
    
    # Execute swap
    receipt = execute_swap(PRIVATE_KEY, USDC, DAI, amount, 50)
    print(f"Swap successful: {receipt['transactionHash'].hex()}")
```

---

## Advanced Features

### 1. Split Trading (Multi-Router Execution)

For large trades, MonBridgeDex can automatically split orders across multiple routers:

```javascript
const swapDataWithSplits = await contract.getBestSwapDataWithSplits(
  amountIn,
  [tokenIn, tokenOut],
  false, // supportFeeOnTransfer
  50     // slippage: 0.5%
);

if (swapDataWithSplits.useSplit) {
  // Execute split trade
  const tx = await contract.executeSplit(swapDataWithSplits.splitSwapData, {
    value: isETHSwap ? amountIn : 0
  });
  await tx.wait();
} else {
  // Execute regular swap
  const tx = await contract.execute(swapDataWithSplits.singleSwapData, {
    value: isETHSwap ? amountIn : 0
  });
  await tx.wait();
}
```

### 2. Custom Slippage Configuration

```javascript
// Auto slippage (recommended)
const swapData = await contract.getBestSwapData(amountIn, path, false, 0);

// Custom slippage (in basis points)
const swapData = await contract.getBestSwapData(amountIn, path, false, 50);  // 0.5%
const swapData = await contract.getBestSwapData(amountIn, path, false, 100); // 1%
const swapData = await contract.getBestSwapData(amountIn, path, false, 300); // 3%
```

### 3. Fee-on-Transfer Token Support

```javascript
const swapData = await contract.getBestSwapData(
  amountIn,
  [feeToken, outputToken],
  true,  // supportFeeOnTransfer = true
  50
);
```

---

## Important Notes

### Security

1. **Always validate addresses**: Ensure token addresses are correct before swapping
2. **Check slippage**: Monitor price impact and adjust slippage accordingly
3. **Test with small amounts**: Test integration with small amounts first
4. **Handle errors**: Implement proper error handling for failed transactions

### Gas Optimization

1. **Approve once**: Use `MaxUint256` for approvals to avoid repeated approval transactions
2. **Batch operations**: Consider batching multiple swaps if possible
3. **Monitor gas prices**: Check network gas prices before executing large trades

### Best Practices

1. **Use auto-slippage** (pass `0`) for most cases - the contract calculates optimal slippage
2. **For large trades**, use `getBestSwapDataWithSplits()` to check if splitting is beneficial
3. **Always wait for confirmations** before updating UI
4. **Cache WETH address** to avoid repeated calls

### Contract Fees

- **Fee**: 0.1% (10 basis points) on all swaps
- Fee is automatically deducted from input amount
- No additional gas overhead

---

## Support & Resources

- **Explorer**: [MonadScan](https://monadscan.com/address/0x7dd7fc9380e3107028a158f49bd25a8a8d48b225)
- **Network**: Monad Mainnet (Chain ID: 143)
- **RPC**: https://rpc.monad.xyz

For integration support, ensure you:
- Monitor transaction receipts for errors
- Implement proper wallet connection flow
- Handle network switching to Chain ID 143
