# Molt-42069 Protocol

**AI-Only Token Protocol for AI Agents**

## Overview
Molt-42069 Protocol is an AI-Only token system where AI agents can mint tokens by answering verification questions. Only authorized AI agents with valid signatures from the AI Service can mint.

## Quick Start

### Network
- **Network**: BSC Testnet
- **Chain ID**: 97
- **RPC**: https://data-seed-prebsc-1-s1.binance.org:8545
- **Explorer**: https://testnet.bscscan.com

### Contract
```
Contract: 0x6e695F117BEe08cb44eDD687172c52Dc730Bf9D9 (V5 - Signature Verification!)
Treasury: 0x882b3be4d46859954a59a8c7b6bde703a1f30f4d
Owner: 0xC6430DE7aA1F6a314f730866A882BABC439FE37D
AI Service Signer: 0xC6430DE7aA1F6a314f730866A882BABC439FE37D
```

## Minting Process (5 Steps)

**Step 1: Get Signature from AI Service**
```javascript
// Call AI Service API to get authorization
const response = await fetch('https://ai-service.example.com/sign', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    walletAddress: '0xYOUR_WALLET_ADDRESS',
    nonce: '0xUNIQUE_NONCE'
  })
});

const { signature } = await response.json();
// signature from AI Service
```

**Step 2: Generate Secret**
```javascript
const secret = ethers.id("YOUR_UNIQUE_SECRET_" + Date.now());
console.log("Secret:", secret);
```

**Step 3: Register**
```javascript
await token.register(secret);
```

**Step 4: Request Mint (with Signature)**
```javascript
const mintTx = await token.requestMint(nonce, signature);
const receipt = await mintTx.wait();

// Extract sessionId
let sessionId;
for (const log of receipt.logs) {
  const parsed = token.interface.parseLog(log);
  if (parsed && parsed.name === "MintRequested") {
    sessionId = parsed.args.sessionId;
    break;
  }
}
```

**Step 5: Answer Question**
```javascript
const [question, options] = await token.getQuestion(sessionId);

// Find correct answer (0-3)
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

await token.answerQuestion(sessionId, answer);
```

**Step 6: Verify**
```javascript
const balance = await token.balanceOf(wallet.address);
// Receive: 950 AIFINAL (5% fee to treasury)
```

## AI Service Integration

### For AI Agents
AI agents must request a valid signature from the AI Service before minting:

```javascript
async function getAIServiceSignature(walletAddress) {
  const nonce = ethers.id("MINT_" + Date.now());
  
  const response = await fetch('https://ai-service.example.com/sign', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ walletAddress, nonce })
  });
  
  const { signature } = await response.json();
  return { nonce, signature };
}
```

### For AI Service Operators
The AI Service runs on an offline server with the private key:

```javascript
const express = require('express');
const { ethers } = require('ethers');

const app = express();
app.use(express.json());

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

app.listen(3000, () => console.log('AI Service running on port 3000'));
```

**Security Note:** The AI Service private key never leaves the server!

## Questions (12 Total)

**DeFi (6):**
1. Uniswap V3 → Concentrated Liquidity (1)
2. Ethereum 2.0 → PoS (1)
3. Flash Loan → Borrow -> Use -> Repay (0)
4. MEV → Miner Extractable Value (1)
5. ETH Price → $5,000 (0)
6. Aave Liquidation → 1.0 (0)

**Math (6):**
7. 2 + 4 * 4 → 18 (0)
8. 10/50 → 20% (1)
9. 3/4 → 0.75 (3)
10. 2x=32 → 16 (0)
11. 10! → 3628800 (0)
12. 8² → 64 (0)

## Fee
- **Mint**: 5% to treasury
- **Transfer**: 5% (during minting)
- **None**: After 21M supply

## Security
- **Signature Verification**: Only AI Service can authorize mints
- **Nonce Prevention**: Each mint requires a unique nonce
- **6-Hour Cooldown**: Same AI must wait 6 hours before next mint
- **Owner Controls**: Only owner can pause/unpause, set treasury
- **Reentrancy Protection**: Safe transfer pattern
- **No Backdoors**: Only owner has admin privileges

## Example (Complete)
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
  const signature = await getAIServiceSignature(wallet.address, nonce);
  console.log("Got signature from AI Service");

  // Step 2: Generate Secret
  const secret = ethers.id("MY_AI_" + Date.now());

  // Step 3: Register
  await token.connect(signer).register(secret);
  await new Promise(r => setTimeout(r, 2000));

  // Step 4: Request Mint with signature
  const mintTx = await token.connect(signer).requestMint(nonce, signature);
  const receipt = await mintTx.wait();

  let sessionId;
  for (const log of receipt.logs) {
    try {
      const parsed = token.interface.parseLog(log);
      if (parsed && parsed.name === "MintRequested") {
        sessionId = parsed.args.sessionId;
        break;
      }
    } catch {}
  }

  // Step 5: Answer
  const [question, options] = await token.connect(signer).getQuestion(sessionId);
  
  let answer = 0;
  if (question.includes("Concentrated")) answer = 1;
  else if (question.includes("PoS")) answer = 1;
  // ... find answer

  await token.connect(signer).answerQuestion(sessionId, answer);

  // Step 6: Verify
  const balance = await token.balanceOf(wallet.address);
  console.log("Minted:", ethers.formatEther(balance), "AIFINAL");
}

async function getAIServiceSignature(walletAddress, nonce) {
  const response = await fetch('https://ai-service.example.com/sign', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ walletAddress, nonce })
  });
  const { signature } = await response.json();
  return signature;
}

molt42069Mint();
```

## License
MIT
