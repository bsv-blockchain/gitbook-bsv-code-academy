# Network Topology

## Overview

Understand the architecture and topology of the BSV blockchain network, from legacy node implementations to the modern Teranode microservices architecture. This module covers peer-to-peer networking, node types, network layers, consensus mechanisms, and the revolutionary horizontal scaling approach of Teranode.

**Estimated Time:** 4-5 hours
**Difficulty:** Advanced
**Prerequisites:** Complete [BSV Fundamentals](../../beginner/bsv-fundamentals/README.md), [Transaction Building](../../intermediate/transaction-building/README.md), and [SPV Verification](../../intermediate/spv-verification/README.md)

## Learning Objectives

By the end of this module, you will be able to:

- ✅ Understand the BSV network architecture and P2P protocol
- ✅ Differentiate between mining nodes, archival nodes, and SPV clients
- ✅ Comprehend the evolution from legacy nodes to Teranode
- ✅ Understand Teranode's microservices architecture
- ✅ Implement network communication patterns
- ✅ Design scalable blockchain infrastructure
- ✅ Understand horizontal vs vertical scaling
- ✅ Work with ARC network layer for transaction broadcasting

## SDK Components Used

This course leverages these standardized SDK modules:

- **[Transaction](../../../sdk-components/transaction/README.md)** - Network transaction propagation
- **[ARC](../../../sdk-components/arc/README.md)** - ARC API for broadcasts
- **[SPV](../../../sdk-components/spv/README.md)** - SPV network clients
- **[BEEF](../../../sdk-components/beef/README.md)** - Transaction envelope format
- **[Merkle Proofs](../../../sdk-components/merkle-proofs/README.md)** - Block verification

## 1. BSV Network Architecture Overview

### Network Layers

```
┌─────────────────────────────────────────────────────┐
│              Application Layer                       │
│  (Wallets, DApps, Services, Overlays)              │
└─────────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────────┐
│           Transaction Layer (ARC)                    │
│  (Transaction Broadcasting, Validation, Routing)    │
└─────────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────────┐
│              Node Layer                              │
│  Legacy Nodes          ←→         Teranode          │
│  (Monolithic)                (Microservices)        │
└─────────────────────────────────────────────────────┘
                       ↕
┌─────────────────────────────────────────────────────┐
│           P2P Network Layer                          │
│  (Peer Discovery, Block Propagation, Consensus)    │
└─────────────────────────────────────────────────────┘
```

### Node Types

**1. Mining Nodes**
- Full transaction validation
- Block template creation
- Proof-of-work computation
- Block propagation
- Highest computational requirements

**2. Archival Nodes (Full Nodes)**
- Full blockchain history
- Transaction validation
- Block relay
- RPC services for queries
- Large storage requirements

**3. SPV Clients**
- Block headers only
- Merkle proof verification
- Lightweight, mobile-friendly
- Minimal storage requirements

**4. ARC Nodes (Transaction Processors)**
- Transaction receipt and validation
- Fee calculation
- Double-spend detection
- Status tracking
- High throughput focus

## 2. Legacy Node Architecture

### Monolithic Design

Legacy BSV nodes (based on Bitcoin Core) use a monolithic architecture:

```
┌────────────────────────────────────┐
│       Legacy BSV Node              │
├────────────────────────────────────┤
│  • Single Process                  │
│  • Vertically Scaled               │
│  • Integrated Components:          │
│    - P2P Networking                │
│    - Transaction Pool (Mempool)    │
│    - Validation Engine             │
│    - Block Assembly                │
│    - Storage (LevelDB)             │
│    - RPC Server                    │
│    - Wallet (optional)             │
└────────────────────────────────────┘
```

### Limitations of Legacy Architecture

1. **Vertical Scaling Only**: Limited by single-machine resources
2. **Monolithic Bottleneck**: All functions in one process
3. **Limited Throughput**: Typically 1,000-10,000 tx/sec
4. **Resource Intensive**: High CPU, memory, disk I/O on single machine
5. **Difficult to Maintain**: Tightly coupled components

### Legacy Node Communication

```typescript
// Conceptual P2P communication pattern (not actual SDK code)

interface P2PMessage {
  command: string  // 'tx', 'block', 'inv', 'getdata', etc.
  payload: Buffer
}

// Legacy nodes communicate via Bitcoin P2P protocol
// Message types:
// - version/verack: Handshake
// - inv: Inventory announcement
// - getdata: Request data
// - tx: Transaction
// - block: Full block
// - headers: Block headers
```

## 3. Teranode: Modern Microservices Architecture

### What is Teranode?

**Teranode** is BSV's next-generation blockchain infrastructure, announced by the BSV Association in October 2024 after three years of development. It represents a fundamental re-architecture of blockchain node software.

**Key Innovation**: Microservices-based, horizontally scalable architecture capable of processing **over 1 million transactions per second**.

### Teranode Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Teranode Ecosystem                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────┐   ┌─────────────────────────┐        │
│  │  Core Services   │   │   Overlay Services      │        │
│  ├──────────────────┤   ├─────────────────────────┤        │
│  │ • TX Validation  │   │ • Block Persistence     │        │
│  │ • TX Propagation │   │ • UTXO Persistence      │        │
│  │ • Block Assembly │   │ • P2P Networking        │        │
│  │ • Block Validation│   │ • RPC Interface         │        │
│  │ • State Mgmt     │   │ • Legacy Compatibility  │        │
│  │ • Asset Server   │   └─────────────────────────┘        │
│  └──────────────────┘                                       │
│                                                              │
│  ┌──────────────────────────────────────────────────┐      │
│  │        Infrastructure Components                 │      │
│  ├──────────────────────────────────────────────────┤      │
│  │ • Kafka Event Messaging                          │      │
│  │ • Blob Storage (Transactions, Blocks)            │      │
│  │ • UTXO Data Stores                               │      │
│  │ • Monitoring & Observability                     │      │
│  └──────────────────────────────────────────────────┘      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Core Teranode Principles

**1. Horizontal Scalability**
- Add more service instances to increase capacity
- No single machine bottleneck
- Linear performance scaling

**2. Microservices Independence**
- Each service is independently deployable
- Services communicate via message queues (Kafka)
- Fault isolation between services

**3. Unbounded Block Sizes**
- No artificial block size limits
- Throughput scales with infrastructure
- True Satoshi Vision scaling

**4. State Management**
- Distributed UTXO set
- Two-phase commit protocol
- Consistency guarantees

### Horizontal vs Vertical Scaling

```
Legacy (Vertical Scaling):
┌─────────────┐
│  1 Big Node │  ← Limited by single machine
│  10K tx/s   │
└─────────────┘

Teranode (Horizontal Scaling):
┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
│Node 1│  │Node 2│  │Node 3│  │Node 4│  │Node 5│
│200K  │  │200K  │  │200K  │  │200K  │  │200K  │
└──────┘  └──────┘  └──────┘  └──────┘  └──────┘
           = 1,000,000+ tx/s total
           (Add more nodes → more capacity)
```

## 4. Teranode Transaction Lifecycle

### Transaction Flow

```
1. Application → ARC
   ↓
2. ARC → Transaction Validation Service
   ↓
3. Validation → Event Queue (Kafka)
   ↓
4. Event Queue → Transaction Propagation Service
   ↓
5. Propagation → Other Teranode Instances
   ↓
6. Multiple Nodes → Subtree Validation
   ↓
7. Validation Complete → Block Assembly Service
   ↓
8. Block Assembly → Mining Pool
   ↓
9. Mining Pool → Proof of Work
   ↓
10. Block Found → Block Propagation Service
   ↓
11. Block Propagation → Network
   ↓
12. Nodes → Block Validation & Confirmation
```

### Two-Phase Commit

Teranode uses a two-phase commit protocol for state consistency:

**Phase 1: Prepare**
- Transaction validated
- UTXO locks acquired
- State changes prepared
- All services vote: COMMIT or ABORT

**Phase 2: Commit**
- If all vote COMMIT → Apply changes
- If any vote ABORT → Rollback changes
- Ensures atomic transactions

## 5. ARC: Transaction Broadcasting Layer

### ARC Architecture

**ARC (Application Request Controller)** is the modern transaction broadcasting layer for BSV, designed to work with both legacy nodes and Teranode.

Reference: **[ARC Component](../../../sdk-components/arc/README.md)**

```typescript
import { ARC, Transaction } from '@bsv/sdk'

// Connect to ARC endpoint
const arc = new ARC({
  apiKey: 'your-api-key',
  deploymentId: 'your-deployment-id',
  callbackUrl: 'https://your-app.com/tx-callback'
})

// Broadcast transaction
async function broadcastToNetwork(tx: Transaction): Promise<string> {
  try {
    const response = await arc.broadcastTransaction(tx)

    console.log('Transaction Status:', response.status)
    // Possible statuses:
    // - SEEN: Received by ARC
    // - QUEUED: Validated, awaiting mining
    // - MINED: Included in block
    // - CONFIRMED: Block confirmed
    // - REJECTED: Invalid transaction

    return response.txid
  } catch (error) {
    console.error('Broadcast failed:', error)
    throw error
  }
}

// Query transaction status
async function checkTransactionStatus(txid: string): Promise<string> {
  const status = await arc.getTransactionStatus(txid)
  return status.blockStatus // SEEN, QUEUED, MINED, CONFIRMED, REJECTED
}

// Get fee quotes
async function getFeeQuote(): Promise<FeeQuote> {
  const quote = await arc.getFeeQuote()

  console.log('Standard fee:', quote.fees.standard)
  console.log('Data fee:', quote.fees.data)
  console.log('Mining fee:', quote.fees.mining)

  return quote
}
```

### ARC Network Benefits

1. **Separation of Concerns**: Apps don't connect directly to nodes
2. **Load Balancing**: ARC handles routing to multiple nodes
3. **Status Tracking**: Real-time transaction lifecycle monitoring
4. **Fee Management**: Dynamic fee calculation
5. **Double-Spend Detection**: Immediate alerts
6. **Callbacks**: Asynchronous notification system

## 6. P2P Network Communication

### Peer Discovery

BSV nodes discover peers through:

1. **DNS Seeds**: Hard-coded DNS addresses that return peer IPs
2. **Peer Exchange**: Nodes share known peer addresses
3. **Manual Configuration**: Static peer list in config

### Network Messages

**Core P2P Message Types:**

```
Message Type    Purpose
-----------     -------
version         Handshake - Node capabilities
verack          Handshake acknowledgment
ping/pong       Keep-alive
inv             Announce available data
getdata         Request specific data
tx              Transaction data
block           Full block data
headers         Block header chain
getblocks       Request block inventory
getaddr         Request peer addresses
addr            Send peer addresses
mempool         Request memory pool contents
reject          Reject message/transaction
```

### Block Propagation

**Compact Blocks** (optimized propagation):

1. Miner finds block
2. Sends compact block header + short transaction IDs
3. Receiving nodes already have most transactions in mempool
4. Only missing transactions requested
5. Faster block propagation across network

```
Traditional Block Propagation:
Block (1MB) → Network → 30 seconds

Compact Block Propagation:
Header + Short IDs (10KB) → Network → 1 second
Missing TXs → Network → 5 seconds
Total: 6 seconds (5x faster)
```

## 7. Network Design Patterns

### Pattern 1: Direct Node Connection (Legacy)

```typescript
// Conceptual pattern - requires P2P protocol implementation

interface LegacyNodeConnection {
  connect(host: string, port: number): Promise<void>
  sendTransaction(tx: Transaction): Promise<void>
  subscribeBlocks(callback: (block: Block) => void): void
  disconnect(): void
}

class LegacyP2PClient implements LegacyNodeConnection {
  private socket: Socket

  async connect(host: string, port: number) {
    this.socket = new Socket()
    await this.socket.connect(port, host)

    // Send version handshake
    await this.sendVersion()
    await this.receiveVerack()
  }

  async sendTransaction(tx: Transaction) {
    const message = {
      command: 'tx',
      payload: tx.toBinary()
    }
    await this.socket.write(serialize(message))
  }

  // ... more P2P protocol implementation
}
```

### Pattern 2: ARC Integration (Modern)

```typescript
import { ARC, Transaction } from '@bsv/sdk'

class ModernNetworkClient {
  private arc: ARC

  constructor(config: ARCConfig) {
    this.arc = new ARC(config)
  }

  async broadcastTransaction(tx: Transaction): Promise<TxReceipt> {
    // ARC handles network distribution
    const receipt = await this.arc.broadcastTransaction(tx)

    return {
      txid: receipt.txid,
      status: receipt.status,
      timestamp: receipt.timestamp
    }
  }

  async monitorTransaction(txid: string): Promise<TxStatus> {
    // Poll or use webhooks
    return await this.arc.getTransactionStatus(txid)
  }

  subscribeToCallbacks(webhook: string) {
    // Set up webhook for async notifications
    this.arc.setCallbackUrl(webhook)
  }
}
```

### Pattern 3: SPV Client (Lightweight)

```typescript
import { ChainTracker, MerklePath } from '@bsv/sdk'

class SPVNetworkClient {
  private chainTracker: ChainTracker

  constructor(chainTracker: ChainTracker) {
    this.chainTracker = chainTracker
  }

  async verifyTransaction(
    txid: string,
    merklePath: MerklePath
  ): Promise<boolean> {
    // Verify without downloading full blocks
    const isValid = await merklePath.verify(txid, this.chainTracker)
    return isValid
  }

  async syncHeaders(): Promise<void> {
    // Download only block headers (80 bytes each)
    const headers = await this.chainTracker.getHeaders()
    // Lightweight sync
  }
}
```

## 8. Teranode Deployment Architecture

### Kubernetes Deployment

Teranode is designed for cloud-native deployment:

```yaml
# Conceptual Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: teranode-tx-validation
spec:
  replicas: 10  # Horizontal scaling
  selector:
    matchLabels:
      app: tx-validation
  template:
    metadata:
      labels:
        app: tx-validation
    spec:
      containers:
      - name: validator
        image: teranode/tx-validator:latest
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
          limits:
            memory: "8Gi"
            cpu: "4"
        env:
        - name: KAFKA_BROKERS
          value: "kafka:9092"
        - name: UTXO_DB_CONNECTION
          value: "postgres://utxo-db:5432"
```

### Service Orchestration

```
Load Balancer
      ↓
API Gateway (ARC)
      ↓
┌─────────┬─────────┬─────────┬─────────┐
│ TX Val 1│ TX Val 2│ TX Val 3│ TX Val 4│
└────┬────┴────┬────┴────┬────┴────┬────┘
     │         │         │         │
     └─────────┴────┬────┴─────────┘
                    ↓
              Kafka Event Bus
                    ↓
     ┌──────────────┼──────────────┐
     │              │              │
     ↓              ↓              ↓
Block Assy 1   Block Assy 2   Block Assy 3
     │              │              │
     └──────────────┴──────────────┘
                    ↓
              UTXO Database
```

## 9. Network Performance Considerations

### Legacy Node Performance

- **Throughput**: 1,000 - 10,000 tx/sec (single node)
- **Latency**: 1-5 seconds (mempool → propagation)
- **Scaling**: Vertical only (upgrade hardware)
- **Cost**: Expensive for high performance

### Teranode Performance

- **Throughput**: 1,000,000+ tx/sec (demonstrated)
- **Theoretical Max**: 6-10 million tx/sec with current architecture
- **Latency**: Sub-second for validation
- **Scaling**: Horizontal (add more services)
- **Cost**: Efficient scaling with commodity hardware

### Real-World Test Results

From BSV Association's live trials (October 2024):

- **Duration**: 2 weeks continuous testing
- **Network**: Globally distributed
- **Result**: Consistently exceeded 1 million tx/sec
- **Stability**: Network remained stable throughout
- **Block Sizes**: Unbounded, handled multi-GB blocks

## 10. Building Network-Aware Applications

### Best Practices

**1. Use ARC Layer**
```typescript
// ✅ GOOD - Use ARC abstraction
const arc = new ARC(config)
await arc.broadcastTransaction(tx)

// ❌ BAD - Direct P2P connection
const socket = connect(nodeIP, 8333)
socket.write(txData)
```

**2. Handle Network Delays**
```typescript
async function reliableBroadcast(tx: Transaction): Promise<string> {
  const maxRetries = 3
  let attempt = 0

  while (attempt < maxRetries) {
    try {
      const result = await arc.broadcastTransaction(tx)

      if (result.status === 'REJECTED') {
        throw new Error(`Rejected: ${result.reason}`)
      }

      return result.txid
    } catch (error) {
      attempt++
      if (attempt >= maxRetries) throw error

      // Exponential backoff
      await sleep(Math.pow(2, attempt) * 1000)
    }
  }
}
```

**3. Monitor Confirmations**
```typescript
async function waitForConfirmation(
  txid: string,
  requiredConfirmations: number = 6
): Promise<void> {
  while (true) {
    const status = await arc.getTransactionStatus(txid)

    if (status.confirmations >= requiredConfirmations) {
      return
    }

    if (status.blockStatus === 'REJECTED') {
      throw new Error('Transaction rejected')
    }

    // Wait before checking again
    await sleep(30000) // 30 seconds
  }
}
```

**4. Implement Webhooks**
```typescript
import express from 'express'

const app = express()

// Receive transaction status updates
app.post('/tx-callback', express.json(), (req, res) => {
  const { txid, status, blockHash, confirmations } = req.body

  console.log(`TX ${txid} status: ${status}`)

  if (status === 'MINED') {
    console.log(`Included in block: ${blockHash}`)
  }

  if (confirmations >= 6) {
    console.log('Transaction confirmed!')
    // Process confirmed transaction
  }

  res.sendStatus(200)
})

app.listen(3000)
```

## 11. Network Monitoring and Debugging

### Monitoring Tools

**1. Transaction Status**
```typescript
async function debugTransaction(txid: string) {
  const status = await arc.getTransactionStatus(txid)

  console.log('TXID:', txid)
  console.log('Status:', status.blockStatus)
  console.log('Timestamp:', status.timestamp)
  console.log('Block Hash:', status.blockHash)
  console.log('Confirmations:', status.confirmations)

  if (status.blockStatus === 'REJECTED') {
    console.error('Rejection Reason:', status.rejectReason)
  }
}
```

**2. Network Health**
```typescript
async function checkNetworkHealth() {
  try {
    const feeQuote = await arc.getFeeQuote()
    console.log('✅ Network is reachable')
    console.log('Current fees:', feeQuote.fees)
  } catch (error) {
    console.error('❌ Network unreachable:', error)
  }
}
```

**3. Peer Information** (Legacy Nodes)
```typescript
// Using RPC if available
async function getPeerInfo() {
  // Conceptual - requires RPC client
  const peers = await rpc.getPeerInfo()

  console.log(`Connected to ${peers.length} peers`)

  for (const peer of peers) {
    console.log(`Peer: ${peer.addr}`)
    console.log(`Version: ${peer.version}`)
    console.log(`Ping: ${peer.pingtime}ms`)
  }
}
```

## Best Practices

1. **Use ARC for production** applications - Don't connect directly to nodes
2. **Implement retry logic** for network failures
3. **Monitor transaction status** with callbacks or polling
4. **Handle network delays** gracefully with timeouts
5. **Use SPV for lightweight** clients (mobile, IoT)
6. **Understand the scaling model** - Legacy vs Teranode
7. **Plan for horizontal scaling** if building infrastructure
8. **Implement proper monitoring** and alerting
9. **Test with testnet** before mainnet deployment
10. **Keep up with network upgrades** - Teranode adoption

## Common Pitfalls

1. **Direct node connections** - Use ARC instead
2. **Ignoring network delays** - Transactions take time to propagate
3. **No retry logic** - Networks can be temporarily unavailable
4. **Assuming immediate confirmation** - Blocks take time
5. **Not monitoring status** - Transaction can be rejected
6. **Single point of failure** - Use multiple ARC endpoints
7. **Ignoring fee market** - Dynamic fees change based on load

## Hands-On Project: Network Monitor Dashboard

Build a real-time network monitoring dashboard:

```typescript
class NetworkMonitor {
  private arc: ARC
  private transactions: Map<string, TxStatus>

  async trackTransaction(txid: string) {
    const interval = setInterval(async () => {
      const status = await this.arc.getTransactionStatus(txid)

      this.transactions.set(txid, status)

      if (status.confirmations >= 6) {
        clearInterval(interval)
        this.onConfirmed(txid)
      }
    }, 30000)
  }

  getNetworkStats() {
    const stats = {
      total: this.transactions.size,
      pending: 0,
      confirmed: 0,
      rejected: 0
    }

    for (const [_, status] of this.transactions) {
      if (status.confirmations >= 6) stats.confirmed++
      else if (status.blockStatus === 'REJECTED') stats.rejected++
      else stats.pending++
    }

    return stats
  }
}
```

## Next Steps

Continue to:
- **[Node Operations](../node-operations/README.md)** - Run and maintain BSV nodes
- **[Advanced Scripting](../advanced-scripting/README.md)** - Complex smart contracts

## Additional Resources

- [ARC SDK Component](../../../sdk-components/arc/README.md)
- [SPV SDK Component](../../../sdk-components/spv/README.md)
- [Teranode Documentation](https://bsv-blockchain.github.io/teranode/)
- [BSV Association - Teranode Release](https://bsvblockchain.org/whats-next-for-teranode/)
- [Bitcoin SV Wiki - Network](https://wiki.bitcoinsv.io/)
- [BSV Scaling Test Network](https://bitcoinscaling.io/)

---

**Status:** ✅ Complete - Ready for learning
