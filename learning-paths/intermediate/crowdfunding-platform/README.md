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
│  - internalizeAction for token claiming │
│  - listOutputs for token viewing        │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│         Backend (Next.js API)           │
│  - Auth middleware (BRC-103)            │
│  - Payment middleware (BRC-103/104)     │
│  - Wallet Toolbox for server wallet     │
│  - PushDrop token creation              │
│  - JSON file state persistence          │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│           BSV Blockchain                │
│  - Investment transactions              │
│  - Individual token claim transactions  │
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

Server-side wallet using Wallet Toolbox (`src/wallet.ts`):

```typescript
import { PrivateKey, KeyDeriver, WalletInterface } from '@bsv/sdk'
import { Wallet, WalletStorageManager, WalletSigner, Services, StorageClient, Chain } from '@bsv/wallet-toolbox'
import { config } from 'dotenv'

config() // Load .env file

// Initialize wallet configuration
const privateKeyHex = process.env.PRIVATE_KEY
const storageUrl = process.env.STORAGE_URL || 'https://storage.babbage.systems'
const network = (process.env.NETWORK || 'main') as Chain

if (!privateKeyHex) {
  throw new Error('PRIVATE_KEY not found in .env. Run: npm run setup')
}

// Initialize wallet from private key
const privateKey = PrivateKey.fromHex(privateKeyHex)
const keyDeriver = new KeyDeriver(privateKey)
const storageManager = new WalletStorageManager(keyDeriver.identityKey)
const signer = new WalletSigner(network, keyDeriver, storageManager)
const services = new Services(network)
const walletInstance = new Wallet(signer, services)

// Setup storage
const client = new StorageClient(walletInstance, storageUrl)
await client.makeAvailable()
await storageManager.addWalletStorageProvider(client)

console.log('✓ Backend wallet initialized')
console.log(`✓ Identity: ${keyDeriver.identityKey}`)

export const wallet: WalletInterface = walletInstance
```

> **Reference**: [Private Keys](../../../sdk-components/private-keys/README.md) | [HD Wallets](../../../sdk-components/hd-wallets/README.md)

### 5. Payment Middleware

Express middleware for handling 402 payments (`lib/middleware.ts`):

```typescript
import { createPaymentMiddleware, createAuthMiddleware } from '@bsv/payment-express-middleware'
import { wallet } from '../src/wallet'

// Price calculator for payment middleware
// Returns 1 sat minimum to trigger the payment flow
// The actual amount validation happens after internalization
export function calculateInvestmentPrice(req: any): number {
  // Try to extract amount from payment header if it exists
  const paymentHeader = req.headers['x-bsv-payment']
  if (paymentHeader && typeof paymentHeader === 'string') {
    try {
      const paymentData = JSON.parse(paymentHeader)
      if (paymentData.amount && typeof paymentData.amount === 'number') {
        return paymentData.amount
      }
    } catch (e) {
      console.error('Failed to parse payment header:', e)
    }
  }
  // Return 1 sat minimum to trigger the payment flow
  return 1
}

// Derivation parameters for BRC-29
export const BRC29_PROTOCOL_ID: [number, string] = [2, '3241645161d8']
export const DERIVATION_PREFIX = 'crowdfunding'

// Create auth middleware instance
export async function getAuthMiddleware() {
  return createAuthMiddleware({
    wallet,
    allowUnauthenticated: true, // Allow unauthenticated - we'll get identity from payment
    logger: console,
    logLevel: 'info'
  })
}

// Create payment middleware instance
export async function getPaymentMiddleware() {
  return createPaymentMiddleware({
    wallet,
    calculateRequestPrice: calculateInvestmentPrice
  })
}
```

### 6. Token Distribution

Individual token claiming flow (pages/api/complete.ts):

When the crowdfunding goal is reached, each investor can claim their token individually:

**Backend** (`pages/api/complete.ts`):

```typescript
import { PushDrop, Utils } from '@bsv/sdk'

// Verify goal is reached
if (crowdfunding.raised < crowdfunding.goal) {
  return res.status(400).json({
    error: 'Goal not reached',
    raised: crowdfunding.raised,
    goal: crowdfunding.goal
  })
}

// Find investor and check if already redeemed
const investor = crowdfunding.investors.find(
  (inv) => inv.identityKey === identityKey
)

if (!investor) {
  return res.status(400).json({ error: 'Investor not found' })
}

if (investor.redeemed === true) {
  return res.status(400).json({ error: 'Investor already redeemed' })
}

// Create PushDrop token for investor
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

// Create and send token
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

// Mark investor as redeemed
investor.redeemed = true
saveCrowdfundingData(walletIdentity.publicKey, crowdfunding)

// Check if all investors have redeemed
const allRedeemed = crowdfunding.investors.every(inv => inv.redeemed)
if (allRedeemed) {
  crowdfunding.isComplete = true
  crowdfunding.completionTxid = result?.txid
  saveCrowdfundingData(walletIdentity.publicKey, crowdfunding)
}
```

**Frontend** (`pages/index.tsx`) internalizes the received token:

```typescript
const response = await fetch('/api/complete', {
  method: 'POST',
  body: JSON.stringify({ identityKey: investorKey, paymentKey }),
  headers: { 'Content-Type': 'application/json' }
})

const data = await response.json()

if (response.ok) {
  await wallet.internalizeAction({
    tx: data.tx,
    outputs: [{
      outputIndex: 0,
      protocol: 'basket insertion',
      insertionRemittance: { basket: 'crowdfunding' }
    }],
    description: 'Internalize crowdfunding token'
  })
}
```

### 7. State Persistence

Crowdfunding state is saved to a JSON file (`lib/storage.ts`):

```typescript
import { readFileSync, writeFileSync, existsSync } from 'fs'
import { CrowdfundingState } from '../src/types'

const DATA_FILE = 'crowdfunding-data.json'

interface StoredData {
  walletIdentity: string
  crowdfunding: CrowdfundingState
}

export function loadCrowdfundingData(walletIdentity: string): CrowdfundingState {
  if (existsSync(DATA_FILE)) {
    try {
      const data = readFileSync(DATA_FILE, 'utf-8')
      const stored: StoredData = JSON.parse(data)

      // Check if wallet matches
      if (stored.walletIdentity === walletIdentity) {
        console.log('Loaded existing crowdfunding data for current wallet')
        return stored.crowdfunding
      } else {
        console.log('Wallet changed - starting fresh crowdfunding')
      }
    } catch (error) {
      console.error('Error loading crowdfunding data:', error)
    }
  }

  // Default state
  return {
    goal: 100,
    raised: 0,
    investors: [],
    isComplete: false,
    completionTxid: undefined
  }
}

export function saveCrowdfundingData(walletIdentity: string, state: CrowdfundingState): void {
  const stored: StoredData = {
    walletIdentity,
    crowdfunding: state
  }
  writeFileSync(DATA_FILE, JSON.stringify(stored, null, 2), 'utf-8')
}
```

This ensures:
- State survives server restarts
- Multiple wallets can run on same system
- Historical data is preserved
- Completion transaction TXID is saved

### 8. Token Viewing

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
| `/api/wallet-info` | GET | Backend wallet identity key |
| `/api/invest` | POST | Submit investment (402 payment flow) |
| `/api/status` | GET | Campaign progress and investor list |
| `/api/complete` | POST | Claim individual investor token |

---

## Important Concepts

### TypeScript Types

The application uses well-defined TypeScript types (`src/types.ts`):

```typescript
export interface Investor {
  identityKey: string
  amount: number
  timestamp: number
  redeemed?: boolean  // Tracks if investor has claimed their token
}

export interface CrowdfundingState {
  goal: number
  raised: number
  investors: Investor[]
  isComplete: boolean
  completionTxid?: string
}
```

### Individual Token Claiming

Unlike batch distributions, this implementation uses an **individual claiming pattern**:

1. Investors make investments via the 402 payment flow
2. When goal is reached, each investor can claim their token
3. Backend creates one token per claim request
4. Each investor is marked as `redeemed` after claiming
5. Campaign is marked complete when all investors have redeemed

This approach:
- Reduces backend transaction fees (spread across investors)
- Gives investors control over when they claim
- Prevents a single large transaction failure from blocking all tokens

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

When the backend receives a payment, it internalizes the transaction to track the UTXO. The payment middleware handles this automatically:

```typescript
// Payment middleware automatically internalizes with these parameters
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
│       ├── invest.ts       # Investment endpoint with payment middleware
│       ├── complete.ts     # Individual token claiming
│       ├── status.ts       # Campaign status
│       └── wallet-info.ts  # Backend wallet identity
├── src/
│   ├── wallet.ts           # Backend wallet initialization
│   ├── setupWallet.ts      # Setup script for backend wallet
│   └── types.ts            # TypeScript type definitions
├── lib/
│   ├── wallet.ts           # Frontend wallet hook
│   ├── middleware.ts       # Auth & payment middleware
│   ├── crowdfunding.ts     # Crowdfunding state management
│   └── storage.ts          # JSON file persistence
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

2. **Setup Backend Wallet**

   This creates a backend wallet and funds it with 10,000 satoshis from your local wallet:

   ```bash
   npm run setup
   ```

   **What this does:**
   - Creates a new private key (or uses existing from `.env`)
   - Initializes a backend wallet using BSV Wallet Toolbox
   - Connects to your BSV Desktop Wallet
   - Sends 10,000 satoshis to the backend wallet via BRC-29 payment
   - Saves wallet configuration to `.env`

3. **Start the application**
   ```bash
   npm run dev
   ```

   Open [http://localhost:3000](http://localhost:3000) in your browser.

4. **Connect BSV Desktop Wallet** and start investing

---

## Summary

This project demonstrates:

- **402 Payment Protocol (BRC-103/104)** - HTTP-based micropayments with auth and payment middleware
- **BRC-29 Key Derivation** - Secure payment addressing with protocol ID `[2, '3241645161d8']`
- **WalletClient** - Frontend wallet integration with `createAction` and `internalizeAction`
- **Wallet Toolbox** - Server-side wallet management with storage providers
- **PushDrop Tokens** - Individual token claiming pattern for investors
- **State Persistence** - JSON file storage keyed by wallet identity
- **Individual Token Claiming** - Distributed fee model where investors claim tokens individually

These patterns form the foundation for building real-world BSV payment applications with proper state management and token distribution.

---

## Related Resources

- [WalletClient Integration](../../beginner/wallet-client-integration/README.md)
- [Transaction Component](../../../sdk-components/transaction/README.md)
- [BRC-29 Standard](../../../sdk-components/brc-29/README.md)
- [Private Keys](../../../sdk-components/private-keys/README.md)
- [HD Wallets](../../../sdk-components/hd-wallets/README.md)
- [Wallet Toolbox Documentation](https://fast.brc.dev/)
