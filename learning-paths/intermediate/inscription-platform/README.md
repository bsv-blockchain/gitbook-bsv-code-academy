# Project: Inscription Platform

**Minimal BSV Inscriptions Platform with WalletClient**

Build an inscription platform that allows users to store data permanently on the BSV blockchain using OP_RETURN outputs. This project demonstrates on-chain data storage patterns, basket organization, and WalletClient integration.

**Repository**: [github.com/bsv-blockchain-demos/inscription-platform](https://github.com/bsv-blockchain-demos/inscription-platform)

---

## What You'll Build

A production-ready inscription platform featuring:

- Text and JSON data inscriptions on-chain
- Document and image hash storage for verification
- Basket-based inscription organization
- Transaction history with localStorage persistence
- Frontend WalletClient integration for user wallets

---

## Learning Objectives

By completing this project, you will learn:

- **OP_RETURN Inscriptions** - Storing arbitrary data on-chain using OP_FALSE OP_RETURN patterns
- **Basket Management** - Organizing inscriptions by type (text, json, hash-document, hash-image)
- **WalletClient** - Frontend wallet integration with `createAction`
- **Data Hashing** - Creating SHA-256 hashes for file verification
- **LocalStorage** - Client-side transaction history persistence

---

## Architecture Overview

```
┌─────────────────────────────────────────┐
│         Frontend (React + Vite)         │
│  - WalletClient for user wallet         │
│  - createAction for inscriptions        │
│  - Basket selection UI                  │
│  - LocalStorage history                 │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│           BSV Blockchain                │
│  - OP_RETURN inscription transactions   │
│  - Organized by basket type             │
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
  const [walletClient, setWalletClient] = useState<WalletClient | null>(null)
  const [identityKey, setIdentityKey] = useState<string | null>(null)
  const [isConnected, setIsConnected] = useState(false)

  async function connect() {
    const client = new WalletClient()
    const { publicKey } = await client.getPublicKey({ identityKey: true })
    setWalletClient(client)
    setIdentityKey(publicKey)
    setIsConnected(true)
  }

  useEffect(() => {
    connect()
  }, [])

  return { walletClient, identityKey, isConnected }
}
```

> **Reference**: [WalletClient Integration](../../beginner/wallet-client-integration/README.md)

### 2. Creating Inscriptions with OP_RETURN

Inscriptions use OP_FALSE OP_RETURN to store data on-chain:

```typescript
import { Script, Utils, WalletClient } from '@bsv/sdk'

export type BasketType = 'text' | 'json' | 'hash-document' | 'hash-image'

export class InscriptionService {
  constructor(private walletClient: WalletClient | null) {}

  async createInscription(data: string, basketType: BasketType) {
    if (!this.walletClient) {
      throw new Error('Wallet not connected')
    }

    // Create OP_RETURN script with data
    const dataBytes = Utils.toArray(data, 'utf8')
    const script = new Script()
      .writeOpCode(0)  // OP_FALSE
      .writeOpCode(106) // OP_RETURN
      .writeBin(dataBytes)

    // Create inscription transaction
    const result = await this.walletClient.createAction({
      outputs: [{
        lockingScript: script.toHex(),
        satoshis: 0,  // OP_RETURN outputs have 0 satoshis
        basket: basketType,
        outputDescription: `Inscription: ${basketType}`
      }],
      description: `Create ${basketType} inscription`,
      options: { randomizeOutputs: false }
    })

    return {
      txid: result.txid,
      basket: basketType,
      size: dataBytes.length
    }
  }
}
```

> **Reference**: [OP_RETURN](../../../code-features/op-return/README.md) | [Script](../../../sdk-components/script/README.md)

### 3. Basket Types

The platform supports four basket types for organizing inscriptions:

| Basket Type | Description | Use Case |
|-------------|-------------|----------|
| `text` | Plain text data | Messages, notes, public statements |
| `json` | JSON data structures | Structured data, metadata, configurations |
| `hash-document` | Document SHA-256 hash | Document verification, timestamping |
| `hash-image` | Image SHA-256 hash | Image verification, proof of existence |

### 4. File Hashing

Create SHA-256 hashes for document/image verification:

```typescript
const hashFile = async (file: File): Promise<string> => {
  const arrayBuffer = await file.arrayBuffer()
  const hashBuffer = await crypto.subtle.digest('SHA-256', arrayBuffer)
  const hashArray = Array.from(new Uint8Array(hashBuffer))
  const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('')
  return hashHex
}

// Usage
const file = e.target.files?.[0]
const hash = await hashFile(file)
// Store hash on-chain as inscription
```

### 5. Transaction History

Persist inscription history in localStorage:

```typescript
export interface InscriptionRecord {
  txid: string
  basketType: BasketType
  data: string
  timestamp: number
  size: number
}

export class HistoryManager {
  private static STORAGE_KEY = 'bsv-inscriptions-history'

  static saveTransaction(record: InscriptionRecord): void {
    const history = this.loadHistory()
    history.unshift(record)
    // Keep last 50 transactions
    const trimmed = history.slice(0, 50)
    localStorage.setItem(this.STORAGE_KEY, JSON.stringify(trimmed))
  }

  static loadHistory(): InscriptionRecord[] {
    try {
      const data = localStorage.getItem(this.STORAGE_KEY)
      return data ? JSON.parse(data) : []
    } catch {
      return []
    }
  }

  static clearHistory(): void {
    localStorage.removeItem(this.STORAGE_KEY)
  }
}
```

---

## Important Concepts

### OP_RETURN Outputs

OP_RETURN is a Bitcoin script opcode that marks outputs as provably unspendable:

- **OP_FALSE OP_RETURN pattern**: Creates a data-carrying output
- **0 satoshis**: OP_RETURN outputs carry no value
- **Data limit**: Practically unlimited on BSV (unlike BTC's 80 bytes)
- **Basket organization**: Group inscriptions by type for easy retrieval

### Data Permanence

Data stored in OP_RETURN outputs is:
- **Immutable**: Cannot be changed once on-chain
- **Permanent**: Stored in the blockchain forever
- **Timestamped**: Block timestamp proves when data was inscribed
- **Verifiable**: Anyone can verify data existence via transaction ID

### Use Cases

This inscription pattern enables:
- **Document timestamping**: Prove document existed at a specific time
- **Copyright registration**: On-chain proof of creation
- **Data anchoring**: Store cryptographic commitments
- **Public statements**: Immutable messages and declarations
- **Metadata storage**: NFT metadata, token information
- **Audit trails**: Tamper-proof record keeping

---

## Project Structure

```
inscription-platform/
├── src/
│   ├── App.tsx              # Main UI component
│   ├── main.tsx             # React entry point
│   ├── index.css            # Tailwind styles
│   ├── hooks/
│   │   └── useWallet.ts     # WalletClient hook
│   └── services/
│       ├── InscriptionService.ts  # Inscription creation logic
│       └── HistoryManager.ts      # LocalStorage history
├── package.json
├── vite.config.ts
├── tailwind.config.js
└── tsconfig.json
```

---

## Setup

1. **Navigate to the project**
   ```bash
   cd inscription_platform
   npm install
   ```

2. **Start the development server**
   ```bash
   npm run dev
   ```

   Open [http://localhost:5173](http://localhost:5173) in your browser.

3. **Connect BSV Desktop Wallet** and create inscriptions

---

## Features

### Text Inscriptions
- Store plain text messages on-chain
- Use for public statements, notes, or messages
- Example: "Hello BSV blockchain!"

### JSON Inscriptions
- Store structured data as JSON
- Automatic JSON validation
- Use for metadata, configurations, or structured records
- Example: `{"name": "My Document", "version": "1.0"}`

### Hash Inscriptions
- Upload a file to generate SHA-256 hash
- Store only the hash on-chain (not the file itself)
- Verify file authenticity by comparing hashes
- Supports documents (PDF, DOC, etc.) and images (JPG, PNG, etc.)

### History View
- View all your inscriptions
- Transaction IDs link to blockchain explorers
- Shows basket type, timestamp, and data size
- Persists across browser sessions

---

## Summary

This project demonstrates:

- **OP_RETURN Inscriptions** - Storing arbitrary data on-chain with OP_FALSE OP_RETURN pattern
- **Basket Management** - Organizing inscriptions by type for easy retrieval
- **WalletClient** - Frontend wallet integration with `createAction`
- **File Hashing** - Creating SHA-256 hashes for document/image verification
- **LocalStorage** - Client-side persistence for transaction history
- **Data Permanence** - Understanding immutability and timestamping on blockchain

These patterns form the foundation for building data storage applications, timestamping services, and verification systems on BSV.

---

## Related Resources

- [WalletClient Integration](../../beginner/wallet-client-integration/README.md)
- [OP_RETURN](../../../code-features/op-return/README.md)
- [Script Component](../../../sdk-components/script/README.md)
- [Transaction Component](../../../sdk-components/transaction/README.md)
- [BSV Desktop Wallet](https://desktop.bsvb.tech/)
