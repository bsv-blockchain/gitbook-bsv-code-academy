# Overlay Networks

## Overview

Overlay networks are application-specific networks built on top of the BSV blockchain. They provide efficient indexing, querying, and state management for specific use cases while maintaining the security guarantees of the base layer.

## What is an Overlay Network?

An overlay network is a layer that:
- **Filters** relevant transactions from the blockchain
- **Indexes** data for efficient querying
- **Maintains** application-specific state
- **Provides** APIs for application access
- **Ensures** consistency with base layer

Think of overlays as specialized views of the blockchain optimized for specific applications.

## Why Use Overlays?

### Performance
- Fast queries without scanning entire blockchain
- Application-specific indexing
- Efficient data retrieval
- Reduced bandwidth requirements

### Scalability
- Horizontal scaling of overlay services
- Parallel processing of transactions
- Cached state for instant access
- Partitioning by application domain

### Functionality
- Complex queries beyond base protocol
- Real-time notifications
- Historical data analytics
- Cross-transaction relationships

## Overlay Architecture

```
┌─────────────────────────────────────────┐
│         Application Layer               │
│  (Wallets, Exchanges, Social Apps)      │
└─────────────────────────────────────────┘
                  ↕ API
┌─────────────────────────────────────────┐
│         Overlay Service Layer           │
│  (Indexing, State, Business Logic)      │
└─────────────────────────────────────────┘
                  ↕ Filter
┌─────────────────────────────────────────┐
│         BSV Blockchain Layer            │
│  (Transactions, Blocks, Consensus)      │
└─────────────────────────────────────────┘
```

## Core Concepts

### 1. Transaction Tagging

Mark transactions as part of your overlay:

```typescript
interface OverlayTransaction {
  // Standard BSV transaction
  transaction: Transaction

  // Overlay identification
  overlayProtocol: string  // e.g., "token-v1"

  // Application data (often in OP_RETURN)
  metadata: {
    action: string
    data: Record<string, any>
  }
}

// Create overlay transaction
function createOverlayTx(
  protocol: string,
  action: string,
  data: any
): Transaction {
  const tx = new Transaction()

  // Add inputs (from UTXO)
  // ...

  // Add payment outputs
  // ...

  // Add overlay metadata in OP_RETURN
  const metadata = {
    protocol,
    action,
    data
  }

  tx.addOutput({
    satoshis: 0,
    lockingScript: Script.fromASM(`OP_FALSE OP_RETURN ${JSON.stringify(metadata)}`)
  })

  return tx
}
```

### 2. State Management

Maintain application state from UTXO set:

```typescript
interface TokenState {
  tokenId: string
  utxos: Map<string, {
    txid: string
    vout: number
    owner: string
    amount: number
  }>
  totalSupply: number
  holders: Map<string, number>
}

class TokenOverlay {
  private state: TokenState

  async processTransaction(tx: Transaction) {
    // Parse transaction
    const action = this.parseAction(tx)

    switch (action.type) {
      case 'mint':
        this.handleMint(action)
        break
      case 'transfer':
        this.handleTransfer(action)
        break
      case 'burn':
        this.handleBurn(action)
        break
    }
  }

  private handleTransfer(action: any) {
    // Update UTXO set
    // Remove spent inputs
    action.inputs.forEach(input => {
      this.state.utxos.delete(`${input.txid}:${input.vout}`)
    })

    // Add new outputs
    action.outputs.forEach((output, index) => {
      this.state.utxos.set(`${action.txid}:${index}`, {
        txid: action.txid,
        vout: index,
        owner: output.owner,
        amount: output.amount
      })
    })

    // Update holder balances
    this.recalculateBalances()
  }

  async getBalance(address: string): Promise<number> {
    return this.state.holders.get(address) || 0
  }

  async getUTXOs(address: string) {
    return Array.from(this.state.utxos.values())
      .filter(utxo => utxo.owner === address)
  }
}
```

### 3. Indexing Strategy

Efficient data indexing:

```typescript
interface OverlayIndex {
  // By transaction ID
  txIndex: Map<string, OverlayTransaction>

  // By address
  addressIndex: Map<string, Set<string>>  // address -> txids

  // By timestamp
  timeIndex: Map<number, Set<string>>     // block height -> txids

  // By action type
  actionIndex: Map<string, Set<string>>   // action -> txids
}

class OverlayIndexer {
  private index: OverlayIndex = {
    txIndex: new Map(),
    addressIndex: new Map(),
    timeIndex: new Map(),
    actionIndex: new Map()
  }

  async indexTransaction(
    tx: OverlayTransaction,
    blockHeight: number
  ) {
    const txid = tx.transaction.id('hex')

    // Index by txid
    this.index.txIndex.set(txid, tx)

    // Index by addresses involved
    const addresses = this.extractAddresses(tx)
    addresses.forEach(addr => {
      if (!this.index.addressIndex.has(addr)) {
        this.index.addressIndex.set(addr, new Set())
      }
      this.index.addressIndex.get(addr)!.add(txid)
    })

    // Index by time
    if (!this.index.timeIndex.has(blockHeight)) {
      this.index.timeIndex.set(blockHeight, new Set())
    }
    this.index.timeIndex.get(blockHeight)!.add(txid)

    // Index by action
    const action = tx.metadata.action
    if (!this.index.actionIndex.has(action)) {
      this.index.actionIndex.set(action, new Set())
    }
    this.index.actionIndex.get(action)!.add(txid)
  }

  async queryByAddress(address: string): Promise<OverlayTransaction[]> {
    const txids = this.index.addressIndex.get(address) || new Set()
    return Array.from(txids).map(txid =>
      this.index.txIndex.get(txid)!
    )
  }

  async queryByTimeRange(
    startHeight: number,
    endHeight: number
  ): Promise<OverlayTransaction[]> {
    const results: OverlayTransaction[] = []

    for (let height = startHeight; height <= endHeight; height++) {
      const txids = this.index.timeIndex.get(height)
      if (txids) {
        txids.forEach(txid => {
          results.push(this.index.txIndex.get(txid)!)
        })
      }
    }

    return results
  }
}
```

## Building an Overlay Service

### Complete Token Overlay Example

```typescript
interface TokenConfig {
  tokenId: string
  name: string
  symbol: string
  decimals: number
}

class TokenOverlayService {
  private config: TokenConfig
  private state: TokenState
  private indexer: OverlayIndexer
  private chainTracker: ChainTracker

  constructor(config: TokenConfig) {
    this.config = config
    this.state = this.initializeState()
    this.indexer = new OverlayIndexer()
    this.chainTracker = new ChainTracker()
  }

  // Start monitoring blockchain
  async start() {
    // Get current block height
    const height = await this.chainTracker.getCurrentHeight()

    // Subscribe to new blocks
    this.chainTracker.on('block', async (block) => {
      await this.processBlock(block)
    })

    // Subscribe to mempool
    this.chainTracker.on('mempool', async (tx) => {
      await this.processMempoolTx(tx)
    })
  }

  private async processBlock(block: Block) {
    // Filter transactions for this overlay
    const overlayTxs = block.transactions.filter(tx =>
      this.isOverlayTransaction(tx)
    )

    // Process each transaction
    for (const tx of overlayTxs) {
      await this.processTransaction(tx, block.height)
    }

    // Emit events
    this.emit('block-processed', {
      height: block.height,
      txCount: overlayTxs.length
    })
  }

  private isOverlayTransaction(tx: Transaction): boolean {
    // Check for overlay protocol marker
    const outputs = tx.outputs

    for (const output of outputs) {
      if (output.lockingScript.isOpReturn()) {
        const data = this.parseOpReturn(output.lockingScript)
        if (data?.protocol === this.config.tokenId) {
          return true
        }
      }
    }

    return false
  }

  // API Methods
  async getBalance(address: string): Promise<number> {
    return this.state.holders.get(address) || 0
  }

  async getTransactionHistory(
    address: string,
    limit = 50
  ): Promise<OverlayTransaction[]> {
    const txs = await this.indexer.queryByAddress(address)
    return txs.slice(0, limit)
  }

  async getTotalSupply(): Promise<number> {
    return this.state.totalSupply
  }

  async getHolders(): Promise<Array<{ address: string, balance: number }>> {
    return Array.from(this.state.holders.entries()).map(
      ([address, balance]) => ({ address, balance })
    )
  }

  // Create transfer transaction
  async createTransfer(
    from: PrivateKey,
    to: string,
    amount: number
  ): Promise<Transaction> {
    const fromAddr = from.toPublicKey().toAddress()

    // Get UTXOs for this address
    const utxos = await this.getUTXOs(fromAddr)

    // Select UTXOs
    const selected = this.selectUTXOs(utxos, amount)

    // Build transaction
    const tx = new Transaction()

    // Add inputs
    for (const utxo of selected) {
      await tx.addInput({
        sourceTransaction: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(from)
      })
    }

    // Add token transfer output
    tx.addOutput({
      satoshis: 1000,
      lockingScript: new P2PKH().lock(to)
    })

    // Add metadata
    const metadata = {
      protocol: this.config.tokenId,
      action: 'transfer',
      amount: amount
    }

    tx.addOutput({
      satoshis: 0,
      lockingScript: Script.fromASM(
        `OP_FALSE OP_RETURN ${JSON.stringify(metadata)}`
      )
    })

    // Add change if needed
    const inputAmount = selected.reduce((sum, u) => sum + u.amount, 0)
    if (inputAmount > amount) {
      const changeAmount = inputAmount - amount
      tx.addOutput({
        satoshis: 1000,
        lockingScript: new P2PKH().lock(fromAddr)
      })

      // Change metadata
      tx.addOutput({
        satoshis: 0,
        lockingScript: Script.fromASM(
          `OP_FALSE OP_RETURN ${JSON.stringify({
            protocol: this.config.tokenId,
            action: 'change',
            amount: changeAmount
          })}`
        )
      })
    }

    await tx.sign()
    return tx
  }
}
```

## Overlay Discovery

Enable clients to find overlay services:

```typescript
interface OverlayInfo {
  protocol: string
  version: string
  endpoints: {
    api: string
    websocket?: string
  }
  capabilities: string[]
  documentation: string
}

// Registry of available overlays
class OverlayRegistry {
  private overlays: Map<string, OverlayInfo> = new Map()

  register(info: OverlayInfo) {
    this.overlays.set(info.protocol, info)
  }

  discover(protocol: string): OverlayInfo | undefined {
    return this.overlays.get(protocol)
  }

  list(): OverlayInfo[] {
    return Array.from(this.overlays.values())
  }
}
```

## Real-World Overlay Examples

### 1. Token Overlay
- Track token balances and transfers
- Provide token history
- Calculate holder statistics

### 2. Social Media Overlay
- Index posts and comments
- Build follower graphs
- Rank content by engagement

### 3. Supply Chain Overlay
- Track product movements
- Verify authenticity
- Maintain custody chain

### 4. Identity Overlay
- Manage certificates
- Verify credentials
- Track reputation

### 5. Data Marketplace Overlay
- List available datasets
- Track purchases
- Manage access rights

## Overlay Interoperability

Enable overlays to interact:

```typescript
interface CrossOverlayMessage {
  sourceOverlay: string
  targetOverlay: string
  message: any
  txid: string
}

// Bridge between overlays
class OverlayBridge {
  async sendMessage(
    from: string,
    to: string,
    message: any
  ): Promise<Transaction> {
    const tx = new Transaction()

    // Create cross-overlay message
    const crossOverlayMsg: CrossOverlayMessage = {
      sourceOverlay: from,
      targetOverlay: to,
      message,
      txid: '' // Will be filled after creation
    }

    // Add to OP_RETURN
    tx.addOutput({
      satoshis: 0,
      lockingScript: Script.fromASM(
        `OP_FALSE OP_RETURN ${JSON.stringify(crossOverlayMsg)}`
      )
    })

    return tx
  }
}
```

## Performance Optimization

### Caching Strategy
```typescript
class OverlayCache {
  private redis: RedisClient

  async getBalance(address: string): Promise<number | null> {
    const cached = await this.redis.get(`balance:${address}`)
    return cached ? parseInt(cached) : null
  }

  async setBalance(address: string, balance: number) {
    await this.redis.set(`balance:${address}`, balance.toString())
    await this.redis.expire(`balance:${address}`, 3600) // 1 hour
  }
}
```

### Database Design
```sql
-- Transactions table
CREATE TABLE overlay_transactions (
  txid VARCHAR(64) PRIMARY KEY,
  block_height INTEGER,
  action VARCHAR(50),
  data JSONB,
  created_at TIMESTAMP
);

-- Address index
CREATE INDEX idx_addresses ON overlay_transactions
USING GIN ((data->'addresses'));

-- Time index
CREATE INDEX idx_block_height ON overlay_transactions (block_height);

-- Action index
CREATE INDEX idx_action ON overlay_transactions (action);
```

## Testing Overlays

```typescript
describe('Token Overlay', () => {
  let overlay: TokenOverlayService

  beforeEach(() => {
    overlay = new TokenOverlayService({
      tokenId: 'TEST-TOKEN',
      name: 'Test Token',
      symbol: 'TST',
      decimals: 8
    })
  })

  it('should track token transfers', async () => {
    const tx = await overlay.createTransfer(
      aliceKey,
      bobAddress,
      100
    )

    await overlay.processTransaction(tx, 1000)

    const aliceBalance = await overlay.getBalance(aliceAddress)
    const bobBalance = await overlay.getBalance(bobAddress)

    expect(bobBalance).toBe(100)
  })

  it('should index by address', async () => {
    const history = await overlay.getTransactionHistory(aliceAddress)
    expect(history.length).toBeGreaterThan(0)
  })
})
```

## Related Components

- [Transaction](../../../sdk-components/transaction/README.md)
- [SPV](../../../sdk-components/spv/README.md)

## Next Module

Continue to: [Network Topology](../network-topology/README.md) to learn about P2P networking.

## Summary

Overlay networks enable efficient, scalable applications on BSV by:
- ✅ Filtering relevant transactions
- ✅ Maintaining application state
- ✅ Providing fast queries
- ✅ Supporting complex interactions
- ✅ Scaling horizontally

Master overlays to build production-grade BSV applications!
