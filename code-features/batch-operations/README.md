# Batch Operations

Complete examples for performing batch payment processing and batch UTXO operations efficiently using the BSV SDK.

## Overview

Batch operations allow you to process multiple payments or UTXO operations in a single transaction, significantly reducing fees and improving efficiency. This code feature demonstrates practical patterns for batch processing, from simple multi-recipient payments to complex UTXO consolidation and distribution strategies.

**Related SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md)
- [Transaction Input](../../sdk-components/transaction-input/README.md)
- [Transaction Output](../../sdk-components/transaction-output/README.md)
- [UTXO Management](../../sdk-components/utxo-management/README.md)

## Batch Payment Processor

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Batch Payment Processor
 *
 * Efficiently process multiple payments in a single transaction,
 * reducing fees and blockchain footprint.
 */
class BatchPaymentProcessor {
  private readonly feePerByte = 0.5

  /**
   * Process batch payments to multiple recipients
   *
   * @param privateKey - Private key to sign the transaction
   * @param utxos - Available UTXOs to fund the payments
   * @param recipients - Array of recipients with addresses and amounts
   * @returns Signed transaction ready for broadcasting
   */
  async processBatchPayment(
    privateKey: PrivateKey,
    utxos: Array<{
      txid: string
      vout: number
      satoshis: number
      script: Script
    }>,
    recipients: Array<{
      address: string
      amount: number
      memo?: string
    }>
  ): Promise<Transaction> {
    try {
      // Validate recipients
      if (recipients.length === 0) {
        throw new Error('At least one recipient is required')
      }

      // Calculate total payment amount
      const totalPayment = recipients.reduce((sum, r) => sum + r.amount, 0)

      // Estimate transaction size and fee
      const estimatedSize = this.estimateTransactionSize(
        utxos.length,
        recipients.length + 1 // +1 for change output
      )
      const estimatedFee = Math.ceil(estimatedSize * this.feePerByte)

      // Select UTXOs to cover payment + fees
      const selectedUTXOs = this.selectUTXOs(utxos, totalPayment + estimatedFee)
      const totalInput = selectedUTXOs.reduce((sum, u) => sum + u.satoshis, 0)

      // Create transaction
      const tx = new Transaction()

      // Add inputs
      for (const utxo of selectedUTXOs) {
        tx.addInput({
          sourceTXID: utxo.txid,
          sourceOutputIndex: utxo.vout,
          unlockingScriptTemplate: new P2PKH().unlock(privateKey),
          sequence: 0xffffffff
        })
      }

      // Add recipient outputs
      for (const recipient of recipients) {
        // Validate amount is above dust threshold
        if (recipient.amount < 546) {
          throw new Error(`Amount ${recipient.amount} is below dust threshold (546 satoshis)`)
        }

        tx.addOutput({
          satoshis: recipient.amount,
          lockingScript: Script.fromAddress(recipient.address)
        })

        // Optional: Add OP_RETURN memo if provided
        if (recipient.memo) {
          tx.addOutput({
            satoshis: 0,
            lockingScript: Script.fromASM(`OP_FALSE OP_RETURN ${Buffer.from(recipient.memo).toString('hex')}`)
          })
        }
      }

      // Calculate and add change output
      const actualFee = this.calculateActualFee(tx)
      const changeAmount = totalInput - totalPayment - actualFee

      if (changeAmount > 546) {
        tx.addOutput({
          satoshis: changeAmount,
          lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
        })
      } else if (changeAmount < 0) {
        throw new Error('Insufficient funds to cover payment and fees')
      }

      // Sign the transaction
      await tx.sign()

      return tx
    } catch (error) {
      throw new Error(`Batch payment processing failed: ${error.message}`)
    }
  }

  /**
   * Estimate transaction size in bytes
   */
  private estimateTransactionSize(numInputs: number, numOutputs: number): number {
    const baseSize = 10
    const inputSize = 148 // Average P2PKH input size
    const outputSize = 34 // Average P2PKH output size

    return baseSize + (numInputs * inputSize) + (numOutputs * outputSize)
  }

  /**
   * Select UTXOs to meet target amount
   */
  private selectUTXOs(
    utxos: Array<{ satoshis: number; [key: string]: any }>,
    targetAmount: number
  ): typeof utxos {
    // Sort UTXOs by size (largest first)
    const sorted = [...utxos].sort((a, b) => b.satoshis - a.satoshis)

    const selected: typeof utxos = []
    let total = 0

    for (const utxo of sorted) {
      selected.push(utxo)
      total += utxo.satoshis

      if (total >= targetAmount) {
        return selected
      }
    }

    throw new Error(`Insufficient funds: need ${targetAmount}, have ${total}`)
  }

  /**
   * Calculate actual transaction fee based on size
   */
  private calculateActualFee(tx: Transaction): number {
    const size = tx.toHex().length / 2 // Convert hex string length to bytes
    return Math.ceil(size * this.feePerByte)
  }
}

/**
 * Usage Example
 */
async function example() {
  const processor = new BatchPaymentProcessor()
  const privateKey = PrivateKey.fromWif('your-wif-key')

  const utxos = [
    {
      txid: 'abc123...',
      vout: 0,
      satoshis: 100000,
      script: new P2PKH().lock(privateKey.toPublicKey().toHash())
    },
    {
      txid: 'def456...',
      vout: 0,
      satoshis: 50000,
      script: new P2PKH().lock(privateKey.toPublicKey().toHash())
    }
  ]

  const recipients = [
    { address: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa', amount: 10000 },
    { address: '1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2', amount: 20000, memo: 'Payment 1' },
    { address: '1JfbZRwdDHKZmuiZgYArJZhcuuzuw2HuMu', amount: 15000 }
  ]

  const tx = await processor.processBatchPayment(privateKey, utxos, recipients)

  console.log('Transaction ID:', tx.id('hex'))
  console.log('Total recipients:', recipients.length)
}
```

## Batch UTXO Consolidation

```typescript
import { Transaction, PrivateKey, P2PKH } from '@bsv/sdk'

/**
 * Batch UTXO Consolidator
 *
 * Consolidates multiple small UTXOs into larger ones,
 * optimizing wallet management and reducing future transaction fees.
 */
class BatchUTXOConsolidator {
  /**
   * Consolidate multiple UTXOs into a single output
   *
   * @param privateKey - Private key controlling the UTXOs
   * @param utxos - UTXOs to consolidate
   * @param targetAddress - Optional address for consolidated output (defaults to privateKey address)
   * @returns Signed consolidation transaction
   */
  async consolidateUTXOs(
    privateKey: PrivateKey,
    utxos: Array<{
      txid: string
      vout: number
      satoshis: number
      script: Script
    }>,
    targetAddress?: string
  ): Promise<Transaction> {
    try {
      if (utxos.length === 0) {
        throw new Error('No UTXOs provided for consolidation')
      }

      const tx = new Transaction()

      // Add all UTXOs as inputs
      for (const utxo of utxos) {
        tx.addInput({
          sourceTXID: utxo.txid,
          sourceOutputIndex: utxo.vout,
          unlockingScriptTemplate: new P2PKH().unlock(privateKey),
          sequence: 0xffffffff
        })
      }

      // Calculate total input amount
      const totalInput = utxos.reduce((sum, utxo) => sum + utxo.satoshis, 0)

      // Calculate fee (base + per-input cost)
      const baseFee = 500
      const perInputFee = 150
      const totalFee = baseFee + (utxos.length * perInputFee)

      const consolidatedAmount = totalInput - totalFee

      if (consolidatedAmount < 546) {
        throw new Error('Consolidated amount would be below dust threshold')
      }

      // Create consolidated output
      const outputScript = targetAddress
        ? Script.fromAddress(targetAddress)
        : new P2PKH().lock(privateKey.toPublicKey().toHash())

      tx.addOutput({
        satoshis: consolidatedAmount,
        lockingScript: outputScript
      })

      // Sign transaction
      await tx.sign()

      console.log(`Consolidated ${utxos.length} UTXOs into 1 output`)
      console.log(`Total input: ${totalInput} satoshis`)
      console.log(`Fee: ${totalFee} satoshis`)
      console.log(`Output: ${consolidatedAmount} satoshis`)

      return tx
    } catch (error) {
      throw new Error(`UTXO consolidation failed: ${error.message}`)
    }
  }

  /**
   * Consolidate UTXOs into multiple evenly-sized outputs
   *
   * Useful for creating a set of standard-sized UTXOs for future use
   */
  async consolidateIntoMultipleOutputs(
    privateKey: PrivateKey,
    utxos: Array<{
      txid: string
      vout: number
      satoshis: number
      script: Script
    }>,
    numOutputs: number
  ): Promise<Transaction> {
    try {
      if (numOutputs < 1) {
        throw new Error('Number of outputs must be at least 1')
      }

      const tx = new Transaction()

      // Add all inputs
      for (const utxo of utxos) {
        tx.addInput({
          sourceTXID: utxo.txid,
          sourceOutputIndex: utxo.vout,
          unlockingScriptTemplate: new P2PKH().unlock(privateKey),
          sequence: 0xffffffff
        })
      }

      // Calculate total and fee
      const totalInput = utxos.reduce((sum, u) => sum + u.satoshis, 0)
      const fee = 500 + (utxos.length * 150) + (numOutputs * 50)
      const amountPerOutput = Math.floor((totalInput - fee) / numOutputs)

      if (amountPerOutput < 546) {
        throw new Error('Output amounts would be below dust threshold')
      }

      // Create equal outputs
      const outputScript = new P2PKH().lock(privateKey.toPublicKey().toHash())

      for (let i = 0; i < numOutputs; i++) {
        tx.addOutput({
          satoshis: amountPerOutput,
          lockingScript: outputScript
        })
      }

      await tx.sign()

      console.log(`Split ${utxos.length} UTXOs into ${numOutputs} equal outputs of ${amountPerOutput} satoshis each`)

      return tx
    } catch (error) {
      throw new Error(`UTXO splitting failed: ${error.message}`)
    }
  }

  /**
   * Smart consolidation: Group UTXOs by size and consolidate efficiently
   */
  async smartConsolidate(
    privateKey: PrivateKey,
    utxos: Array<{
      txid: string
      vout: number
      satoshis: number
      script: Script
    }>,
    targetOutputSize: number = 100000
  ): Promise<Transaction[]> {
    try {
      // Sort UTXOs by size
      const sortedUTXOs = [...utxos].sort((a, b) => a.satoshis - b.satoshis)

      const transactions: Transaction[] = []
      let currentBatch: typeof utxos = []
      let currentTotal = 0

      for (const utxo of sortedUTXOs) {
        currentBatch.push(utxo)
        currentTotal += utxo.satoshis

        // When batch reaches target size, consolidate it
        if (currentTotal >= targetOutputSize) {
          const tx = await this.consolidateUTXOs(privateKey, currentBatch)
          transactions.push(tx)

          currentBatch = []
          currentTotal = 0
        }
      }

      // Handle remaining UTXOs
      if (currentBatch.length > 0) {
        const tx = await this.consolidateUTXOs(privateKey, currentBatch)
        transactions.push(tx)
      }

      console.log(`Created ${transactions.length} consolidation transactions`)

      return transactions
    } catch (error) {
      throw new Error(`Smart consolidation failed: ${error.message}`)
    }
  }
}

/**
 * Usage Example
 */
async function consolidationExample() {
  const consolidator = new BatchUTXOConsolidator()
  const privateKey = PrivateKey.fromWif('your-wif-key')

  // Multiple small UTXOs
  const dustyUTXOs = [
    { txid: 'utxo1...', vout: 0, satoshis: 1000, script: new P2PKH().lock(privateKey.toPublicKey().toHash()) },
    { txid: 'utxo2...', vout: 0, satoshis: 2000, script: new P2PKH().lock(privateKey.toPublicKey().toHash()) },
    { txid: 'utxo3...', vout: 0, satoshis: 1500, script: new P2PKH().lock(privateKey.toPublicKey().toHash()) },
    { txid: 'utxo4...', vout: 0, satoshis: 3000, script: new P2PKH().lock(privateKey.toPublicKey().toHash()) }
  ]

  // Consolidate all into one
  const consolidatedTx = await consolidator.consolidateUTXOs(privateKey, dustyUTXOs)
  console.log('Consolidated Transaction:', consolidatedTx.id('hex'))

  // Or split into multiple equal outputs
  const splitTx = await consolidator.consolidateIntoMultipleOutputs(privateKey, dustyUTXOs, 2)
  console.log('Split Transaction:', splitTx.id('hex'))
}
```

## Batch UTXO Distribution

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Batch UTXO Distributor
 *
 * Create multiple outputs for distribution to different addresses
 * or for creating a pool of UTXOs with specific characteristics.
 */
class BatchUTXODistributor {
  /**
   * Distribute a large UTXO into multiple smaller UTXOs
   *
   * @param privateKey - Private key to sign the transaction
   * @param sourceUTXO - Large UTXO to split
   * @param distribution - Array of addresses and amounts
   * @returns Signed distribution transaction
   */
  async distributeUTXO(
    privateKey: PrivateKey,
    sourceUTXO: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    distribution: Array<{
      address: string
      amount: number
      label?: string
    }>
  ): Promise<Transaction> {
    try {
      const tx = new Transaction()

      // Add source UTXO as input
      tx.addInput({
        sourceTXID: sourceUTXO.txid,
        sourceOutputIndex: sourceUTXO.vout,
        unlockingScriptTemplate: new P2PKH().unlock(privateKey),
        sequence: 0xffffffff
      })

      // Calculate total distribution
      const totalDistribution = distribution.reduce((sum, d) => sum + d.amount, 0)
      const fee = 500 + (distribution.length * 50)

      if (totalDistribution + fee > sourceUTXO.satoshis) {
        throw new Error('Distribution amount exceeds source UTXO')
      }

      // Add distribution outputs
      for (const dist of distribution) {
        if (dist.amount < 546) {
          throw new Error(`Amount ${dist.amount} is below dust threshold`)
        }

        tx.addOutput({
          satoshis: dist.amount,
          lockingScript: Script.fromAddress(dist.address)
        })
      }

      // Add change output if needed
      const change = sourceUTXO.satoshis - totalDistribution - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log(`Distributed UTXO into ${distribution.length} outputs`)

      return tx
    } catch (error) {
      throw new Error(`UTXO distribution failed: ${error.message}`)
    }
  }

  /**
   * Create a batch of identical UTXOs for future use
   *
   * Useful for creating a pool of standardized UTXOs
   */
  async createUTXOPool(
    privateKey: PrivateKey,
    sourceUTXO: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    utxoSize: number,
    count: number
  ): Promise<Transaction> {
    try {
      const tx = new Transaction()

      // Add source
      tx.addInput({
        sourceTXID: sourceUTXO.txid,
        sourceOutputIndex: sourceUTXO.vout,
        unlockingScriptTemplate: new P2PKH().unlock(privateKey),
        sequence: 0xffffffff
      })

      // Calculate if we have enough
      const totalNeeded = (utxoSize * count)
      const fee = 500 + (count * 50)

      if (totalNeeded + fee > sourceUTXO.satoshis) {
        throw new Error(`Not enough satoshis to create ${count} UTXOs of ${utxoSize} satoshis`)
      }

      // Create identical outputs
      const outputScript = new P2PKH().lock(privateKey.toPublicKey().toHash())

      for (let i = 0; i < count; i++) {
        tx.addOutput({
          satoshis: utxoSize,
          lockingScript: outputScript
        })
      }

      // Add change
      const change = sourceUTXO.satoshis - totalNeeded - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: outputScript
        })
      }

      await tx.sign()

      console.log(`Created pool of ${count} UTXOs, each ${utxoSize} satoshis`)

      return tx
    } catch (error) {
      throw new Error(`UTXO pool creation failed: ${error.message}`)
    }
  }
}

/**
 * Usage Example
 */
async function distributionExample() {
  const distributor = new BatchUTXODistributor()
  const privateKey = PrivateKey.fromWif('your-wif-key')

  const largeUTXO = {
    txid: 'large-utxo...',
    vout: 0,
    satoshis: 1000000,
    script: new P2PKH().lock(privateKey.toPublicKey().toHash())
  }

  // Distribute to multiple addresses
  const distribution = [
    { address: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa', amount: 100000, label: 'Partner A' },
    { address: '1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2', amount: 200000, label: 'Partner B' },
    { address: '1JfbZRwdDHKZmuiZgYArJZhcuuzuw2HuMu', amount: 150000, label: 'Partner C' }
  ]

  const distTx = await distributor.distributeUTXO(privateKey, largeUTXO, distribution)
  console.log('Distribution Transaction:', distTx.id('hex'))

  // Create a pool of identical UTXOs
  const poolTx = await distributor.createUTXOPool(privateKey, largeUTXO, 10000, 50)
  console.log('Pool Transaction:', poolTx.id('hex'))
}
```

## Optimized Batch Processing

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Optimized Batch Processor
 *
 * Advanced batch processing with optimization strategies
 * for minimizing fees and transaction size.
 */
class OptimizedBatchProcessor {
  private readonly feePerByte = 0.5
  private readonly dustThreshold = 546
  private readonly maxOutputsPerTx = 1000 // Practical limit

  /**
   * Process large batch with automatic transaction splitting
   *
   * @param privateKey - Private key for signing
   * @param utxos - Available UTXOs
   * @param recipients - All recipients (may exceed single transaction limits)
   * @returns Array of signed transactions
   */
  async processLargeBatch(
    privateKey: PrivateKey,
    utxos: Array<{
      txid: string
      vout: number
      satoshis: number
      script: Script
    }>,
    recipients: Array<{
      address: string
      amount: number
    }>
  ): Promise<Transaction[]> {
    try {
      // Split recipients into manageable chunks
      const chunks = this.chunkRecipients(recipients, this.maxOutputsPerTx)
      const transactions: Transaction[] = []

      let availableUTXOs = [...utxos]

      for (let i = 0; i < chunks.length; i++) {
        const chunk = chunks[i]
        const totalAmount = chunk.reduce((sum, r) => sum + r.amount, 0)
        const estimatedFee = this.estimateFee(1, chunk.length + 1)

        // Select UTXOs for this chunk
        const { selected, remaining } = this.selectUTXOsOptimized(
          availableUTXOs,
          totalAmount + estimatedFee
        )

        // Build transaction for this chunk
        const tx = await this.buildBatchTransaction(
          privateKey,
          selected,
          chunk
        )

        transactions.push(tx)

        // Update available UTXOs (remove used, add change if created)
        availableUTXOs = remaining

        // If change was created, add it to available UTXOs
        if (tx.outputs.length > chunk.length) {
          const changeOutput = tx.outputs[tx.outputs.length - 1]
          availableUTXOs.push({
            txid: tx.id('hex'),
            vout: tx.outputs.length - 1,
            satoshis: changeOutput.satoshis,
            script: changeOutput.lockingScript
          })
        }

        console.log(`Created batch transaction ${i + 1}/${chunks.length}`)
      }

      return transactions
    } catch (error) {
      throw new Error(`Large batch processing failed: ${error.message}`)
    }
  }

  /**
   * Build a single batch transaction
   */
  private async buildBatchTransaction(
    privateKey: PrivateKey,
    utxos: Array<{
      txid: string
      vout: number
      satoshis: number
      script: Script
    }>,
    recipients: Array<{
      address: string
      amount: number
    }>
  ): Promise<Transaction> {
    const tx = new Transaction()

    // Add inputs
    for (const utxo of utxos) {
      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(privateKey),
        sequence: 0xffffffff
      })
    }

    // Add recipient outputs
    for (const recipient of recipients) {
      tx.addOutput({
        satoshis: recipient.amount,
        lockingScript: Script.fromAddress(recipient.address)
      })
    }

    // Calculate and add change
    const totalInput = utxos.reduce((sum, u) => sum + u.satoshis, 0)
    const totalOutput = recipients.reduce((sum, r) => sum + r.amount, 0)
    const fee = this.estimateFee(utxos.length, recipients.length + 1)
    const change = totalInput - totalOutput - fee

    if (change > this.dustThreshold) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
      })
    }

    await tx.sign()

    return tx
  }

  /**
   * Chunk recipients into manageable groups
   */
  private chunkRecipients<T>(array: T[], chunkSize: number): T[][] {
    const chunks: T[][] = []
    for (let i = 0; i < array.length; i += chunkSize) {
      chunks.push(array.slice(i, i + chunkSize))
    }
    return chunks
  }

  /**
   * Select UTXOs with optimization
   */
  private selectUTXOsOptimized(
    utxos: Array<{ satoshis: number; [key: string]: any }>,
    targetAmount: number
  ): { selected: typeof utxos; remaining: typeof utxos } {
    const sorted = [...utxos].sort((a, b) => b.satoshis - a.satoshis)
    const selected: typeof utxos = []
    const remaining: typeof utxos = []
    let total = 0

    for (const utxo of sorted) {
      if (total < targetAmount) {
        selected.push(utxo)
        total += utxo.satoshis
      } else {
        remaining.push(utxo)
      }
    }

    if (total < targetAmount) {
      throw new Error(`Insufficient funds: need ${targetAmount}, have ${total}`)
    }

    return { selected, remaining }
  }

  /**
   * Estimate transaction fee
   */
  private estimateFee(numInputs: number, numOutputs: number): number {
    const baseSize = 10
    const inputSize = 148
    const outputSize = 34
    const size = baseSize + (numInputs * inputSize) + (numOutputs * outputSize)
    return Math.ceil(size * this.feePerByte)
  }
}
```

## Related Examples

- [Transaction Building](../transaction-building/README.md)
- [UTXO Management](../utxo-management/README.md)
- [Payment Processing](../payment-processing/README.md)
- [Transaction Broadcasting](../transaction-broadcasting/README.md)

## See Also

**SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md) - Core transaction building
- [UTXO Management](../../sdk-components/utxo-management/README.md) - UTXO selection strategies
- [Transaction Output](../../sdk-components/transaction-output/README.md) - Creating outputs

**Learning Paths:**
- [Transaction Building](../../learning-paths/intermediate/transaction-building/README.md)
- [UTXO Management](../../learning-paths/intermediate/utxo-management/README.md)
- [Advanced Transactions](../../learning-paths/advanced/complex-transactions/README.md)
