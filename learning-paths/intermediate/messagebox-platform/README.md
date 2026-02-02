# Project: MessageBox Platform

**Peer-to-Peer Encrypted Messaging and Payments with BSV SDK**

Build a peer-to-peer messaging and payment platform using BSV's MessageBox network for encrypted, ephemeral message delivery and BRC-29 identity-based payments.

> **📦 Complete Code Repository**: [https://github.com/bsv-blockchain-demos/messagebox-platform](https://github.com/bsv-blockchain-demos/messagebox-platform)
>
> This guide references the full production implementation. Clone the repository to follow along with working code examples.

---

## What You'll Build

A production-ready MessageBox platform featuring:

- Identity certification on the BSV blockchain
- Encrypted peer-to-peer messaging via MessageBox network
- BRC-29 compliant identity-based payments
- Transaction internalization for payment tracking
- Frontend and server implementation patterns

---

## Learning Objectives

By completing this project, you will learn:

- **Identity Management** - Using BSV public keys as universal identities
- **MessageBox Protocol** - Encrypted, ephemeral message delivery between identities
- **PeerPay Protocol** - BRC-29 payment tokens with key derivation
- **Transaction Internalization** - Adding received payments to wallet balance
- **WalletClient** - Browser-based signing without exposing private keys
- **Certification** - Creating blockchain proofs of identity registration

---

## Architecture Overview

```
┌─────────────────────────────────────────┐
│         Frontend (Browser)              │
│  - WalletClient for identity            │
│  - MessageBoxClient for certification    │
│  - PeerPayClient for payments           │
│  - Encrypted message handling           │
└─────────────────┬───────────────────────┘
                  │
                  │ Encrypted Messages
                  │
┌─────────────────▼───────────────────────┐
│      MessageBox Network                 │
│  (messagebox.babbage.systems)           │
│  - Identity certification                │
│  - Ephemeral message storage            │
│  - End-to-end encryption                │
└─────────────────┬───────────────────────┘
                  │
                  │ Transactions
                  │
┌─────────────────▼───────────────────────┐
│         BSV Blockchain                  │
│  - Identity certification txs            │
│  - Payment transactions (BRC-29)        │
│  - Permanent, immutable storage         │
└─────────────────────────────────────────┘
```

---

## Implementation Guides

This project is split into two implementation guides:

### [Frontend Usage](./frontend.md)
Learn how to integrate MessageBox functionality into your browser-based applications:
- Connecting to user wallets with WalletClient
- Certifying identities on the blockchain
- Sending BRC-29 payments with PeerPayClient
- Receiving and internalizing payments
- Handling encrypted messages

### [Server Usage](./server.md)
Learn how to implement MessageBox patterns on the server side:
- Server-side wallet management
- Session handling for connected wallets
- Backend certification and payment endpoints
- Database storage for certified users
- API design for MessageBox applications

---

## Key Concepts

### Identity-Based Architecture

Unlike traditional systems that use usernames/passwords, BSV MessageBox uses public keys as identities:

**Traditional Auth:**
```
User → Username/Password → Database → Session
```

**BSV Identity:**
```
User → Wallet → Public Key (Identity) → Blockchain Certification
```

**Benefits:**
- ✅ No password management
- ✅ Cryptographically provable identity
- ✅ Works across all BSV applications
- ✅ No central authority can revoke

### MessageBox Network

The MessageBox network provides encrypted, ephemeral message delivery:

- **End-to-end encrypted**: Only recipient can decrypt (ECIES)
- **Ephemeral storage**: Messages deleted when acknowledged
- **Identity-based**: Send to public keys, not addresses
- **Privacy-preserving**: No permanent message history

### BRC-29 Payments

BRC-29 enables sending payments directly to identities (public keys):

```typescript
// Traditional Bitcoin: Send to address
sendToAddress("1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa")

// BRC-29: Send to identity (public key)
sendToIdentity("03abc123...def456")
```

**Key Advantages:**
- New derived address per payment (privacy)
- No address management needed
- Universal identity across apps
- Recipient derives spending key with payment remittance data

### Overlay Integration

MessageBox can be integrated with [Overlay Services](../overlay-services/README.md) for whitelabel deployments:

- Run your own MessageBox infrastructure
- Custom validation rules
- Private messaging networks
- Enterprise deployments

See the [Overlay Services guide](../overlay-services/README.md) for implementing custom overlay networks with Topic Managers and Lookup Services.

---

## Use Cases

### Peer-to-Peer Applications
- Direct payments between users
- Encrypted messaging
- Social media with built-in monetization
- Content creator platforms

### Decentralized Identity
- Universal identity across applications
- Blockchain-certified credentials
- Privacy-preserving authentication
- Cross-platform interoperability

### Micropayments
- Pay-per-message services
- Content tips and donations
- API access payments
- Subscription services

### Enterprise Communication
- Secure team messaging
- Encrypted document sharing
- Payment reconciliation
- Audit trails on blockchain

---

## Why BSV for MessageBox?

### Low Transaction Fees
- Average tx fee: ~$0.000006 USD
- Viable for micropayments
- No need for off-chain solutions

### Scalability
- 50,000+ transactions per second
- No Layer 2 complexity
- All transactions on-chain

### Identity Standards
- BRC-29: Identity-based payments
- BRC-42: Key derivation protocol
- BRC-48: PushDrop token format
- Interoperable across wallets

### Protocol Stability
- Frozen protocol (no more hard forks)
- Build once, run forever
- Long-term application viability

---

## Getting Started

### Quick Start with Demo Repository

```bash
# Clone the complete working implementation
git clone https://github.com/bsv-blockchain-demos/messagebox-platform.git
cd messagebox-platform

# Install dependencies
npm run install:all

# Set up environment variables
cp .env.example .env
# Edit .env with your MongoDB URI and MessageBox host

# Start backend server
npm run dev:server

# In another terminal, start frontend
npm run dev:frontend
```

### Choose Your Learning Path

1. **[Frontend Implementation](./frontend.md)** - Start here if you're building a browser-based app
2. **[Server Implementation](./server.md)** - Start here if you're building backend services

Both guides use the BSV SDK (`@bsv/sdk`) and MessageBox client (`@bsv/message-box-client`) and reference the [complete working code](https://github.com/bsv-blockchain-demos/messagebox-platform).

---

## Related Resources

- [WalletClient Integration](../../beginner/wallet-client-integration/README.md)
- [Overlay Services](../overlay-services/README.md)
- [BRC-29: Payment Addressing](https://brc.dev/29)
- [BRC-42: Key Derivation](https://brc.dev/42)
- [MessageBox Protocol](https://github.com/bitcoin-sv/message-box)
- [BSV SDK Documentation](https://docs.bsvblockchain.org/)
