# SPV Verification

Complete examples for Simplified Payment Verification (SPV) using merkle proofs.

## Overview

This code feature demonstrates Simplified Payment Verification (SPV), which allows lightweight clients to verify that a transaction is included in a block without downloading the entire blockchain. SPV uses merkle proofs to cryptographically prove transaction inclusion with minimal data.

**Related SDK Components:**
- [Merkle Proof](../../sdk-components/merkle-proof/README.md)
- [Block Headers](../../sdk-components/block-headers/README.md)
- [Transaction](../../sdk-components/transaction/README.md)
- [Chain Tracker](../../sdk-components/chain-tracker/README.md)

## Basic Merkle Proof Verification

```typescript
import { MerkleProof, Transaction, Hash } from '@bsv/sdk'

/**
 * Basic SPV Verification
 *
 * Verify transaction inclusion using merkle proofs
 */
class BasicSPVVerification {
  /**
   * Verify a transaction is included in a block
   */
  verifyTransactionInclusion(
    txid: string,
    merkleProof: {
      index: number
      txOrId: string
      target: string
      nodes: string[]
    },
    blockHeader: {
      version: number
      prevBlockHash: string
      merkleRoot: string
      time: number
      bits: number
      nonce: number
    }
  ): boolean {
    try {
      // Create MerkleProof instance
      const proof = MerkleProof.fromHex(this.serializeMerkleProof(merkleProof))

      // Verify the proof
      const isValid = proof.verify(
        txid,
        Buffer.from(blockHeader.merkleRoot, 'hex')
      )

      return isValid
    } catch (error) {
      console.error('Verification failed:', error)
      return false
    }
  }

  /**
   * Serialize merkle proof to hex
   */
  private serializeMerkleProof(proof: {
    index: number
    txOrId: string
    target: string
    nodes: string[]
  }): string {
    // Implementation depends on SDK's MerkleProof format
    // This is a simplified example
    return JSON.stringify(proof)
  }

  /**
   * Calculate merkle root from transaction and proof
   */
  calculateMerkleRoot(txid: string, proof: string[]): string {
    let hash = Buffer.from(txid, 'hex')

    for (const node of proof) {
      const nodeBuffer = Buffer.from(node, 'hex')

      // Concatenate and hash
      const combined = Buffer.concat([hash, nodeBuffer])
      hash = Hash.sha256(Hash.sha256(combined))
    }

    return hash.toString('hex')
  }

  /**
   * Verify merkle root matches block header
   */
  verifyMerkleRoot(
    calculatedRoot: string,
    blockHeaderRoot: string
  ): boolean {
    return calculatedRoot === blockHeaderRoot
  }
}

/**
 * Usage Example
 */
async function basicSPVExample() {
  const spv = new BasicSPVVerification()

  // Transaction ID to verify
  const txid = 'abc123...'

  // Merkle proof (typically obtained from a block explorer or node)
  const merkleProof = {
    index: 0,
    txOrId: txid,
    target: 'merkle-root-hash...',
    nodes: [
      'hash1...',
      'hash2...',
      'hash3...'
    ]
  }

  // Block header (from chain)
  const blockHeader = {
    version: 1,
    prevBlockHash: 'prev-hash...',
    merkleRoot: 'merkle-root-hash...',
    time: 1234567890,
    bits: 0x1d00ffff,
    nonce: 12345
  }

  // Verify transaction inclusion
  const isValid = spv.verifyTransactionInclusion(
    txid,
    merkleProof,
    blockHeader
  )

  console.log('Transaction verified:', isValid)
}
```

## SPV Proof Construction and Verification

```typescript
import { MerkleProof, Transaction, Hash, BlockHeader } from '@bsv/sdk'

/**
 * SPV Proof Construction
 *
 * Build and verify merkle proofs for SPV
 */
class SPVProofConstruction {
  /**
   * Build merkle tree from transactions
   */
  buildMerkleTree(txids: string[]): {
    root: string
    tree: string[][]
  } {
    if (txids.length === 0) {
      throw new Error('No transactions provided')
    }

    // Start with transaction IDs as leaf nodes
    let currentLevel: string[] = txids.map(txid => {
      // Double SHA256 of txid
      const hash = Hash.sha256(Hash.sha256(Buffer.from(txid, 'hex')))
      return hash.toString('hex')
    })

    const tree: string[][] = [currentLevel]

    // Build tree levels
    while (currentLevel.length > 1) {
      const nextLevel: string[] = []

      for (let i = 0; i < currentLevel.length; i += 2) {
        const left = currentLevel[i]
        const right = i + 1 < currentLevel.length
          ? currentLevel[i + 1]
          : currentLevel[i] // Duplicate if odd number

        // Concatenate and hash
        const combined = Buffer.concat([
          Buffer.from(left, 'hex'),
          Buffer.from(right, 'hex')
        ])

        const parentHash = Hash.sha256(Hash.sha256(combined))
        nextLevel.push(parentHash.toString('hex'))
      }

      currentLevel = nextLevel
      tree.push(currentLevel)
    }

    return {
      root: currentLevel[0],
      tree
    }
  }

  /**
   * Generate merkle proof for specific transaction
   */
  generateMerkleProof(
    txid: string,
    txids: string[]
  ): {
    txid: string
    index: number
    proof: string[]
  } | null {
    const index = txids.indexOf(txid)
    if (index === -1) {
      return null
    }

    const { tree } = this.buildMerkleTree(txids)
    const proof: string[] = []

    let currentIndex = index

    // Collect sibling hashes at each level
    for (let level = 0; level < tree.length - 1; level++) {
      const levelNodes = tree[level]
      const isRightNode = currentIndex % 2 === 1
      const siblingIndex = isRightNode ? currentIndex - 1 : currentIndex + 1

      if (siblingIndex < levelNodes.length) {
        proof.push(levelNodes[siblingIndex])
      } else {
        // Duplicate node if no sibling
        proof.push(levelNodes[currentIndex])
      }

      currentIndex = Math.floor(currentIndex / 2)
    }

    return {
      txid,
      index,
      proof
    }
  }

  /**
   * Verify merkle proof
   */
  verifyMerkleProof(
    txid: string,
    index: number,
    proof: string[],
    merkleRoot: string
  ): boolean {
    try {
      // Start with transaction hash
      let hash = Hash.sha256(Hash.sha256(Buffer.from(txid, 'hex')))
      let currentIndex = index

      // Climb the tree
      for (const sibling of proof) {
        const isRightNode = currentIndex % 2 === 1
        const siblingBuffer = Buffer.from(sibling, 'hex')

        // Concatenate in correct order
        const combined = isRightNode
          ? Buffer.concat([siblingBuffer, hash])
          : Buffer.concat([hash, siblingBuffer])

        hash = Hash.sha256(Hash.sha256(combined))
        currentIndex = Math.floor(currentIndex / 2)
      }

      // Compare with merkle root
      return hash.toString('hex') === merkleRoot
    } catch (error) {
      console.error('Proof verification failed:', error)
      return false
    }
  }

  /**
   * Verify transaction with full SPV check
   */
  verifySPV(
    transaction: Transaction,
    blockHeader: BlockHeader,
    merkleProof: {
      index: number
      proof: string[]
    }
  ): {
    valid: boolean
    txInBlock: boolean
    blockValid: boolean
  } {
    const txid = transaction.id('hex')

    // Verify merkle proof
    const txInBlock = this.verifyMerkleProof(
      txid,
      merkleProof.index,
      merkleProof.proof,
      blockHeader.merkleRoot
    )

    // Verify block header (simplified - would check difficulty, timestamp, etc.)
    const blockValid = this.verifyBlockHeader(blockHeader)

    return {
      valid: txInBlock && blockValid,
      txInBlock,
      blockValid
    }
  }

  /**
   * Verify block header (simplified)
   */
  private verifyBlockHeader(header: BlockHeader): boolean {
    // In production, would verify:
    // - Proof of work (hash meets difficulty target)
    // - Timestamp is reasonable
    // - Block connects to known chain
    // - etc.

    try {
      // Basic validation
      if (!header.merkleRoot || !header.prevBlockHash) {
        return false
      }

      // Would verify hash meets difficulty target
      // const blockHash = header.hash()
      // const target = this.calculateTarget(header.bits)
      // return blockHash <= target

      return true
    } catch (error) {
      return false
    }
  }
}

/**
 * Usage Example
 */
async function proofConstructionExample() {
  const spv = new SPVProofConstruction()

  // Block with 8 transactions
  const txids = [
    'tx1-hash...',
    'tx2-hash...',
    'tx3-hash...',
    'tx4-hash...',
    'tx5-hash...',
    'tx6-hash...',
    'tx7-hash...',
    'tx8-hash...'
  ]

  // Build merkle tree
  const { root, tree } = spv.buildMerkleTree(txids)
  console.log('Merkle root:', root)

  // Generate proof for transaction 3
  const proof = spv.generateMerkleProof('tx3-hash...', txids)
  if (proof) {
    console.log('Proof for tx3:', proof)

    // Verify the proof
    const isValid = spv.verifyMerkleProof(
      proof.txid,
      proof.index,
      proof.proof,
      root
    )

    console.log('Proof valid:', isValid)
  }
}
```

## Lightweight SPV Client

```typescript
import { Transaction, BlockHeader, MerkleProof, Hash } from '@bsv/sdk'

/**
 * Lightweight SPV Client
 *
 * Minimal client that verifies transactions without full blockchain
 */
class LightweightSPVClient {
  private blockHeaders: Map<string, BlockHeader> = new Map()
  private verifiedTransactions: Map<string, boolean> = new Map()

  /**
   * Add block header to local storage
   */
  addBlockHeader(header: BlockHeader): void {
    const blockHash = header.hash()
    this.blockHeaders.set(blockHash, header)
  }

  /**
   * Get block header by hash
   */
  getBlockHeader(blockHash: string): BlockHeader | undefined {
    return this.blockHeaders.get(blockHash)
  }

  /**
   * Verify transaction with SPV proof
   */
  async verifyTransaction(
    tx: Transaction,
    blockHash: string,
    merkleProof: {
      index: number
      nodes: string[]
    }
  ): Promise<{
    verified: boolean
    confirmations?: number
    error?: string
  }> {
    try {
      // Get block header
      const header = this.getBlockHeader(blockHash)
      if (!header) {
        return {
          verified: false,
          error: 'Block header not found'
        }
      }

      // Verify merkle proof
      const txid = tx.id('hex')
      const merkleRoot = header.merkleRoot

      const proofValid = this.verifyMerkleProof(
        txid,
        merkleProof.index,
        merkleProof.nodes,
        merkleRoot
      )

      if (!proofValid) {
        return {
          verified: false,
          error: 'Invalid merkle proof'
        }
      }

      // Calculate confirmations (simplified)
      const confirmations = this.calculateConfirmations(blockHash)

      // Cache verification result
      this.verifiedTransactions.set(txid, true)

      return {
        verified: true,
        confirmations
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
    index: number,
    nodes: string[],
    merkleRoot: string
  ): boolean {
    let hash = Hash.sha256(Hash.sha256(Buffer.from(txid, 'hex')))
    let currentIndex = index

    for (const node of nodes) {
      const nodeBuffer = Buffer.from(node, 'hex')
      const isRightNode = currentIndex % 2 === 1

      const combined = isRightNode
        ? Buffer.concat([nodeBuffer, hash])
        : Buffer.concat([hash, nodeBuffer])

      hash = Hash.sha256(Hash.sha256(combined))
      currentIndex = Math.floor(currentIndex / 2)
    }

    return hash.toString('hex') === merkleRoot
  }

  /**
   * Calculate confirmations (simplified)
   */
  private calculateConfirmations(blockHash: string): number {
    // In production, would track chain height and calculate
    // confirmations based on blocks built on top
    return 6 // Example
  }

  /**
   * Check if transaction has been verified
   */
  isTransactionVerified(txid: string): boolean {
    return this.verifiedTransactions.get(txid) || false
  }

  /**
   * Verify payment to address
   */
  async verifyPayment(
    toAddress: string,
    expectedAmount: number,
    tx: Transaction,
    blockHash: string,
    merkleProof: { index: number; nodes: string[] }
  ): Promise<{
    verified: boolean
    paid: boolean
    amount?: number
  }> {
    // Verify transaction in block
    const spvResult = await this.verifyTransaction(tx, blockHash, merkleProof)

    if (!spvResult.verified) {
      return {
        verified: false,
        paid: false
      }
    }

    // Check if transaction pays to address
    for (const output of tx.outputs) {
      const outputAddress = output.lockingScript.toAddress()

      if (outputAddress === toAddress && output.satoshis >= expectedAmount) {
        return {
          verified: true,
          paid: true,
          amount: output.satoshis
        }
      }
    }

    return {
      verified: true,
      paid: false
    }
  }

  /**
   * Sync block headers from a source
   */
  async syncHeaders(
    startHeight: number,
    endHeight: number,
    headerSource: (height: number) => Promise<BlockHeader>
  ): Promise<number> {
    let synced = 0

    for (let height = startHeight; height <= endHeight; height++) {
      try {
        const header = await headerSource(height)
        this.addBlockHeader(header)
        synced++
      } catch (error) {
        console.error(`Failed to sync header at height ${height}:`, error)
        break
      }
    }

    return synced
  }

  /**
   * Get verification statistics
   */
  getStats(): {
    headersStored: number
    transactionsVerified: number
  } {
    return {
      headersStored: this.blockHeaders.size,
      transactionsVerified: this.verifiedTransactions.size
    }
  }
}

/**
 * Usage Example
 */
async function lightweightClientExample() {
  const client = new LightweightSPVClient()

  // Add block header
  const header = new BlockHeader({
    version: 1,
    prevBlockHash: 'prev-hash...',
    merkleRoot: 'merkle-root...',
    time: 1234567890,
    bits: 0x1d00ffff,
    nonce: 12345
  })

  client.addBlockHeader(header)

  // Verify transaction
  const tx = Transaction.fromHex('transaction-hex...')
  const blockHash = header.hash()

  const merkleProof = {
    index: 0,
    nodes: ['node1...', 'node2...', 'node3...']
  }

  const result = await client.verifyTransaction(tx, blockHash, merkleProof)

  console.log('Verification result:', result)

  if (result.verified) {
    console.log('Confirmations:', result.confirmations)
  }

  // Check payment
  const payment = await client.verifyPayment(
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    50000,
    tx,
    blockHash,
    merkleProof
  )

  console.log('Payment verified:', payment)
}
```

## Advanced SPV Patterns

```typescript
import { Transaction, BlockHeader, MerkleProof } from '@bsv/sdk'

/**
 * Advanced SPV Patterns
 *
 * Complex SPV verification scenarios
 */
class AdvancedSPVPatterns {
  /**
   * Batch verify multiple transactions
   */
  async batchVerify(
    transactions: Array<{
      tx: Transaction
      blockHash: string
      merkleProof: { index: number; nodes: string[] }
    }>,
    headers: Map<string, BlockHeader>
  ): Promise<Map<string, boolean>> {
    const results = new Map<string, boolean>()

    for (const { tx, blockHash, merkleProof } of transactions) {
      const header = headers.get(blockHash)
      if (!header) {
        results.set(tx.id('hex'), false)
        continue
      }

      const verified = this.verifyMerkleProof(
        tx.id('hex'),
        merkleProof.index,
        merkleProof.nodes,
        header.merkleRoot
      )

      results.set(tx.id('hex'), verified)
    }

    return results
  }

  /**
   * Verify transaction chain (tx depends on previous tx)
   */
  async verifyTransactionChain(
    transactions: Transaction[],
    proofs: Map<string, { blockHash: string; merkleProof: any }>,
    headers: Map<string, BlockHeader>
  ): Promise<boolean> {
    // Verify each transaction in sequence
    for (let i = 0; i < transactions.length; i++) {
      const tx = transactions[i]
      const txid = tx.id('hex')
      const proof = proofs.get(txid)

      if (!proof) return false

      const header = headers.get(proof.blockHash)
      if (!header) return false

      const verified = this.verifyMerkleProof(
        txid,
        proof.merkleProof.index,
        proof.merkleProof.nodes,
        header.merkleRoot
      )

      if (!verified) return false

      // Verify this tx spends from previous tx
      if (i > 0) {
        const prevTxid = transactions[i - 1].id('hex')
        const spendsFromPrev = tx.inputs.some(
          input => input.sourceTXID === prevTxid
        )

        if (!spendsFromPrev) return false
      }
    }

    return true
  }

  /**
   * Verify double-spend protection
   */
  async verifyNoDoubleSpend(
    tx1: Transaction,
    tx2: Transaction,
    proofs: Map<string, any>,
    headers: Map<string, BlockHeader>
  ): Promise<{
    bothValid: boolean
    doubleSpend: boolean
  }> {
    // Verify both transactions
    const tx1Verified = this.verifySingle(tx1, proofs, headers)
    const tx2Verified = this.verifySingle(tx2, proofs, headers)

    if (!tx1Verified || !tx2Verified) {
      return {
        bothValid: false,
        doubleSpend: false
      }
    }

    // Check if they spend the same UTXO
    const tx1Inputs = new Set(
      tx1.inputs.map(i => `${i.sourceTXID}:${i.sourceOutputIndex}`)
    )

    const hasDoubleSpend = tx2.inputs.some(input => {
      const key = `${input.sourceTXID}:${input.sourceOutputIndex}`
      return tx1Inputs.has(key)
    })

    return {
      bothValid: true,
      doubleSpend: hasDoubleSpend
    }
  }

  /**
   * Helper: Verify single transaction
   */
  private verifySingle(
    tx: Transaction,
    proofs: Map<string, any>,
    headers: Map<string, BlockHeader>
  ): boolean {
    const txid = tx.id('hex')
    const proof = proofs.get(txid)
    if (!proof) return false

    const header = headers.get(proof.blockHash)
    if (!header) return false

    return this.verifyMerkleProof(
      txid,
      proof.merkleProof.index,
      proof.merkleProof.nodes,
      header.merkleRoot
    )
  }

  /**
   * Verify merkle proof
   */
  private verifyMerkleProof(
    txid: string,
    index: number,
    nodes: string[],
    merkleRoot: string
  ): boolean {
    let hash = Hash.sha256(Hash.sha256(Buffer.from(txid, 'hex')))
    let currentIndex = index

    for (const node of nodes) {
      const nodeBuffer = Buffer.from(node, 'hex')
      const isRightNode = currentIndex % 2 === 1

      const combined = isRightNode
        ? Buffer.concat([nodeBuffer, hash])
        : Buffer.concat([hash, nodeBuffer])

      hash = Hash.sha256(Hash.sha256(combined))
      currentIndex = Math.floor(currentIndex / 2)
    }

    return hash.toString('hex') === merkleRoot
  }

  /**
   * Estimate proof size
   */
  estimateProofSize(blockSize: number, txCount: number): number {
    // Merkle tree depth
    const depth = Math.ceil(Math.log2(txCount))

    // Each node is 32 bytes
    const proofSize = depth * 32

    // Add metadata overhead
    const overhead = 20

    return proofSize + overhead
  }
}

/**
 * Usage Example
 */
async function advancedPatternsExample() {
  const patterns = new AdvancedSPVPatterns()

  // Estimate proof sizes for different block sizes
  console.log('1,000 txs:', patterns.estimateProofSize(1000000, 1000), 'bytes')
  console.log('10,000 txs:', patterns.estimateProofSize(10000000, 10000), 'bytes')
  console.log('100,000 txs:', patterns.estimateProofSize(100000000, 100000), 'bytes')
}
```

## Related Examples

- [Transaction Building](../transaction-building/README.md)
- [Block Headers](../block-headers/README.md)
- [Chain Tracker](../chain-tracker/README.md)

## See Also

**SDK Components:**
- [Merkle Proof](../../sdk-components/merkle-proof/README.md) - Merkle proof construction and verification
- [Block Headers](../../sdk-components/block-headers/README.md) - Block header validation
- [Chain Tracker](../../sdk-components/chain-tracker/README.md) - Chain synchronization
- [Transaction](../../sdk-components/transaction/README.md) - Transaction handling

**Learning Paths:**
- [SPV Basics](../../learning-paths/intermediate/spv-basics/README.md)
- [Merkle Trees](../../learning-paths/intermediate/merkle-trees/README.md)
- [Lightweight Clients](../../learning-paths/advanced/lightweight-clients/README.md)
