# SPV Implementation

Complete examples for implementing Simplified Payment Verification (SPV) clients and systems for BSV.

## Overview

Simplified Payment Verification (SPV) allows lightweight clients to verify transactions without downloading the entire blockchain. This guide demonstrates building SPV clients, managing block headers, creating merkle proofs, and implementing efficient payment verification systems.

**Related SDK Components:**
- [Merkle Proof](../../sdk-components/merkle-proof/README.md)
- [Block Headers](../../sdk-components/block-headers/README.md)
- [Chain Tracker](../../sdk-components/chain-tracker/README.md)
- [Transaction](../../sdk-components/transaction/README.md)

## Basic SPV Client

```typescript
import { Transaction, BlockHeader, MerkleProof, Hash } from '@bsv/sdk'

/**
 * Basic SPV Client
 *
 * Lightweight client for verifying transactions using SPV
 */
class BasicSPVClient {
  private headers: Map<string, StoredHeader> = new Map()
  private verifiedTxs: Set<string> = new Set()
  private chainTip: string | null = null

  /**
   * Add block header to the client
   */
  addHeader(header: BlockHeader): void {
    try {
      const blockHash = header.hash()

      console.log('Adding block header:', blockHash)

      // Store header
      this.headers.set(blockHash, {
        header,
        height: this.calculateHeight(header),
        addedAt: Date.now()
      })

      // Update chain tip
      if (!this.chainTip || this.isNewTip(header)) {
        this.chainTip = blockHash
        console.log('New chain tip:', blockHash)
      }

      console.log('Header added successfully')
    } catch (error) {
      throw new Error(`Failed to add header: ${error.message}`)
    }
  }

  /**
   * Get block header by hash
   */
  getHeader(blockHash: string): BlockHeader | null {
    const stored = this.headers.get(blockHash)
    return stored ? stored.header : null
  }

  /**
   * Verify transaction with merkle proof
   */
  async verifyTransaction(
    tx: Transaction,
    blockHash: string,
    merkleProof: MerkleProofData
  ): Promise<VerificationResult> {
    try {
      const txid = tx.id('hex')

      console.log('Verifying transaction:', txid)
      console.log('Block:', blockHash)

      // Get header
      const stored = this.headers.get(blockHash)
      if (!stored) {
        return {
          verified: false,
          error: 'Block header not found'
        }
      }

      // Verify merkle proof
      const proofValid = this.verifyMerkleProof(
        txid,
        merkleProof,
        stored.header.merkleRoot
      )

      if (!proofValid) {
        return {
          verified: false,
          error: 'Invalid merkle proof'
        }
      }

      // Mark as verified
      this.verifiedTxs.add(txid)

      console.log('Transaction verified successfully')

      return {
        verified: true,
        blockHeight: stored.height,
        confirmations: this.calculateConfirmations(blockHash)
      }
    } catch (error) {
      return {
        verified: false,
        error: error.message
      }
    }
  }

  /**
   * Verify merkle proof
   */
  private verifyMerkleProof(
    txid: string,
    proof: MerkleProofData,
    merkleRoot: string
  ): boolean {
    try {
      let hash = Hash.sha256(Hash.sha256(Buffer.from(txid, 'hex')))
      let index = proof.index

      // Climb the merkle tree
      for (const sibling of proof.nodes) {
        const siblingBuffer = Buffer.from(sibling, 'hex')
        const isRightNode = index % 2 === 1

        const combined = isRightNode
          ? Buffer.concat([siblingBuffer, hash])
          : Buffer.concat([hash, siblingBuffer])

        hash = Hash.sha256(Hash.sha256(combined))
        index = Math.floor(index / 2)
      }

      return hash.toString('hex') === merkleRoot
    } catch (error) {
      console.error('Merkle proof verification failed:', error.message)
      return false
    }
  }

  /**
   * Calculate block height
   */
  private calculateHeight(header: BlockHeader): number {
    // Find previous header and increment height
    const prevHeader = this.headers.get(header.prevBlockHash)
    return prevHeader ? prevHeader.height + 1 : 0
  }

  /**
   * Check if header is new chain tip
   */
  private isNewTip(header: BlockHeader): boolean {
    if (!this.chainTip) return true

    const currentTip = this.headers.get(this.chainTip)
    if (!currentTip) return true

    const newHeight = this.calculateHeight(header)
    return newHeight > currentTip.height
  }

  /**
   * Calculate confirmations for a block
   */
  private calculateConfirmations(blockHash: string): number {
    if (!this.chainTip) return 0

    const block = this.headers.get(blockHash)
    const tip = this.headers.get(this.chainTip)

    if (!block || !tip) return 0

    return Math.max(0, tip.height - block.height + 1)
  }

  /**
   * Check if transaction has been verified
   */
  isVerified(txid: string): boolean {
    return this.verifiedTxs.has(txid)
  }

  /**
   * Get client statistics
   */
  getStats(): SPVStats {
    return {
      headersCount: this.headers.size,
      verifiedTxsCount: this.verifiedTxs.size,
      chainTip: this.chainTip,
      tipHeight: this.chainTip
        ? this.headers.get(this.chainTip)?.height || 0
        : 0
    }
  }

  /**
   * Sync headers from a source
   */
  async syncHeaders(
    startHeight: number,
    count: number,
    headerSource: (height: number) => Promise<BlockHeader>
  ): Promise<number> {
    console.log(`Syncing ${count} headers from height ${startHeight}`)

    let synced = 0

    for (let i = 0; i < count; i++) {
      const height = startHeight + i

      try {
        const header = await headerSource(height)
        this.addHeader(header)
        synced++

        if ((i + 1) % 100 === 0) {
          console.log(`Synced ${i + 1}/${count} headers`)
        }
      } catch (error) {
        console.error(`Failed to sync header at height ${height}:`, error.message)
        break
      }
    }

    console.log(`Sync complete: ${synced} headers synced`)

    return synced
  }
}

interface StoredHeader {
  header: BlockHeader
  height: number
  addedAt: number
}

interface MerkleProofData {
  index: number
  nodes: string[]
}

interface VerificationResult {
  verified: boolean
  blockHeight?: number
  confirmations?: number
  error?: string
}

interface SPVStats {
  headersCount: number
  verifiedTxsCount: number
  chainTip: string | null
  tipHeight: number
}

/**
 * Usage Example
 */
async function basicSPVExample() {
  const client = new BasicSPVClient()

  console.log('=== Basic SPV Client ===')

  // Add block headers
  const header = new BlockHeader({
    version: 1,
    prevBlockHash: '0000000000000000000000000000000000000000000000000000000000000000',
    merkleRoot: 'merkle-root-hash...',
    time: Date.now(),
    bits: 0x1d00ffff,
    nonce: 12345
  })

  client.addHeader(header)

  // Verify transaction
  const tx = Transaction.fromHex('tx-hex...')
  const blockHash = header.hash()

  const merkleProof: MerkleProofData = {
    index: 0,
    nodes: ['node1...', 'node2...']
  }

  const result = await client.verifyTransaction(tx, blockHash, merkleProof)

  console.log('Verification result:', result)

  // Get statistics
  const stats = client.getStats()
  console.log('SPV Client stats:', stats)
}
```

## Advanced SPV Client with Header Chain Management

```typescript
import { BlockHeader, Transaction, MerkleProof, Hash } from '@bsv/sdk'

/**
 * Advanced SPV Client
 *
 * Full-featured SPV client with header chain validation and reorganization handling
 */
class AdvancedSPVClient {
  private headers: Map<string, ChainHeader> = new Map()
  private heightIndex: Map<number, string> = new Map()
  private chainTip: ChainHeader | null = null
  private genesisHash: string

  constructor(genesisHash: string) {
    this.genesisHash = genesisHash
  }

  /**
   * Add and validate block header
   */
  async addHeader(header: BlockHeader): Promise<AddHeaderResult> {
    try {
      const blockHash = header.hash()

      console.log('Processing header:', blockHash)

      // Check if already exists
      if (this.headers.has(blockHash)) {
        return {
          success: true,
          action: 'duplicate',
          blockHash
        }
      }

      // Validate header
      const validation = this.validateHeader(header)
      if (!validation.valid) {
        return {
          success: false,
          error: validation.error
        }
      }

      // Find previous header
      const prevHeader = this.headers.get(header.prevBlockHash)
      if (!prevHeader && header.prevBlockHash !== this.genesisHash) {
        return {
          success: false,
          error: 'Previous header not found'
        }
      }

      // Calculate height and work
      const height = prevHeader ? prevHeader.height + 1 : 0
      const chainWork = this.calculateChainWork(header, prevHeader)

      // Create chain header
      const chainHeader: ChainHeader = {
        header,
        blockHash,
        height,
        chainWork,
        timestamp: Date.now()
      }

      // Add to storage
      this.headers.set(blockHash, chainHeader)
      this.heightIndex.set(height, blockHash)

      console.log('Header added at height:', height)

      // Check for reorganization
      const reorg = await this.checkReorganization(chainHeader)

      if (reorg) {
        console.log('Chain reorganization detected')
        return {
          success: true,
          action: 'reorganization',
          blockHash,
          height,
          reorgHeight: reorg.reorgHeight
        }
      }

      // Update chain tip if this is the new best chain
      if (!this.chainTip || chainWork > this.chainTip.chainWork) {
        this.chainTip = chainHeader
        console.log('New chain tip:', blockHash)

        return {
          success: true,
          action: 'extended',
          blockHash,
          height
        }
      }

      return {
        success: true,
        action: 'side_chain',
        blockHash,
        height
      }
    } catch (error) {
      return {
        success: false,
        error: error.message
      }
    }
  }

  /**
   * Validate block header
   */
  private validateHeader(header: BlockHeader): { valid: boolean; error?: string } {
    try {
      // Check proof of work
      const blockHash = header.hash()
      const target = this.calculateTarget(header.bits)

      if (!this.hashMeetsTarget(blockHash, target)) {
        return {
          valid: false,
          error: 'Insufficient proof of work'
        }
      }

      // Check timestamp (not too far in future)
      const maxTime = Date.now() + 2 * 60 * 60 * 1000 // 2 hours
      if (header.time > maxTime) {
        return {
          valid: false,
          error: 'Timestamp too far in future'
        }
      }

      return { valid: true }
    } catch (error) {
      return {
        valid: false,
        error: error.message
      }
    }
  }

  /**
   * Calculate chain work
   */
  private calculateChainWork(
    header: BlockHeader,
    prevHeader?: ChainHeader
  ): number {
    const blockWork = this.calculateBlockWork(header.bits)
    const prevWork = prevHeader ? prevHeader.chainWork : 0

    return prevWork + blockWork
  }

  /**
   * Calculate block work from bits
   */
  private calculateBlockWork(bits: number): number {
    // Simplified work calculation
    const target = this.calculateTarget(bits)
    return Math.pow(2, 256) / (target + 1)
  }

  /**
   * Calculate target from bits
   */
  private calculateTarget(bits: number): number {
    // Simplified target calculation
    const exponent = bits >> 24
    const mantissa = bits & 0xffffff
    return mantissa * Math.pow(256, exponent - 3)
  }

  /**
   * Check if hash meets target
   */
  private hashMeetsTarget(hash: string, target: number): boolean {
    // Simplified check
    const hashNum = parseInt(hash.substring(0, 16), 16)
    return hashNum <= target
  }

  /**
   * Check for chain reorganization
   */
  private async checkReorganization(
    newHeader: ChainHeader
  ): Promise<ReorgResult | null> {
    if (!this.chainTip) return null

    // If new header extends current tip, no reorg
    if (newHeader.header.prevBlockHash === this.chainTip.blockHash) {
      return null
    }

    // If new chain has more work, reorganize
    if (newHeader.chainWork > this.chainTip.chainWork) {
      const forkPoint = this.findForkPoint(newHeader, this.chainTip)

      if (forkPoint) {
        console.log('Reorganizing from height:', forkPoint.height)

        // Update chain tip
        this.chainTip = newHeader

        return {
          reorgHeight: forkPoint.height,
          oldTip: this.chainTip.blockHash,
          newTip: newHeader.blockHash
        }
      }
    }

    return null
  }

  /**
   * Find fork point between two chains
   */
  private findForkPoint(
    header1: ChainHeader,
    header2: ChainHeader
  ): ChainHeader | null {
    let h1 = header1
    let h2 = header2

    // Walk back to same height
    while (h1.height > h2.height) {
      const prev = this.headers.get(h1.header.prevBlockHash)
      if (!prev) return null
      h1 = prev
    }

    while (h2.height > h1.height) {
      const prev = this.headers.get(h2.header.prevBlockHash)
      if (!prev) return null
      h2 = prev
    }

    // Walk back until we find common ancestor
    while (h1.blockHash !== h2.blockHash) {
      const prev1 = this.headers.get(h1.header.prevBlockHash)
      const prev2 = this.headers.get(h2.header.prevBlockHash)

      if (!prev1 || !prev2) return null

      h1 = prev1
      h2 = prev2
    }

    return h1
  }

  /**
   * Get header by height
   */
  getHeaderByHeight(height: number): ChainHeader | null {
    const blockHash = this.heightIndex.get(height)
    return blockHash ? this.headers.get(blockHash) || null : null
  }

  /**
   * Get headers in range
   */
  getHeadersRange(startHeight: number, count: number): ChainHeader[] {
    const headers: ChainHeader[] = []

    for (let i = 0; i < count; i++) {
      const header = this.getHeaderByHeight(startHeight + i)
      if (header) {
        headers.push(header)
      } else {
        break
      }
    }

    return headers
  }

  /**
   * Get chain tip
   */
  getChainTip(): ChainHeader | null {
    return this.chainTip
  }

  /**
   * Get confirmations for a block
   */
  getConfirmations(blockHash: string): number {
    const header = this.headers.get(blockHash)
    if (!header || !this.chainTip) return 0

    return Math.max(0, this.chainTip.height - header.height + 1)
  }

  /**
   * Verify transaction with enhanced validation
   */
  async verifyTransactionAdvanced(
    tx: Transaction,
    blockHash: string,
    merkleProof: MerkleProofData,
    requiredConfirmations: number = 6
  ): Promise<AdvancedVerificationResult> {
    try {
      const header = this.headers.get(blockHash)
      if (!header) {
        return {
          verified: false,
          error: 'Block header not found'
        }
      }

      // Verify merkle proof
      const txid = tx.id('hex')
      const proofValid = this.verifyMerkleProof(
        txid,
        merkleProof,
        header.header.merkleRoot
      )

      if (!proofValid) {
        return {
          verified: false,
          error: 'Invalid merkle proof'
        }
      }

      // Check confirmations
      const confirmations = this.getConfirmations(blockHash)

      if (confirmations < requiredConfirmations) {
        return {
          verified: true,
          confirmed: false,
          confirmations,
          requiredConfirmations,
          blockHeight: header.height
        }
      }

      return {
        verified: true,
        confirmed: true,
        confirmations,
        requiredConfirmations,
        blockHeight: header.height
      }
    } catch (error) {
      return {
        verified: false,
        error: error.message
      }
    }
  }

  /**
   * Verify merkle proof
   */
  private verifyMerkleProof(
    txid: string,
    proof: MerkleProofData,
    merkleRoot: string
  ): boolean {
    try {
      let hash = Hash.sha256(Hash.sha256(Buffer.from(txid, 'hex')))
      let index = proof.index

      for (const sibling of proof.nodes) {
        const siblingBuffer = Buffer.from(sibling, 'hex')
        const isRightNode = index % 2 === 1

        const combined = isRightNode
          ? Buffer.concat([siblingBuffer, hash])
          : Buffer.concat([hash, siblingBuffer])

        hash = Hash.sha256(Hash.sha256(combined))
        index = Math.floor(index / 2)
      }

      return hash.toString('hex') === merkleRoot
    } catch (error) {
      return false
    }
  }

  /**
   * Get client status
   */
  getStatus(): SPVClientStatus {
    return {
      headersCount: this.headers.size,
      tipHeight: this.chainTip?.height || 0,
      tipHash: this.chainTip?.blockHash || null,
      chainWork: this.chainTip?.chainWork || 0
    }
  }
}

interface ChainHeader {
  header: BlockHeader
  blockHash: string
  height: number
  chainWork: number
  timestamp: number
}

interface AddHeaderResult {
  success: boolean
  action?: 'duplicate' | 'extended' | 'side_chain' | 'reorganization'
  blockHash?: string
  height?: number
  reorgHeight?: number
  error?: string
}

interface ReorgResult {
  reorgHeight: number
  oldTip: string
  newTip: string
}

interface MerkleProofData {
  index: number
  nodes: string[]
}

interface AdvancedVerificationResult {
  verified: boolean
  confirmed?: boolean
  confirmations?: number
  requiredConfirmations?: number
  blockHeight?: number
  error?: string
}

interface SPVClientStatus {
  headersCount: number
  tipHeight: number
  tipHash: string | null
  chainWork: number
}

/**
 * Usage Example
 */
async function advancedSPVExample() {
  const client = new AdvancedSPVClient('genesis-hash...')

  console.log('=== Advanced SPV Client ===')

  // Add headers
  const header1 = new BlockHeader({
    version: 1,
    prevBlockHash: 'genesis-hash...',
    merkleRoot: 'merkle-root-1...',
    time: Date.now(),
    bits: 0x1d00ffff,
    nonce: 12345
  })

  const result1 = await client.addHeader(header1)
  console.log('Add header result:', result1)

  // Check status
  const status = client.getStatus()
  console.log('Client status:', status)

  // Verify transaction
  const tx = Transaction.fromHex('tx-hex...')
  const proof: MerkleProofData = {
    index: 0,
    nodes: ['node1...', 'node2...']
  }

  const verification = await client.verifyTransactionAdvanced(
    tx,
    header1.hash(),
    proof,
    6
  )

  console.log('Verification result:', verification)
}
```

## SPV Payment Verification System

```typescript
import { Transaction, BlockHeader, Hash } from '@bsv/sdk'

/**
 * SPV Payment Verification System
 *
 * Verify payments to specific addresses using SPV
 */
class SPVPaymentVerifier {
  private spvClient: BasicSPVClient
  private payments: Map<string, Payment> = new Map()

  constructor(spvClient: BasicSPVClient) {
    this.spvClient = spvClient
  }

  /**
   * Wait for payment to address
   */
  async waitForPayment(
    toAddress: string,
    expectedAmount: number,
    timeout: number = 60000
  ): Promise<PaymentResult> {
    console.log('Waiting for payment...')
    console.log('Address:', toAddress)
    console.log('Amount:', expectedAmount, 'satoshis')

    const paymentId = `${toAddress}:${expectedAmount}`

    const payment: Payment = {
      address: toAddress,
      expectedAmount,
      startTime: Date.now(),
      timeout,
      status: 'waiting'
    }

    this.payments.set(paymentId, payment)

    return new Promise((resolve, reject) => {
      const checkInterval = setInterval(async () => {
        // Check timeout
        if (Date.now() - payment.startTime > timeout) {
          clearInterval(checkInterval)
          payment.status = 'timeout'

          reject(new Error('Payment timeout'))
          return
        }

        // In production, would check for new transactions
        // This is simplified for demonstration
      }, 1000)
    })
  }

  /**
   * Verify payment transaction
   */
  async verifyPayment(
    tx: Transaction,
    toAddress: string,
    expectedAmount: number,
    blockHash: string,
    merkleProof: MerkleProofData
  ): Promise<PaymentVerification> {
    try {
      console.log('Verifying payment transaction')

      // Verify transaction in blockchain
      const verification = await this.spvClient.verifyTransaction(
        tx,
        blockHash,
        merkleProof
      )

      if (!verification.verified) {
        return {
          valid: false,
          error: verification.error
        }
      }

      // Check if transaction pays to address
      let paidAmount = 0

      for (const output of tx.outputs) {
        try {
          const outputAddress = output.lockingScript.toAddress()

          if (outputAddress === toAddress) {
            paidAmount += output.satoshis
          }
        } catch (error) {
          // Skip outputs that can't be converted to addresses
          continue
        }
      }

      // Check if amount meets expectation
      if (paidAmount < expectedAmount) {
        return {
          valid: false,
          error: `Insufficient amount: got ${paidAmount}, expected ${expectedAmount}`
        }
      }

      console.log('Payment verified successfully')
      console.log('Paid:', paidAmount, 'satoshis')

      return {
        valid: true,
        txid: tx.id('hex'),
        paidAmount,
        confirmations: verification.confirmations
      }
    } catch (error) {
      return {
        valid: false,
        error: error.message
      }
    }
  }

  /**
   * Batch verify multiple payments
   */
  async batchVerifyPayments(
    payments: PaymentToVerify[]
  ): Promise<BatchPaymentResult[]> {
    console.log(`Batch verifying ${payments.length} payments`)

    const results: BatchPaymentResult[] = []

    for (let i = 0; i < payments.length; i++) {
      const payment = payments[i]

      try {
        console.log(`Verifying payment ${i + 1}/${payments.length}`)

        const result = await this.verifyPayment(
          payment.tx,
          payment.toAddress,
          payment.expectedAmount,
          payment.blockHash,
          payment.merkleProof
        )

        results.push({
          index: i,
          success: result.valid,
          txid: payment.tx.id('hex'),
          paidAmount: result.paidAmount,
          error: result.error
        })
      } catch (error) {
        results.push({
          index: i,
          success: false,
          txid: payment.tx.id('hex'),
          error: error.message
        })
      }
    }

    const successful = results.filter(r => r.success).length
    console.log(`Batch verification complete: ${successful}/${payments.length} successful`)

    return results
  }

  /**
   * Get payment status
   */
  getPaymentStatus(paymentId: string): Payment | null {
    return this.payments.get(paymentId) || null
  }
}

interface Payment {
  address: string
  expectedAmount: number
  startTime: number
  timeout: number
  status: 'waiting' | 'verified' | 'timeout'
  txid?: string
}

interface PaymentResult {
  verified: boolean
  txid: string
  amount: number
  confirmations: number
}

interface MerkleProofData {
  index: number
  nodes: string[]
}

interface PaymentVerification {
  valid: boolean
  txid?: string
  paidAmount?: number
  confirmations?: number
  error?: string
}

interface PaymentToVerify {
  tx: Transaction
  toAddress: string
  expectedAmount: number
  blockHash: string
  merkleProof: MerkleProofData
}

interface BatchPaymentResult {
  index: number
  success: boolean
  txid: string
  paidAmount?: number
  error?: string
}

/**
 * Usage Example
 */
async function paymentVerificationExample() {
  const spvClient = new BasicSPVClient()
  const verifier = new SPVPaymentVerifier(spvClient)

  console.log('=== SPV Payment Verification ===')

  // Wait for payment
  const myAddress = '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa'
  const expectedAmount = 50000

  try {
    const payment = await verifier.waitForPayment(
      myAddress,
      expectedAmount,
      60000
    )

    console.log('Payment received:', payment)
  } catch (error) {
    console.error('Payment failed:', error.message)
  }

  // Verify specific payment
  const tx = Transaction.fromHex('tx-hex...')
  const blockHash = 'block-hash...'
  const merkleProof: MerkleProofData = {
    index: 0,
    nodes: ['node1...', 'node2...']
  }

  const verification = await verifier.verifyPayment(
    tx,
    myAddress,
    expectedAmount,
    blockHash,
    merkleProof
  )

  console.log('Verification result:', verification)
}
```

## Related Examples

- [SPV Verification](../spv-verification/README.md)
- [SPV Validation](../spv-validation/README.md)
- [Transaction Building](../transaction-building/README.md)
- [Block Headers](../block-headers/README.md)

## See Also

**SDK Components:**
- [Merkle Proof](../../sdk-components/merkle-proof/README.md) - Merkle proof operations
- [Block Headers](../../sdk-components/block-headers/README.md) - Block header handling
- [Chain Tracker](../../sdk-components/chain-tracker/README.md) - Chain synchronization
- [Transaction](../../sdk-components/transaction/README.md) - Transaction handling

**Learning Paths:**
- [SPV Basics](../../learning-paths/intermediate/spv-basics/README.md)
- [Lightweight Clients](../../learning-paths/advanced/lightweight-clients/README.md)
- [Payment Verification](../../learning-paths/intermediate/payment-verification/README.md)
