# Project: Crowdfunding Platform

**Full-Stack BSV Application with Payment Protocol**

Build a crowdfunding platform where users invest with BSV micropayments and receive PushDrop tokens when the campaign completes. This project demonstrates real-world BSV patterns including the 402 Payment Protocol (BRC-103/104), key derivation (BRC-29), and token distribution.

**Repository**: [github.com/bsv-blockchain-demos/crowfunding-workshop-demo](https://github.com/bsv-blockchain-demos/crowfunding-workshop-demo)

---

## What You'll Build

A production-ready crowdfunding platform featuring:

- Campaign progress tracking with live updates
- BSV micropayments via 402 Payment Protocol
- Automatic token distribution to investors
- Both frontend (WalletClient) and backend (Wallet Toolbox) implementations

---

## Learning Objectives

By completing this project, you will learn:

- **Payment Protocol** - Implementing BRC-103/104 for HTTP payments
- **Key Derivation** - Using BRC-29 for secure payment addressing
- **WalletClient** - Frontend wallet integration with `createAction`
- **Wallet Toolbox** - Server-side wallet management
- **PushDrop Tokens** - Creating and distributing tokens to investors

---

## Architecture Overview

```
┌─────────────────────────────────────────┐
│         Frontend (Next.js/React)        │
│  - WalletClient for user payments       │
│  - createAction for transactions        │
│  - listOutputs for token viewing        │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│         Backend (Next.js API)           │
│  - Payment middleware (BRC-103/104)     │
│  - Wallet Toolbox for server wallet     │
│  - PushDrop token creation              │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│           BSV Blockchain                │
│  - Investment transactions              │
│  - Token distribution transactions      │
└─────────────────────────────────────────┘
```

---

## Key Patterns

### 1. Frontend Wallet Integration

Connect to the user's BSV Desktop Wallet using a React hook:

```typescript
import { useState, useEffect } from 'react'
import { WalletClient } from '@bsv/sdk'

export const useWallet = () => {
  const [wallet, setWallet] = useState<WalletClient | null>(null)
  const [identityKey, setIdentityKey] = useState<string | null>(null)

  async function initWallet() {
    const w = new WalletClient()
    const { publicKey } = await w.getPublicKey({ identityKey: true })
    setWallet(w)
    setIdentityKey(publicKey)
  }

  useEffect(() => {
    initWallet()
  }, [])

  return { wallet, identityKey }
}
```

> **Reference**: [WalletClient Integration](../../beginner/wallet-client-integration/README.md)

### 2. Payment Protocol (402 Flow)

The investment flow uses HTTP 402 Payment Required:

1. Frontend sends POST to `/api/invest`
2. Backend responds with 402 and derivation prefix
3. Frontend derives payment key and creates transaction
4. Frontend resubmits with `x-bsv-payment` header
5. Backend validates and internalizes payment

```typescript
// Frontend: Derive payment key and create transaction
const { publicKey: derivedPublicKey } = await wallet.getPublicKey({
  counterparty: backendIdentityKey,
  protocolID: brc29ProtocolID,
  keyID: `${derivationPrefix} ${derivationSuffix}`,
  forSelf: false
})

const lockingScript = new P2PKH().lock(
  PublicKey.fromString(derivedPublicKey).toAddress()
).toHex()

const result = await wallet.createAction({
  outputs: [{
    lockingScript,
    satoshis: investmentAmount,
    outputDescription: 'Crowdfunding investment'
  }],
  description: 'Investment in crowdfunding',
  options: { randomizeOutputs: false }  // Required for middleware
})
```

> **Reference**: [BRC-29](../../../sdk-components/brc-29/README.md) | [Transaction](../../../sdk-components/transaction/README.md)

### 3. BRC-29 Key Derivation

The protocol ID and derivation parameters are defined in the middleware:

```typescript
// lib/middleware.ts
export const BRC29_PROTOCOL_ID: [number, string] = [2, '3241645161d8']
export const DERIVATION_PREFIX = 'crowdfunding'
```

Frontend derives the payment key for the payee:

```typescript
const brc29ProtocolID: WalletProtocol = [2, '3241645161d8']

const { publicKey: derivedPublicKey } = await wallet.getPublicKey({
  counterparty: backendIdentityKey,
  protocolID: brc29ProtocolID,
  keyID: `${derivationPrefix} ${derivationSuffix}`,
  forSelf: false  // Critical: deriving for payee
})
```

> **Reference**: [BRC-29](../../../sdk-components/brc-29/README.md)

### 4. Backend Wallet Setup

Server-side wallet using Wallet Toolbox:

```typescript
import { PrivateKey, KeyDeriver } from '@bsv/sdk'
import { Wallet, WalletStorageManager, WalletSigner, Services, StorageClient } from '@bsv/wallet-toolbox'

const privateKey = PrivateKey.fromHex(process.env.PRIVATE_KEY)
const keyDeriver = new KeyDeriver(privateKey)
const storageManager = new WalletStorageManager(keyDeriver.identityKey)
const signer = new WalletSigner('main', keyDeriver, storageManager)
const services = new Services('main')

export const wallet = new Wallet(signer, services)

// Connect to storage provider
const client = new StorageClient(wallet, 'https://storage.babbage.systems')
await client.makeAvailable()
await storageManager.addWalletStorageProvider(client)
```

> **Reference**: [Private Keys](../../../sdk-components/private-keys/README.md) | [HD Wallets](../../../sdk-components/hd-wallets/README.md)

### 5. Payment Middleware

Express middleware for handling 402 payments:

```typescript
import { createPaymentMiddleware, createAuthMiddleware } from '@bsv/payment-express-middleware'

export async function getPaymentMiddleware() {
  return createPaymentMiddleware({
    wallet,
    calculateRequestPrice: () => 1  // Minimum to trigger flow
  })
}
```

### 6. Token Distribution

Create PushDrop tokens for investors (backend):

```typescript
const tokenDescription = `Crowdfunding token for ${investor.amount} sats`
const pushdrop = new PushDrop(wallet)

// Encrypt token metadata
const { ciphertext } = await wallet.encrypt({
  plaintext: Utils.toArray(tokenDescription, 'utf8'),
  protocolID: [0, 'token list'],
  keyID: '1',
  counterparty: 'anyone'
})

// Create locking script for investor
const lockingScript = await pushdrop.lock(
  [ciphertext],
  [0, 'token list'],
  '1',
  identityKey  // investor's identity key
)

// Create and distribute token
const result = await wallet.createAction({
  description: `Create token: ${tokenDescription}`,
  outputs: [{
    lockingScript: lockingScript.toHex(),
    satoshis: 1,
    basket: 'crowdfunding',
    outputDescription: 'Crowdfunding token'
  }],
  options: { randomizeOutputs: false }
})
```

Frontend internalizes the received token:

```typescript
await wallet.internalizeAction({
  tx: data.tx,
  outputs: [{
    outputIndex: 0,
    protocol: 'basket insertion',
    insertionRemittance: { basket: 'crowdfunding' }
  }],
  description: 'Internalize crowdfunding token'
})
```

### 7. Token Viewing

Investors view their tokens using `listOutputs`:

```typescript
const outputs = await wallet.listOutputs({
  basket: 'crowdfunding',
  include: 'locking scripts'
})

const tokens = []
for (const output of outputs.outputs) {
  if (!output.lockingScript) continue

  const script = LockingScript.fromHex(output.lockingScript)
  const decodedToken = PushDrop.decode(script)

  const txid = output.outpoint.split('.')[0]
  const vout = parseInt(output.outpoint.split('.')[1])

  tokens.push({
    txid,
    vout,
    satoshis: output.satoshis,
    lockingScript: output.lockingScript
  })
}
```

---

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/wallet-info` | GET | Backend wallet identity |
| `/api/invest` | POST | Submit investment (402 flow) |
| `/api/status` | GET | Campaign status |
| `/api/complete` | POST | Distribute tokens |
| `/api/my-tokens` | GET | User's tokens |

---

## Important Concepts

### randomizeOutputs: false

**Critical for payment middleware**. The middleware needs to know the exact output index for transaction internalization.

```typescript
await wallet.createAction({
  outputs: [...],
  options: { randomizeOutputs: false }
})
```

### P2PK vs P2PKH

PushDrop tokens use **P2PK** (Pay-to-Public-Key), not P2PKH:
- Script contains raw public key
- Cannot be found by searching address on explorer
- Must use transaction ID to find tokens

### Transaction Internalization

When the backend receives a payment, it internalizes the transaction to track the UTXO:

```typescript
await wallet.internalizeAction({
  tx: atomicBEEF,
  outputs: [{
    outputIndex: 0,
    protocol: 'wallet payment',
    paymentRemittance: {
      derivationPrefix,
      derivationSuffix,
      senderIdentityKey
    }
  }]
})
```

---

## Project Structure

```
crowdfunding-project/
├── pages/
│   ├── index.tsx           # Main UI
│   ├── tokens.tsx          # Token viewer
│   └── api/
│       ├── invest.ts       # Payment endpoint
│       ├── complete.ts     # Token distribution
│       └── my-tokens.ts    # Token discovery
├── src/
│   ├── wallet.ts           # Backend wallet
│   └── pushdrop.ts         # Token creation
├── lib/
│   ├── wallet.ts           # Frontend wallet hook
│   ├── middleware.ts       # Payment middleware
│   └── storage.ts          # State persistence
└── .env                    # Configuration
```

---

## Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/bsv-blockchain-demos/crowfunding-workshop-demo
   cd crowfunding-workshop-demo
   npm install
   ```

2. **Configure environment**
   ```env
   PRIVATE_KEY=<your-private-key-hex>
   STORAGE_URL=https://storage.babbage.systems
   NETWORK=main
   ```

3. **Run the application**
   ```bash
   npm run dev
   ```

4. **Connect BSV Desktop Wallet** and start investing

---

## Summary

This project demonstrates:

- **402 Payment Protocol** - HTTP-based micropayments
- **BRC-29 Key Derivation** - Secure payment addressing
- **WalletClient** - Frontend wallet integration
- **Wallet Toolbox** - Server-side wallet management
- **PushDrop Tokens** - Token creation and distribution

These patterns form the foundation for building real-world BSV payment applications.

---

## Related Resources

- [WalletClient Integration](../../beginner/wallet-client-integration/README.md)
- [Transaction Component](../../../sdk-components/transaction/README.md)
- [BRC-29 Standard](../../../sdk-components/brc-29/README.md)
- [Private Keys](../../../sdk-components/private-keys/README.md)
- [HD Wallets](../../../sdk-components/hd-wallets/README.md)
- [Wallet Toolbox Documentation](https://fast.brc.dev/)
