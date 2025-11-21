# Transaction Building

Complete examples for building various types of BSV transactions using the SDK.

## Overview

This code feature demonstrates practical transaction building patterns, from simple single-input/output transactions to complex multi-party transactions with custom scripts.

**Related SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md)
- [Transaction Input](../../sdk-components/transaction-input/README.md)
- [Transaction Output](../../sdk-components/transaction-output/README.md)
- [UTXO Management](../../sdk-components/utxo-management/README.md)

## Simple Transaction Builder

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Simple Transaction Builder
 *
 * Builds a basic transaction with one input and two outputs (payment + change)
 */
class SimpleTransactionBuilder {
  /**
   * Create a simple payment transaction
   */
  async buildPayment(
    fromPrivateKey: PrivateKey,
    toAddress: string,
    amount: number,
    utxo: UTXO
  ): Promise<Transaction> {
    const tx = new Transaction()

    // Add input from UTXO
    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(fromPrivateKey),
      sequence: 0xffffffff
    })

    // Add payment output
    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    // Add change output
    const fee = 500
    const changeAmount = utxo.satoshis - amount - fee

    if (changeAmount > 0) {
      tx.addOutput({
        satoshis: changeAmount,
        lockingScript: new P2PKH().lock(fromPrivateKey.toPublicKey().toHash())
      })
    }

    // Sign transaction
    await tx.sign()

    return tx
  }
}

/**
 * Usage Example
 */
async function example() {
  const builder = new SimpleTransactionBuilder()
  const privateKey = PrivateKey.fromWif('your-wif-key')

  const utxo: UTXO = {
    txid: 'abc123...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(privateKey.toPublicKey().toHash())
  }

  const tx = await builder.buildPayment(
    privateKey,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    50000,
    utxo
  )

  console.log('Transaction ID:', tx.id('hex'))
}
```

## Multi-Input Transaction Builder

```typescript
import { Transaction, PrivateKey, P2PKH } from '@bsv/sdk'

/**
 * Multi-Input Transaction Builder
 *
 * Builds transactions with multiple inputs (UTXO consolidation)
 */
class MultiInputBuilder {
  /**
   * Consolidate multiple UTXOs into a single output
   */
  async consolidateUTXOs(
    privateKey: PrivateKey,
    utxos: UTXO[]
  ): Promise<Transaction> {
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

    // Calculate fee (base + per-input)
    const fee = 500 + (utxos.length * 150)

    // Add single consolidated output
    tx.addOutput({
      satoshis: totalInput - fee,
      lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
    })

    await tx.sign()
    return tx
  }

  /**
   * Build transaction from multiple UTXOs to multiple outputs
   */
  async buildMultiInputMultiOutput(
    privateKey: PrivateKey,
    utxos: UTXO[],
    outputs: Array<{ address: string; amount: number }>
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

    // Add payment outputs
    for (const output of outputs) {
      tx.addOutput({
        satoshis: output.amount,
        lockingScript: Script.fromAddress(output.address)
      })
    }

    // Calculate change
    const totalInput = utxos.reduce((sum, u) => sum + u.satoshis, 0)
    const totalOutput = outputs.reduce((sum, o) => sum + o.amount, 0)
    const fee = 500 + (utxos.length * 150) + (outputs.length * 50)
    const change = totalInput - totalOutput - fee

    if (change > 0) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }
}
```

## Advanced Transaction Builder

```typescript
import { Transaction, PrivateKey, P2PKH, Script, OP } from '@bsv/sdk'

/**
 * Advanced Transaction Builder
 *
 * Includes fee estimation, UTXO selection, and optimization
 */
class AdvancedTransactionBuilder {
  private readonly feePerByte = 0.5 // satoshis per byte

  /**
   * Estimate transaction size
   */
  estimateSize(numInputs: number, numOutputs: number): number {
    // Base size
    let size = 10

    // Inputs: ~148 bytes each (typical P2PKH)
    size += numInputs * 148

    // Outputs: ~34 bytes each (typical P2PKH)
    size += numOutputs * 34

    return size
  }

  /**
   * Calculate required fee
   */
  calculateFee(numInputs: number, numOutputs: number): number {
    const size = this.estimateSize(numInputs, numOutputs)
    return Math.ceil(size * this.feePerByte)
  }

  /**
   * Select UTXOs using branch-and-bound algorithm
   */
  selectUTXOs(
    utxos: UTXO[],
    targetAmount: number,
    feePerInput: number
  ): UTXO[] {
    // Sort UTXOs by value (descending)
    const sorted = [...utxos].sort((a, b) => b.satoshis - a.satoshis)

    // Try to find exact match first
    const exactMatch = this.findExactMatch(sorted, targetAmount, feePerInput)
    if (exactMatch) return exactMatch

    // Otherwise, use largest-first selection
    const selected: UTXO[] = []
    let total = 0
    let totalFee = 0

    for (const utxo of sorted) {
      selected.push(utxo)
      total += utxo.satoshis
      totalFee = feePerInput * selected.length

      if (total >= targetAmount + totalFee) {
        break
      }
    }

    if (total < targetAmount + totalFee) {
      throw new Error('Insufficient funds')
    }

    return selected
  }

  /**
   * Find exact UTXO match (minimize change)
   */
  private findExactMatch(
    utxos: UTXO[],
    targetAmount: number,
    feePerInput: number
  ): UTXO[] | null {
    // Try single UTXO matches
    for (const utxo of utxos) {
      const fee = feePerInput
      if (utxo.satoshis >= targetAmount + fee &&
          utxo.satoshis <= targetAmount + fee + 1000) {
        return [utxo]
      }
    }

    // Try two-UTXO combinations
    for (let i = 0; i < utxos.length; i++) {
      for (let j = i + 1; j < utxos.length; j++) {
        const total = utxos[i].satoshis + utxos[j].satoshis
        const fee = feePerInput * 2

        if (total >= targetAmount + fee &&
            total <= targetAmount + fee + 1000) {
          return [utxos[i], utxos[j]]
        }
      }
    }

    return null
  }

  /**
   * Build optimized transaction with automatic UTXO selection
   */
  async buildOptimized(
    privateKey: PrivateKey,
    availableUTXOs: UTXO[],
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    // Estimate fee for initial selection
    const feePerInput = this.calculateFee(1, 2)

    // Select optimal UTXOs
    const selectedUTXOs = this.selectUTXOs(
      availableUTXOs,
      amount,
      feePerInput
    )

    // Calculate actual fee
    const fee = this.calculateFee(selectedUTXOs.length, 2)

    // Build transaction
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

    // Add payment output
    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    // Add change output
    const totalInput = selectedUTXOs.reduce((sum, u) => sum + u.satoshis, 0)
    const change = totalInput - amount - fee

    if (change > 546) { // Dust threshold
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }
}
```

## Batch Payment Builder

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Batch Payment Builder
 *
 * Efficiently send payments to multiple recipients in a single transaction
 */
class BatchPaymentBuilder {
  /**
   * Create a batch payment transaction
   */
  async buildBatchPayment(
    privateKey: PrivateKey,
    utxos: UTXO[],
    recipients: Array<{ address: string; amount: number }>,
    changeAddress?: string
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

    // Calculate change
    const totalInput = utxos.reduce((sum, u) => sum + u.satoshis, 0)
    const totalOutput = recipients.reduce((sum, r) => sum + r.amount, 0)
    const fee = 500 + (utxos.length * 150) + (recipients.length * 50)
    const change = totalInput - totalOutput - fee

    if (change > 546) {
      const changeScript = changeAddress
        ? Script.fromAddress(changeAddress)
        : new P2PKH().lock(privateKey.toPublicKey().toHash())

      tx.addOutput({
        satoshis: change,
        lockingScript: changeScript
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * Split a UTXO into multiple equal outputs
   */
  async splitUTXO(
    privateKey: PrivateKey,
    utxo: UTXO,
    numOutputs: number
  ): Promise<Transaction> {
    const tx = new Transaction()

    // Add input
    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(privateKey),
      sequence: 0xffffffff
    })

    // Calculate amount per output
    const fee = 500 + (numOutputs * 50)
    const amountPerOutput = Math.floor((utxo.satoshis - fee) / numOutputs)

    // Add equal outputs
    for (let i = 0; i < numOutputs; i++) {
      tx.addOutput({
        satoshis: amountPerOutput,
        lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }
}
```

## Related Examples

- [Transaction Signing](../transaction-signing/README.md)
- [UTXO Management](../utxo-management/README.md)
- [Batch Operations](../batch-operations/README.md)
- [Transaction Broadcasting](../transaction-broadcasting/README.md)

## See Also

**SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md) - Core transaction building
- [UTXO Management](../../sdk-components/utxo-management/README.md) - UTXO selection strategies

**Learning Paths:**
- [First Transaction](../../learning-paths/beginner/first-transaction/README.md)
- [Transaction Building](../../learning-paths/intermediate/transaction-building/README.md)
