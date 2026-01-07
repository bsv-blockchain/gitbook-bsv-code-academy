# Project: Token Overlay Service

**BSV Overlay Indexing Node with Protocol Validation**

Build an overlay service that acts as a specialized transaction broadcaster, validating token protocol rules and indexing admitted UTXOs. This project demonstrates overlay architecture (BRC-57), Topic Managers, Lookup Services, and MongoDB integration for fast token queries.

---

## What You'll Build

A production-ready overlay service featuring:

- Transaction reception via BEEF format (BRC-62)
- Protocol-specific validation (balance conservation, mint detection)
- Token UTXO indexing in MongoDB
- REST API for querying tokens by ID or outpoint
- Integration with ARC broadcasters

---

## Learning Objectives

By completing this project, you will learn:

- **Overlay Architecture** - Building specialized BSV indexing nodes (BRC-57)
- **Topic Managers** - Validating and filtering transaction outputs
- **Lookup Services** - Indexing and querying blockchain data
- **BEEF Parsing** - Extracting transactions from BRC-62 format
- **PushDrop Decoding** - Reading token data from locking scripts (BRC-48)
- **MongoDB Integration** - Efficient UTXO storage and retrieval
- **Balance Validation** - Enforcing fungible token rules

---

## Architecture Overview

```
┌─────────────────────────────────────────┐
│           Client Application             │
│  - Creates token transaction (BEEF)     │
│  - Tags with topic: 'tm_tokendemo'      │
└─────────────────────────────────────────┘
                    ↓ HTTP POST
┌─────────────────────────────────────────┐
│      Overlay Service (Express.js)       │
│  - /submit endpoint                     │
│  - OverlayExpressNode router            │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│        Topic Manager (Validator)        │
│  - Parse BEEF transaction               │
│  - Decode PushDrop tokens               │
│  - Track token balances                 │
│  - Validate conservation rules          │
│  - Return admit/reject decision         │
└─────────────────────────────────────────┘
                    ↓ (if admitted)
┌─────────────────────────────────────────┐
│     Lookup Service (Indexer)            │
│  - outputAdmittedByTopic hook           │
│  - Extract token details                │
│  - Store in MongoDB                     │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│           MongoDB Database              │
│  - Token UTXOs indexed                  │
│  - Query by tokenId/outpoint            │
│  - Fast pagination support              │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│         Lookup API (/lookup)            │
│  - Query tokens by ID                   │
│  - Query by outpoint                    │
│  - List all tokens (paginated)          │
└─────────────────────────────────────────┘
```

---

## Key Patterns

### 1. Overlay Service Setup

Initialize the overlay Express node with topic managers and lookup services:

```typescript
import express from 'express'
import { OverlayExpressNode } from '@bsv/overlay-express'
import { TokenDemoTopicManager } from './token-demo/TokenDemoTopicManager'
import { TokenDemoLookupServiceFactory } from './token-demo/TokenDemoLookupServiceFactory'

const app = express()

// Initialize overlay node
const overlayNode = new OverlayExpressNode({
  port: 3000,
  app,
  nodeImpl: {
    getTopicManagers: () => [new TokenDemoTopicManager()],
    getLookupServiceFactory: () => new TokenDemoLookupServiceFactory()
  }
})

app.listen(3000, () => {
  console.log('Overlay service running on http://localhost:3000')
})
```

**Key Components**:
- **OverlayExpressNode**: Handles `/submit` and `/lookup` endpoints
- **Topic Managers**: Validate and filter outputs
- **Lookup Services**: Index and query admitted outputs

> **Reference**: [BRC-57: Overlay Services](https://hub.bsvblockchain.org/brc/overlays/0088)

### 2. Transaction Reception

Client submits transactions in tagged BEEF format:

```typescript
// Client-side submission
const taggedBEEF = {
  beef: transaction.toBEEF(),
  topics: ['tm_tokendemo']  // Topic manager ID
}

const response = await fetch('http://localhost:3000/submit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(taggedBEEF)
})

// Response format
{
  "tm_tokendemo": {
    "outputsToAdmit": [0, 1],  // Admitted output indices
    "coinsToRetain": []        // Retained input indices
  }
}
```

**Flow**:
1. Client creates transaction
2. Packages as BEEF (includes parent txs + merkle proofs)
3. Tags with topic manager ID
4. POSTs to `/submit`
5. Overlay validates and returns admission decision

> **Reference**: [BRC-62: BEEF Format](https://hub.bsvblockchain.org/brc/transactions/0062)

### 3. Topic Manager Validation

Validate token protocol rules before admitting outputs:

```typescript
import { Transaction, PushDrop, Utils, OP } from '@bsv/sdk'

class TokenDemoTopicManager implements TopicManager {
  async identifyAdmissibleOutputs(beef: number[]): Promise<AdmittanceInstructions> {
    // Parse BEEF transaction
    const tx = Transaction.fromBEEF(beef)
    const txid = tx.id('hex')

    // Track token balances (inputs add, outputs subtract)
    const balances = new Map<string, { amount: number, isMint: boolean }>()

    // Process inputs (credits)
    for (const input of tx.inputs) {
      const sourceOutput = input.sourceTransaction.outputs[input.sourceOutputIndex]

      // Decode token from locking script
      const token = PushDrop.decode(sourceOutput.lockingScript)
      let tokenId = Utils.toUTF8(token.fields[0])

      // Handle mint marker
      if (tokenId === '___mint___') {
        const sourceTxid = input.sourceTransaction.id('hex')
        tokenId = `${sourceTxid}.${input.sourceOutputIndex}`
      }

      const amount = new Utils.Reader(token.fields[1]).readUInt64LEBn()

      // Add input amount (credit balance)
      balances.set(tokenId, {
        amount: (balances.get(tokenId)?.amount || 0) + amount,
        isMint: balances.get(tokenId)?.isMint || false
      })
    }

    // Process outputs (debits)
    const outputsToAdmit: number[] = []

    for (let vout = 0; vout < tx.outputs.length; vout++) {
      const output = tx.outputs[vout]

      // Validate P2PKH-style script
      if (output.lockingScript.chunks[1].op !== OP.OP_CHECKSIG) continue

      try {
        const token = PushDrop.decode(output.lockingScript)
        let tokenId = Utils.toUTF8(token.fields[0])

        // Check for mint
        if (tokenId === '___mint___') {
          tokenId = `${txid}.${vout}`
          balances.set(tokenId, { amount: 0, isMint: true })
        }

        const amount = new Utils.Reader(token.fields[1]).readUInt64LEBn()

        // Subtract output amount (debit balance)
        const current = balances.get(tokenId) || { amount: 0, isMint: false }
        balances.set(tokenId, {
          amount: current.amount - amount,
          isMint: current.isMint
        })

        outputsToAdmit.push(vout)
      } catch (err) {
        // Invalid PushDrop format - skip
        continue
      }
    }

    // Validate balance conservation
    for (const [tokenId, balance] of balances) {
      // Mints can create new tokens (positive balance OK)
      if (balance.isMint) continue

      // Regular transfers must balance (inputs = outputs)
      if (balance.amount !== 0) {
        return { outputsToAdmit: [], coinsToRetain: [] }  // REJECT
      }
    }

    return { outputsToAdmit, coinsToRetain: [] }
  }

  getDocumentation(): string {
    return 'Token Demo Topic Manager - validates fungible token transfers'
  }

  getMetaData(): TopicManagerMetadata {
    return {
      name: 'tm_tokendemo',
      shortDescription: 'Token Demo Protocol',
      iconURL: 'https://example.com/icon.png',
      version: '1.0.0',
      informationURL: 'https://example.com/docs'
    }
  }
}
```

**Validation Rules**:
1. **Mint tokens** (`___mint___`): Create new balance (no inputs required)
2. **Regular transfers**: Total inputs MUST equal total outputs
3. **Reject unbalanced**: Any non-zero balance after accounting = rejection

> **Reference**: [PushDrop Decoding](https://hub.bsvblockchain.org/brc/scripts/0048#implementations)

### 4. Lookup Service Indexing

Index admitted outputs in MongoDB for fast queries:

```typescript
import { LookupService } from '@bsv/overlay-tools'

class TokenDemoLookupService implements LookupService {
  constructor(private storage: TokenDemoStorage) {}

  async outputAdmittedByTopic(payload: OutputAdmittedByTopic): Promise<void> {
    const { txid, outputIndex, lockingScript } = payload

    // Decode token details
    const token = PushDrop.decode(LockingScript.fromHex(lockingScript))

    let tokenId = Utils.toUTF8(token.fields[0])
    if (tokenId === '___mint___') {
      tokenId = `${txid}.${outputIndex}`
    }

    const amount = new Utils.Reader(token.fields[1]).readUInt64LEBn()
    const customFields = JSON.parse(Utils.toUTF8(token.fields[2]))

    // Store in MongoDB
    await this.storage.storeRecord(txid, outputIndex, {
      tokenId,
      amount: amount.toString(),  // Store as string for large numbers
      customFields,
      createdAt: new Date()
    })
  }

  async lookup(question: LookupQuestion): Promise<LookupAnswer> {
    const { outpoint, tokenId, limit, skip, sortOrder } = question.query

    // Query by outpoint
    if (outpoint) {
      const [txid, vout] = outpoint.split('.')
      return await this.storage.findByOutpoint(txid, parseInt(vout))
    }

    // Query by tokenId
    if (tokenId) {
      return await this.storage.findByTokenId(tokenId, limit, skip, sortOrder)
    }

    // List all tokens
    return await this.storage.findAll(limit, skip, sortOrder)
  }

  getDocumentation(): string {
    return 'Query token UTXOs by tokenId or outpoint'
  }

  getMetaData(): LookupServiceMetadata {
    return {
      name: 'ls_tokendemo',
      shortDescription: 'Token Demo Lookup',
      version: '1.0.0'
    }
  }
}
```

**Key Events**:
- **outputAdmittedByTopic**: Triggered when Topic Manager admits output
- **lookup**: Triggered by client queries via `/lookup` endpoint

> **Reference**: [Overlay Tools](https://hub.bsvblockchain.org/brc/overlays/0022)

### 5. MongoDB Storage

Efficient UTXO storage with composite and hashed indices:

```typescript
import { MongoClient, Db, Collection } from 'mongodb'

class TokenDemoStorage {
  private collection: Collection

  async initialize() {
    const client = new MongoClient('mongodb://localhost:27017')
    await client.connect()
    this.collection = client.db('overlay').collection('token_demo')

    // Create indices for fast queries
    await this.collection.createIndex(
      { txid: 1, outputIndex: 1 },
      { unique: true }
    )
    await this.collection.createIndex({ tokenId: 'hashed' })
  }

  async storeRecord(txid: string, outputIndex: number, details: TokenDetails) {
    await this.collection.insertOne({
      txid,
      outputIndex,
      ...details
    })
  }

  async findByOutpoint(txid: string, vout: number): Promise<UTXOReference[]> {
    const record = await this.collection.findOne({ txid, outputIndex: vout })

    if (!record) return []

    return [{ txid: record.txid, outputIndex: record.outputIndex }]
  }

  async findByTokenId(
    tokenId: string,
    limit = 50,
    skip = 0,
    sortOrder = 'desc'
  ): Promise<UTXOReference[]> {
    const sort = sortOrder === 'desc' ? -1 : 1

    const records = await this.collection
      .find({ tokenId })
      .sort({ createdAt: sort })
      .skip(skip)
      .limit(limit)
      .toArray()

    return records.map(r => ({ txid: r.txid, outputIndex: r.outputIndex }))
  }

  async findAll(limit = 100, skip = 0, sortOrder = 'desc') {
    const sort = sortOrder === 'desc' ? -1 : 1

    const records = await this.collection
      .find({})
      .sort({ createdAt: sort })
      .skip(skip)
      .limit(limit)
      .toArray()

    return records.map(r => ({ txid: r.txid, outputIndex: r.outputIndex }))
  }
}
```

**Indices**:
- **Composite** `{txid, outputIndex}`: Fast outpoint lookups
- **Hashed** `{tokenId}`: Efficient tokenId searches

> **Reference**: [MongoDB Indexing](https://www.mongodb.com/docs/manual/indexes/)

### 6. Querying from Frontend

Use `LookupResolver` to query indexed tokens:

```typescript
import { LookupResolver } from '@bsv/sdk'

const resolver = new LookupResolver({
  url: 'http://localhost:3000/lookup'
})

// Query all tokens of a specific ID
const utxos = await resolver.query({
  service: 'ls_tokendemo',
  query: {
    tokenId: 'abc123.0',
    limit: 50,
    skip: 0,
    sortOrder: 'desc'
  }
})

// Returns: [{ txid: "...", outputIndex: 0 }, ...]
```

**Query Types**:
1. **By tokenId**: Get all UTXOs of a token (with pagination)
2. **By outpoint**: Get specific UTXO (exact lookup)
3. **List all**: Get all tokens (paginated)

---

## Important Concepts

### Balance Accounting Model

The Topic Manager uses double-entry bookkeeping:

```
Inputs (Credits):  +amount
Outputs (Debits):  -amount
Final Balance:     must = 0 (except mints)
```

**Example Transfer**:
```
Input:   tokenId "abc.0" → +1000 sats
Output1: tokenId "abc.0" → -700 sats (recipient)
Output2: tokenId "abc.0" → -300 sats (change)
Balance: +1000 - 700 - 300 = 0 ✓ ADMITTED
```

**Example Invalid**:
```
Input:   tokenId "abc.0" → +1000 sats
Output1: tokenId "abc.0" → -1200 sats
Balance: +1000 - 1200 = -200 ✗ REJECTED
```

### Mint Detection

Special case where new tokens are created:

```typescript
// Output with '___mint___' marker
if (tokenId === '___mint___') {
  tokenId = `${txid}.${vout}`  // Assign outpoint as tokenId
  balances.set(tokenId, { amount: 0, isMint: true })
}
```

**No input validation required** for mints (creates new supply).

### UTXO Lifecycle

```
1. Transaction submitted → Topic Manager validates
2. Output admitted → Lookup Service indexes in MongoDB
3. Output spent → (optionally) remove from index
4. Spent transaction submitted → New outputs indexed
```

**Current Implementation**: No spend tracking (`spendNotificationMode: 'none'`)

### Topic Manager vs Lookup Service

| Component | Purpose | Timing |
|-----------|---------|--------|
| **Topic Manager** | Validate protocol rules | Before admission |
| **Lookup Service** | Index and query data | After admission |

Topic Managers are **gatekeepers**, Lookup Services are **librarians**.

### BRC Standards Integration

- **BRC-48**: PushDrop token format (3-field structure)
- **BRC-57**: Overlay service architecture
- **BRC-62**: BEEF transaction packaging

---

## Project Structure

```
overlay/
├── index.ts                                   # Express server setup
├── token-demo/
│   ├── TokenDemoTopicManager.ts               # Validation logic
│   ├── TokenDemoLookupServiceFactory.ts       # Service factory
│   ├── TokenDemoStorage.ts                    # MongoDB operations
│   ├── types.ts                               # TypeScript interfaces
│   └── TokenDemoTopicDocs.ts                  # Documentation
├── package.json                               # Dependencies
└── tsconfig.json                              # TypeScript config
```

---

## Setup

1. **Install MongoDB**

   ```bash
   brew install mongodb-community  # macOS
   brew services start mongodb-community
   ```

2. **Install dependencies**
   ```bash
   cd overlay
   npm install
   ```

3. **Build and start**
   ```bash
   npm run build
   npm start
   ```

4. **Verify service**

   The overlay runs on [http://localhost:3000](http://localhost:3000)

   **Endpoints**:
   - `POST /submit` - Submit transactions
   - `POST /lookup` - Query tokens

---

## Summary

This project demonstrates:

- **Overlay Architecture (BRC-57)** - Specialized blockchain indexing nodes
- **Topic Managers** - Protocol validation and output filtering
- **Lookup Services** - Efficient UTXO indexing and querying
- **BEEF Parsing (BRC-62)** - Extracting transactions with SPV proofs
- **PushDrop Decoding (BRC-48)** - Reading token data from locking scripts
- **Balance Validation** - Enforcing fungible token conservation rules
- **MongoDB Integration** - Fast queries with composite and hashed indices
- **REST API** - Querying tokens by ID, outpoint, or listing all

These patterns form the foundation for building application-specific blockchain indexers that enforce custom protocol rules and provide fast data access.

---

## Related Resources

- [BRC-48: PushDrop Token Standard](https://hub.bsvblockchain.org/brc/scripts/0048)
- [BRC-57: Overlay Services Protocol](https://hub.bsvblockchain.org/brc/overlays/0088)
- [BRC-62: BEEF Transaction Format](https://hub.bsvblockchain.org/brc/transactions/0062)
- [BSV SDK Documentation](https://docs.bsvblockchain.org/)
- [Overlay Tools](https://github.com/bitcoin-sv/overlay-tools)
- [MongoDB Documentation](https://www.mongodb.com/docs/)
