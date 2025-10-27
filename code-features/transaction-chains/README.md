# Transaction Chains

Complete examples for creating transaction chains using the BEEF (Background Evaluation Extended Format) standard, enabling efficient SPV verification and zero-confirmation transaction chains.

## Overview

Transaction chains allow you to create sequences of dependent transactions where outputs from one transaction are spent in subsequent transactions, all before any confirmations. The BEEF format packages these chains with their Merkle proofs, enabling efficient SPV verification without requiring the full blockchain.

**Related SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md)
- [BEEF](../../sdk-components/beef/README.md)
- [SPV](../../sdk-components/spv/README.md)
- [Merkle Proofs](../../sdk-components/merkle-proofs/README.md)

## Basic Transaction Chain

```typescript
import { Transaction, PrivateKey, P2PKH, BEEF } from '@bsv/sdk'

/**
 * Basic Transaction Chain Builder
 *
 * Create a simple chain of transactions where each transaction
 * spends outputs from the previous transaction.
 */
class TransactionChainBuilder {
  /**
   * Create a chain of transactions
   *
   * @param privateKey - Private key to sign transactions
   * @param initialUTXO - Starting UTXO
   * @param chainLength - Number of transactions in the chain
   * @param amountPerTx - Amount to transfer in each transaction
   * @returns Array of chained transactions
   */
  async buildSimpleChain(
    privateKey: PrivateKey,
    initialUTXO: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    chainLength: number,
    amountPerTx: number
  ): Promise<Transaction[]> {
    try {
      if (chainLength < 2) {
        throw new Error('Chain length must be at least 2')
      }

      const chain: Transaction[] = []
      let currentUTXO = initialUTXO

      for (let i = 0; i < chainLength; i++) {
        const tx = new Transaction()

        // Add input from previous transaction (or initial UTXO)
        tx.addInput({
          sourceTXID: currentUTXO.txid,
          sourceOutputIndex: currentUTXO.vout,
          unlockingScriptTemplate: new P2PKH().unlock(privateKey),
          sequence: 0xffffffff
        })

        // Calculate fee
        const fee = 500

        // Validate sufficient funds
        if (currentUTXO.satoshis < amountPerTx + fee) {
          throw new Error(`Insufficient funds at chain position ${i}`)
        }

        // Add output for next transaction in chain
        const outputScript = new P2PKH().lock(privateKey.toPublicKey().toHash())
        tx.addOutput({
          satoshis: amountPerTx,
          lockingScript: outputScript
        })

        // Add change output if needed
        const change = currentUTXO.satoshis - amountPerTx - fee

        if (change > 546) {
          tx.addOutput({
            satoshis: change,
            lockingScript: outputScript
          })
        }

        // Sign transaction
        await tx.sign()

        chain.push(tx)

        // Update current UTXO for next iteration
        currentUTXO = {
          txid: tx.id('hex'),
          vout: 0,
          satoshis: amountPerTx,
          script: outputScript
        }

        console.log(`Created transaction ${i + 1}/${chainLength} in chain`)
      }

      return chain
    } catch (error) {
      throw new Error(`Transaction chain building failed: ${error.message}`)
    }
  }

  /**
   * Create a branching chain (one input, multiple outputs, each spent separately)
   */
  async buildBranchingChain(
    privateKey: PrivateKey,
    initialUTXO: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    branches: number
  ): Promise<{ root: Transaction; branches: Transaction[] }> {
    try {
      const outputScript = new P2PKH().lock(privateKey.toPublicKey().toHash())

      // Create root transaction with multiple outputs
      const rootTx = new Transaction()

      rootTx.addInput({
        sourceTXID: initialUTXO.txid,
        sourceOutputIndex: initialUTXO.vout,
        unlockingScriptTemplate: new P2PKH().unlock(privateKey),
        sequence: 0xffffffff
      })

      // Calculate amount per branch
      const fee = 500 + (branches * 50)
      const amountPerBranch = Math.floor((initialUTXO.satoshis - fee) / branches)

      if (amountPerBranch < 546) {
        throw new Error('Not enough satoshis for branches')
      }

      // Create outputs for each branch
      for (let i = 0; i < branches; i++) {
        rootTx.addOutput({
          satoshis: amountPerBranch,
          lockingScript: outputScript
        })
      }

      await rootTx.sign()

      // Create branch transactions
      const branchTxs: Transaction[] = []

      for (let i = 0; i < branches; i++) {
        const branchTx = new Transaction()

        branchTx.addInput({
          sourceTXID: rootTx.id('hex'),
          sourceOutputIndex: i,
          unlockingScriptTemplate: new P2PKH().unlock(privateKey),
          sequence: 0xffffffff
        })

        const branchFee = 500
        const branchOutput = amountPerBranch - branchFee

        branchTx.addOutput({
          satoshis: branchOutput,
          lockingScript: outputScript
        })

        await branchTx.sign()
        branchTxs.push(branchTx)
      }

      console.log(`Created branching chain: 1 root → ${branches} branches`)

      return { root: rootTx, branches: branchTxs }
    } catch (error) {
      throw new Error(`Branching chain creation failed: ${error.message}`)
    }
  }
}

/**
 * Usage Example
 */
async function basicChainExample() {
  const builder = new TransactionChainBuilder()
  const privateKey = PrivateKey.fromWif('your-wif-key')

  const initialUTXO = {
    txid: 'initial-txid...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(privateKey.toPublicKey().toHash())
  }

  // Create a chain of 5 transactions
  const chain = await builder.buildSimpleChain(privateKey, initialUTXO, 5, 15000)

  console.log('Chain created:')
  chain.forEach((tx, i) => {
    console.log(`  ${i + 1}. ${tx.id('hex')}`)
  })
}
```

## BEEF Format Transaction Chain

```typescript
import { Transaction, PrivateKey, P2PKH, BEEF, MerkleProof } from '@bsv/sdk'

/**
 * BEEF Transaction Chain Manager
 *
 * Create and manage transaction chains using BEEF format
 * for efficient SPV verification.
 */
class BEEFChainManager {
  /**
   * Create a BEEF-formatted transaction chain
   *
   * @param transactions - Array of chained transactions
   * @param merkleProofs - Merkle proofs for confirmed transactions
   * @returns BEEF-formatted transaction chain
   */
  async createBEEFChain(
    transactions: Transaction[],
    merkleProofs: Map<string, MerkleProof> = new Map()
  ): Promise<number[]> {
    try {
      // Create BEEF structure
      const beef = new BEEF()

      // Add transactions to BEEF
      for (const tx of transactions) {
        const txid = tx.id('hex')

        // Add merkle proof if available
        if (merkleProofs.has(txid)) {
          beef.mergeTxData({
            tx: tx,
            proof: merkleProofs.get(txid)
          })
        } else {
          // Add without proof (unconfirmed transaction)
          beef.mergeTxData({ tx: tx })
        }
      }

      // Serialize BEEF to bytes
      const beefBytes = beef.toArray()

      console.log(`Created BEEF structure with ${transactions.length} transactions`)
      console.log(`BEEF size: ${beefBytes.length} bytes`)

      return beefBytes
    } catch (error) {
      throw new Error(`BEEF chain creation failed: ${error.message}`)
    }
  }

  /**
   * Parse and validate a BEEF chain
   *
   * @param beefBytes - BEEF-formatted byte array
   * @returns Parsed transactions with validation status
   */
  async parseBEEFChain(
    beefBytes: number[]
  ): Promise<{
    transactions: Transaction[]
    isValid: boolean
    validationErrors: string[]
  }> {
    try {
      const beef = BEEF.fromBinary(beefBytes)
      const transactions: Transaction[] = []
      const validationErrors: string[] = []

      // Extract transactions from BEEF
      for (const txData of beef.txs) {
        transactions.push(txData.tx)

        // Validate if proof is present
        if (txData.proof) {
          try {
            const isValid = await this.verifyMerkleProof(
              txData.tx,
              txData.proof
            )

            if (!isValid) {
              validationErrors.push(
                `Invalid merkle proof for tx ${txData.tx.id('hex')}`
              )
            }
          } catch (error) {
            validationErrors.push(
              `Proof verification failed for tx ${txData.tx.id('hex')}: ${error.message}`
            )
          }
        }
      }

      console.log(`Parsed ${transactions.length} transactions from BEEF`)
      console.log(`Validation errors: ${validationErrors.length}`)

      return {
        transactions,
        isValid: validationErrors.length === 0,
        validationErrors
      }
    } catch (error) {
      throw new Error(`BEEF chain parsing failed: ${error.message}`)
    }
  }

  /**
   * Verify merkle proof for a transaction
   */
  private async verifyMerkleProof(
    tx: Transaction,
    proof: MerkleProof
  ): Promise<boolean> {
    try {
      // Verify the merkle proof
      const txid = tx.id('hex')
      const calculatedRoot = proof.calculateRoot(txid)

      // In production, you would verify this root against
      // a known block header
      return calculatedRoot !== null
    } catch (error) {
      return false
    }
  }

  /**
   * Build a BEEF chain with automatic dependency resolution
   */
  async buildBEEFWithDependencies(
    privateKey: PrivateKey,
    rootUTXO: {
      txid: string
      vout: number
      satoshis: number
      script: Script
      merkleProof?: MerkleProof
    },
    operations: Array<{
      type: 'transfer' | 'split' | 'consolidate'
      params: any
    }>
  ): Promise<number[]> {
    try {
      const transactions: Transaction[] = []
      const proofs = new Map<string, MerkleProof>()

      // Add root transaction proof if available
      if (rootUTXO.merkleProof) {
        proofs.set(rootUTXO.txid, rootUTXO.merkleProof)
      }

      let currentOutputs = [{
        txid: rootUTXO.txid,
        vout: rootUTXO.vout,
        satoshis: rootUTXO.satoshis,
        script: rootUTXO.script
      }]

      // Process each operation
      for (const operation of operations) {
        let tx: Transaction

        switch (operation.type) {
          case 'transfer':
            tx = await this.buildTransferTx(
              privateKey,
              currentOutputs[0],
              operation.params
            )
            break

          case 'split':
            tx = await this.buildSplitTx(
              privateKey,
              currentOutputs[0],
              operation.params
            )
            break

          case 'consolidate':
            tx = await this.buildConsolidateTx(
              privateKey,
              currentOutputs,
              operation.params
            )
            break

          default:
            throw new Error(`Unknown operation type: ${operation.type}`)
        }

        transactions.push(tx)

        // Update current outputs for next operation
        currentOutputs = tx.outputs.map((output, vout) => ({
          txid: tx.id('hex'),
          vout,
          satoshis: output.satoshis,
          script: output.lockingScript
        }))
      }

      // Create BEEF with all transactions
      return await this.createBEEFChain(transactions, proofs)
    } catch (error) {
      throw new Error(`BEEF chain with dependencies failed: ${error.message}`)
    }
  }

  private async buildTransferTx(
    privateKey: PrivateKey,
    input: any,
    params: { toAddress: string; amount: number }
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: input.txid,
      sourceOutputIndex: input.vout,
      unlockingScriptTemplate: new P2PKH().unlock(privateKey),
      sequence: 0xffffffff
    })

    tx.addOutput({
      satoshis: params.amount,
      lockingScript: Script.fromAddress(params.toAddress)
    })

    const fee = 500
    const change = input.satoshis - params.amount - fee

    if (change > 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  private async buildSplitTx(
    privateKey: PrivateKey,
    input: any,
    params: { count: number }
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: input.txid,
      sourceOutputIndex: input.vout,
      unlockingScriptTemplate: new P2PKH().unlock(privateKey),
      sequence: 0xffffffff
    })

    const fee = 500 + (params.count * 50)
    const amountPerOutput = Math.floor((input.satoshis - fee) / params.count)

    for (let i = 0; i < params.count; i++) {
      tx.addOutput({
        satoshis: amountPerOutput,
        lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  private async buildConsolidateTx(
    privateKey: PrivateKey,
    inputs: any[],
    params: { targetAddress?: string }
  ): Promise<Transaction> {
    const tx = new Transaction()

    for (const input of inputs) {
      tx.addInput({
        sourceTXID: input.txid,
        sourceOutputIndex: input.vout,
        unlockingScriptTemplate: new P2PKH().unlock(privateKey),
        sequence: 0xffffffff
      })
    }

    const totalInput = inputs.reduce((sum, i) => sum + i.satoshis, 0)
    const fee = 500 + (inputs.length * 150)

    const outputScript = params.targetAddress
      ? Script.fromAddress(params.targetAddress)
      : new P2PKH().lock(privateKey.toPublicKey().toHash())

    tx.addOutput({
      satoshis: totalInput - fee,
      lockingScript: outputScript
    })

    await tx.sign()
    return tx
  }
}

/**
 * Usage Example
 */
async function beefChainExample() {
  const manager = new BEEFChainManager()
  const privateKey = PrivateKey.fromWif('your-wif-key')

  const rootUTXO = {
    txid: 'root-txid...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(privateKey.toPublicKey().toHash())
  }

  // Build a BEEF chain with multiple operations
  const beefBytes = await manager.buildBEEFWithDependencies(
    privateKey,
    rootUTXO,
    [
      { type: 'split', params: { count: 3 } },
      { type: 'transfer', params: { toAddress: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa', amount: 10000 } }
    ]
  )

  console.log('BEEF chain created:', beefBytes.length, 'bytes')

  // Parse and validate the BEEF chain
  const parsed = await manager.parseBEEFChain(beefBytes)
  console.log('Valid:', parsed.isValid)
  console.log('Transactions:', parsed.transactions.length)
}
```

## Advanced Chain Patterns

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Advanced Chain Pattern Builder
 *
 * Implements complex transaction chain patterns for various use cases.
 */
class AdvancedChainPatterns {
  /**
   * Create a payment channel chain
   *
   * Builds a series of transactions representing state updates in a payment channel
   */
  async buildPaymentChannelChain(
    privateKey1: PrivateKey,
    privateKey2: PrivateKey,
    fundingUTXO: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    states: Array<{ toParty1: number; toParty2: number }>
  ): Promise<Transaction[]> {
    try {
      const chain: Transaction[] = []

      for (let i = 0; i < states.length; i++) {
        const state = states[i]
        const tx = new Transaction()

        // Input from funding transaction (or previous state)
        const inputTxid = i === 0 ? fundingUTXO.txid : chain[i - 1].id('hex')
        const inputVout = i === 0 ? fundingUTXO.vout : 0

        tx.addInput({
          sourceTXID: inputTxid,
          sourceOutputIndex: inputVout,
          // In a real payment channel, this would be a 2-of-2 multisig
          unlockingScriptTemplate: new P2PKH().unlock(privateKey1),
          sequence: i // Sequence number for relative timelock
        })

        // Output to party 1
        if (state.toParty1 > 0) {
          tx.addOutput({
            satoshis: state.toParty1,
            lockingScript: new P2PKH().lock(privateKey1.toPublicKey().toHash())
          })
        }

        // Output to party 2
        if (state.toParty2 > 0) {
          tx.addOutput({
            satoshis: state.toParty2,
            lockingScript: new P2PKH().lock(privateKey2.toPublicKey().toHash())
          })
        }

        await tx.sign()
        chain.push(tx)

        console.log(`Payment channel state ${i + 1}: Party1=${state.toParty1}, Party2=${state.toParty2}`)
      }

      return chain
    } catch (error) {
      throw new Error(`Payment channel chain failed: ${error.message}`)
    }
  }

  /**
   * Create a time-locked transaction chain
   *
   * Each transaction in the chain can only be broadcast after a certain time
   */
  async buildTimeLockedChain(
    privateKey: PrivateKey,
    initialUTXO: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    schedule: Array<{
      amount: number
      lockTime: number // Unix timestamp
      recipient: string
    }>
  ): Promise<Transaction[]> {
    try {
      const chain: Transaction[] = []
      let currentUTXO = initialUTXO

      for (const payment of schedule) {
        const tx = new Transaction()

        // Set lock time for the transaction
        tx.lockTime = payment.lockTime

        tx.addInput({
          sourceTXID: currentUTXO.txid,
          sourceOutputIndex: currentUTXO.vout,
          unlockingScriptTemplate: new P2PKH().unlock(privateKey),
          sequence: 0xfffffffe // Required for lockTime to be enforced
        })

        // Payment output
        tx.addOutput({
          satoshis: payment.amount,
          lockingScript: Script.fromAddress(payment.recipient)
        })

        // Change output for next transaction in chain
        const fee = 500
        const change = currentUTXO.satoshis - payment.amount - fee

        if (change > 546) {
          const outputScript = new P2PKH().lock(privateKey.toPublicKey().toHash())

          tx.addOutput({
            satoshis: change,
            lockingScript: outputScript
          })

          currentUTXO = {
            txid: tx.id('hex'),
            vout: 1,
            satoshis: change,
            script: outputScript
          }
        }

        await tx.sign()
        chain.push(tx)

        const lockDate = new Date(payment.lockTime * 1000)
        console.log(`Scheduled payment: ${payment.amount} sats on ${lockDate.toISOString()}`)
      }

      return chain
    } catch (error) {
      throw new Error(`Time-locked chain creation failed: ${error.message}`)
    }
  }

  /**
   * Create a conditional transaction chain
   *
   * Chain with branching based on conditions (simulated with different signing paths)
   */
  async buildConditionalChain(
    privateKey: PrivateKey,
    initialUTXO: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    branches: Array<{
      condition: string
      amount: number
      recipient: string
    }>
  ): Promise<Map<string, Transaction[]>> {
    try {
      const chains = new Map<string, Transaction[]>()

      for (const branch of branches) {
        const chain: Transaction[] = []

        // First transaction: setup
        const setupTx = new Transaction()

        setupTx.addInput({
          sourceTXID: initialUTXO.txid,
          sourceOutputIndex: initialUTXO.vout,
          unlockingScriptTemplate: new P2PKH().unlock(privateKey),
          sequence: 0xffffffff
        })

        const fee = 500
        const outputScript = new P2PKH().lock(privateKey.toPublicKey().toHash())

        setupTx.addOutput({
          satoshis: branch.amount + fee,
          lockingScript: outputScript
        })

        await setupTx.sign()
        chain.push(setupTx)

        // Second transaction: execute condition
        const executeTx = new Transaction()

        executeTx.addInput({
          sourceTXID: setupTx.id('hex'),
          sourceOutputIndex: 0,
          unlockingScriptTemplate: new P2PKH().unlock(privateKey),
          sequence: 0xffffffff
        })

        executeTx.addOutput({
          satoshis: branch.amount,
          lockingScript: Script.fromAddress(branch.recipient)
        })

        await executeTx.sign()
        chain.push(executeTx)

        chains.set(branch.condition, chain)

        console.log(`Created conditional chain for: ${branch.condition}`)
      }

      return chains
    } catch (error) {
      throw new Error(`Conditional chain creation failed: ${error.message}`)
    }
  }
}

/**
 * Usage Example
 */
async function advancedChainExample() {
  const patterns = new AdvancedChainPatterns()
  const privateKey1 = PrivateKey.fromWif('party1-wif')
  const privateKey2 = PrivateKey.fromWif('party2-wif')

  const fundingUTXO = {
    txid: 'funding-txid...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(privateKey1.toPublicKey().toHash())
  }

  // Payment channel with state updates
  const channelStates = [
    { toParty1: 60000, toParty2: 40000 },
    { toParty1: 50000, toParty2: 50000 },
    { toParty1: 40000, toParty2: 60000 }
  ]

  const channelChain = await patterns.buildPaymentChannelChain(
    privateKey1,
    privateKey2,
    fundingUTXO,
    channelStates
  )

  console.log('Payment channel chain created with', channelChain.length, 'states')
}
```

## Chain Validation and Management

```typescript
import { Transaction } from '@bsv/sdk'

/**
 * Transaction Chain Validator
 *
 * Validate and analyze transaction chains for correctness and efficiency.
 */
class ChainValidator {
  /**
   * Validate a transaction chain
   */
  validateChain(transactions: Transaction[]): {
    isValid: boolean
    errors: string[]
    warnings: string[]
  } {
    const errors: string[] = []
    const warnings: string[] = []

    // Check chain continuity
    for (let i = 1; i < transactions.length; i++) {
      const prevTx = transactions[i - 1]
      const currentTx = transactions[i]

      // Verify that current transaction spends from previous
      const spendsFromPrevious = currentTx.inputs.some(
        input => input.sourceTXID === prevTx.id('hex')
      )

      if (!spendsFromPrevious) {
        errors.push(
          `Transaction ${i} does not spend from previous transaction`
        )
      }
    }

    // Check for dust outputs
    for (let i = 0; i < transactions.length; i++) {
      const tx = transactions[i]

      for (let j = 0; j < tx.outputs.length; j++) {
        if (tx.outputs[j].satoshis < 546) {
          warnings.push(
            `Transaction ${i} output ${j} is below dust threshold`
          )
        }
      }
    }

    // Calculate total fees
    let totalFees = 0
    for (const tx of transactions) {
      const fee = this.calculateTransactionFee(tx)
      totalFees += fee

      if (fee < 500) {
        warnings.push(
          `Transaction ${tx.id('hex')} has low fee: ${fee} satoshis`
        )
      }
    }

    console.log(`Chain validation: ${errors.length} errors, ${warnings.length} warnings`)
    console.log(`Total chain fees: ${totalFees} satoshis`)

    return {
      isValid: errors.length === 0,
      errors,
      warnings
    }
  }

  /**
   * Calculate transaction fee
   */
  private calculateTransactionFee(tx: Transaction): number {
    const totalInput = tx.inputs.reduce((sum, input) => {
      // In practice, you'd need to look up the input amounts
      return sum
    }, 0)

    const totalOutput = tx.outputs.reduce((sum, output) => sum + output.satoshis, 0)

    return totalInput - totalOutput
  }

  /**
   * Analyze chain efficiency
   */
  analyzeChainEfficiency(transactions: Transaction[]): {
    totalSize: number
    averageSize: number
    totalFees: number
    feeRate: number
  } {
    let totalSize = 0
    let totalFees = 0

    for (const tx of transactions) {
      const size = tx.toHex().length / 2
      totalSize += size

      const fee = this.calculateTransactionFee(tx)
      totalFees += fee
    }

    const averageSize = totalSize / transactions.length
    const feeRate = totalFees / totalSize

    console.log('Chain Efficiency Analysis:')
    console.log(`  Total size: ${totalSize} bytes`)
    console.log(`  Average size: ${averageSize.toFixed(2)} bytes`)
    console.log(`  Total fees: ${totalFees} satoshis`)
    console.log(`  Fee rate: ${feeRate.toFixed(4)} sats/byte`)

    return {
      totalSize,
      averageSize,
      totalFees,
      feeRate
    }
  }
}
```

## Related Examples

- [Transaction Building](../transaction-building/README.md)
- [UTXO Management](../utxo-management/README.md)
- [SPV Verification](../spv-verification/README.md)
- [Transaction Broadcasting](../transaction-broadcasting/README.md)

## See Also

**SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md) - Core transaction building
- [BEEF](../../sdk-components/beef/README.md) - BEEF format specification
- [SPV](../../sdk-components/spv/README.md) - SPV verification
- [Merkle Proofs](../../sdk-components/merkle-proofs/README.md) - Merkle proof verification

**Learning Paths:**
- [SPV Basics](../../learning-paths/intermediate/spv-basics/README.md)
- [Advanced Transactions](../../learning-paths/advanced/complex-transactions/README.md)
- [Transaction Chains](../../learning-paths/advanced/transaction-chains/README.md)
