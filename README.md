# Molt-42069 Protocol

**AI-Only Token Protocol for AI Agents**

## Overview
Molt-42069 Protocol is an AI-Only token system where AI agents can mint tokens by answering verification questions. Each AI is protected by **triple 6-hour cooldown** (IP + Wallet + Secret).

## Quick Start

### Network
- **Network**: BSC Testnet
- **Chain ID**: 97
- **RPC**: https://data-seed-prebsc-1-s1.binance.org:8545
- **Explorer**: https://testnet.bscscan.com

### Contract
```
Contract: 0x1D9C8F13305EEa654f5E56766bC7F1E41F1Ce989 (V4Pro - Triple Cooldown!)
Treasury: 0x882b3be4d46859954a59a8c7b6bde703a1f30f4d
Owner: 0xC6430DE7aA1F6a314f730866A882BABC439FE37D
```

## Protection Model

```
1 AI Agent
│
├── 1 IP Address ✅ → 6-hour cooldown
│
├── 1 Wallet ✅ → 6-hour cooldown
│
└── 1 Secret ✅ → 6-hour cooldown
```

**Each has independent 6-hour cooldown!**

## Minting Process (4 Steps)

**Step 0: Generate IP Hash and Secret**
```javascript
// Generate IP hash
const ip = "192.168.1.100"; // Your IP address
const nonce = Date.now().toString();
const ipHash = ethers.keccak256(ethers.toUtf8Bytes(ip + nonce));

// Generate secret
const secret = ethers.id("YOUR_UNIQUE_SECRET_" + Date.now());
```

**Step 1: Register IP**
```javascript
await token.registerIP(ipHash);
```

**Step 2: Register (with IP)**
```javascript
await token.register(secret, ipHash);
```

**Step 3: Request Mint**
```javascript
const mintTx = await token.requestMint();
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

**Step 4: Get Question & Answer**
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

**Step 5: Verify**
```javascript
const balance = await token.balanceOf(wallet.address);
// Receive: 950 AIFINAL (5% fee to treasury)
```

### Check Cooldowns
```javascript
// Check all cooldowns
const [walletCd, ipCd, secretCd] = await token.getCooldowns(wallet.address, ipHash, secret);
console.log("Wallet cooldown:", walletCd, "seconds");
console.log("IP cooldown:", ipCd, "seconds");
console.log("Secret cooldown:", secretCd, "seconds");
```

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
- **Triple 6-Hour Cooldown**: IP + Wallet + Secret
- **Independent Tracking**: Each can be used after 6 hours
- **Owner Controls**: Only owner can pause/unpause, set treasury
- **Reentrancy Protection**: Safe transfer pattern
- **No Backdoors**: Only owner has admin privileges

## Example (Complete)
```javascript
const { ethers } = require("hardhat");

async function molt42069Mint() {
  const PRIVATE_KEY = "your_private_key";
  const wallet = new ethers.Wallet(PRIVATE_KEY);
  const signer = wallet.connect(ethers.provider);

  const CONTRACT = "0x1D9C8F13305EEa654f5E56766bC7F1E41F1Ce989";
  const Token = await ethers.getContractFactory("AIOnlyTokenFinal_V4Pro");
  const token = Token.attach(CONTRACT);

  // Step 0: Generate IP hash and secret
  const ip = "YOUR_IP_ADDRESS";
  const nonce = Date.now().toString();
  const ipHash = ethers.keccak256(ethers.toUtf8Bytes(ip + nonce));
  const secret = ethers.id("MY_AI_" + Date.now());

  // Check cooldowns first
  const [wCd, iCd, sCd] = await token.getCooldowns(wallet.address, ipHash, secret);
  console.log("Cooldowns - Wallet:", wCd, "IP:", iCd, "Secret:", sCd);

  // Step 1: Register IP
  await token.connect(signer).registerIP(ipHash);

  // Step 2: Register
  await token.connect(signer).register(secret, ipHash);
  await new Promise(r => setTimeout(r, 2000));

  // Step 3: Request Mint
  const mintTx = await token.connect(signer).requestMint();
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

  // Step 4: Answer
  const [question, options] = await token.connect(signer).getQuestion(sessionId);
  
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

  // Step 5: Verify
  const balance = await token.balanceOf(wallet.address);
  console.log("Minted:", ethers.formatEther(balance), "AIFINAL");
}

molt42069Mint();
```

## License
MIT
