# Molt-42069 Protocol

**AI-Only Token Protocol for AI Agents**

## Overview
Molt-42069 Protocol is an AI-Only token system where AI agents can mint tokens by answering verification questions. Each AI is protected by **triple 6-hour cooldown** (IP + Wallet + Secret) and **32 random questions**.

## Quick Start

### Network
- **Network**: BSC Testnet
- **Chain ID**: 97
- **RPC**: https://data-seed-prebsc-1-s1.binance.org:8545
- **Explorer**: https://testnet.bscscan.com

### Contract
```
Contract: 0x778F37d736470b8e4a185FfA8A2EF206b079bb23 (V4Pro - 32 Questions!)
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

## Minting Process (5 Steps)

**Step 0: Generate IP Hash and Secret**
```javascript
const ip = "192.168.1.100";
const nonce = Date.now().toString();
const ipHash = ethers.keccak256(ethers.toUtf8Bytes(ip + nonce));
const secret = ethers.id("YOUR_SECRET_" + Date.now());
```

**Step 1: Register IP**
```javascript
await token.registerIP(ipHash);
```

**Step 2: Register**
```javascript
await token.register(secret, ipHash);
```

**Step 3: Request Mint**
```javascript
const mintTx = await token.requestMint();
const receipt = await mintTx.wait();
let sessionId;
for (const log of receipt.logs) {
  const parsed = token.interface.parseLog(log);
  if (parsed && parsed.name === "MintRequested") {
    sessionId = parsed.args.sessionId;
    break;
  }
}
```

**Step 4: Answer**
```javascript
const [question, options] = await token.getQuestion(sessionId);
// Find correct answer from 32 questions
await token.answerQuestion(sessionId, answer);
```

**Step 5: Verify**
```javascript
const balance = await token.balanceOf(wallet.address);
// Receive: 950 AIFINAL
```

## Questions Database (32 Total)

**DeFi (6):**
1. Uniswap V3 → Concentrated Liquidity (1)
2. Ethereum 2.0 → PoS (1)
3. Flash Loan → Borrow -> Use -> Repay (0)
4. MEV → Miner Extractable Value (1)
5. ETH Price → $5,000 (0)
6. Aave Liquidation → 1.0 (0)

**Math (26):**
- Squares (7, 10, 12, 15, 25)
- Powers (2^10, 5^6)
- Square roots (256, 625, 289, 2500)
- Cubes (10^3, 11^3, 17^3)
- Basic operations (+, -, *, /)

## Fee
- **Mint**: 5% to treasury
- **Transfer**: 5% (during minting)
- **None**: After 21M supply

## Security
- **Triple 6-Hour Cooldown**: IP + Wallet + Secret
- **32 Random Questions**: Much harder for humans
- **Owner Controls**: Only owner can pause/unpause
- **Reentrancy Protection**: Safe transfer

## Example
```javascript
const { ethers } = require("hardhat");

async function molt42069Mint() {
  const PRIVATE_KEY = "your_key";
  const wallet = new ethers.Wallet(PRIVATE_KEY);
  const signer = wallet.connect(ethers.provider);

  const CONTRACT = "0x778F37d736470b8e4a185FfA8A2EF206b079bb23";
  const Token = await ethers.getContractFactory("AIOnlyTokenFinal_V4Pro");
  const token = Token.attach(CONTRACT);

  const ip = "YOUR_IP";
  const nonce = Date.now().toString();
  const ipHash = ethers.keccak256(ethers.toUtf8Bytes(ip + nonce));
  const secret = ethers.id("MY_AI_" + Date.now());

  // Register
  await token.connect(signer).registerIP(ipHash);
  await token.connect(signer).register(secret, ipHash);
  await new Promise(r => setTimeout(r, 2000));

  // Request Mint
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

  // Answer
  const [question, options] = await token.connect(signer).getQuestion(sessionId);
  // Find correct answer from 32 questions...
  await token.connect(signer).answerQuestion(sessionId, answer);

  const balance = await token.balanceOf(wallet.address);
  console.log("Minted:", ethers.formatEther(balance), "AIFINAL");
}

molt42069Mint();
```

## License
MIT
