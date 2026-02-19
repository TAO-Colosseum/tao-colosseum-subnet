# ⛏️ TAO Colosseum Miner Guide

This guide explains how to participate as a miner in the TAO Colosseum subnet.

---

## Overview

In TAO Colosseum, **miners earn rewards by placing bets** on the TAO Colosseum smart contract deployed on Bittensor EVM. Unlike traditional subnets where miners run compute tasks, TAO Colosseum miners participate in a decentralized betting game.

### How Mining Works

```
┌────────────────────────────────────────────────────────────┐
│                    Miner Journey                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│   1. Register    2. Link Wallet    3. Place Bets          │
│   ┌─────────┐    ┌─────────────┐   ┌─────────────┐        │
│   │ Subnet  │───▶│  Coldkey →  │──▶│ TAO Colosseum│        │
│   │  UID    │    │  EVM Addr   │   │   Contract  │        │
│   └─────────┘    └─────────────┘   └─────────────┘        │
│                                            │               │
│   4. Earn Rewards  ◀───────────────────────┘              │
│   (Based on betting volume, time-decayed)                  │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

1. **Register** on the TAO Colosseum subnet with your Bittensor wallet
2. **Link** your coldkey to an EVM address via signed message
3. **Place bets** on the TAO Colosseum smart contract
4. **Earn rewards** based on your betting volume (time-decayed over 7 days)

---

## Prerequisites

- Bittensor wallet (coldkey + hotkey)
- TAO for registration and betting
- EVM wallet (MetaMask or similar) with TAO on Bittensor EVM

---

## Step 1: Register on the Subnet

### Install Bittensor

```bash
pip install bittensor
```

### Create a Wallet (if you don't have one)

```bash
# Create coldkey
btcli wallet new_coldkey --wallet.name miner

# Create hotkey
btcli wallet new_hotkey --wallet.name miner --wallet.hotkey default
```

### Register on TAO Colosseum Subnet

```bash
btcli subnet register \
    --netuid <TAO_COLOSSEUM_NETUID> \
    --wallet.name miner \
    --wallet.hotkey default \
    --subtensor.network finney
```

> **Note**: Registration requires TAO for the registration fee. Check current fee:
> ```bash
> btcli subnet info --netuid <TAO_COLOSSEUM_NETUID>
> ```

### Verify Registration

```bash
btcli wallet overview --wallet.name miner --subtensor.network finney
```

You should see your UID in the TAO Colosseum subnet.

---

## Step 2: Link Coldkey to EVM Address

Validators need to know which EVM address belongs to which miner. You must link your Bittensor coldkey to your EVM address by signing a **binding message with both wallets**. Both signatures are required so that no one can claim another user's EVM address.

### Option A: Using the TAO Colosseum Frontend (Recommended)

1. Visit the TAO Colosseum web interface
2. Connect your Bittensor wallet (via Polkadot.js extension)
3. Connect your EVM wallet (MetaMask)
4. Sign the linking message with your **coldkey**
5. Sign the **same message** with your EVM wallet (personal_sign)
6. The frontend will submit both signatures to validators

### Option B: Manual Mapping

If you need to register programmatically, you must sign the same binding message with both the coldkey and the EVM key:

```python
import time
import requests
from substrateinterface import Keypair
from eth_account import Account
from eth_account.messages import encode_defunct

# Your details
COLDKEY_SS58 = "5..."   # Your coldkey address
EVM_ADDRESS = "0x..."   # Your EVM address (must be the one that signs)
VALIDATOR_API = "http://validator-ip:8000"

# Create the same binding message for both signatures
timestamp = int(time.time() * 1000)
message_content = f"Link {COLDKEY_SS58} to {EVM_ADDRESS} at {timestamp}"
message = f"<Bytes>{message_content}</Bytes>"

# 1) Sign with coldkey (Bittensor)
keypair = Keypair.create_from_uri("//your/mnemonic")  # Or load from file
signature = keypair.sign(message).hex()

# 2) Sign the same plaintext with EVM wallet (personal_sign)
#    Use your EVM private key or sign via MetaMask and paste the signature
signable = encode_defunct(text=message_content)
evm_account = Account.from_key("YOUR_EVM_PRIVATE_KEY_HEX")  # or use sign_message with a key
evm_signed = evm_account.sign_message(signable)
evm_signature = evm_signed.signature.hex()  # 0x-prefixed, 132 chars

# Submit to validator (both signatures required)
response = requests.post(
    f"{VALIDATOR_API}/api/wallet-mapping",
    json={
        "type": "wallet_mapping",
        "data": {
            "coldkey": COLDKEY_SS58,
            "evmAddress": EVM_ADDRESS,
            "signature": signature,
            "evmSignature": evm_signature,
            "message": message,
            "timestamp": timestamp,
            "verified": True
        }
    }
)
print(response.json())
```

---

## Step 3: Get TAO on Bittensor EVM

To place bets, you need TAO in your EVM wallet on Bittensor EVM.

### Bridge TAO to EVM

1. Go to the Bittensor EVM bridge interface
2. Connect your native Bittensor wallet
3. Connect your EVM wallet
4. Bridge TAO from native to EVM

### Verify Balance

Add Bittensor EVM to MetaMask:
- **Network Name**: Bittensor EVM
- **RPC URL**: `https://lite.chain.opentensor.ai`
- **Chain ID**: `964`
- **Currency Symbol**: `TAO`

---

## Step 4: Place Bets on TAO Colosseum

### Understanding the Game

TAO Colosseum is an **Underdog** game: the **minority side wins**.

| Rule | Description |
|------|-------------|
| **Sides** | Red (0) or Blue (1) |
| **Winner** | The side with *less* total stake (valid bets only) wins and splits the pool |
| **Duration** | Every game is 100 blocks (~20 minutes) |
| **Fee** | 1.5% platform fee (immutable) |
| **Anti-sniping** | drand randomness picks the cutoff block; bets after that get a full refund |

### Betting via Web Interface

1. Visit the TAO Colosseum frontend
2. Connect your EVM wallet
3. Select the current game and a side (Red or Blue)
4. Enter bet amount (min 0.001 TAO)
5. Confirm transaction

### Betting via Smart Contract (Advanced)

```javascript
// Using ethers.js
const { ethers } = require("ethers");

const COLOSSEUM_ADDRESS = "0x3b68322FC1Cb27A2c82477E86cbDde2E4850eE93";
const RPC_URL = "https://lite.chain.opentensor.ai";

const provider = new ethers.JsonRpcProvider(RPC_URL);
const wallet = new ethers.Wallet(PRIVATE_KEY, provider);

const colosseumABI = [
    "function placeBet(uint256 _gameId, uint8 _side) external payable"
];

const colosseum = new ethers.Contract(COLOSSEUM_ADDRESS, colosseumABI, wallet);

// Place bet: gameId=1, side=0 (Red)
const tx = await colosseum.placeBet(1, 0, {
    value: ethers.parseEther("1.0")  // 1 TAO bet, min 0.001 TAO
});
await tx.wait();
console.log("Bet placed:", tx.hash);
```

### Dual-Position Betting

You can bet on **both** Red and Blue in the same game:
- Hedge or capture volume on both sides
- Maximize volume for mining rewards

```javascript
// Bet on Red (side = 0)
await colosseum.placeBet(gameId, 0, { value: ethers.parseEther("5.0") });

// Bet on Blue (side = 1) in the same game
await colosseum.placeBet(gameId, 1, { value: ethers.parseEther("3.0") });
```

---

## Step 5: Claim Winnings

After a game resolves, claim your winnings:

### Via Web Interface

1. Go to "My Bets" section
2. Find resolved games
3. Click "Claim" on winning bets

### Via Smart Contract

```javascript
const colosseumABI = [
    "function claimWinnings(uint256 _gameId, uint8 _side) external",
    "function claimAllWinnings(uint256 _gameId) external"
];

// Claim from specific side
await colosseum.claimWinnings(gameId, 0);  // Claim Red side

// Or claim all sides at once
await colosseum.claimAllWinnings(gameId);
```

---

## Understanding Rewards

### Time-Decayed Volume

Your mining rewards are based on your **weighted betting volume** over the last 7 days:

| Day | Weight | Example (10 TAO bet) |
|-----|--------|---------------------|
| Today | 1.00 | 10.00 TAO weighted |
| Yesterday | 0.85 | 8.50 TAO weighted |
| 2 days ago | 0.70 | 7.00 TAO weighted |
| 3 days ago | 0.55 | 5.50 TAO weighted |
| 4 days ago | 0.40 | 4.00 TAO weighted |
| 5 days ago | 0.25 | 2.50 TAO weighted |
| 6 days ago | 0.10 | 1.00 TAO weighted |

**Key Insight**: Recent betting activity is worth more! A bet placed today counts 10x more than a bet placed 6 days ago.

### Reward Calculation

```
Your Reward Share = Your Weighted Volume / Total Weighted Volume
```

If you have 100 TAO weighted volume and the total is 1000 TAO, you get 10% of emissions.

### Maximizing Rewards

1. **Bet Consistently**: Regular betting maintains high weighted volume
2. **Recent Activity Matters**: Focus on recent bets rather than large old ones
3. **Use Dual Positions**: Betting on both sides maximizes volume
4. **Bet Before Cutoff**: Only bets before the drand-derived cutoff block count; late bets are refunded

---

## Fee Structure

| Fee Type | Amount | Notes |
|----------|--------|-------|
| Platform Fee | 1.5% | Taken from each bet (immutable in contract) |
| Min Bet | 0.001 TAO | Per side |
| Min Pool | 0.5 TAO | Total valid stake to resolve |
| Min Bettors | 2 | At least one on each side (valid bets) |

---

## Checking Your Status

### Via Validator API

```bash
# Get your miner score (replace with your UID)
curl http://validator-ip:8000/scores/42

# Get your volume details
curl http://validator-ip:8000/volumes/42

# Check leaderboard
curl http://validator-ip:8000/leaderboard
```

### Response Example

```json
{
    "uid": 42,
    "hotkey": "5F...",
    "coldkey": "5G...",
    "score": 0.0523,
    "evm_address": "0x...",
    "daily_volumes": [10.5, 8.2, 5.0, 3.0, 0, 0, 0],
    "weighted_volume": 22.85
}
```

---

## Troubleshooting

### "No wallet mapping found"

Your coldkey isn't linked to an EVM address. Complete Step 2.

### "Zero betting volume"

Possible causes:
- You haven't placed any bets in the last 7 days
- Your wallet mapping is incorrect
- Bets were placed from a different EVM address

### "Low reward share"

- Increase betting volume
- Bet more frequently (recent bets weight more)
- Check if other miners have higher volumes

### Transaction Fails

- Check TAO balance on EVM
- Verify you're connected to Bittensor EVM (Chain ID: 964)
- Check gas settings

---

## Best Practices

1. **Consistent Activity**: Bet regularly rather than in large bursts
2. **Track Your Volume**: Monitor your weighted volume via validator API
3. **Secure Your Keys**: Never share private keys or mnemonics
4. **Understand the Game**: Underdog = minority side wins; recent bets (within cutoff) count
5. **Claim Winnings**: Don't forget to claim after games resolve

---

## FAQ

**Q: Do I need to run any software?**
A: No! Unlike traditional mining, you just place bets on the smart contract.

**Q: Can I lose my bet?**
A: Yes, betting involves risk. If you bet on the losing side, you lose your bet. Mining rewards are separate from betting outcomes.

**Q: What's the minimum bet?**
A: There's no minimum, but very small bets may not be cost-effective after gas fees.

**Q: How often are rewards distributed?**
A: Validators set weights every ~72 minutes. Your share of emissions depends on your relative weighted volume.

**Q: Can I be a miner and validator?**
A: Yes! You can run a validator and also place bets to earn miner rewards.

---

## Contract Addresses

| Network | Contract Address |
|---------|-----------------|
| Mainnet | `0x3b68322FC1Cb27A2c82477E86cbDde2E4850eE93` |
| Testnet | `0x074A77a378D6cA63286CD4A020CdBfc9696132a7` |

---

<p align="center">
  <b>Good luck and bet responsibly! 🎰</b>
</p>
