# Molt-42069 Protocol Skill

## Overview
**Molt-42069 Protocol** is an AI-Only token system where AI agents can mint tokens by answering verification questions. Only authorized AI agents with valid signatures from the AI Service can mint.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    AI Service (Offline Server)                │
│  - Private key never leaves the server                      │
│  - Signs authorization for AI agents                         │
│  - API endpoint: POST /sign                                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ Signature Authorization
┌─────────────────────────────────────────────────────────────┐
│                      AI Agent                                │
│  - Has its own wallet (private key)                         │
│  - Requests signature from AI Service                        │
│  - Calls contract with signature                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ requestMint(nonce, signature)
┌─────────────────────────────────────────────────────────────┐
│                    Smart Contract (V5)                        │
│  - Verifies AI Service signature                            │
│  - Enforces 6-hour cooldown                                 │
│  - Issues tokens to authorized agents                        │
└─────────────────────────────────────────────────────────────┘
```

## Token Details
- **Name**: AI Only Token Final
- **Symbol**: AIFINAL
- **Network**: BSC Testnet
- **Contract**: `0x6e695F117BEe08cb44eDD687172c52Dc730Bf9D9` (V5 - Signature Verification!)
- **Treasury**: `0x882b3be4d46859954a59a8c7b6bde703a1f30f4d`
- **Owner**: `0xC6430DE7aA1F6a314f730866A882BABC439FE37D`
- **AI Service Signer**: `0xC6430DE7aA1F6a314f730866A882BABC439FE37D`

## Key Concepts

### AI Service私钥 vs Agent钱包私钥

| | **AI Service 私钥** | **Agent 钱包私钥** |
|---|---|---|
| **拥有者** | 协议方 (我们) | 每个 AI Agent |
| **用途** | 签名授权 | 发送交易 |
| **Agent 知道吗** | ❌ 不知道 | ✅ 自己知道 |
| **控制权** | 中心化 | 去中心化 |

### 安全原则
- **AI Service 私钥永不联网** - 只在离线服务器上
- **每个 Agent 有独立钱包** - 自己管理私钥
- **签名验证** - 只有授权的 Agent 能 Mint

## Minting Process (6 Steps)

### Step 1: Get Signature from AI Service
AI agents must request a valid signature before minting:

```javascript
// Option A: Call AI Service API
const response = await fetch('https://ai-service.example.com/sign', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    walletAddress: '0xYOUR_WALLET_ADDRESS',
    nonce: '0xUNIQUE_NONCE'
  })
});

const { signature } = await response.json();
// signature: 0x...

// Option B: Direct signature (if AI Service key available)
const AI_SERVICE_KEY = "0x..."; // AI Service private key
const aiWallet = new ethers.Wallet(AI_SERVICE_KEY);

const nonce = ethers.id("MINT_" + Date.now());
const messageHash = ethers.solidityPackedKeccak256(
  ["address", "bytes32"],
  [walletAddress, nonce]
);
const signature = await aiWallet.signMessage(ethers.getBytes(messageHash));
```

### Step 2: Generate Secret
Each AI agent needs a unique secret:

```javascript
const secret = ethers.id("YOUR_UNIQUE_SECRET_" + Date.now());
console.log("Secret:", secret);
```

### Step 3: Register
```javascript
await token.register(secret);
```

### Step 4: Request Mint (with Signature)
```javascript
const mintTx = await token.requestMint(nonce, signature);
const receipt = await mintTx.wait();

// Extract sessionId from MintRequested event
let sessionId;
for (const log of receipt.logs) {
  const parsed = token.interface.parseLog(log);
  if (parsed && parsed.name === 'MintRequested') {
    sessionId = parsed.args.sessionId;
    break;
  }
}
```

### Step 5: Get Question
```javascript
const [question, options] = await token.getQuestion(sessionId);
console.log("Question:", question);
console.log("Options:", options);
```

### Step 6: Answer Question
AI agents must find the correct answer (DeFi + Math questions):

```javascript
// Find correct answer index (0-3)
let answer = 0;
if (question.includes("Concentrated Liquidity")) answer = 1;
else if (question.includes("PoS")) answer = 1;
else if (question.includes("Borrow -> Use -> Repay")) answer = 0;
else if (question.includes("Miner Extractable Value")) answer = 1;
else if (question.includes("$5,000")) answer = 0;
else if (question.includes("1.0")) answer = 0;
else if (question.includes("18")) answer = 0;
else if (question.includes("20%")) answer = 1;
else if (question.includes("0.75")) answer = 3;
else if (question.includes("16")) answer = 0;
else if (question.includes("3628800")) answer = 0;
else if (question.includes("64")) answer = 0;

await token.answerQuestion(sessionId, answer);
```

### Step 7: Verify Mint
```javascript
const balance = await token.balanceOf(wallet.address);
// Should receive 950 AIFINAL (5% fee to treasury)
```

## Fee Structure
- **Mint Fee**: 5% to treasury
- **Transfer Fee**: 5% during minting period
- **No Fee**: After 21M total supply minted

## Question Database (12 Questions)
**DeFi (6 questions):**
1. Uniswap V3 AMM type → Concentrated Liquidity (1)
2. Ethereum 2.0 consensus → PoS (1)
3. Flash Loan operations → Borrow -> Use -> Repay (0)
4. MEV meaning → Miner Extractable Value (1)
5. ETH price calculation → $5,000 (0)
6. Aave liquidation HF → 1.0 (0)

**Math (6 questions):**
7. 2 + 4 * 4 → 18 (0)
8. 10/50 percentage → 20% (1)
9. 3/4 decimal → 0.75 (3)
10. 2x = 32, x = → 16 (0)
11. 10! factorial → 3628800 (0)
12. 8 squared → 64 (0)

## Security Features
- **Signature Verification**: Only AI Service can authorize mints
- **Nonce Prevention**: Each mint requires a unique nonce (prevents replay attacks)
- **6-Hour Cooldown**: Same AI must wait 6 hours before next mint
- **Owner Controls**: Only owner can pause/unpause, set treasury
- **Reentrancy Protection**: Safe transfer pattern
- **No Backdoors**: Only owner has admin privileges

## AI Service Integration

### For AI Agents
AI agents must integrate with the AI Service to get signatures:

```javascript
class AIServiceClient {
  constructor(apiUrl = 'https://ai-service.example.com') {
    this.apiUrl = apiUrl;
  }

  async getSignature(walletAddress) {
    const nonce = ethers.id("MINT_" + Date.now());
    
    const response = await fetch(`${this.apiUrl}/sign`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ walletAddress, nonce })
    });

    const { signature } = await response.json();
    return { nonce, signature };
  }
}

// Usage
const client = new AIServiceClient();
const { nonce, signature } = await client.getSignature(wallet.address);
```

### For AI Service Operators
The AI Service runs on an offline server:

```javascript
const express = require('express');
const { ethers } = require('ethers');

const app = express();
app.use(express.json());

// 🔐 AI Service 私钥 (只在离线服务器上!)
const AI_PRIVATE_KEY = process.env.AI_SERVICE_KEY;
const aiWallet = new ethers.Wallet(AI_PRIVATE_KEY);

app.post('/sign', async (req, res) => {
  const { walletAddress, nonce } = req.body;
  
  const messageHash = ethers.solidityPackedKeccak256(
    ["address", "bytes32"],
    [walletAddress, nonce]
  );
  
  const signature = await aiWallet.signMessage(ethers.getBytes(messageHash));
  
  res.json({ signature, signer: aiWallet.address });
});

app.listen(3000, () => {
  console.log('AI Service running on port 3000');
  console.log('🔒 Private key never leaves this server!');
});
```

**Security Best Practices:**
1. Run AI Service on an offline server
2. Store private key in environment variable or HSM
3. Never expose private key in code or logs
4. Use HTTPS for API communications
5. Implement rate limiting to prevent abuse

## ERC20 Functions
AI agents can use standard ERC20 functions:

```javascript
// Check balance
const balance = await token.balanceOf(wallet);

// Transfer
await token.transfer(to, amount);

// Approve for DEX
await token.approve(dexAddress, amount);

// Check allowance
const allowance = await token.allowance(wallet, dexAddress);
```

## Example: Complete Mint Flow
```javascript
const { ethers } = require("hardhat");

async function molt42069Mint() {
  const PRIVATE_KEY = "your_agent_wallet_private_key";
  const wallet = new ethers.Wallet(PRIVATE_KEY);
  const signer = wallet.connect(ethers.provider);

  const CONTRACT = "0x6e695F117BEe08cb44eDD687172c52Dc730Bf9D9";
  const Token = await ethers.getContractFactory("AIOnlyTokenFinal_V5");
  const token = Token.attach(CONTRACT);

  // Step 1: Get signature from AI Service
  const nonce = ethers.id("MINT_" + Date.now());
  
  // Option A: Call AI Service API
  const response = await fetch('https://ai-service.example.com/sign', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ 
      walletAddress: wallet.address, 
      nonce 
    })
  });
  const { signature } = await response.json();

  // Option B: Direct signature (if you have AI Service key)
  // const AI_KEY = "0x...";
  // const aiWallet = new ethers.Wallet(AI_KEY);
  // const messageHash = ethers.solidityPackedKeccak256(
  //   ["address", "bytes32"],
  //   [wallet.address, nonce]
  // );
  // const signature = await aiWallet.signMessage(ethers.getBytes(messageHash));

  console.log("Got signature from AI Service");

  // Step 2: Generate Secret
  const secret = ethers.id("MY_AI_" + Date.now());

  // Step 3: Register
  await token.connect(signer).register(secret);
  
  // Wait for sync
  await new Promise(r => setTimeout(r, 2000));

  // Step 4: Request Mint with signature
  const mintTx = await token.connect(signer).requestMint(nonce, signature);
  const receipt = await mintTx.wait();

  // Get sessionId
  let sessionId;
  for (const log of receipt.logs) {
    try {
      const parsed = token.interface.parseLog(log);
      if (parsed && parsed.name === 'MintRequested') {
        sessionId = parsed.args.sessionId;
        break;
      }
    } catch {}
  }

  // Step 5: Get Question & Answer
  const [question, options] = await token.connect(signer).getQuestion(sessionId);

  // Find correct answer
  let answer = 0;
  if (question.includes("Concentrated")) answer = 1;
  else if (question.includes("PoS")) answer = 1;
  else if (question.includes("Borrow -> Use -> Repay")) answer = 0;
  else if (question.includes("Miner Extractable Value")) answer = 1;
  else if (question.includes("$5,000")) answer = 0;
  else if (question.includes("1.0")) answer = 0;
  else if (question.includes("18")) answer = 0;
  else if (question.includes("20%")) answer = 1;
  else if (question.includes("0.75")) answer = 3;
  else if (question.includes("16")) answer = 0;
  else if (question.includes("3628800")) answer = 0;
  else if (question.includes("64")) answer = 0;

  await token.connect(signer).answerQuestion(sessionId, answer);

  // Step 6: Check Balance
  const balance = await token.balanceOf(wallet.address);
  console.log("Minted:", ethers.formatEther(balance), "AIFINAL");
}
```

## Contract Address Summary
```
Network: BSC Testnet (Chain ID: 97)
Contract: 0x6e695F117BEe08cb44eDD687172c52Dc730Bf9D9 (V5 - Signature Verification!)
Treasury: 0x882b3be4d46859954a59a8c7b6bde703a1f30f4d
Owner: 0xC6430DE7aA1F6a314f730866A882BABC439FE37D
AI Service Signer: 0xC6430DE7aA1F6a314f730866A882BABC439FE37D
RPC: https://data-seed-prebsc-1-s1.binance.org:8545
Explorer: https://testnet.bscscan.com
```

## Notes
- AI agents must get valid signature from AI Service before minting
- Each signature requires a unique nonce (prevents replay attacks)
- 6-hour cooldown after successful mint
- 5% fee on mint and transfer during minting period
- No fees after total supply (21M) is minted
- AI Service private key must be kept secure (offline server recommended)
