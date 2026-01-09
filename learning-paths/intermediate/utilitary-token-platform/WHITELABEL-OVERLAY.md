# Whitelabel Token Overlay Service

Build a token validation overlay using BSV SDK's overlay architecture, Topic Managers, and Lookup Services.

---

## What You'll Build

- Transaction validation with balance conservation
- BEEF parsing (BRC-62) and PushDrop decoding (BRC-48)
- MongoDB UTXO indexing
- REST API for token queries

**Validation Rules:**
1. Mints (`'___mint___'`) create new supply
2. Transfers: `Σ inputs = Σ outputs`
3. Reject any balance mismatch

---

## Setup

```bash
mkdir my-token-overlay && cd my-token-overlay
npm init -y
npm install @bsv/sdk @bsv/overlay-express mongodb express dotenv
npm install -D typescript @types/node @types/express
```

Create `.env`:
```env
PORT=3000
MONGODB_URI=mongodb://localhost:27017
```

Create `tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "outDir": "./dist",
    "strict": true,
    "esModuleInterop": true
  }
}
```

> **BSV SDK Requirement**: Use ES2022+ for BigInt support in amount fields.

---

## 1. Topic Manager (Validator)

`src/MyTokenTopicManager.ts`:

```typescript
import { TopicManager, AdmittanceInstructions } from '@bsv/overlay-tools'
import { Transaction, PushDrop, Utils, OP } from '@bsv/sdk'

export class MyTokenTopicManager implements TopicManager {
  async identifyAdmissibleOutputs(beef: number[]): Promise<AdmittanceInstructions> {
    try {
      const tx = Transaction.fromBEEF(beef)
      const txid = tx.id('hex')

      // Track balances: Map<tokenId, { amount: number, isMint: boolean }>
      const balances = new Map<string, { amount: number, isMint: boolean }>()

      // Process inputs (credits)
      for (const input of tx.inputs) {
        if (!input.sourceTransaction) continue

        try {
          const sourceOutput = input.sourceTransaction.outputs[input.sourceOutputIndex]
          const token = PushDrop.decode(sourceOutput.lockingScript)

          let tokenId = Utils.toUTF8(token.fields[0])
          if (tokenId === '___mint___') {
            tokenId = `${input.sourceTransaction.id('hex')}.${input.sourceOutputIndex}`
          }

          const amount = Number(new Utils.Reader(token.fields[1]).readUInt64LEBn())

          const current = balances.get(tokenId) || { amount: 0, isMint: false }
          balances.set(tokenId, { amount: current.amount + amount, isMint: current.isMint })
        } catch { continue }
      }

      // Process outputs (debits)
      const outputsToAdmit: number[] = []

      for (let vout = 0; vout < tx.outputs.length; vout++) {
        const output = tx.outputs[vout]
        const chunks = output.lockingScript.chunks

        if (chunks.length < 2 || chunks[chunks.length - 1].op !== OP.OP_CHECKSIG) continue

        try {
          const token = PushDrop.decode(output.lockingScript)

          let tokenId = Utils.toUTF8(token.fields[0])
          if (tokenId === '___mint___') {
            tokenId = `${txid}.${vout}`
            balances.set(tokenId, { amount: 0, isMint: true })
          }

          const amount = Number(new Utils.Reader(token.fields[1]).readUInt64LEBn())

          const current = balances.get(tokenId) || { amount: 0, isMint: false }
          balances.set(tokenId, { amount: current.amount - amount, isMint: current.isMint })

          outputsToAdmit.push(vout)
        } catch { continue }
      }

      // Validate balance conservation
      for (const [tokenId, balance] of balances) {
        if (balance.isMint) continue  // Mints can create new supply
        if (balance.amount !== 0) {
          console.log(`❌ Balance mismatch for ${tokenId}: ${balance.amount}`)
          return { outputsToAdmit: [], coinsToRetain: [] }  // REJECT
        }
      }

      console.log(`✅ Admitted ${outputsToAdmit.length} outputs`)
      return { outputsToAdmit, coinsToRetain: [] }
    } catch (error) {
      console.error('Validation error:', error)
      return { outputsToAdmit: [], coinsToRetain: [] }
    }
  }

  getDocumentation(): string {
    return 'Validates fungible token balance conservation using PushDrop (BRC-48)'
  }

  getMetaData() {
    return {
      name: 'tm_mytoken',
      shortDescription: 'My Token Validator',
      version: '1.0.0'
    }
  }
}
```

**Balance Accounting:**
```
Inputs:  +amount (credit)
Outputs: -amount (debit)
Result:  must equal 0 (except mints)
```

> **Reference**: [BRC-48: PushDrop](https://hub.bsvblockchain.org/brc/scripts/0048), [BRC-88: Overlay Services](https://hub.bsvblockchain.org/brc/overlays/0088)

---

## 2. MongoDB Storage

`src/TokenStorage.ts`:

```typescript
import { MongoClient, Collection } from 'mongodb'

interface TokenRecord {
  txid: string
  outputIndex: number
  tokenId: string
  amount: string
  metadata: any
  createdAt: Date
}

export class TokenStorage {
  private collection: Collection<TokenRecord> | null = null

  async initialize(uri: string) {
    const client = await MongoClient.connect(uri)
    this.collection = client.db().collection('tokens')

    // Create indices
    await this.collection.createIndex({ txid: 1, outputIndex: 1 }, { unique: true })
    await this.collection.createIndex({ tokenId: 'hashed' })
    await this.collection.createIndex({ createdAt: -1 })
  }

  async storeRecord(txid: string, outputIndex: number, tokenId: string, amount: string, metadata: any) {
    await this.collection!.insertOne({ txid, outputIndex, tokenId, amount, metadata, createdAt: new Date() })
  }

  async findByTokenId(tokenId: string, limit = 50, skip = 0) {
    return await this.collection!.find({ tokenId }).sort({ createdAt: -1 }).skip(skip).limit(limit).toArray()
  }

  async findByOutpoint(txid: string, vout: number) {
    return await this.collection!.findOne({ txid, outputIndex: vout })
  }
}
```

**Index Strategy:**
- Composite `{txid, outputIndex}` - Fast outpoint lookups
- Hashed `{tokenId}` - Efficient token searches
- Timestamp `{createdAt}` - Sorted queries

---

## 3. Lookup Service (Indexer)

`src/MyTokenLookupService.ts`:

```typescript
import { LookupService, LookupQuestion, LookupAnswer } from '@bsv/overlay-tools'
import { PushDrop, LockingScript, Utils } from '@bsv/sdk'
import { TokenStorage } from './TokenStorage'

export class MyTokenLookupService implements LookupService {
  constructor(private storage: TokenStorage) {}

  async outputAdded(payload: { txid: string, outputIndex: number, outputScript: { script: string } }) {
    try {
      const script = LockingScript.fromHex(payload.outputScript.script)
      const token = PushDrop.decode(script)

      let tokenId = Utils.toUTF8(token.fields[0])
      if (tokenId === '___mint___') tokenId = `${payload.txid}.${payload.outputIndex}`

      const amount = new Utils.Reader(token.fields[1]).readUInt64LEBn().toString()
      const metadata = JSON.parse(Utils.toUTF8(token.fields[2]))

      await this.storage.storeRecord(payload.txid, payload.outputIndex, tokenId, amount, metadata)
      console.log(`📝 Indexed: ${tokenId}`)
    } catch (error) {
      console.error('Indexing error:', error)
    }
  }

  async lookup(question: LookupQuestion): Promise<LookupAnswer> {
    const query = question.query as any

    if (query.outpoint) {
      const [txid, vout] = query.outpoint.split('.')
      const record = await this.storage.findByOutpoint(txid, parseInt(vout))
      return { type: 'output-list', outputs: record ? [record] : [] }
    }

    if (query.tokenId) {
      const records = await this.storage.findByTokenId(query.tokenId, query.limit, query.skip)
      return { type: 'output-list', outputs: records }
    }

    return { type: 'output-list', outputs: [] }
  }

  getDocumentation() { return 'Query tokens by outpoint or tokenId' }
  getMetaData() { return { name: 'ls_mytoken', shortDescription: 'Token Lookup', version: '1.0.0' } }
}
```

**Lifecycle:**
1. Topic Manager admits output → `outputAdded()` called
2. Extract token data → Store in MongoDB
3. Client queries → `lookup()` returns results

---

## 4. Main Server

`src/index.ts`:

```typescript
import express from 'express'
import { config } from 'dotenv'
import { OverlayExpressNode } from '@bsv/overlay-express'
import { MyTokenTopicManager } from './MyTokenTopicManager'
import { MyTokenLookupService } from './MyTokenLookupService'
import { TokenStorage } from './TokenStorage'

config()

async function main() {
  const app = express()
  const storage = new TokenStorage()
  await storage.initialize(process.env.MONGODB_URI!)

  new OverlayExpressNode({
    port: parseInt(process.env.PORT!),
    app,
    nodeImpl: {
      getTopicManagers: () => [new MyTokenTopicManager()],
      getLookupServiceFactory: () => ({ getLookupService: () => new MyTokenLookupService(storage) })
    }
  })

  app.listen(process.env.PORT, () => {
    console.log(`✅ Overlay running on http://localhost:${process.env.PORT}`)
    console.log(`📡 POST /submit - Submit transactions`)
    console.log(`🔍 POST /lookup - Query tokens`)
  })
}

main()
```

---

## 5. SPV Verification (Optional)

Add ChainTracker for merkle proof validation:

```typescript
import { ChaintracksChainTracker } from '@bsv/sdk'

const chainTracker = new ChaintracksChainTracker('main')  // or 'test'
server.configureChainTracker(chainTracker)
```

**Purpose**: Validates merkle paths in BEEF packages without full node.

> **Reference**: [BRC-9: SPV](https://hub.bsvblockchain.org/brc/transactions/0009)

---

## Testing

**Start MongoDB:**
```bash
mongod --dbpath ./data
```

**Start Overlay:**
```bash
npm run build && npm start
```

**Submit Transaction (from client):**
```typescript
import { HTTPSOverlayBroadcastFacilitator } from '@bsv/sdk'

const overlay = new HTTPSOverlayBroadcastFacilitator()
const response = await overlay.send('http://localhost:3000/submit', {
  beef: tx.toBEEF(),
  topics: ['tm_mytoken']
})
// Returns: { tm_mytoken: { outputsToAdmit: [0, 1], coinsToRetain: [] } }
```

**Query Tokens:**
```bash
curl -X POST http://localhost:3000/lookup -H "Content-Type: application/json" \
  -d '{"service":"ls_mytoken","query":{"tokenId":"abc.0","limit":50}}'
```

---

## Key Concepts

**Topic Manager vs Lookup Service:**
- **Topic Manager** - Validates before admission (gatekeeper)
- **Lookup Service** - Indexes after admission (librarian)

**Balance Conservation:**
```
Valid:   Input +1000 → Outputs -700, -300 = 0 ✅
Invalid: Input +1000 → Output -1200 = -200 ❌
Mint:    No inputs → Output +1000 (isMint=true) ✅
```

---

## Enhancements

**1. Spend Tracking:**
```typescript
async outputSpent(payload: { txid: string, outputIndex: number }) {
  await this.storage.markAsSpent(payload.txid, payload.outputIndex)
}
```

**2. Metadata Validation:**
```typescript
const metadata = JSON.parse(Utils.toUTF8(token.fields[2]))
if (!metadata.label) return { outputsToAdmit: [], coinsToRetain: [] }
```

**3. Token Registry:**
```typescript
const allowedTokens = ['credits', 'points']
if (!allowedTokens.includes(label)) return { outputsToAdmit: [], coinsToRetain: [] }
```

---

## Resources

- [BRC-88: Overlay Services](https://hub.bsvblockchain.org/brc/overlays/0088)
- [BRC-48: PushDrop](https://hub.bsvblockchain.org/brc/scripts/0048)
- [BRC-62: BEEF](https://hub.bsvblockchain.org/brc/transactions/0062)
- [BRC-9: SPV](https://hub.bsvblockchain.org/brc/transactions/0009)
- [Overlay Express](https://github.com/bitcoin-sv/overlay-express)
