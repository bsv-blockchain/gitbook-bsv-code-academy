# SPV Validation

Complete examples for validating transactions, merkle proofs, and block headers using Simplified Payment Verification (SPV).

## Overview

SPV validation involves cryptographically verifying that transactions are included in blocks without downloading the entire blockchain. This guide demonstrates merkle proof validation, block header validation, transaction verification, and building secure SPV validation systems with proper error handling.

**Related SDK Components:**
- [Merkle Proof](../../sdk-components/merkle-proof/README.md)
- [Block Headers](../../sdk-components/block-headers/README.md)
- [Transaction](../../sdk-components/transaction/README.md)
- [Chain Tracker](../../sdk-components/chain-tracker/README.md)

## Basic Merkle Proof Validation

```typescript
import { MerkleProof, Transaction, Hash } from '@bsv/sdk'

/**
 * Basic Merkle Proof Validator
 *
 * Validate merkle proofs for SPV
 */
class BasicMerkleValidator {
  /**
   * Validate merkle proof for a transaction
   */
  validateMerkleProof(
    txid: string,
    merkleProof: MerkleProofData,
    merkleRoot: string
  ): ValidationResult {
    try {
      console.log('Validating merkle proof...')
      console.log('TXID:', txid)
      console.log('Merkle root:', merkleRoot)

      // Start with transaction hash
      let hash = Hash.sha256(Hash.sha256(Buffer.from(txid, 'hex')))
      let index = merkleProof.index

      console.log(`Climbing merkle tree (${merkleProof.nodes.length} levels)`)

      // Climb the merkle tree
      for (let i = 0; i < merkleProof.nodes.length; i++) {
        const sibling = merkleProof.nodes[i]
        const siblingBuffer = Buffer.from(sibling, 'hex')

        // Determine if current node is on right or left
        const isRightNode = index % 2 === 1

        console.log(`Level ${i}: index=${index}, isRight=${isRightNode}`)

        // Concatenate in correct order
        const combined = isRightNode
          ? Buffer.concat([siblingBuffer, hash])
          : Buffer.concat([hash, siblingBuffer])

        // Hash the concatenation
        hash = Hash.sha256(Hash.sha256(combined))
        index = Math.floor(index / 2)
      }

      const calculatedRoot = hash.toString('hex')
      const valid = calculatedRoot === merkleRoot

      console.log('Calculated root:', calculatedRoot)
      console.log('Expected root:', merkleRoot)
      console.log('Valid:', valid)

      return {
        valid,
        calculatedRoot,
        expectedRoot: merkleRoot,
        error: valid ? undefined : 'Merkle root mismatch'
      }
    } catch (error) {
      console.error('Validation failed:', error.message)

      return {
        valid: false,
        error: error.message
      }
    }
  }

  /**
   * Validate merkle proof with block header
   */
  validateWithBlockHeader(
    txid: string,
    merkleProof: MerkleProofData,
    blockHeader: BlockHeaderData
  ): ValidationResult {
    try {
      console.log('Validating with block header')

      // Validate merkle proof
      const proofResult = this.validateMerkleProof(
        txid,
        merkleProof,
        blockHeader.merkleRoot
      )

      if (!proofResult.valid) {
        return proofResult
      }

      // Additional header validation could go here

      console.log('Block header validation passed')

      return {
        valid: true,
        blockHash: blockHeader.hash,
        blockHeight: blockHeader.height
      }
    } catch (error) {
      return {
        valid: false,
        error: error.message
      }
    }
  }

  /**
   * Validate multiple proofs in batch
   */
  batchValidate(
    proofs: ProofToValidate[]
  ): BatchValidationResult[] {
    console.log(`Batch validating ${proofs.length} merkle proofs`)

    const results: BatchValidationResult[] = []

    for (let i = 0; i < proofs.length; i++) {
      const proof = proofs[i]

      try {
        const result = this.validateMerkleProof(
          proof.txid,
          proof.merkleProof,
          proof.merkleRoot
        )

        results.push({
          index: i,
          txid: proof.txid,
          valid: result.valid,
          error: result.error
        })
      } catch (error) {
        results.push({
          index: i,
          txid: proof.txid,
          valid: false,
          error: error.message
        })
      }
    }

    const validCount = results.filter(r => r.valid).length
    console.log(`Batch validation complete: ${validCount}/${proofs.length} valid`)

    return results
  }

  /**
   * Estimate proof size for a given block
   */
  estimateProofSize(txCount: number): number {
    // Merkle tree depth
    const depth = Math.ceil(Math.log2(txCount))

    // Each node is 32 bytes
    const proofSize = depth * 32

    // Add overhead for index and metadata
    const overhead = 8

    return proofSize + overhead
  }
}

interface MerkleProofData {
  index: number
  nodes: string[]
}

interface BlockHeaderData {
  hash: string
  merkleRoot: string
  height?: number
}

interface ValidationResult {
  valid: boolean
  calculatedRoot?: string
  expectedRoot?: string
  blockHash?: string
  blockHeight?: number
  error?: string
}

interface ProofToValidate {
  txid: string
  merkleProof: MerkleProofData
  merkleRoot: string
}

interface BatchValidationResult {
  index: number
  txid: string
  valid: boolean
  error?: string
}

/**
 * Usage Example
 */
async function basicMerkleValidationExample() {
  const validator = new BasicMerkleValidator()

  console.log('=== Basic Merkle Proof Validation ===')

  // Validate merkle proof
  const txid = 'transaction-id...'
  const merkleProof: MerkleProofData = {
    index: 2,
    nodes: [
      'sibling1...',
      'parent-sibling...',
      'grandparent-sibling...'
    ]
  }
  const merkleRoot = 'merkle-root...'

  const result = validator.validateMerkleProof(txid, merkleProof, merkleRoot)

  if (result.valid) {
    console.log('Proof is valid!')
  } else {
    console.log('Proof is invalid:', result.error)
  }

  // Estimate proof size
  const proofSize = validator.estimateProofSize(10000)
  console.log('Estimated proof size for 10,000 txs:', proofSize, 'bytes')
}
```

## Advanced SPV Validation

```typescript
import { Transaction, BlockHeader, Hash } from '@bsv/sdk'

/**
 * Advanced SPV Validator
 *
 * Comprehensive validation with header chain verification
 */
class AdvancedSPVValidator {
  private minimumWork: number
  private requiredConfirmations: number

  constructor(options: ValidatorOptions = {}) {
    this.minimumWork = options.minimumWork || 0
    this.requiredConfirmations = options.requiredConfirmations || 6
  }

  /**
   * Comprehensive transaction validation
   */
  async validateTransaction(
    tx: Transaction,
    validationData: TransactionValidationData
  ): Promise<ComprehensiveValidationResult> {
    const errors: string[] = []
    const warnings: string[] = []

    try {
      console.log('Starting comprehensive validation')
      console.log('TXID:', tx.id('hex'))

      // 1. Validate merkle proof
      console.log('Step 1: Validating merkle proof')
      const merkleValid = this.validateMerkleProof(
        tx.id('hex'),
        validationData.merkleProof,
        validationData.blockHeader.merkleRoot
      )

      if (!merkleValid) {
        errors.push('Invalid merkle proof')
      }

      // 2. Validate block header
      console.log('Step 2: Validating block header')
      const headerValidation = this.validateBlockHeader(validationData.blockHeader)

      if (!headerValidation.valid) {
        errors.push(`Invalid block header: ${headerValidation.error}`)
      }

      // 3. Check confirmations
      console.log('Step 3: Checking confirmations')
      const confirmations = validationData.confirmations || 0

      if (confirmations < this.requiredConfirmations) {
        warnings.push(
          `Insufficient confirmations: ${confirmations}/${this.requiredConfirmations}`
        )
      }

      // 4. Validate transaction structure
      console.log('Step 4: Validating transaction structure')
      const structureValidation = this.validateTransactionStructure(tx)

      if (!structureValidation.valid) {
        errors.push(`Invalid transaction structure: ${structureValidation.error}`)
      }

      // 5. Check chain work
      console.log('Step 5: Checking chain work')
      if (validationData.chainWork < this.minimumWork) {
        warnings.push('Chain work below recommended minimum')
      }

      const valid = errors.length === 0

      console.log('Validation complete')
      console.log('Valid:', valid)
      console.log('Errors:', errors.length)
      console.log('Warnings:', warnings.length)

      return {
        valid,
        errors,
        warnings,
        confirmations,
        blockHeight: validationData.blockHeight
      }
    } catch (error) {
      errors.push(`Validation error: ${error.message}`)

      return {
        valid: false,
        errors,
        warnings
      }
    }
  }

  /**
   * Validate merkle proof
   */
  private validateMerkleProof(
    txid: string,
    merkleProof: MerkleProofData,
    merkleRoot: string
  ): boolean {
    try {
      let hash = Hash.sha256(Hash.sha256(Buffer.from(txid, 'hex')))
      let index = merkleProof.index

      for (const sibling of merkleProof.nodes) {
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
      console.error('Merkle proof validation failed:', error.message)
      return false
    }
  }

  /**
   * Validate block header
   */
  private validateBlockHeader(
    header: BlockHeaderData
  ): { valid: boolean; error?: string } {
    try {
      // Check required fields
      if (!header.hash || !header.merkleRoot) {
        return {
          valid: false,
          error: 'Missing required header fields'
        }
      }

      // Validate proof of work (simplified)
      if (header.bits) {
        const target = this.calculateTarget(header.bits)
        if (!this.hashMeetsTarget(header.hash, target)) {
          return {
            valid: false,
            error: 'Insufficient proof of work'
          }
        }
      }

      // Validate timestamp
      if (header.time) {
        const now = Date.now()
        const maxFutureTime = now + 2 * 60 * 60 * 1000 // 2 hours

        if (header.time > maxFutureTime) {
          return {
            valid: false,
            error: 'Timestamp too far in future'
          }
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
   * Validate transaction structure
   */
  private validateTransactionStructure(
    tx: Transaction
  ): { valid: boolean; error?: string } {
    try {
      // Check inputs
      if (tx.inputs.length === 0) {
        return {
          valid: false,
          error: 'Transaction has no inputs'
        }
      }

      // Check outputs
      if (tx.outputs.length === 0) {
        return {
          valid: false,
          error: 'Transaction has no outputs'
        }
      }

      // Check output amounts
      for (let i = 0; i < tx.outputs.length; i++) {
        const output = tx.outputs[i]

        if (output.satoshis < 0) {
          return {
            valid: false,
            error: `Output ${i} has negative amount`
          }
        }

        // Check dust threshold
        if (output.satoshis > 0 && output.satoshis < 546) {
          return {
            valid: false,
            error: `Output ${i} below dust threshold`
          }
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
   * Calculate target from bits
   */
  private calculateTarget(bits: number): number {
    const exponent = bits >> 24
    const mantissa = bits & 0xffffff
    return mantissa * Math.pow(256, exponent - 3)
  }

  /**
   * Check if hash meets target
   */
  private hashMeetsTarget(hash: string, target: number): boolean {
    const hashNum = parseInt(hash.substring(0, 16), 16)
    return hashNum <= target
  }

  /**
   * Validate payment to address
   */
  async validatePayment(
    tx: Transaction,
    toAddress: string,
    expectedAmount: number,
    validationData: TransactionValidationData
  ): Promise<PaymentValidationResult> {
    try {
      console.log('Validating payment')
      console.log('To:', toAddress)
      console.log('Expected amount:', expectedAmount)

      // First validate transaction
      const txValidation = await this.validateTransaction(tx, validationData)

      if (!txValidation.valid) {
        return {
          valid: false,
          errors: txValidation.errors
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
          // Skip outputs that can't be converted to address
          continue
        }
      }

      // Check amount
      if (paidAmount < expectedAmount) {
        return {
          valid: false,
          errors: [`Insufficient amount: paid ${paidAmount}, expected ${expectedAmount}`]
        }
      }

      console.log('Payment validation successful')
      console.log('Paid amount:', paidAmount)

      return {
        valid: true,
        paidAmount,
        confirmations: txValidation.confirmations
      }
    } catch (error) {
      return {
        valid: false,
        errors: [error.message]
      }
    }
  }

  /**
   * Validate transaction chain
   */
  async validateTransactionChain(
    transactions: Transaction[],
    validationData: ChainValidationData
  ): Promise<ChainValidationResult> {
    console.log(`Validating transaction chain of ${transactions.length} transactions`)

    const errors: string[] = []

    // Validate each transaction
    for (let i = 0; i < transactions.length; i++) {
      const tx = transactions[i]
      const data = validationData.transactions[i]

      if (!data) {
        errors.push(`Missing validation data for transaction ${i}`)
        continue
      }

      const result = await this.validateTransaction(tx, data)

      if (!result.valid) {
        errors.push(`Transaction ${i} invalid: ${result.errors.join(', ')}`)
      }

      // Verify chain links
      if (i > 0) {
        const prevTxid = transactions[i - 1].id('hex')
        const spendsFromPrev = tx.inputs.some(
          input => input.sourceTXID === prevTxid
        )

        if (!spendsFromPrev) {
          errors.push(`Transaction ${i} does not spend from transaction ${i - 1}`)
        }
      }
    }

    const valid = errors.length === 0

    console.log('Chain validation complete')
    console.log('Valid:', valid)

    return {
      valid,
      errors,
      transactionCount: transactions.length
    }
  }
}

interface ValidatorOptions {
  minimumWork?: number
  requiredConfirmations?: number
}

interface MerkleProofData {
  index: number
  nodes: string[]
}

interface BlockHeaderData {
  hash: string
  merkleRoot: string
  height?: number
  time?: number
  bits?: number
}

interface TransactionValidationData {
  merkleProof: MerkleProofData
  blockHeader: BlockHeaderData
  confirmations?: number
  chainWork: number
  blockHeight: number
}

interface ComprehensiveValidationResult {
  valid: boolean
  errors: string[]
  warnings: string[]
  confirmations?: number
  blockHeight?: number
}

interface PaymentValidationResult {
  valid: boolean
  paidAmount?: number
  confirmations?: number
  errors?: string[]
}

interface ChainValidationData {
  transactions: TransactionValidationData[]
}

interface ChainValidationResult {
  valid: boolean
  errors: string[]
  transactionCount: number
}

/**
 * Usage Example
 */
async function advancedValidationExample() {
  const validator = new AdvancedSPVValidator({
    minimumWork: 1000000,
    requiredConfirmations: 6
  })

  console.log('=== Advanced SPV Validation ===')

  // Validate transaction
  const tx = Transaction.fromHex('tx-hex...')

  const validationData: TransactionValidationData = {
    merkleProof: {
      index: 0,
      nodes: ['node1...', 'node2...']
    },
    blockHeader: {
      hash: 'block-hash...',
      merkleRoot: 'merkle-root...',
      height: 700000,
      time: Date.now(),
      bits: 0x1d00ffff
    },
    confirmations: 6,
    chainWork: 2000000,
    blockHeight: 700000
  }

  const result = await validator.validateTransaction(tx, validationData)

  console.log('Validation result:', result)

  if (result.valid) {
    console.log('Transaction is valid!')
  } else {
    console.log('Errors:', result.errors)
  }

  if (result.warnings.length > 0) {
    console.log('Warnings:', result.warnings)
  }

  // Validate payment
  const paymentResult = await validator.validatePayment(
    tx,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    50000,
    validationData
  )

  console.log('Payment result:', paymentResult)
}
```

## SPV Validation Service

```typescript
import { Transaction, BlockHeader } from '@bsv/sdk'

/**
 * SPV Validation Service
 *
 * Production-ready SPV validation service with caching and monitoring
 */
class SPVValidationService {
  private validator: AdvancedSPVValidator
  private validationCache: Map<string, CachedValidation> = new Map()
  private cacheExpiry: number = 3600000 // 1 hour

  constructor(options: ServiceOptions = {}) {
    this.validator = new AdvancedSPVValidator(options.validatorOptions)
    this.cacheExpiry = options.cacheExpiry || 3600000
  }

  /**
   * Validate with caching
   */
  async validateWithCache(
    tx: Transaction,
    validationData: TransactionValidationData
  ): Promise<ComprehensiveValidationResult> {
    const txid = tx.id('hex')

    // Check cache
    const cached = this.validationCache.get(txid)

    if (cached && Date.now() - cached.timestamp < this.cacheExpiry) {
      console.log('Returning cached validation result')
      return cached.result
    }

    // Validate
    console.log('Performing new validation')
    const result = await this.validator.validateTransaction(tx, validationData)

    // Cache result
    this.validationCache.set(txid, {
      result,
      timestamp: Date.now()
    })

    return result
  }

  /**
   * Validate and monitor
   */
  async validateAndMonitor(
    tx: Transaction,
    validationData: TransactionValidationData,
    onUpdate?: (result: ComprehensiveValidationResult) => void
  ): Promise<ComprehensiveValidationResult> {
    // Initial validation
    let result = await this.validateWithCache(tx, validationData)

    // If not enough confirmations, monitor
    if (result.valid && result.confirmations && result.confirmations < 6) {
      console.log('Monitoring for additional confirmations')

      // In production, would set up monitoring
      // This is simplified for demonstration
      if (onUpdate) {
        onUpdate(result)
      }
    }

    return result
  }

  /**
   * Batch validation with progress tracking
   */
  async batchValidateWithProgress(
    items: ValidationItem[],
    onProgress?: (completed: number, total: number) => void
  ): Promise<BatchValidationServiceResult[]> {
    console.log(`Batch validating ${items.length} items`)

    const results: BatchValidationServiceResult[] = []

    for (let i = 0; i < items.length; i++) {
      const item = items[i]

      try {
        const result = await this.validateWithCache(item.tx, item.validationData)

        results.push({
          index: i,
          txid: item.tx.id('hex'),
          valid: result.valid,
          errors: result.errors,
          warnings: result.warnings,
          confirmations: result.confirmations
        })

        // Progress callback
        if (onProgress) {
          onProgress(i + 1, items.length)
        }
      } catch (error) {
        results.push({
          index: i,
          txid: item.tx.id('hex'),
          valid: false,
          errors: [error.message],
          warnings: []
        })
      }
    }

    const validCount = results.filter(r => r.valid).length
    console.log(`Batch validation complete: ${validCount}/${items.length} valid`)

    return results
  }

  /**
   * Clear validation cache
   */
  clearCache(): void {
    this.validationCache.clear()
    console.log('Validation cache cleared')
  }

  /**
   * Clean expired cache entries
   */
  cleanExpiredCache(): number {
    const now = Date.now()
    let cleaned = 0

    for (const [txid, cached] of this.validationCache.entries()) {
      if (now - cached.timestamp >= this.cacheExpiry) {
        this.validationCache.delete(txid)
        cleaned++
      }
    }

    console.log(`Cleaned ${cleaned} expired cache entries`)

    return cleaned
  }

  /**
   * Get service statistics
   */
  getStats(): ServiceStats {
    let validCount = 0
    let invalidCount = 0

    for (const cached of this.validationCache.values()) {
      if (cached.result.valid) {
        validCount++
      } else {
        invalidCount++
      }
    }

    return {
      cacheSize: this.validationCache.size,
      validCached: validCount,
      invalidCached: invalidCount,
      cacheHitRate: 0 // Would track in production
    }
  }
}

interface ServiceOptions {
  validatorOptions?: ValidatorOptions
  cacheExpiry?: number
}

interface ValidatorOptions {
  minimumWork?: number
  requiredConfirmations?: number
}

interface TransactionValidationData {
  merkleProof: MerkleProofData
  blockHeader: BlockHeaderData
  confirmations?: number
  chainWork: number
  blockHeight: number
}

interface MerkleProofData {
  index: number
  nodes: string[]
}

interface BlockHeaderData {
  hash: string
  merkleRoot: string
  height?: number
  time?: number
  bits?: number
}

interface ComprehensiveValidationResult {
  valid: boolean
  errors: string[]
  warnings: string[]
  confirmations?: number
  blockHeight?: number
}

interface CachedValidation {
  result: ComprehensiveValidationResult
  timestamp: number
}

interface ValidationItem {
  tx: Transaction
  validationData: TransactionValidationData
}

interface BatchValidationServiceResult {
  index: number
  txid: string
  valid: boolean
  errors: string[]
  warnings: string[]
  confirmations?: number
}

interface ServiceStats {
  cacheSize: number
  validCached: number
  invalidCached: number
  cacheHitRate: number
}

/**
 * Usage Example
 */
async function validationServiceExample() {
  const service = new SPVValidationService({
    validatorOptions: {
      minimumWork: 1000000,
      requiredConfirmations: 6
    },
    cacheExpiry: 3600000
  })

  console.log('=== SPV Validation Service ===')

  // Validate with caching
  const tx = Transaction.fromHex('tx-hex...')

  const validationData: TransactionValidationData = {
    merkleProof: {
      index: 0,
      nodes: ['node1...', 'node2...']
    },
    blockHeader: {
      hash: 'block-hash...',
      merkleRoot: 'merkle-root...',
      height: 700000
    },
    confirmations: 6,
    chainWork: 2000000,
    blockHeight: 700000
  }

  const result = await service.validateWithCache(tx, validationData)
  console.log('Validation result:', result)

  // Batch validation with progress
  const items: ValidationItem[] = [] // Array of items to validate

  const batchResults = await service.batchValidateWithProgress(
    items,
    (completed, total) => {
      console.log(`Progress: ${completed}/${total}`)
    }
  )

  console.log('Batch results:', batchResults)

  // Get statistics
  const stats = service.getStats()
  console.log('Service stats:', stats)

  // Clean cache
  const cleaned = service.cleanExpiredCache()
  console.log('Cleaned entries:', cleaned)
}
```

## Related Examples

- [SPV Implementation](../spv/README.md)
- [SPV Verification](../spv-verification/README.md)
- [Transaction Building](../transaction-building/README.md)
- [Block Headers](../block-headers/README.md)

## See Also

**SDK Components:**
- [Merkle Proof](../../sdk-components/merkle-proof/README.md) - Merkle proof validation
- [Block Headers](../../sdk-components/block-headers/README.md) - Block header validation
- [Transaction](../../sdk-components/transaction/README.md) - Transaction validation
- [Chain Tracker](../../sdk-components/chain-tracker/README.md) - Chain validation

**Learning Paths:**
- [SPV Basics](../../learning-paths/intermediate/spv-basics/README.md)
- [Transaction Validation](../../learning-paths/intermediate/transaction-validation/README.md)
- [Security Best Practices](../../learning-paths/advanced/security/README.md)
