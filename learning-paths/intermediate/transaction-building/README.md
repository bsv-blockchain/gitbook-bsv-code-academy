# Transaction Building

## Overview

Learn to construct complex, efficient BSV transactions for real-world applications. This module teaches you how to build multi-input transactions, implement batch payments, optimize fees, manage change outputs, and create transaction chains using the BSV SDK.

**Estimated Time:** 3-4 hours
**Difficulty:** Intermediate
**Prerequisites:** Complete [BSV Fundamentals](../../beginner/bsv-fundamentals/README.md) and [Your First Transaction](../../beginner/first-transaction/README.md)

## Learning Objectives

By the end of this module, you will be able to:

- ✅ Build multi-input transactions efficiently
- ✅ Create batch payment transactions with multiple outputs
- ✅ Implement advanced fee calculation and optimization
- ✅ Handle change management properly
- ✅ Create transaction chains with dependencies
- ✅ Use BEEF format for complex transaction bundles
- ✅ Estimate transaction sizes before building

## SDK Components Used

This course leverages these standardized SDK modules:

- **[Transaction](../../../sdk-components/transaction/README.md)** - Core transaction building
- **[Transaction Input](../../../sdk-components/transaction-input/README.md)** - Managing inputs and UTXOs
- **[Transaction Output](../../../sdk-components/transaction-output/README.md)** - Creating outputs
- **[UTXO Management](../../../sdk-components/utxo-management/README.md)** - UTXO selection strategies
- **[BEEF](../../../sdk-components/beef/README.md)** - Transaction envelopes
- **[ARC](../../../sdk-components/arc/README.md)** - Transaction broadcasting
- **[Signatures](../../../sdk-components/signatures/README.md)** - Transaction signing
- **[Script Templates](../../../sdk-components/script-templates/README.md)** - P2PKH and custom scripts

## 1. Multi-Input Transactions

### Understanding Multiple Inputs

When you need to spend more than one UTXO, you create a multi-input transaction. This is common when:
- Your balance is spread across multiple UTXOs
- You need to consolidate small UTXOs
- The payment amount exceeds any single UTXO

### Basic Multi-Input Transaction

Reference: **[Transaction Component](../../../sdk-components/transaction/README.md#key-features)** and **[UTXO Management](../../../sdk-components/utxo-management/README.md)**

```typescript
import { Transaction, PrivateKey, P2PKH } from '@bsv/sdk'

async function createMultiInputTransaction(
  privateKey: PrivateKey,
  utxos: Array<{ txid: string; vout: number; satoshis: number }>,
  toAddress: string,
  amount: number
): Promise<Transaction> {
  // Create new transaction
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

  // Add payment output
  const lockingScript = new P2PKH().lock(toAddress)
  tx.addOutput({
    lockingScript,
    satoshis: amount
  })

  // Calculate total input
  const totalInput = utxos.reduce((sum, utxo) => sum + utxo.satoshis, 0)

  // Calculate fee (estimated size * fee rate)
  await tx.fee()

  // Add change output
  const changeAmount = totalInput - amount - tx.getFee()
  if (changeAmount > 0) {
    const changeAddress = privateKey.toPublicKey().toAddress()
    const changeLockingScript = new P2PKH().lock(changeAddress)
    tx.addOutput({
      lockingScript: changeLockingScript,
      satoshis: changeAmount
    })
  }

  // Sign all inputs
  await tx.sign()

  return tx
}

// Usage
const utxos = [
  { txid: 'abc123...', vout: 0, satoshis: 10000 },
  { txid: 'def456...', vout: 1, satoshis: 15000 },
  { txid: 'ghi789...', vout: 0, satoshis: 20000 }
]

const tx = await createMultiInputTransaction(
  privateKey,
  utxos,
  recipientAddress,
  40000
)
```

### UTXO Selection Strategy

Reference: **[UTXO Management - Coin Selection](../../../sdk-components/utxo-management/README.md#common-patterns)**

```typescript
import { Transaction, PrivateKey, P2PKH } from '@bsv/sdk'

interface UTXO {
  txid: string
  vout: number
  satoshis: number
  script: string
}

class UTXOSelector {
  /**
   * Largest-first selection (minimizes inputs)
   */
  selectLargestFirst(utxos: UTXO[], targetAmount: number): UTXO[] {
    const sorted = [...utxos].sort((a, b) => b.satoshis - a.satoshis)
    const selected: UTXO[] = []
    let total = 0

    for (const utxo of sorted) {
      selected.push(utxo)
      total += utxo.satoshis

      if (total >= targetAmount) {
        break
      }
    }

    return selected
  }

  /**
   * Smallest-first selection (consolidates dust)
   */
  selectSmallestFirst(utxos: UTXO[], targetAmount: number): UTXO[] {
    const sorted = [...utxos].sort((a, b) => a.satoshis - b.satoshis)
    const selected: UTXO[] = []
    let total = 0

    for (const utxo of sorted) {
      selected.push(utxo)
      total += utxo.satoshis

      if (total >= targetAmount) {
        break
      }
    }

    return selected
  }

  /**
   * Branch-and-bound for optimal selection
   */
  selectOptimal(utxos: UTXO[], targetAmount: number): UTXO[] {
    // Sort by value
    const sorted = [...utxos].sort((a, b) => b.satoshis - a.satoshis)

    // Try to find exact match or minimal overage
    const result = this.branchAndBound(sorted, targetAmount, 0, [], 0)
    return result || this.selectLargestFirst(utxos, targetAmount)
  }

  private branchAndBound(
    utxos: UTXO[],
    target: number,
    index: number,
    selected: UTXO[],
    currentTotal: number
  ): UTXO[] | null {
    // Found exact match
    if (currentTotal === target) {
      return selected
    }

    // Exceeded target but close enough
    if (currentTotal >= target && currentTotal - target < 1000) {
      return selected
    }

    // No more UTXOs to try
    if (index >= utxos.length) {
      return null
    }

    // Try including current UTXO
    const withCurrent = this.branchAndBound(
      utxos,
      target,
      index + 1,
      [...selected, utxos[index]],
      currentTotal + utxos[index].satoshis
    )

    if (withCurrent) return withCurrent

    // Try without current UTXO
    return this.branchAndBound(utxos, target, index + 1, selected, currentTotal)
  }
}

// Usage
const selector = new UTXOSelector()
const selectedUTXOs = selector.selectOptimal(availableUTXOs, paymentAmount)
```

## 2. Batch Payments

### Multiple Output Transactions

Reference: **[Transaction Output Component](../../../sdk-components/transaction-output/README.md)**

```typescript
import { Transaction, PrivateKey, P2PKH } from '@bsv/sdk'

interface Payment {
  address: string
  amount: number
  description?: string
}

async function createBatchPayment(
  privateKey: PrivateKey,
  utxos: UTXO[],
  payments: Payment[]
): Promise<Transaction> {
  const tx = new Transaction()

  // Add inputs
  for (const utxo of utxos) {
    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(privateKey)
    })
  }

  // Add payment outputs
  for (const payment of payments) {
    const lockingScript = new P2PKH().lock(payment.address)
    tx.addOutput({
      lockingScript,
      satoshis: payment.amount
    })
  }

  // Calculate fee
  await tx.fee()

  // Add change if needed
  const totalInput = utxos.reduce((sum, u) => sum + u.satoshis, 0)
  const totalOutput = payments.reduce((sum, p) => sum + p.amount, 0)
  const changeAmount = totalInput - totalOutput - tx.getFee()

  if (changeAmount > 0) {
    const changeAddress = privateKey.toPublicKey().toAddress()
    tx.addOutput({
      lockingScript: new P2PKH().lock(changeAddress),
      satoshis: changeAmount
    })
  }

  // Sign transaction
  await tx.sign()

  return tx
}

// Usage - Pay multiple recipients
const payments = [
  { address: '1Address1...', amount: 10000 },
  { address: '1Address2...', amount: 15000 },
  { address: '1Address3...', amount: 20000 }
]

const batchTx = await createBatchPayment(privateKey, selectedUTXOs, payments)
```

### Production Batch Payment Processor

```typescript
class BatchPaymentProcessor {
  private privateKey: PrivateKey
  private maxOutputs: number = 100 // Limit outputs per transaction

  constructor(privateKey: PrivateKey) {
    this.privateKey = privateKey
  }

  async processBatch(
    utxos: UTXO[],
    payments: Payment[]
  ): Promise<Transaction[]> {
    const transactions: Transaction[] = []

    // Split into batches if needed
    for (let i = 0; i < payments.length; i += this.maxOutputs) {
      const batch = payments.slice(i, i + this.maxOutputs)
      const tx = await this.createBatchTransaction(utxos, batch)
      transactions.push(tx)
    }

    return transactions
  }

  private async createBatchTransaction(
    utxos: UTXO[],
    payments: Payment[]
  ): Promise<Transaction> {
    const totalNeeded = payments.reduce((sum, p) => sum + p.amount, 0)

    // Select UTXOs
    const selector = new UTXOSelector()
    const selectedUTXOs = selector.selectOptimal(utxos, totalNeeded)

    // Create transaction
    return await createBatchPayment(this.privateKey, selectedUTXOs, payments)
  }
}
```

## 3. Fee Calculation and Optimization

### Understanding Transaction Fees

Reference: **[Transaction Component - Fee Calculation](../../../sdk-components/transaction/README.md#key-features)**

BSV fees are calculated based on transaction size:
- **Fee Rate**: Satoshis per byte (typically 0.05 sat/byte)
- **Transaction Size**: Based on inputs, outputs, and scripts
- **Formula**: `fee = size_in_bytes * fee_rate`

### Accurate Fee Estimation

```typescript
class FeeCalculator {
  private feeRate: number = 0.05 // satoshis per byte

  /**
   * Estimate transaction size before building
   */
  estimateSize(numInputs: number, numOutputs: number): number {
    const BASE_SIZE = 10 // version, locktime, etc.
    const INPUT_SIZE = 148 // Average P2PKH input size
    const OUTPUT_SIZE = 34 // Average P2PKH output size

    return BASE_SIZE + (numInputs * INPUT_SIZE) + (numOutputs * OUTPUT_SIZE)
  }

  /**
   * Calculate fee for estimated size
   */
  calculateFee(numInputs: number, numOutputs: number): number {
    const estimatedSize = this.estimateSize(numInputs, numOutputs)
    return Math.ceil(estimatedSize * this.feeRate)
  }

  /**
   * Optimize UTXO selection for minimal fees
   */
  optimizeForFees(
    utxos: UTXO[],
    targetAmount: number
  ): { utxos: UTXO[]; fee: number } {
    // Try different combinations to minimize fee
    const sorted = [...utxos].sort((a, b) => b.satoshis - a.satoshis)

    let bestSelection: UTXO[] = []
    let bestFee = Infinity

    for (let i = 1; i <= sorted.length; i++) {
      const selected = sorted.slice(0, i)
      const total = selected.reduce((sum, u) => sum + u.satoshis, 0)

      if (total >= targetAmount) {
        const fee = this.calculateFee(i, 2) // 2 outputs (payment + change)

        if (fee < bestFee && total >= targetAmount + fee) {
          bestSelection = selected
          bestFee = fee
          break // Found optimal
        }
      }
    }

    return { utxos: bestSelection, fee: bestFee }
  }
}

// Usage
const feeCalc = new FeeCalculator()
const { utxos, fee } = feeCalc.optimizeForFees(availableUTXOs, 100000)
```

### Dynamic Fee Adjustment

```typescript
async function createTransactionWithDynamicFee(
  privateKey: PrivateKey,
  utxos: UTXO[],
  toAddress: string,
  amount: number,
  feeRate: number = 0.05
): Promise<Transaction> {
  const tx = new Transaction()

  // Add inputs
  for (const utxo of utxos) {
    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(privateKey)
    })
  }

  // Add payment output
  tx.addOutput({
    lockingScript: new P2PKH().lock(toAddress),
    satoshis: amount
  })

  // Set custom fee rate
  await tx.fee(undefined, feeRate)

  // Add change
  const totalInput = utxos.reduce((sum, u) => sum + u.satoshis, 0)
  const changeAmount = totalInput - amount - tx.getFee()

  if (changeAmount > 0) {
    tx.addOutput({
      lockingScript: new P2PKH().lock(privateKey.toPublicKey().toAddress()),
      satoshis: changeAmount
    })
  }

  await tx.sign()
  return tx
}
```

## 4. Change Management

### Optimal Change Output Handling

Reference: **[Transaction Output - Change Management](../../../sdk-components/transaction-output/README.md#key-features)**

```typescript
class ChangeManager {
  private readonly DUST_LIMIT = 546 // Minimum output size
  private privateKey: PrivateKey

  constructor(privateKey: PrivateKey) {
    this.privateKey = privateKey
  }

  /**
   * Determine if change output should be created
   */
  shouldCreateChange(changeAmount: number, fee: number): boolean {
    // Don't create dust outputs
    if (changeAmount < this.DUST_LIMIT) {
      return false
    }

    // Change must cover its own cost
    const changeOutputCost = 34 * 0.05 // ~1.7 satoshis
    return changeAmount > changeOutputCost
  }

  /**
   * Add change output to transaction
   */
  addChangeOutput(
    tx: Transaction,
    totalInput: number,
    totalOutput: number,
    fee: number
  ): boolean {
    const changeAmount = totalInput - totalOutput - fee

    if (!this.shouldCreateChange(changeAmount, fee)) {
      // Add to fee instead
      return false
    }

    const changeAddress = this.privateKey.toPublicKey().toAddress()
    tx.addOutput({
      lockingScript: new P2PKH().lock(changeAddress),
      satoshis: changeAmount
    })

    return true
  }

  /**
   * Get next change address (for HD wallets)
   */
  getNextChangeAddress(index: number): string {
    // In production, use HD wallet derivation
    // This is simplified
    return this.privateKey.toPublicKey().toAddress()
  }
}
```

## 5. Transaction Chaining

### Creating Dependent Transactions

Reference: **[BEEF Component](../../../sdk-components/beef/README.md)**

```typescript
import { Transaction, PrivateKey, P2PKH, Beef } from '@bsv/sdk'

async function createTransactionChain(
  privateKey: PrivateKey,
  initialUTXO: UTXO,
  chainLength: number
): Promise<Transaction[]> {
  const transactions: Transaction[] = []
  let currentTx: Transaction | null = null
  let currentTXID: string = initialUTXO.txid
  let currentVout: number = initialUTXO.vout
  let currentSatoshis: number = initialUTXO.satoshis

  for (let i = 0; i < chainLength; i++) {
    const tx = new Transaction()

    // Add input from previous transaction
    tx.addInput({
      sourceTXID: currentTXID,
      sourceOutputIndex: currentVout,
      unlockingScriptTemplate: new P2PKH().unlock(privateKey),
      sequence: 0xffffffff
    })

    // Create output for next transaction in chain
    const outputAmount = currentSatoshis - 500 // Deduct fee
    tx.addOutput({
      lockingScript: new P2PKH().lock(privateKey.toPublicKey().toAddress()),
      satoshis: outputAmount
    })

    await tx.sign()

    transactions.push(tx)

    // Update for next iteration
    currentTXID = tx.id('hex') as string
    currentVout = 0
    currentSatoshis = outputAmount
  }

  return transactions
}

// Usage
const chain = await createTransactionChain(privateKey, initialUTXO, 5)
console.log(`Created chain of ${chain.length} transactions`)
```

### BEEF Transaction Bundles

Reference: **[BEEF Component - Atomic Bundles](../../../sdk-components/beef/README.md#key-features)**

```typescript
import { Beef, Transaction, ARC } from '@bsv/sdk'

async function createAtomicBundle(
  transactions: Transaction[]
): Promise<number[]> {
  // Create BEEF envelope
  const beef = new Beef()

  // Add each transaction to the BEEF
  for (const tx of transactions) {
    beef.mergeRawTx(tx.toBinary())
  }

  // Serialize to binary format
  return beef.toBinary()
}

async function broadcastChainWithBEEF(
  transactions: Transaction[]
): Promise<void> {
  // Package as BEEF
  const beefBinary = await createAtomicBundle(transactions)

  // Convert to hex for broadcasting
  const beefHex = Buffer.from(beefBinary).toString('hex')

  // Broadcast BEEF bundle using the SDK's ARC client
  const arc = new ARC('https://api.taal.com/arc', {
    apiKey: 'your-api-key',
    deploymentId: 'your-deployment-id'
  })

  const response = await arc.broadcastBEEF(beefHex)

  console.log('BEEF bundle broadcast:', response)
}
```

## 6. Complete Transaction Builder

### Production-Ready Builder Class

```typescript
import { Transaction, PrivateKey, P2PKH } from '@bsv/sdk'

interface BuildOptions {
  feeRate?: number
  dustLimit?: number
  changeAddress?: string
}

class TransactionBuilder {
  private privateKey: PrivateKey
  private options: Required<BuildOptions>

  constructor(privateKey: PrivateKey, options: BuildOptions = {}) {
    this.privateKey = privateKey
    this.options = {
      feeRate: options.feeRate ?? 0.05,
      dustLimit: options.dustLimit ?? 546,
      changeAddress: options.changeAddress ?? privateKey.toPublicKey().toAddress()
    }
  }

  async buildPayment(
    utxos: UTXO[],
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    // Select UTXOs
    const selector = new UTXOSelector()
    const selected = selector.selectOptimal(utxos, amount)

    if (selected.length === 0) {
      throw new Error('Insufficient funds')
    }

    // Create transaction
    const tx = new Transaction()

    // Add inputs
    for (const utxo of selected) {
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

    // Calculate fee
    await tx.fee(undefined, this.options.feeRate)

    // Add change if needed
    const totalInput = selected.reduce((sum, u) => sum + u.satoshis, 0)
    const changeAmount = totalInput - amount - tx.getFee()

    if (changeAmount >= this.options.dustLimit) {
      tx.addOutput({
        lockingScript: new P2PKH().lock(this.options.changeAddress),
        satoshis: changeAmount
      })
    }

    // Sign transaction
    await tx.sign()

    return tx
  }

  async buildBatchPayment(
    utxos: UTXO[],
    payments: Payment[]
  ): Promise<Transaction> {
    const totalAmount = payments.reduce((sum, p) => sum + p.amount, 0)

    // Select UTXOs
    const selector = new UTXOSelector()
    const selected = selector.selectOptimal(utxos, totalAmount)

    const tx = new Transaction()

    // Add inputs
    for (const utxo of selected) {
      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(this.privateKey)
      })
    }

    // Add payment outputs
    for (const payment of payments) {
      tx.addOutput({
        lockingScript: new P2PKH().lock(payment.address),
        satoshis: payment.amount
      })
    }

    // Fee and change
    await tx.fee(undefined, this.options.feeRate)

    const totalInput = selected.reduce((sum, u) => sum + u.satoshis, 0)
    const changeAmount = totalInput - totalAmount - tx.getFee()

    if (changeAmount >= this.options.dustLimit) {
      tx.addOutput({
        lockingScript: new P2PKH().lock(this.options.changeAddress),
        satoshis: changeAmount
      })
    }

    await tx.sign()
    return tx
  }
}

// Usage
const builder = new TransactionBuilder(privateKey, {
  feeRate: 0.05,
  dustLimit: 546
})

const tx = await builder.buildPayment(utxos, recipientAddress, 100000)
```

## Hands-On Project: Multi-Payment Platform

Build a complete payment platform that handles:
- Multiple payment types
- Batch processing
- Fee optimization
- Transaction tracking

```typescript
class PaymentPlatform {
  private builder: TransactionBuilder
  private utxoManager: UTXOManager

  async processPayments(payments: Payment[]): Promise<string[]> {
    const utxos = await this.utxoManager.getUTXOs()
    const txids: string[] = []

    // Process in batches of 100
    for (let i = 0; i < payments.length; i += 100) {
      const batch = payments.slice(i, i + 100)
      const tx = await this.builder.buildBatchPayment(utxos, batch)

      // Broadcast using SDK's built-in broadcast
      const response = await tx.broadcast()
      txids.push(response.txid)
    }

    return txids
  }
}
```

## Best Practices

1. **Always validate UTXOs** before building transactions
2. **Use coin selection strategies** appropriate for your use case
3. **Estimate fees accurately** to avoid overpaying
4. **Handle change properly** - avoid dust outputs
5. **Test with testnet** before using real funds
6. **Use BEEF format** for transaction chains
7. **Implement retry logic** for broadcast failures
8. **Monitor confirmation** status
9. **Keep private keys secure** - never log or expose them
10. **Validate addresses** before creating outputs

## Common Pitfalls

1. **Insufficient fees** - Transaction won't be mined
2. **Dust outputs** - Creating outputs below dust limit
3. **Missing change** - Forgetting to add change output
4. **Wrong UTXO selection** - Inefficient coin selection
5. **Not signing all inputs** - Transaction invalid

## Next Steps

Continue to:
- **[Script Templates](../script-templates/README.md)** - Learn custom locking scripts
- **[SPV Verification](../spv-verification/README.md)** - Verify transactions with SPV

## Additional Resources

- [Transaction SDK Component](../../../sdk-components/transaction/README.md)
- [UTXO Management Guide](../../../sdk-components/utxo-management/README.md)
- [BEEF Format Specification](../../../sdk-components/beef/README.md)
- [BRC-8: Transaction Envelopes](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0008.md)
- [BRC-62: BEEF Format](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0062.md)

---

**Status:** ✅ Complete - Ready for learning
