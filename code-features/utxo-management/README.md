# UTXO Management

Complete examples for managing Unspent Transaction Outputs (UTXOs) efficiently.

## Overview

UTXO management is critical for wallet performance and transaction building. This code feature demonstrates practical UTXO tracking, selection strategies, and optimization techniques.

**Related SDK Components:**
- [UTXO Management](../../sdk-components/utxo-management/README.md)
- [Transaction](../../sdk-components/transaction/README.md)
- [Transaction Input](../../sdk-components/transaction-input/README.md)

## UTXO Tracker

```typescript
import { Transaction, PrivateKey, PublicKey, Script } from '@bsv/sdk'

interface UTXO {
  txid: string
  vout: number
  satoshis: number
  script: Script
  confirmations?: number
}

/**
 * UTXO Tracker
 *
 * Track and manage UTXOs for a wallet
 */
class UTXOTracker {
  private utxos: Map<string, UTXO> = new Map()

  /**
   * Add UTXO to tracker
   */
  addUTXO(utxo: UTXO): void {
    const key = `${utxo.txid}:${utxo.vout}`
    this.utxos.set(key, utxo)
  }

  /**
   * Remove spent UTXO
   */
  removeUTXO(txid: string, vout: number): void {
    const key = `${txid}:${vout}`
    this.utxos.delete(key)
  }

  /**
   * Get all UTXOs
   */
  getAllUTXOs(): UTXO[] {
    return Array.from(this.utxos.values())
  }

  /**
   * Get total balance
   */
  getTotalBalance(): number {
    return Array.from(this.utxos.values())
      .reduce((sum, utxo) => sum + utxo.satoshis, 0)
  }

  /**
   * Get confirmed balance
   */
  getConfirmedBalance(minConfirmations: number = 6): number {
    return Array.from(this.utxos.values())
      .filter(utxo => (utxo.confirmations || 0) >= minConfirmations)
      .reduce((sum, utxo) => sum + utxo.satoshis, 0)
  }

  /**
   * Get UTXOs by minimum value
   */
  getUTXOsByMinValue(minValue: number): UTXO[] {
    return Array.from(this.utxos.values())
      .filter(utxo => utxo.satoshis >= minValue)
  }

  /**
   * Process transaction to update UTXO set
   */
  processTransaction(
    tx: Transaction,
    publicKeyHash: Buffer
  ): void {
    // Remove spent UTXOs (inputs)
    for (const input of tx.inputs) {
      this.removeUTXO(input.sourceTXID!, input.sourceOutputIndex!)
    }

    // Add new UTXOs (outputs)
    tx.outputs.forEach((output, index) => {
      if (this.isOwnedByKey(output.lockingScript, publicKeyHash)) {
        this.addUTXO({
          txid: tx.id('hex'),
          vout: index,
          satoshis: output.satoshis!,
          script: output.lockingScript,
          confirmations: 0
        })
      }
    })
  }

  /**
   * Check if output is owned by key
   */
  private isOwnedByKey(script: Script, publicKeyHash: Buffer): boolean {
    // Simple P2PKH detection
    const chunks = script.chunks
    if (chunks.length !== 5) return false

    if (chunks[0].op !== OP.OP_DUP) return false
    if (chunks[1].op !== OP.OP_HASH160) return false
    if (chunks[3].op !== OP.OP_EQUALVERIFY) return false
    if (chunks[4].op !== OP.OP_CHECKSIG) return false

    return chunks[2].buf?.equals(publicKeyHash) || false
  }
}
```

## UTXO Selection Strategies

```typescript
/**
 * UTXO Selection Strategies
 *
 * Different algorithms for selecting UTXOs optimally
 */
class UTXOSelector {
  /**
   * Largest-first selection
   * Minimizes number of inputs, but may create large change
   */
  selectLargestFirst(
    utxos: UTXO[],
    targetAmount: number,
    feePerInput: number = 150
  ): UTXO[] {
    const sorted = [...utxos].sort((a, b) => b.satoshis - a.satoshis)
    const selected: UTXO[] = []
    let total = 0

    for (const utxo of sorted) {
      selected.push(utxo)
      total += utxo.satoshis

      const fee = feePerInput * selected.length
      if (total >= targetAmount + fee) {
        break
      }
    }

    const finalFee = feePerInput * selected.length
    if (total < targetAmount + finalFee) {
      throw new Error('Insufficient funds')
    }

    return selected
  }

  /**
   * Smallest-first selection
   * Cleans up small UTXOs (dust consolidation)
   */
  selectSmallestFirst(
    utxos: UTXO[],
    targetAmount: number,
    feePerInput: number = 150
  ): UTXO[] {
    const sorted = [...utxos].sort((a, b) => a.satoshis - b.satoshis)
    const selected: UTXO[] = []
    let total = 0

    for (const utxo of sorted) {
      selected.push(utxo)
      total += utxo.satoshis

      const fee = feePerInput * selected.length
      if (total >= targetAmount + fee) {
        break
      }
    }

    const finalFee = feePerInput * selected.length
    if (total < targetAmount + finalFee) {
      throw new Error('Insufficient funds')
    }

    return selected
  }

  /**
   * Branch-and-bound selection
   * Finds optimal UTXO combination with minimal change
   */
  selectOptimal(
    utxos: UTXO[],
    targetAmount: number,
    feePerInput: number = 150
  ): UTXO[] {
    // Try to find exact match first
    const exactMatch = this.findExactMatch(utxos, targetAmount, feePerInput)
    if (exactMatch) return exactMatch

    // Try branch-and-bound for near-perfect match
    const sorted = [...utxos].sort((a, b) => b.satoshis - a.satoshis)
    const bestMatch = this.branchAndBound(
      sorted,
      targetAmount,
      feePerInput,
      0,
      [],
      0
    )

    if (bestMatch) return bestMatch

    // Fallback to largest-first
    return this.selectLargestFirst(utxos, targetAmount, feePerInput)
  }

  /**
   * Find exact match (no change needed)
   */
  private findExactMatch(
    utxos: UTXO[],
    targetAmount: number,
    feePerInput: number
  ): UTXO[] | null {
    // Try single UTXO
    for (const utxo of utxos) {
      const fee = feePerInput
      if (Math.abs(utxo.satoshis - targetAmount - fee) < 546) {
        return [utxo]
      }
    }

    // Try pairs
    for (let i = 0; i < utxos.length; i++) {
      for (let j = i + 1; j < utxos.length; j++) {
        const total = utxos[i].satoshis + utxos[j].satoshis
        const fee = feePerInput * 2

        if (Math.abs(total - targetAmount - fee) < 546) {
          return [utxos[i], utxos[j]]
        }
      }
    }

    return null
  }

  /**
   * Branch-and-bound algorithm for optimal selection
   */
  private branchAndBound(
    utxos: UTXO[],
    targetAmount: number,
    feePerInput: number,
    index: number,
    current: UTXO[],
    currentTotal: number
  ): UTXO[] | null {
    const fee = feePerInput * current.length

    // Success: found near-perfect match
    if (currentTotal >= targetAmount + fee &&
        currentTotal - targetAmount - fee < 1000) {
      return current
    }

    // Exceeded or out of UTXOs
    if (index >= utxos.length || currentTotal > targetAmount + fee + 10000) {
      return null
    }

    // Try including current UTXO
    const withCurrent = this.branchAndBound(
      utxos,
      targetAmount,
      feePerInput,
      index + 1,
      [...current, utxos[index]],
      currentTotal + utxos[index].satoshis
    )

    if (withCurrent) return withCurrent

    // Try excluding current UTXO
    return this.branchAndBound(
      utxos,
      targetAmount,
      feePerInput,
      index + 1,
      current,
      currentTotal
    )
  }

  /**
   * Select UTXOs with change minimization
   */
  selectMinimizeChange(
    utxos: UTXO[],
    targetAmount: number,
    feePerInput: number = 150
  ): UTXO[] {
    let bestSelection: UTXO[] = []
    let smallestChange = Infinity

    // Try all combinations up to 4 UTXOs
    const maxCombinations = Math.min(utxos.length, 4)

    for (let size = 1; size <= maxCombinations; size++) {
      const combinations = this.getCombinations(utxos, size)

      for (const combo of combinations) {
        const total = combo.reduce((sum, u) => sum + u.satoshis, 0)
        const fee = feePerInput * combo.length
        const change = total - targetAmount - fee

        if (change >= 0 && change < smallestChange) {
          smallestChange = change
          bestSelection = combo
        }
      }
    }

    if (bestSelection.length === 0) {
      throw new Error('Insufficient funds')
    }

    return bestSelection
  }

  /**
   * Get all combinations of given size
   */
  private getCombinations(utxos: UTXO[], size: number): UTXO[][] {
    if (size === 1) {
      return utxos.map(u => [u])
    }

    const result: UTXO[][] = []

    for (let i = 0; i <= utxos.length - size; i++) {
      const smaller = this.getCombinations(utxos.slice(i + 1), size - 1)
      for (const combo of smaller) {
        result.push([utxos[i], ...combo])
      }
    }

    return result
  }
}
```

## UTXO Consolidation

```typescript
/**
 * UTXO Consolidation
 *
 * Combine many small UTXOs into fewer larger ones
 */
class UTXOConsolidator {
  /**
   * Consolidate UTXOs when count exceeds threshold
   */
  async consolidate(
    privateKey: PrivateKey,
    utxos: UTXO[],
    threshold: number = 100
  ): Promise<Transaction | null> {
    if (utxos.length < threshold) {
      return null
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

    // Calculate total and fee
    const totalInput = utxos.reduce((sum, u) => sum + u.satoshis, 0)
    const fee = 500 + (utxos.length * 150)

    // Single consolidated output
    tx.addOutput({
      satoshis: totalInput - fee,
      lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
    })

    await tx.sign()
    return tx
  }

  /**
   * Consolidate dust UTXOs
   */
  async consolidateDust(
    privateKey: PrivateKey,
    utxos: UTXO[],
    dustThreshold: number = 10000
  ): Promise<Transaction | null> {
    const dustUTXOs = utxos.filter(u => u.satoshis < dustThreshold)

    if (dustUTXOs.length === 0) {
      return null
    }

    return this.consolidate(privateKey, dustUTXOs, 0)
  }

  /**
   * Smart consolidation - balance UTXO count and sizes
   */
  async smartConsolidate(
    privateKey: PrivateKey,
    utxos: UTXO[],
    targetCount: number = 10,
    targetSize: number = 100000
  ): Promise<Transaction[]> {
    const transactions: Transaction[] = []
    const sorted = [...utxos].sort((a, b) => a.satoshis - b.satoshis)

    while (sorted.length > targetCount) {
      // Take smallest UTXOs
      const batch = sorted.splice(0, Math.min(100, sorted.length - targetCount + 1))

      const tx = new Transaction()

      for (const utxo of batch) {
        tx.addInput({
          sourceTXID: utxo.txid,
          sourceOutputIndex: utxo.vout,
          unlockingScriptTemplate: new P2PKH().unlock(privateKey),
          sequence: 0xffffffff
        })
      }

      const totalInput = batch.reduce((sum, u) => sum + u.satoshis, 0)
      const fee = 500 + (batch.length * 150)

      tx.addOutput({
        satoshis: totalInput - fee,
        lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
      })

      await tx.sign()
      transactions.push(tx)

      // Add consolidated UTXO back to sorted list
      sorted.push({
        txid: tx.id('hex'),
        vout: 0,
        satoshis: totalInput - fee,
        script: tx.outputs[0].lockingScript
      })
      sorted.sort((a, b) => a.satoshis - b.satoshis)
    }

    return transactions
  }
}
```

## Related Examples

- [Transaction Building](../transaction-building/README.md)
- [Batch Operations](../batch-operations/README.md)
- [Create Wallet](../create-wallet/README.md)

## See Also

**SDK Components:**
- [UTXO Management](../../sdk-components/utxo-management/README.md)
- [Transaction](../../sdk-components/transaction/README.md)

**Learning Paths:**
- [First Wallet](../../learning-paths/beginner/first-wallet/README.md)
- [Transaction Building](../../learning-paths/intermediate/transaction-building/README.md)
