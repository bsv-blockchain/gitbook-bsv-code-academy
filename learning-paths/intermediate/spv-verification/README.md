# SPV Verification

## Overview

Master Simplified Payment Verification (SPV) to build lightweight, scalable BSV blockchain clients. This module teaches you how to verify transactions without downloading the entire blockchain using merkle proofs, block headers, and proof-of-work validation. Learn to implement production-ready SPV clients following BRC-9 and BRC-67 standards.

**Estimated Time:** 3-4 hours
**Difficulty:** Intermediate
**Prerequisites:** Complete [BSV Fundamentals](../../beginner/bsv-fundamentals/README.md) and [Transaction Building](../transaction-building/README.md)

## Learning Objectives

By the end of this module, you will be able to:

- ✅ Understand SPV principles and how it enables lightweight clients
- ✅ Create and verify merkle proofs for transaction inclusion
- ✅ Validate block headers and verify proof-of-work
- ✅ Implement the ChainTracker interface for blockchain state
- ✅ Work with SPV envelopes following BRC-8 specification
- ✅ Build practical SPV clients for production use
- ✅ Combine SPV verification with BEEF transaction bundles
- ✅ Deploy complete SPV implementations with proper security

## SDK Components Used

This course leverages these standardized SDK modules:

- **[SPV](../../../sdk-components/spv/README.md)** - SPV verification implementation
- **[Merkle Proofs](../../../sdk-components/merkle-proofs/README.md)** - Transaction inclusion proofs
- **[Transaction](../../../sdk-components/transaction/README.md)** - Transaction structure and validation
- **[ChainTracker](../../../sdk-components/chain-tracker/README.md)** - Blockchain state interface
- **[BEEF](../../../sdk-components/beef/README.md)** - Transaction envelopes with SPV
- **[Block Headers](../../../sdk-components/block-headers/README.md)** - Header validation

## 1. Understanding SPV

### What is SPV?

Reference: **[SPV Component](../../../sdk-components/spv/README.md)**

Simplified Payment Verification (SPV) allows lightweight clients to verify transaction inclusion in the blockchain without downloading all blocks. Instead of validating every transaction, SPV clients:

1. **Download block headers only** (~80 bytes each vs. megabytes per block)
2. **Verify proof-of-work** in the header chain
3. **Use merkle proofs** to verify specific transactions
4. **Trust the longest chain** with the most accumulated work

```typescript
import { MerklePath, Transaction, ChainTracker } from '@bsv/sdk'

/**
 * SPV Verification Flow
 *
 * 1. Receive transaction + merkle proof
 * 2. Verify merkle proof computes to block's merkle root
 * 3. Verify block header has valid proof-of-work
 * 4. Verify block is in the longest chain
 * 5. Transaction is verified!
 */

async function verifySPV(
  tx: Transaction,
  merklePath: MerklePath,
  chainTracker: ChainTracker
): Promise<boolean> {
  // Step 1: Compute merkle root from transaction + proof
  const computedRoot = merklePath.computeRoot(tx.id('hex') as string)

  // Step 2: Get block height from merkle path
  const blockHeight = merklePath.blockHeight

  // Step 3: Verify computed root matches the blockchain
  const isValid = await chainTracker.isValidRootForHeight(
    computedRoot,
    blockHeight
  )

  return isValid
}
```

### Why SPV Matters

SPV enables:

- **Mobile Wallets** - Run on phones without gigabytes of storage
- **IoT Devices** - Verify payments with minimal resources
- **Scalability** - Process millions of transactions without full nodes
- **Privacy** - Query specific transactions without revealing interests
- **Speed** - Instant verification without blockchain sync

### How SPV Works

```
┌─────────────────────────────────────────────────────┐
│              Full Node (Miner/Archive)              │
│  • Stores all transactions                          │
│  • Validates every transaction                      │
│  • Maintains full UTXO set                          │
│  • ~2+ TB storage                                   │
└─────────────────────────────────────────────────────┘
                      │
                      │ Provides merkle proofs
                      ▼
┌─────────────────────────────────────────────────────┐
│              SPV Client (Lightweight)               │
│  • Stores block headers only                        │
│  • Verifies specific transactions                   │
│  • Uses merkle proofs                               │
│  • ~100 MB storage                                  │
└─────────────────────────────────────────────────────┘
```

### Security Model

Reference: **[SPV Component - Security](../../../sdk-components/spv/README.md#security-considerations)**

**What SPV Guarantees:**
- Transaction was included in a block
- Block has valid proof-of-work
- Block is part of the longest chain

**What SPV Does NOT Guarantee:**
- Transaction inputs are valid (could be double-spend)
- All rules were followed (rely on miners for full validation)
- Full network state (only verified transactions)

**BAD Example - Trusting Without Verification:**
```typescript
// ❌ BAD: Accepting transaction without proof
async function acceptPayment(tx: Transaction): Promise<void> {
  // Just trust the transaction is valid
  await processPayment(tx)
}
```

**GOOD Example - Proper SPV Verification:**
```typescript
// ✅ GOOD: Verify with merkle proof
async function acceptPayment(
  tx: Transaction,
  merklePath: MerklePath,
  chainTracker: ChainTracker
): Promise<void> {
  // Verify transaction is in a valid block
  const isValid = await verifySPV(tx, merklePath, chainTracker)

  if (!isValid) {
    throw new Error('Transaction not verified in blockchain')
  }

  // Now safe to process
  await processPayment(tx)
}
```

## 2. Merkle Proof Verification

### Understanding Merkle Trees

Reference: **[Merkle Proofs Component](../../../sdk-components/merkle-proofs/README.md)**

A merkle tree is a binary tree of hashes where:
- **Leaves** are transaction IDs (hashes)
- **Nodes** are hashes of their children
- **Root** is the final hash included in the block header

```
                    Merkle Root
                   /            \
              Hash(AB)         Hash(CD)
              /    \           /    \
         Hash(A) Hash(B)  Hash(C) Hash(D)
           |       |        |       |
          Tx A    Tx B     Tx C    Tx D
```

To prove Tx B is in the block, provide:
1. Hash(A) - sibling
2. Hash(CD) - uncle
3. Merkle Root - from block header

### Creating Merkle Proofs

```typescript
import { MerklePath } from '@bsv/sdk'

/**
 * Create a merkle proof for a transaction
 * (Usually provided by the miner/transaction processor)
 */
function createMerkleProof(
  txid: string,
  blockTransactions: string[],
  blockHeight: number
): MerklePath {
  // Find transaction index in block
  const txIndex = blockTransactions.indexOf(txid)

  if (txIndex === -1) {
    throw new Error('Transaction not in block')
  }

  // Build merkle path
  const path: Array<{ offset: number; hash?: string; txid?: boolean; duplicate?: boolean }> = []

  let level = blockTransactions.map(id => Buffer.from(id, 'hex'))
  let currentIndex = txIndex

  while (level.length > 1) {
    const newLevel: Buffer[] = []

    for (let i = 0; i < level.length; i += 2) {
      const left = level[i]
      const right = i + 1 < level.length ? level[i + 1] : left

      // Record sibling for proof
      if (i === currentIndex || i + 1 === currentIndex) {
        const siblingIndex = i === currentIndex ? i + 1 : i
        const sibling = level[siblingIndex] || left

        path.push({
          offset: siblingIndex,
          hash: sibling.toString('hex')
        })
      }

      // Compute parent hash
      const combined = Buffer.concat([left, right])
      const parent = Hash.hash256(combined)
      newLevel.push(parent)
    }

    currentIndex = Math.floor(currentIndex / 2)
    level = newLevel
  }

  // Create MerklePath from path data
  return MerklePath.fromBinary([
    blockHeight,
    ...path
  ])
}
```

### Verifying Merkle Proofs

Reference: **[Merkle Proofs - Verification](../../../sdk-components/merkle-proofs/README.md#key-features)**

```typescript
import { MerklePath, Transaction, Hash } from '@bsv/sdk'

/**
 * Verify a transaction is included in a block using merkle proof
 */
function verifyMerkleProof(
  tx: Transaction,
  merklePath: MerklePath
): { valid: boolean; computedRoot: string } {
  // Get transaction ID
  const txid = tx.id('hex') as string

  // Compute merkle root from transaction + proof
  const computedRoot = merklePath.computeRoot(txid)

  // In production, verify this root against block header
  return {
    valid: true, // Would check against blockchain
    computedRoot
  }
}

// Usage
const tx = Transaction.fromHex('...')
const proof = MerklePath.fromHex('...')

const { computedRoot } = verifyMerkleProof(tx, proof)
console.log('Computed merkle root:', computedRoot)
```

### Working with TSC Format

The Transaction Signature Component (TSC) format is the standard binary format for merkle paths:

```typescript
import { MerklePath } from '@bsv/sdk'

/**
 * Parse TSC-format merkle proof
 */
function parseTSCProof(tscHex: string): {
  blockHeight: number
  path: Array<{ offset: number; hash: string }>
} {
  const merklePath = MerklePath.fromHex(tscHex)

  return {
    blockHeight: merklePath.blockHeight,
    path: merklePath.path.map(node => ({
      offset: node.offset,
      hash: node.hash || ''
    }))
  }
}

/**
 * Serialize merkle path to TSC format
 */
function toTSCProof(merklePath: MerklePath): string {
  return merklePath.toHex()
}

// Usage
const tscHex = 'fed7c509000a02fddd01...'
const parsed = parseTSCProof(tscHex)

console.log('Block height:', parsed.blockHeight)
console.log('Proof path length:', parsed.path.length)
```

### Batch Merkle Verification

```typescript
/**
 * Verify multiple transactions efficiently
 */
async function verifyBatchMerkleProofs(
  transactions: Array<{ tx: Transaction; proof: MerklePath }>,
  chainTracker: ChainTracker
): Promise<Map<string, boolean>> {
  const results = new Map<string, boolean>()

  // Group by block height for efficient verification
  const byHeight = new Map<number, Array<{ tx: Transaction; proof: MerklePath }>>()

  for (const item of transactions) {
    const height = item.proof.blockHeight
    if (!byHeight.has(height)) {
      byHeight.set(height, [])
    }
    byHeight.get(height)!.push(item)
  }

  // Verify each block's transactions
  for (const [height, items] of byHeight) {
    for (const { tx, proof } of items) {
      const txid = tx.id('hex') as string
      const computedRoot = proof.computeRoot(txid)

      const isValid = await chainTracker.isValidRootForHeight(
        computedRoot,
        height
      )

      results.set(txid, isValid)
    }
  }

  return results
}
```

## 3. Header Validation

### Block Header Structure

Reference: **[Block Headers Component](../../../sdk-components/block-headers/README.md)**

A Bitcoin block header contains:
- **Version** (4 bytes) - Block version number
- **Previous Block Hash** (32 bytes) - Links to previous block
- **Merkle Root** (32 bytes) - Root of transaction merkle tree
- **Timestamp** (4 bytes) - Block creation time
- **Bits** (4 bytes) - Difficulty target
- **Nonce** (4 bytes) - Proof-of-work solution

Total: 80 bytes

```typescript
interface BlockHeader {
  version: number
  previousBlockHash: string
  merkleRoot: string
  timestamp: number
  bits: number
  nonce: number
}

/**
 * Parse block header from binary
 */
function parseBlockHeader(headerHex: string): BlockHeader {
  const buffer = Buffer.from(headerHex, 'hex')

  return {
    version: buffer.readUInt32LE(0),
    previousBlockHash: buffer.slice(4, 36).reverse().toString('hex'),
    merkleRoot: buffer.slice(36, 68).reverse().toString('hex'),
    timestamp: buffer.readUInt32LE(68),
    bits: buffer.readUInt32LE(72),
    nonce: buffer.readUInt32LE(76)
  }
}

/**
 * Serialize block header to binary
 */
function serializeBlockHeader(header: BlockHeader): Buffer {
  const buffer = Buffer.allocUnsafe(80)

  buffer.writeUInt32LE(header.version, 0)
  Buffer.from(header.previousBlockHash, 'hex').reverse().copy(buffer, 4)
  Buffer.from(header.merkleRoot, 'hex').reverse().copy(buffer, 36)
  buffer.writeUInt32LE(header.timestamp, 68)
  buffer.writeUInt32LE(header.bits, 72)
  buffer.writeUInt32LE(header.nonce, 76)

  return buffer
}
```

### Proof-of-Work Validation

```typescript
import { Hash } from '@bsv/sdk'

/**
 * Verify block header has valid proof-of-work
 */
function verifyProofOfWork(header: BlockHeader): boolean {
  // Serialize header
  const headerBytes = serializeBlockHeader(header)

  // Double SHA-256 hash
  const hash = Hash.hash256(headerBytes)

  // Convert hash to number for comparison
  const hashValue = BigInt('0x' + hash.toString('hex'))

  // Calculate target from bits
  const target = bitsToTarget(header.bits)

  // Hash must be less than target
  return hashValue < target
}

/**
 * Convert compact bits representation to target
 */
function bitsToTarget(bits: number): bigint {
  const exponent = bits >> 24
  const mantissa = bits & 0x00ffffff

  return BigInt(mantissa) * (2n ** (8n * BigInt(exponent - 3)))
}

/**
 * Calculate difficulty from bits
 */
function calculateDifficulty(bits: number): number {
  const maxTarget = 0x00000000ffff0000000000000000000000000000000000000000000000000000n
  const target = bitsToTarget(bits)

  return Number(maxTarget / target)
}

// Usage
const header = parseBlockHeader(headerHex)
const isValidPOW = verifyProofOfWork(header)
const difficulty = calculateDifficulty(header.bits)

console.log('Valid proof-of-work:', isValidPOW)
console.log('Block difficulty:', difficulty)
```

### Header Chain Validation

Reference: **[SPV Component - Header Validation](../../../sdk-components/spv/README.md#key-features)**

```typescript
/**
 * Verify a chain of block headers
 */
class HeaderChainValidator {
  /**
   * Validate headers form a valid chain
   */
  validateChain(headers: BlockHeader[]): boolean {
    if (headers.length === 0) {
      return false
    }

    for (let i = 1; i < headers.length; i++) {
      const prev = headers[i - 1]
      const current = headers[i]

      // Verify proof-of-work
      if (!verifyProofOfWork(current)) {
        console.error(`Invalid POW at height ${i}`)
        return false
      }

      // Verify links to previous block
      const prevHash = this.getBlockHash(prev)
      if (current.previousBlockHash !== prevHash) {
        console.error(`Broken chain at height ${i}`)
        return false
      }

      // Verify timestamp progression
      if (current.timestamp <= prev.timestamp) {
        console.error(`Invalid timestamp at height ${i}`)
        return false
      }
    }

    return true
  }

  /**
   * Calculate block hash from header
   */
  getBlockHash(header: BlockHeader): string {
    const headerBytes = serializeBlockHeader(header)
    const hash = Hash.hash256(headerBytes)
    return hash.reverse().toString('hex')
  }

  /**
   * Calculate total accumulated work
   */
  calculateChainWork(headers: BlockHeader[]): bigint {
    let totalWork = 0n

    for (const header of headers) {
      const target = bitsToTarget(header.bits)
      const work = (2n ** 256n) / (target + 1n)
      totalWork += work
    }

    return totalWork
  }
}

// Usage
const validator = new HeaderChainValidator()
const isValidChain = validator.validateChain(blockHeaders)
const chainWork = validator.calculateChainWork(blockHeaders)

console.log('Valid chain:', isValidChain)
console.log('Total chain work:', chainWork.toString())
```

### Checkpoint Validation

```typescript
/**
 * Use checkpoints for faster validation
 */
class CheckpointValidator {
  private checkpoints: Map<number, string> // height -> block hash

  constructor() {
    this.checkpoints = new Map([
      // Known valid blocks
      [0, '000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f'],
      [100000, '000000000003ba27aa200b1cecaad478d2b00432346c3f1f3986da1afd33e506'],
      [500000, '000000000000000000024bead8df69990852c202db0e0097c1a12ea637d7e96d']
    ])
  }

  /**
   * Verify header against known checkpoints
   */
  verifyAgainstCheckpoints(
    height: number,
    blockHash: string
  ): boolean {
    const checkpoint = this.checkpoints.get(height)

    if (!checkpoint) {
      return true // No checkpoint at this height
    }

    return checkpoint === blockHash
  }

  /**
   * Add new checkpoint
   */
  addCheckpoint(height: number, blockHash: string): void {
    this.checkpoints.set(height, blockHash)
  }
}
```

## 4. ChainTracker Interface

### Implementing ChainTracker

Reference: **[ChainTracker Component](../../../sdk-components/chain-tracker/README.md)**

The ChainTracker interface provides blockchain state for SPV verification:

```typescript
import { ChainTracker } from '@bsv/sdk'

/**
 * ChainTracker interface for blockchain queries
 */
interface ChainTracker {
  /**
   * Verify merkle root is valid for a specific block height
   */
  isValidRootForHeight(root: string, height: number): Promise<boolean>
}
```

### WhatsOnChain ChainTracker

```typescript
/**
 * WhatsOnChain API implementation of ChainTracker
 */
class WhatsOnChainTracker implements ChainTracker {
  private baseURL: string
  private network: 'main' | 'test'

  constructor(network: 'main' | 'test' = 'main') {
    this.network = network
    this.baseURL = `https://api.whatsonchain.com/v1/bsv/${network}`
  }

  /**
   * Verify merkle root for block height
   */
  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    try {
      // Fetch block header at height
      const response = await fetch(
        `${this.baseURL}/block/height/${height}/header`
      )

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`)
      }

      const headerHex = await response.text()
      const header = parseBlockHeader(headerHex)

      // Compare merkle roots
      return header.merkleRoot === root
    } catch (error) {
      console.error('ChainTracker error:', error)
      return false
    }
  }

  /**
   * Get block hash at specific height
   */
  async getBlockHash(height: number): Promise<string> {
    const response = await fetch(`${this.baseURL}/block/height/${height}/hash`)
    return await response.text()
  }

  /**
   * Get current blockchain height
   */
  async getCurrentHeight(): Promise<number> {
    const response = await fetch(`${this.baseURL}/chain/info`)
    const data = await response.json()
    return data.blocks
  }
}

// Usage
const chainTracker = new WhatsOnChainTracker('main')
const isValid = await chainTracker.isValidRootForHeight(
  'a1b2c3...',
  750000
)
```

### Local Header ChainTracker

```typescript
/**
 * Local header store implementation
 * Stores block headers locally for offline verification
 */
class LocalHeaderTracker implements ChainTracker {
  private headers: Map<number, BlockHeader>

  constructor() {
    this.headers = new Map()
  }

  /**
   * Add block header to local store
   */
  addHeader(height: number, header: BlockHeader): void {
    // Verify proof-of-work
    if (!verifyProofOfWork(header)) {
      throw new Error('Invalid proof-of-work')
    }

    // Verify chain continuity
    if (height > 0) {
      const prevHeader = this.headers.get(height - 1)
      if (prevHeader) {
        const validator = new HeaderChainValidator()
        const prevHash = validator.getBlockHash(prevHeader)

        if (header.previousBlockHash !== prevHash) {
          throw new Error('Header does not link to previous block')
        }
      }
    }

    this.headers.set(height, header)
  }

  /**
   * Verify merkle root for height
   */
  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    const header = this.headers.get(height)

    if (!header) {
      console.warn(`Header not found for height ${height}`)
      return false
    }

    return header.merkleRoot === root
  }

  /**
   * Get header at height
   */
  getHeader(height: number): BlockHeader | undefined {
    return this.headers.get(height)
  }

  /**
   * Get current tip height
   */
  getCurrentHeight(): number {
    return Math.max(...this.headers.keys())
  }

  /**
   * Sync headers from remote source
   */
  async syncHeaders(
    startHeight: number,
    endHeight: number,
    source: WhatsOnChainTracker
  ): Promise<void> {
    for (let height = startHeight; height <= endHeight; height++) {
      const headerHex = await this.fetchHeaderHex(source, height)
      const header = parseBlockHeader(headerHex)
      this.addHeader(height, header)
    }
  }

  private async fetchHeaderHex(
    source: WhatsOnChainTracker,
    height: number
  ): Promise<string> {
    const response = await fetch(
      `https://api.whatsonchain.com/v1/bsv/main/block/height/${height}/header`
    )
    return await response.text()
  }
}

// Usage
const localTracker = new LocalHeaderTracker()

// Sync headers
const remoteTracker = new WhatsOnChainTracker()
await localTracker.syncHeaders(750000, 750100, remoteTracker)

// Now verify offline
const isValid = await localTracker.isValidRootForHeight(root, 750050)
```

### Cached ChainTracker

```typescript
/**
 * Caching wrapper for any ChainTracker implementation
 */
class CachedChainTracker implements ChainTracker {
  private tracker: ChainTracker
  private cache: Map<string, boolean>
  private cacheTimeout: number

  constructor(tracker: ChainTracker, cacheTimeoutMs: number = 3600000) {
    this.tracker = tracker
    this.cache = new Map()
    this.cacheTimeout = cacheTimeoutMs
  }

  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    const cacheKey = `${root}:${height}`

    // Check cache
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey)!
    }

    // Query tracker
    const isValid = await this.tracker.isValidRootForHeight(root, height)

    // Cache result
    this.cache.set(cacheKey, isValid)

    // Clear cache after timeout
    setTimeout(() => {
      this.cache.delete(cacheKey)
    }, this.cacheTimeout)

    return isValid
  }

  /**
   * Clear all cached results
   */
  clearCache(): void {
    this.cache.clear()
  }
}

// Usage
const baseTracker = new WhatsOnChainTracker()
const cachedTracker = new CachedChainTracker(baseTracker, 3600000) // 1 hour cache
```

## 5. SPV Envelopes (BRC-8)

### Understanding BRC-8

Reference: **[BEEF Component - SPV Envelopes](../../../sdk-components/beef/README.md)**

BRC-8 defines the Transaction Envelope specification for packaging transactions with their SPV proofs:

```typescript
/**
 * BRC-8 Transaction Envelope
 */
interface TransactionEnvelope {
  rawTx: number[]              // Transaction binary
  proof?: {
    blockHeight: number        // Block height containing tx
    path: MerklePath           // Merkle proof
  }
  inputs?: TransactionEnvelope[] // Input source envelopes
}
```

### Creating SPV Envelopes

```typescript
import { Transaction, MerklePath } from '@bsv/sdk'

/**
 * Create SPV envelope for a transaction
 */
function createSPVEnvelope(
  tx: Transaction,
  merklePath: MerklePath
): TransactionEnvelope {
  return {
    rawTx: Array.from(tx.toBinary()),
    proof: {
      blockHeight: merklePath.blockHeight,
      path: merklePath
    }
  }
}

/**
 * Create envelope with input chain
 */
async function createEnvelopeWithInputs(
  tx: Transaction,
  merklePath: MerklePath,
  sourceEnvelopes: TransactionEnvelope[]
): Promise<TransactionEnvelope> {
  return {
    rawTx: Array.from(tx.toBinary()),
    proof: {
      blockHeight: merklePath.blockHeight,
      path: merklePath
    },
    inputs: sourceEnvelopes
  }
}

// Usage
const tx = new Transaction()
const proof = MerklePath.fromHex('...')

const envelope = createSPVEnvelope(tx, proof)
```

### Verifying SPV Envelopes

```typescript
/**
 * Verify complete SPV envelope
 */
async function verifySPVEnvelope(
  envelope: TransactionEnvelope,
  chainTracker: ChainTracker
): Promise<boolean> {
  // Parse transaction
  const tx = Transaction.fromBinary(envelope.rawTx)

  // Verify has proof
  if (!envelope.proof) {
    console.error('Envelope missing merkle proof')
    return false
  }

  // Compute merkle root
  const txid = tx.id('hex') as string
  const computedRoot = envelope.proof.path.computeRoot(txid)

  // Verify against blockchain
  const isValid = await chainTracker.isValidRootForHeight(
    computedRoot,
    envelope.proof.blockHeight
  )

  if (!isValid) {
    console.error('Invalid merkle proof')
    return false
  }

  // Recursively verify input envelopes
  if (envelope.inputs) {
    for (const inputEnvelope of envelope.inputs) {
      const inputValid = await verifySPVEnvelope(inputEnvelope, chainTracker)
      if (!inputValid) {
        return false
      }
    }
  }

  return true
}

// Usage
const isValid = await verifySPVEnvelope(envelope, chainTracker)
console.log('Envelope valid:', isValid)
```

### Serializing Envelopes

```typescript
/**
 * Serialize envelope to binary format
 */
function serializeEnvelope(envelope: TransactionEnvelope): Buffer {
  const parts: Buffer[] = []

  // Transaction
  parts.push(Buffer.from(envelope.rawTx))

  // Has proof flag
  const hasProof = envelope.proof ? 1 : 0
  parts.push(Buffer.from([hasProof]))

  if (envelope.proof) {
    // Block height
    const heightBuf = Buffer.allocUnsafe(4)
    heightBuf.writeUInt32LE(envelope.proof.blockHeight, 0)
    parts.push(heightBuf)

    // Merkle path
    parts.push(Buffer.from(envelope.proof.path.toBinary()))
  }

  // Input count
  const inputCount = envelope.inputs?.length || 0
  parts.push(Buffer.from([inputCount]))

  // Input envelopes
  if (envelope.inputs) {
    for (const input of envelope.inputs) {
      parts.push(serializeEnvelope(input))
    }
  }

  return Buffer.concat(parts)
}

/**
 * Parse envelope from binary
 */
function parseEnvelope(buffer: Buffer): TransactionEnvelope {
  let offset = 0

  // Parse transaction (read until proof flag)
  // This is simplified - real implementation needs proper parsing
  const txLength = 250 // Would calculate actual length
  const rawTx = Array.from(buffer.slice(offset, offset + txLength))
  offset += txLength

  // Parse proof flag
  const hasProof = buffer.readUInt8(offset) === 1
  offset += 1

  let proof: { blockHeight: number; path: MerklePath } | undefined

  if (hasProof) {
    const blockHeight = buffer.readUInt32LE(offset)
    offset += 4

    // Parse merkle path (simplified)
    const pathLength = 100 // Would calculate actual length
    const pathBinary = buffer.slice(offset, offset + pathLength)
    const path = MerklePath.fromBinary(Array.from(pathBinary))
    offset += pathLength

    proof = { blockHeight, path }
  }

  // Parse input count
  const inputCount = buffer.readUInt8(offset)
  offset += 1

  const inputs: TransactionEnvelope[] = []
  for (let i = 0; i < inputCount; i++) {
    const input = parseEnvelope(buffer.slice(offset))
    inputs.push(input)
    offset += serializeEnvelope(input).length
  }

  return { rawTx, proof, inputs: inputCount > 0 ? inputs : undefined }
}
```

## 6. Practical SPV Client

### Building a Lightweight SPV Client

Reference: **[SPV Component - Common Patterns](../../../sdk-components/spv/README.md#common-patterns)**

```typescript
import { Transaction, MerklePath, ChainTracker } from '@bsv/sdk'

/**
 * Production SPV Client
 */
class SPVClient {
  private chainTracker: ChainTracker
  private verifiedTransactions: Map<string, Transaction>
  private pendingTransactions: Map<string, Transaction>

  constructor(chainTracker: ChainTracker) {
    this.chainTracker = chainTracker
    this.verifiedTransactions = new Map()
    this.pendingTransactions = new Map()
  }

  /**
   * Verify and track a transaction
   */
  async verifyTransaction(
    tx: Transaction,
    merklePath?: MerklePath
  ): Promise<boolean> {
    const txid = tx.id('hex') as string

    // If no proof, mark as pending
    if (!merklePath) {
      this.pendingTransactions.set(txid, tx)
      console.log(`Transaction ${txid} pending (no proof)`)
      return false
    }

    // Verify merkle proof
    const computedRoot = merklePath.computeRoot(txid)
    const isValid = await this.chainTracker.isValidRootForHeight(
      computedRoot,
      merklePath.blockHeight
    )

    if (isValid) {
      // Move from pending to verified
      this.pendingTransactions.delete(txid)
      this.verifiedTransactions.set(txid, tx)
      console.log(`Transaction ${txid} verified at height ${merklePath.blockHeight}`)
      return true
    } else {
      console.error(`Transaction ${txid} failed verification`)
      return false
    }
  }

  /**
   * Get transaction status
   */
  getTransactionStatus(txid: string): 'verified' | 'pending' | 'unknown' {
    if (this.verifiedTransactions.has(txid)) {
      return 'verified'
    }
    if (this.pendingTransactions.has(txid)) {
      return 'pending'
    }
    return 'unknown'
  }

  /**
   * Get verified transaction
   */
  getTransaction(txid: string): Transaction | undefined {
    return this.verifiedTransactions.get(txid) ||
           this.pendingTransactions.get(txid)
  }

  /**
   * Verify transaction chain (tx and all inputs)
   */
  async verifyTransactionChain(
    tx: Transaction,
    merklePath: MerklePath,
    inputProofs: Map<string, MerklePath>
  ): Promise<boolean> {
    // Verify main transaction
    const txValid = await this.verifyTransaction(tx, merklePath)
    if (!txValid) {
      return false
    }

    // Verify all input transactions
    for (const input of tx.inputs) {
      const sourceTXID = input.sourceTXID
      if (!sourceTXID) continue

      const sourceProof = inputProofs.get(sourceTXID)
      if (!sourceProof) {
        console.error(`Missing proof for input ${sourceTXID}`)
        return false
      }

      const sourceTransaction = input.sourceTransaction
      if (!sourceTransaction) {
        console.error(`Missing source transaction ${sourceTXID}`)
        return false
      }

      const sourceValid = await this.verifyTransaction(
        sourceTransaction,
        sourceProof
      )

      if (!sourceValid) {
        return false
      }
    }

    return true
  }

  /**
   * Wait for transaction confirmation
   */
  async waitForConfirmation(
    txid: string,
    maxWaitMs: number = 60000
  ): Promise<boolean> {
    const startTime = Date.now()

    while (Date.now() - startTime < maxWaitMs) {
      const status = this.getTransactionStatus(txid)

      if (status === 'verified') {
        return true
      }

      // Wait 5 seconds before checking again
      await new Promise(resolve => setTimeout(resolve, 5000))
    }

    return false
  }

  /**
   * Get verified transactions count
   */
  getVerifiedCount(): number {
    return this.verifiedTransactions.size
  }

  /**
   * Clear old verified transactions (memory management)
   */
  pruneOldTransactions(maxAge: number = 86400000): void {
    // In production, track timestamps and prune
    // This is simplified
    if (this.verifiedTransactions.size > 10000) {
      // Keep only recent 5000
      const keys = Array.from(this.verifiedTransactions.keys())
      const toRemove = keys.slice(0, keys.length - 5000)
      for (const key of toRemove) {
        this.verifiedTransactions.delete(key)
      }
    }
  }
}

// Usage
const chainTracker = new WhatsOnChainTracker('main')
const spvClient = new SPVClient(chainTracker)

// Verify transaction
const tx = Transaction.fromHex('...')
const proof = MerklePath.fromHex('...')

const isValid = await spvClient.verifyTransaction(tx, proof)

if (isValid) {
  console.log('Payment verified!')
}
```

### SPV Wallet Implementation

```typescript
import { Utils } from '@bsv/sdk'

/**
 * SPV-based wallet for lightweight clients
 */
class SPVWallet {
  private spvClient: SPVClient
  private privateKey: PrivateKey
  private utxos: Map<string, UTXO>

  constructor(
    privateKey: PrivateKey,
    chainTracker: ChainTracker
  ) {
    this.privateKey = privateKey
    this.spvClient = new SPVClient(chainTracker)
    this.utxos = new Map()
  }

  /**
   * Receive payment with SPV verification
   */
  async receivePayment(
    tx: Transaction,
    merklePath: MerklePath
  ): Promise<void> {
    // Verify transaction
    const isValid = await this.spvClient.verifyTransaction(tx, merklePath)

    if (!isValid) {
      throw new Error('Payment verification failed')
    }

    // Find outputs to our address
    const myAddress = this.privateKey.toPublicKey().toAddress()

    tx.outputs.forEach((output, index) => {
      // Check if output is to our address
      const p2pkh = new P2PKH()
      const lockingScript = output.lockingScript

      // Simple check - in production, parse script properly
      if (this.isOutputToAddress(lockingScript, myAddress)) {
        const utxo: UTXO = {
          txid: tx.id('hex') as string,
          vout: index,
          satoshis: output.satoshis || 0,
          script: lockingScript.toHex()
        }

        this.utxos.set(`${utxo.txid}:${utxo.vout}`, utxo)
        console.log(`Received ${utxo.satoshis} satoshis`)
      }
    })
  }

  /**
   * Send payment from SPV wallet
   */
  async sendPayment(
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    // Select UTXOs
    const selectedUTXOs = this.selectUTXOs(amount)

    // Build transaction
    const tx = new Transaction()

    for (const utxo of selectedUTXOs) {
      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(this.privateKey)
      })
    }

    // Add payment output
    tx.addOutput({
      lockingScript: new P2PKH().lock(toAddress),
      satoshis: amount
    })

    // Calculate fee and add change
    await tx.fee()

    const totalInput = selectedUTXOs.reduce((sum, u) => sum + u.satoshis, 0)
    const changeAmount = totalInput - amount - tx.getFee()

    if (changeAmount > 546) { // Dust limit
      tx.addOutput({
        lockingScript: new P2PKH().lock(
          this.privateKey.toPublicKey().toAddress()
        ),
        satoshis: changeAmount
      })
    }

    // Sign transaction
    await tx.sign()

    // Mark UTXOs as spent
    for (const utxo of selectedUTXOs) {
      this.utxos.delete(`${utxo.txid}:${utxo.vout}`)
    }

    return tx
  }

  /**
   * Get wallet balance
   */
  getBalance(): number {
    return Array.from(this.utxos.values())
      .reduce((sum, utxo) => sum + utxo.satoshis, 0)
  }

  private selectUTXOs(amount: number): UTXO[] {
    const utxos = Array.from(this.utxos.values())
    const sorted = utxos.sort((a, b) => b.satoshis - a.satoshis)

    const selected: UTXO[] = []
    let total = 0

    for (const utxo of sorted) {
      selected.push(utxo)
      total += utxo.satoshis

      if (total >= amount + 500) { // Include estimated fee
        break
      }
    }

    return selected
  }

  private isOutputToAddress(
    lockingScript: LockingScript,
    address: string
  ): boolean {
    // Simplified check - in production, properly parse P2PKH script
    const scriptHex = lockingScript.toHex()
    const decoded = Utils.fromBase58Check(address)
    const addressHash = Buffer.from(decoded.data).toString('hex')
    return scriptHex.includes(addressHash)
  }
}

// Usage
const wallet = new SPVWallet(privateKey, chainTracker)

// Receive payment
await wallet.receivePayment(incomingTx, incomingProof)

// Check balance
const balance = wallet.getBalance()
console.log('Balance:', balance, 'satoshis')

// Send payment
const paymentTx = await wallet.sendPayment(recipientAddress, 100000)
```

## 7. SPV with BEEF

### Combining SPV and BEEF

Reference: **[BEEF Component - SPV Integration](../../../sdk-components/beef/README.md#key-features)**

```typescript
import { Beef, Transaction, MerklePath, ChainTracker } from '@bsv/sdk'

/**
 * Verify BEEF bundle with SPV proofs
 */
async function verifyBEEFWithSPV(
  beefHex: string,
  chainTracker: ChainTracker
): Promise<boolean> {
  // Parse BEEF
  const beef = Beef.fromBinary(Array.from(Buffer.from(beefHex, 'hex')))

  // Get all transactions
  const transactions = beef.txs

  // Verify each transaction with its merkle proof
  for (const tx of transactions) {
    if (!tx.merklePath) {
      console.log(`Transaction ${tx.id('hex')} has no merkle proof (unconfirmed)`)
      continue
    }

    // Verify merkle proof
    const txid = tx.id('hex') as string
    const computedRoot = tx.merklePath.computeRoot(txid)

    const isValid = await chainTracker.isValidRootForHeight(
      computedRoot,
      tx.merklePath.blockHeight
    )

    if (!isValid) {
      console.error(`SPV verification failed for ${txid}`)
      return false
    }
  }

  return true
}

/**
 * Create BEEF bundle with SPV proofs
 */
async function createBEEFWithSPV(
  transactions: Transaction[],
  proofs: Map<string, MerklePath>
): Promise<Beef> {
  const beef = new Beef()

  for (const tx of transactions) {
    const txid = tx.id('hex') as string
    const proof = proofs.get(txid)

    if (proof) {
      // Attach merkle proof to transaction
      tx.merklePath = proof
    }

    beef.mergeRawTx(tx.toBinary())
  }

  return beef
}

// Usage
const beef = await createBEEFWithSPV(transactions, merkleProofs)
const beefHex = Buffer.from(beef.toBinary()).toString('hex')

// Later, verify BEEF with SPV
const isValid = await verifyBEEFWithSPV(beefHex, chainTracker)
console.log('BEEF bundle verified:', isValid)
```

### BEEF-Based SPV Client

```typescript
/**
 * SPV client that works with BEEF bundles
 */
class BEEFSPVClient {
  private chainTracker: ChainTracker
  private verifiedBundles: Map<string, BEEF>

  constructor(chainTracker: ChainTracker) {
    this.chainTracker = chainTracker
    this.verifiedBundles = new Map()
  }

  /**
   * Verify complete BEEF bundle
   */
  async verifyBundle(beef: BEEF): Promise<boolean> {
    const transactions = beef.txs

    // Verify each transaction
    for (const tx of transactions) {
      // Check if transaction has SPV proof
      if (!tx.merklePath) {
        // Unconfirmed transaction - verify inputs instead
        const inputsValid = await this.verifyUnconfirmedInputs(tx, beef)
        if (!inputsValid) {
          return false
        }
        continue
      }

      // Verify SPV proof
      const txid = tx.id('hex') as string
      const computedRoot = tx.merklePath.computeRoot(txid)

      const isValid = await this.chainTracker.isValidRootForHeight(
        computedRoot,
        tx.merklePath.blockHeight
      )

      if (!isValid) {
        console.error(`Invalid SPV proof for ${txid}`)
        return false
      }
    }

    // All transactions verified
    const bundleId = this.getBundleId(beef)
    this.verifiedBundles.set(bundleId, beef)

    return true
  }

  /**
   * Verify unconfirmed transaction by checking inputs
   */
  private async verifyUnconfirmedInputs(
    tx: Transaction,
    beef: BEEF
  ): Promise<boolean> {
    // For unconfirmed tx, verify all inputs are in BEEF and verified
    for (const input of tx.inputs) {
      if (!input.sourceTransaction) {
        console.error('Missing source transaction for input')
        return false
      }

      // Check if source transaction is verified
      const sourceTx = input.sourceTransaction

      if (!sourceTx.merklePath) {
        console.error('Input source transaction not verified')
        return false
      }

      // Source must be verified via SPV
      const txid = sourceTx.id('hex') as string
      const computedRoot = sourceTx.merklePath.computeRoot(txid)

      const isValid = await this.chainTracker.isValidRootForHeight(
        computedRoot,
        sourceTx.merklePath.blockHeight
      )

      if (!isValid) {
        return false
      }
    }

    return true
  }

  /**
   * Get transaction from verified bundles
   */
  getTransaction(txid: string): Transaction | undefined {
    for (const beef of this.verifiedBundles.values()) {
      const tx = beef.txs.find(t => t.id('hex') === txid)
      if (tx) return tx
    }
    return undefined
  }

  private getBundleId(beef: BEEF): string {
    // Use hash of BEEF binary as ID
    const binary = beef.toBinary()
    return Hash.hash256(Buffer.from(binary)).toString('hex')
  }
}

// Usage
const beefClient = new BEEFSPVClient(chainTracker)

const beef = Beef.fromBinary(Array.from(Buffer.from(beefHex, 'hex')))
const isValid = await beefClient.verifyBundle(beef)

if (isValid) {
  console.log('BEEF bundle fully verified with SPV')
}
```

## 8. Production SPV Implementation

### Complete SPV System

```typescript
import { Utils } from '@bsv/sdk'

/**
 * Production-ready SPV system
 * Combines header tracking, merkle verification, and transaction management
 */
class ProductionSPVSystem {
  private headerTracker: LocalHeaderTracker
  private chainTracker: ChainTracker
  private spvClient: SPVClient
  private checkpointValidator: CheckpointValidator

  constructor(network: 'main' | 'test' = 'main') {
    this.headerTracker = new LocalHeaderTracker()
    this.chainTracker = new WhatsOnChainTracker(network)
    this.spvClient = new SPVClient(this.headerTracker)
    this.checkpointValidator = new CheckpointValidator()
  }

  /**
   * Initialize SPV system
   */
  async initialize(startHeight: number, endHeight: number): Promise<void> {
    console.log(`Syncing headers from ${startHeight} to ${endHeight}...`)

    // Sync block headers
    await this.syncHeaders(startHeight, endHeight)

    console.log('SPV system initialized')
  }

  /**
   * Sync block headers from remote source
   */
  private async syncHeaders(
    startHeight: number,
    endHeight: number
  ): Promise<void> {
    const batchSize = 100

    for (let height = startHeight; height <= endHeight; height += batchSize) {
      const batchEnd = Math.min(height + batchSize - 1, endHeight)

      console.log(`Syncing headers ${height} to ${batchEnd}`)

      await this.headerTracker.syncHeaders(
        height,
        batchEnd,
        this.chainTracker as WhatsOnChainTracker
      )

      // Validate against checkpoints
      const header = this.headerTracker.getHeader(height)
      if (header) {
        const validator = new HeaderChainValidator()
        const blockHash = validator.getBlockHash(header)

        if (!this.checkpointValidator.verifyAgainstCheckpoints(height, blockHash)) {
          throw new Error(`Checkpoint validation failed at height ${height}`)
        }
      }
    }
  }

  /**
   * Verify transaction with full SPV validation
   */
  async verifyTransaction(
    tx: Transaction,
    merklePath: MerklePath
  ): Promise<{
    valid: boolean
    confirmations: number
    blockHeight: number
  }> {
    // Verify merkle proof
    const isValid = await this.spvClient.verifyTransaction(tx, merklePath)

    if (!isValid) {
      return {
        valid: false,
        confirmations: 0,
        blockHeight: 0
      }
    }

    // Calculate confirmations
    const currentHeight = this.headerTracker.getCurrentHeight()
    const txHeight = merklePath.blockHeight
    const confirmations = currentHeight - txHeight + 1

    return {
      valid: true,
      confirmations,
      blockHeight: txHeight
    }
  }

  /**
   * Verify payment received
   */
  async verifyPayment(
    tx: Transaction,
    merklePath: MerklePath,
    expectedAddress: string,
    expectedAmount: number
  ): Promise<boolean> {
    // Verify transaction is in blockchain
    const { valid, confirmations } = await this.verifyTransaction(tx, merklePath)

    if (!valid) {
      console.error('Transaction not verified in blockchain')
      return false
    }

    // Verify output to expected address
    let foundPayment = false

    for (const output of tx.outputs) {
      if (this.isOutputToAddress(output.lockingScript, expectedAddress)) {
        if (output.satoshis === expectedAmount) {
          foundPayment = true
          break
        }
      }
    }

    if (!foundPayment) {
      console.error('Expected payment output not found')
      return false
    }

    console.log(`Payment verified: ${expectedAmount} satoshis with ${confirmations} confirmations`)
    return true
  }

  /**
   * Monitor transaction until confirmed
   */
  async waitForConfirmation(
    txid: string,
    requiredConfirmations: number = 6,
    timeoutMs: number = 3600000 // 1 hour
  ): Promise<boolean> {
    const startTime = Date.now()

    while (Date.now() - startTime < timeoutMs) {
      const status = this.spvClient.getTransactionStatus(txid)

      if (status === 'verified') {
        // Check confirmations
        const tx = this.spvClient.getTransaction(txid)
        if (tx && tx.merklePath) {
          const currentHeight = this.headerTracker.getCurrentHeight()
          const confirmations = currentHeight - tx.merklePath.blockHeight + 1

          if (confirmations >= requiredConfirmations) {
            console.log(`Transaction ${txid} has ${confirmations} confirmations`)
            return true
          }
        }
      }

      // Wait 30 seconds before checking again
      await new Promise(resolve => setTimeout(resolve, 30000))

      // Sync latest headers
      const currentHeight = this.headerTracker.getCurrentHeight()
      await this.syncHeaders(currentHeight + 1, currentHeight + 10)
    }

    console.error(`Timeout waiting for confirmation of ${txid}`)
    return false
  }

  /**
   * Get current blockchain state
   */
  getBlockchainInfo(): {
    height: number
    verifiedTxCount: number
  } {
    return {
      height: this.headerTracker.getCurrentHeight(),
      verifiedTxCount: this.spvClient.getVerifiedCount()
    }
  }

  private isOutputToAddress(
    lockingScript: LockingScript,
    address: string
  ): boolean {
    // Simplified - in production, properly parse script
    const scriptHex = lockingScript.toHex()
    const decoded = Utils.fromBase58Check(address)
    const addressHash = Buffer.from(decoded.data).toString('hex')
    return scriptHex.includes(addressHash)
  }
}

// Usage
const spvSystem = new ProductionSPVSystem('main')

// Initialize with recent headers
const currentHeight = 800000
await spvSystem.initialize(currentHeight - 1000, currentHeight)

// Verify payment
const paymentTx = Transaction.fromHex('...')
const paymentProof = MerklePath.fromHex('...')

const isValidPayment = await spvSystem.verifyPayment(
  paymentTx,
  paymentProof,
  '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
  100000 // 100,000 satoshis
)

if (isValidPayment) {
  console.log('Payment verified and accepted!')
}

// Wait for confirmations
const txid = paymentTx.id('hex') as string
await spvSystem.waitForConfirmation(txid, 6)
```

### SPV Payment Processor

```typescript
/**
 * Payment processor using SPV for lightweight verification
 */
class SPVPaymentProcessor {
  private spvSystem: ProductionSPVSystem
  private payments: Map<string, PaymentRecord>

  constructor(spvSystem: ProductionSPVSystem) {
    this.spvSystem = spvSystem
    this.payments = new Map()
  }

  /**
   * Process incoming payment
   */
  async processPayment(
    paymentId: string,
    tx: Transaction,
    merklePath: MerklePath,
    expectedAddress: string,
    expectedAmount: number
  ): Promise<void> {
    console.log(`Processing payment ${paymentId}`)

    // Verify payment
    const isValid = await this.spvSystem.verifyPayment(
      tx,
      merklePath,
      expectedAddress,
      expectedAmount
    )

    if (!isValid) {
      this.payments.set(paymentId, {
        status: 'failed',
        txid: tx.id('hex') as string,
        timestamp: Date.now()
      })
      throw new Error('Payment verification failed')
    }

    // Record payment
    this.payments.set(paymentId, {
      status: 'verified',
      txid: tx.id('hex') as string,
      amount: expectedAmount,
      timestamp: Date.now()
    })

    console.log(`Payment ${paymentId} verified`)

    // Wait for confirmations in background
    this.waitForConfirmations(paymentId, tx.id('hex') as string)
  }

  /**
   * Wait for confirmations and update status
   */
  private async waitForConfirmations(
    paymentId: string,
    txid: string
  ): Promise<void> {
    const confirmed = await this.spvSystem.waitForConfirmation(txid, 6)

    if (confirmed) {
      const payment = this.payments.get(paymentId)
      if (payment) {
        payment.status = 'confirmed'
        this.payments.set(paymentId, payment)
        console.log(`Payment ${paymentId} confirmed`)
      }
    }
  }

  /**
   * Get payment status
   */
  getPaymentStatus(paymentId: string): string {
    const payment = this.payments.get(paymentId)
    return payment?.status || 'unknown'
  }
}

interface PaymentRecord {
  status: 'verified' | 'confirmed' | 'failed'
  txid: string
  amount?: number
  timestamp: number
}
```

## Best Practices

1. **Always verify merkle proofs** before trusting transactions
2. **Use checkpoints** for known valid block hashes
3. **Implement header caching** to reduce network requests
4. **Validate proof-of-work** on all block headers
5. **Monitor chain reorganizations** and handle them gracefully
6. **Cache verification results** for performance
7. **Use multiple data sources** for redundancy
8. **Implement timeout mechanisms** for network requests
9. **Validate transaction structure** in addition to SPV proofs
10. **Keep headers synced** with blockchain tip

## Common Pitfalls

1. **Not validating proof-of-work** - Headers could be fake
2. **Trusting single data source** - Network could be compromised
3. **Ignoring chain reorganizations** - Transactions can become invalid
4. **Missing input validation** - SPV doesn't verify transaction rules
5. **Insufficient confirmations** - Transactions could be reversed
6. **Memory leaks** from unbounded caching
7. **Not handling network failures** gracefully

**BAD Example - Insufficient Validation:**
```typescript
// ❌ BAD: Only checking merkle proof exists
async function acceptPayment(tx: Transaction, proof: any): Promise<void> {
  if (proof) {
    await processPayment(tx)
  }
}
```

**GOOD Example - Complete Validation:**
```typescript
// ✅ GOOD: Full SPV verification
async function acceptPayment(
  tx: Transaction,
  merklePath: MerklePath,
  chainTracker: ChainTracker,
  minConfirmations: number = 6
): Promise<void> {
  // Verify merkle proof
  const txid = tx.id('hex') as string
  const computedRoot = merklePath.computeRoot(txid)
  const isValid = await chainTracker.isValidRootForHeight(
    computedRoot,
    merklePath.blockHeight
  )

  if (!isValid) {
    throw new Error('Invalid merkle proof')
  }

  // Check confirmations
  const currentHeight = await chainTracker.getCurrentHeight()
  const confirmations = currentHeight - merklePath.blockHeight + 1

  if (confirmations < minConfirmations) {
    throw new Error(`Insufficient confirmations: ${confirmations}`)
  }

  // Now safe to process
  await processPayment(tx)
}
```

## Hands-On Project: SPV Payment Gateway

Build a complete payment gateway that:
- Accepts payments with SPV verification
- Monitors confirmations
- Handles multiple payments concurrently
- Provides payment status API
- Uses local header caching

```typescript
class SPVPaymentGateway {
  private spvSystem: ProductionSPVSystem
  private processor: SPVPaymentProcessor

  async initialize(): Promise<void> {
    // Initialize SPV system
    const currentHeight = await this.getCurrentHeight()
    await this.spvSystem.initialize(currentHeight - 2000, currentHeight)
  }

  async acceptPayment(
    invoice: Invoice,
    tx: Transaction,
    merklePath: MerklePath
  ): Promise<PaymentReceipt> {
    // Verify and process payment
    await this.processor.processPayment(
      invoice.id,
      tx,
      merklePath,
      invoice.address,
      invoice.amount
    )

    return {
      invoiceId: invoice.id,
      txid: tx.id('hex') as string,
      status: 'verified',
      timestamp: Date.now()
    }
  }

  async getPaymentStatus(invoiceId: string): Promise<string> {
    return this.processor.getPaymentStatus(invoiceId)
  }
}
```

## Next Steps

Continue to:
- **[Advanced Scripting](../../advanced/advanced-scripting/README.md)** - Complex smart contracts
- **[Overlay Networks](../../advanced/overlay-networks/README.md)** - Build scalable applications

## Additional Resources

- [SPV SDK Component](../../../sdk-components/spv/README.md)
- [Merkle Proofs SDK Component](../../../sdk-components/merkle-proofs/README.md)
- [ChainTracker SDK Component](../../../sdk-components/chain-tracker/README.md)
- [BEEF SDK Component](../../../sdk-components/beef/README.md)
- [BRC-8: Transaction Envelopes](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0008.md)
- [BRC-9: SPV Implementation](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0009.md)
- [BRC-67: SPV Validation Rules](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0067.md)
- [Bitcoin Whitepaper - Section 8: Simplified Payment Verification](https://bitcoinsv.io/bitcoin.pdf)

---

**Status:** ✅ Complete - Ready for learning
