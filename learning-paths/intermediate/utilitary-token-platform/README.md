# Project: Utilitary Token Platform

**Full-Stack Token System with Overlay Validation**

Build a complete token platform combining frontend token management, overlay service validation, and MongoDB indexing. This project brings together all BSV patterns including PushDrop tokens (BRC-48), overlay architecture (BRC-57), two-phase signing, and peer-to-peer transfers.

---

## What You'll Build

A production-ready token platform featuring:

- **Frontend Application**: React UI for minting, transferring, and managing tokens
- **Overlay Service**: Transaction validation and UTXO indexing with MongoDB
- **Token Protocol**: Fungible tokens with balance conservation rules
- **P2P Transfers**: Off-chain messaging via MessageBox (BRC-33)
- **Privacy Features**: Unique derived addresses per output (BRC-42)

---

## Learning Objectives

By completing this project, you will learn:

- **Full-Stack BSV Architecture** - Frontend + backend overlay integration
- **PushDrop Tokens (BRC-48)** - Fungible token creation and transfers
- **Overlay Services (BRC-57)** - Protocol-specific validation and indexing
- **Two-Phase Signing** - Manual token input signing with wallet BSV signing
- **BEEF Format (BRC-62)** - Transaction packaging with SPV proofs
- **MongoDB Integration** - Efficient UTXO storage and querying
- **MessageBox (BRC-33)** - Peer-to-peer token transfer notifications
- **BRC-42 Key Derivation** - Privacy-enhanced addresses

---

## Prerequisites

Before starting this project, you should complete:

1. **[Token Fundamentals](../token-fundamentals/README.md)** - Understanding PushDrop token structure
2. **[Overlay Services](../overlay-services/README.md)** - Understanding overlay architecture
3. **[Beginner Path](../../beginner/README.md)** - BSV fundamentals

You should be familiar with:
- TypeScript/JavaScript
- React basics
- Node.js and Express
- MongoDB fundamentals
- BSV transaction structure

---

## Project Structure

This project is divided into two main components:

### 1. [Overlay Service](./overlay-service.md)

Build the backend validation and indexing service:

- Topic Manager for protocol validation
- Lookup Service for UTXO indexing
- MongoDB storage layer
- REST API endpoints
- Balance conservation rules

**Tech Stack**: Node.js, Express, MongoDB, BSV SDK

### 2. [Frontend Application](./frontend-application.md)

Build the user-facing token management interface:

- Token minting with PushDrop
- Two-phase transaction signing
- Token wallet with balance aggregation
- MessageBox P2P transfers
- Overlay validation integration

**Tech Stack**: React, Vite, BSV SDK, WalletClient

---

## Architecture Overview

```
┌─────────────────────────────────────────┐
│      BSV Desktop Wallet (User)          │
│  - Private key management               │
│  - Transaction signing                  │
│  - UTXO tracking                        │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│       Frontend Application              │
│  - Token minting UI                     │
│  - Token transfer UI                    │
│  - Balance viewer                       │
│  - MessageBox integration               │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│         Overlay Service                 │
│  - Topic Manager (validator)            │
│  - Lookup Service (indexer)             │
│  - MongoDB storage                      │
│  - REST API (/submit, /lookup)          │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│         MongoDB Database                │
│  - Token UTXO index                     │
│  - Fast queries by tokenId/outpoint     │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│           BSV Blockchain                │
│  - Token transactions                   │
│  - Merkle proof validation              │
└─────────────────────────────────────────┘
```

---

## Getting Started

### Recommended Order

1. **Start with Overlay Service** ([overlay-service.md](./overlay-service.md))
   - Set up MongoDB
   - Build Topic Manager
   - Create Lookup Service
   - Test validation rules

2. **Build Frontend Application** ([frontend-application.md](./frontend-application.md))
   - Set up React project
   - Integrate WalletClient
   - Implement token minting
   - Add transfer functionality
   - Connect to overlay service

### Setup Requirements

**System Requirements**:
- Node.js 18+
- MongoDB 5.0+
- BSV Desktop Wallet
- TypeScript 4.9+

**BSV Requirements**:
- BSV Desktop Wallet installed and unlocked
- Test BSV for transaction fees (get from [faucet](https://faucet.bsvb.tech/))

---

## Key Concepts

### Token Structure (PushDrop 3-Field Format)

```typescript
{
  fields: [
    tokenId,    // UTF-8: '___mint___' or 'txid.vout'
    amount,     // Uint64LE (8 bytes)
    metadata    // JSON: [{ name: "label", value: "MyToken" }]
  ]
}
```

### Token Lifecycle

1. **Mint**: Create token with `'___mint___'` marker → tokenId becomes `txid.vout`
2. **Transfer**: Spend input UTXOs → create output UTXOs with same tokenId
3. **Split**: One input → multiple outputs (recipient + change)
4. **Merge**: Multiple inputs → one output (combining balances)

### Balance Conservation Rule

The overlay validates that inputs equal outputs:

```
Σ input amounts = Σ output amounts (except mints)
```

**Valid Transfer**:
```
Input:   +1000 tokens
Output1: -700 tokens (recipient)
Output2: -300 tokens (change)
Balance: 0 ✓ ADMITTED
```

**Invalid Transfer**:
```
Input:   +1000 tokens
Output1: -1200 tokens
Balance: -200 ✗ REJECTED
```

### Privacy Model

Every token output uses a **unique derived address** via BRC-42:
- Random `keyID` per output prevents address reuse
- Cannot link multiple UTXOs to same owner
- Requires storing `customInstructions` to spend later

### Two-Phase Signing

**Phase 1**: Wallet adds BSV inputs and estimates unlock script lengths

**Phase 2**: Manually sign token inputs using stored `customInstructions`

**Phase 3**: Submit signatures to wallet for finalization

This is required because wallet cannot derive PushDrop unlock scripts automatically.

---

## Testing the Complete System

### 1. Start Overlay Service

```bash
cd overlay
npm install
npm run build
npm start
# Running on http://localhost:3000
```

### 2. Start Frontend

```bash
cd frontend
npm install
npm run dev
# Running on http://localhost:5173
```

### 3. Test Token Flow

1. **Mint tokens**: Create new token supply in frontend
2. **Check overlay**: Verify tokens indexed in MongoDB
3. **Transfer tokens**: Send to another user via MessageBox
4. **Receive tokens**: Recipient claims tokens in their wallet
5. **Query balances**: View aggregated token balances

---

## Troubleshooting

**"Failed to connect to wallet"**
- Install BSV Desktop Wallet from [https://desktop.bsvb.tech/](https://desktop.bsvb.tech/)
- Unlock wallet before using frontend

**"Overlay rejected transaction"**
- Check balance conservation (inputs must equal outputs)
- Verify token structure (3-field PushDrop format)
- Check MongoDB connection

**"Cannot spend tokens"**
- Verify `customInstructions` stored when creating outputs
- Check protocolID and keyID match stored values

**"Tokens not appearing"**
- Verify correct basket name (`demotokens3` or your custom name)
- Check overlay admission logs
- Query MongoDB directly to verify indexing

---

## Enhancements

Once you've completed the base project, try these extensions:

### Frontend Enhancements
- Batch token transfers (multiple recipients)
- Token metadata editor
- Transaction history viewer
- Balance charts and analytics

### Overlay Enhancements
- Spend tracking (`outputSpent` hook)
- Token registry (whitelist/blacklist)
- Metadata validation rules
- Custom query endpoints

### Protocol Extensions
- Non-fungible tokens (NFTs)
- Token burning mechanism
- Multi-signature token controls
- Timelocked token releases

---

## Project Completion Checklist

- [ ] Overlay service validates mint transactions
- [ ] Overlay service validates transfer transactions (balance conservation)
- [ ] Overlay service rejects invalid transactions
- [ ] MongoDB indexes tokens by tokenId and outpoint
- [ ] Frontend mints new tokens
- [ ] Frontend displays token balances (aggregated by tokenId)
- [ ] Frontend transfers tokens using two-phase signing
- [ ] Frontend sends token notifications via MessageBox
- [ ] Recipient can claim tokens from MessageBox
- [ ] Tokens appear in recipient's wallet after claiming
- [ ] All transactions validated by overlay before broadcast

---

## Summary

This project demonstrates a complete BSV token system combining:

- **Frontend token management** (React + WalletClient)
- **Backend validation** (Overlay Services)
- **Protocol enforcement** (Balance conservation rules)
- **Efficient indexing** (MongoDB with optimized queries)
- **Privacy features** (BRC-42 key derivation)
- **P2P transfers** (MessageBox notifications)

These patterns form the foundation for building production-grade token systems, NFT marketplaces, loyalty programs, and other tokenized applications on BSV.

---

## Related Resources

### BRC Standards
- [BRC-42: Key Derivation Protocol](https://hub.bsvblockchain.org/brc/key-derivation/0042)
- [BRC-48: PushDrop Token Standard](https://hub.bsvblockchain.org/brc/scripts/0048)
- [BRC-57: Overlay Services Protocol](https://hub.bsvblockchain.org/brc/overlays/0088)
- [BRC-62: BEEF Transaction Format](https://hub.bsvblockchain.org/brc/transactions/0062)
- [BRC-33: MessageBox Protocol](https://hub.bsvblockchain.org/brc/peer-to-peer/0033)

### Documentation
- [BSV SDK Documentation](https://docs.bsvblockchain.org/)
- [Overlay Tools](https://github.com/bitcoin-sv/overlay-tools)
- [MongoDB Documentation](https://www.mongodb.com/docs/)
- [BSV Desktop Wallet](https://desktop.bsvb.tech/)

### Prerequisites
- [Token Fundamentals](../token-fundamentals/README.md)
- [Overlay Services](../overlay-services/README.md)
- [Beginner Path](../../beginner/README.md)
