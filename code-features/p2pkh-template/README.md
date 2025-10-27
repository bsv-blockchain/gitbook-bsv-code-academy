# P2PKH Template

Complete examples for using the Pay-to-Public-Key-Hash (P2PKH) template with all SIGHASH types.

## Overview

This code feature demonstrates the P2PKH template, the most common Bitcoin transaction type. P2PKH locks coins to a public key hash (Bitcoin address) and requires a signature from the corresponding private key to unlock them. This guide covers all signature hash types and their practical applications.

**Related SDK Components:**
- [P2PKH Template](../../sdk-components/p2pkh-template/README.md)
- [Transaction](../../sdk-components/transaction/README.md)
- [Signatures](../../sdk-components/signatures/README.md)
- [Script Templates](../../sdk-components/script-templates/README.md)

## Basic P2PKH Transaction

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Basic P2PKH Transaction
 *
 * Creates a standard transaction using P2PKH template for both locking and unlocking
 */
class BasicP2PKH {
  /**
   * Create a simple P2PKH payment
   */
  async createPayment(
    fromPrivateKey: PrivateKey,
    toAddress: string,
    amount: number,
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<Transaction> {
    const tx = new Transaction()

    // Add input with P2PKH unlocking template
    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(fromPrivateKey),
      sequence: 0xffffffff
    })

    // Add output with P2PKH locking script (to address)
    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    // Add change output with P2PKH locking script (to public key hash)
    const fee = 500
    const changeAmount = utxo.satoshis - amount - fee

    if (changeAmount > 546) { // Above dust limit
      tx.addOutput({
        satoshis: changeAmount,
        lockingScript: new P2PKH().lock(fromPrivateKey.toPublicKey().toHash())
      })
    }

    // Sign the transaction
    await tx.sign()

    return tx
  }

  /**
   * Lock coins to a public key hash
   */
  lockToPublicKeyHash(publicKeyHash: number[]): Script {
    return new P2PKH().lock(publicKeyHash)
  }

  /**
   * Lock coins to an address
   */
  lockToAddress(address: string): Script {
    return Script.fromAddress(address)
  }

  /**
   * Create unlocking template for P2PKH
   */
  createUnlockingTemplate(privateKey: PrivateKey): P2PKH {
    return new P2PKH().unlock(privateKey)
  }
}

/**
 * Usage Example
 */
async function example() {
  const p2pkh = new BasicP2PKH()
  const privateKey = PrivateKey.fromWif('your-wif-key')

  const utxo = {
    txid: 'abc123...',
    vout: 0,
    satoshis: 100000
  }

  const tx = await p2pkh.createPayment(
    privateKey,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    50000,
    utxo
  )

  console.log('Transaction ID:', tx.id('hex'))
  console.log('Transaction hex:', tx.toHex())
}
```

## P2PKH with All SIGHASH Types

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * P2PKH SIGHASH Examples
 *
 * Demonstrates all SIGHASH types with P2PKH template
 */
class P2PKHSigHashExamples {
  /**
   * SIGHASH_ALL (default) - Most secure, signs all inputs and outputs
   * Use case: Standard payments where transaction cannot be modified
   */
  async createWithSigHashAll(
    privateKey: PrivateKey,
    utxo: { txid: string; vout: number; satoshis: number },
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(
        privateKey,
        'all',  // SIGHASH_ALL
        false   // Not ANYONECANPAY
      ),
      sequence: 0xffffffff
    })

    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    const change = utxo.satoshis - amount - 500
    if (change > 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * SIGHASH_NONE - Signs all inputs, no outputs
   * Use case: Blank check - recipient determines where funds go
   */
  async createWithSigHashNone(
    privateKey: PrivateKey,
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(
        privateKey,
        'none',  // SIGHASH_NONE
        false
      ),
      sequence: 0xffffffff
    })

    // Outputs can be added/modified by anyone after signing
    // This is intentionally left empty or with placeholder outputs

    await tx.sign()
    return tx
  }

  /**
   * SIGHASH_SINGLE - Signs one input and corresponding output
   * Use case: Multiple parties each sign their input/output pair
   */
  async createWithSigHashSingle(
    privateKey: PrivateKey,
    utxo: { txid: string; vout: number; satoshis: number },
    toAddress: string,
    amount: number,
    inputIndex: number = 0
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(
        privateKey,
        'single',  // SIGHASH_SINGLE
        false
      ),
      sequence: 0xffffffff
    })

    // This output is locked to this input
    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    await tx.sign()
    return tx
  }

  /**
   * SIGHASH_ALL | ANYONECANPAY - Signs one input and all outputs
   * Use case: Crowdfunding - anyone can add inputs to fund the outputs
   */
  async createWithSigHashAllAnyoneCanPay(
    privateKey: PrivateKey,
    utxo: { txid: string; vout: number; satoshis: number },
    fundingGoalAddress: string,
    fundingGoal: number
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(
        privateKey,
        'all',    // SIGHASH_ALL
        true      // ANYONECANPAY - allows others to add inputs
      ),
      sequence: 0xffffffff
    })

    // Funding goal output (others can contribute to reach this)
    tx.addOutput({
      satoshis: fundingGoal,
      lockingScript: Script.fromAddress(fundingGoalAddress)
    })

    // Contributor's change
    const contribution = Math.min(utxo.satoshis - 500, fundingGoal)
    const change = utxo.satoshis - contribution - 500

    if (change > 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * SIGHASH_NONE | ANYONECANPAY - Signs one input, no outputs
   * Use case: Donation where contributor doesn't care about destination
   */
  async createWithSigHashNoneAnyoneCanPay(
    privateKey: PrivateKey,
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(
        privateKey,
        'none',   // SIGHASH_NONE
        true      // ANYONECANPAY
      ),
      sequence: 0xffffffff
    })

    // Outputs can be added/modified by anyone
    // This allows complete flexibility for the recipient

    await tx.sign()
    return tx
  }

  /**
   * SIGHASH_SINGLE | ANYONECANPAY - Signs one input and one output
   * Use case: Atomic swaps - each party signs their input/output independently
   */
  async createWithSigHashSingleAnyoneCanPay(
    privateKey: PrivateKey,
    utxo: { txid: string; vout: number; satoshis: number },
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(
        privateKey,
        'single',  // SIGHASH_SINGLE
        true       // ANYONECANPAY
      ),
      sequence: 0xffffffff
    })

    // This output is paired with this input
    // Other inputs/outputs can be added independently
    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    await tx.sign()
    return tx
  }
}
```

## Advanced P2PKH Patterns

```typescript
import { Transaction, PrivateKey, P2PKH, Script, PublicKey } from '@bsv/sdk'

/**
 * Advanced P2PKH Patterns
 *
 * Complex transaction patterns using P2PKH
 */
class AdvancedP2PKHPatterns {
  /**
   * Batch payments using P2PKH
   */
  async createBatchPayment(
    privateKey: PrivateKey,
    utxos: Array<{ txid: string; vout: number; satoshis: number }>,
    recipients: Array<{ address: string; amount: number }>
  ): Promise<Transaction> {
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
    const fee = 500 + (utxos.length * 150) + (recipients.length * 50)
    const change = totalInput - totalOutput - fee

    if (change > 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * UTXO consolidation with P2PKH
   */
  async consolidateUTXOs(
    privateKey: PrivateKey,
    utxos: Array<{ txid: string; vout: number; satoshis: number }>
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

    // Calculate total and create single output
    const totalInput = utxos.reduce((sum, u) => sum + u.satoshis, 0)
    const fee = 500 + (utxos.length * 150)

    tx.addOutput({
      satoshis: totalInput - fee,
      lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
    })

    await tx.sign()
    return tx
  }

  /**
   * Split UTXO into multiple outputs
   */
  async splitUTXO(
    privateKey: PrivateKey,
    utxo: { txid: string; vout: number; satoshis: number },
    numOutputs: number
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(privateKey),
      sequence: 0xffffffff
    })

    // Calculate amount per output
    const fee = 500 + (numOutputs * 50)
    const amountPerOutput = Math.floor((utxo.satoshis - fee) / numOutputs)

    // Create equal outputs
    for (let i = 0; i < numOutputs; i++) {
      tx.addOutput({
        satoshis: amountPerOutput,
        lockingScript: new P2PKH().lock(privateKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * Create P2PKH output for specific public key
   */
  lockToPublicKey(publicKey: PublicKey, amount: number): {
    satoshis: number
    lockingScript: Script
  } {
    return {
      satoshis: amount,
      lockingScript: new P2PKH().lock(publicKey.toHash())
    }
  }

  /**
   * Verify P2PKH script
   */
  isP2PKH(script: Script): boolean {
    const chunks = script.chunks

    // P2PKH format: OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
    if (chunks.length !== 5) return false

    return (
      chunks[0].opCodeNum === 118 && // OP_DUP
      chunks[1].opCodeNum === 169 && // OP_HASH160
      chunks[2].buf?.length === 20 && // 20-byte hash
      chunks[3].opCodeNum === 136 && // OP_EQUALVERIFY
      chunks[4].opCodeNum === 172    // OP_CHECKSIG
    )
  }

  /**
   * Extract public key hash from P2PKH script
   */
  extractPublicKeyHash(script: Script): number[] | null {
    if (!this.isP2PKH(script)) return null

    const chunks = script.chunks
    const hashBuffer = chunks[2].buf

    if (!hashBuffer || hashBuffer.length !== 20) return null

    return Array.from(hashBuffer)
  }
}
```

## Crowdfunding with P2PKH and ANYONECANPAY

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Crowdfunding Example
 *
 * Demonstrates how multiple contributors can fund a goal using P2PKH with ANYONECANPAY
 */
class CrowdfundingExample {
  /**
   * Create initial crowdfunding transaction
   */
  createCrowdfundingTransaction(
    targetAddress: string,
    targetAmount: number
  ): Transaction {
    const tx = new Transaction()

    // Add the funding goal output
    tx.addOutput({
      satoshis: targetAmount,
      lockingScript: Script.fromAddress(targetAddress)
    })

    return tx
  }

  /**
   * Contributor adds their input to the crowdfunding transaction
   */
  async addContribution(
    tx: Transaction,
    contributorKey: PrivateKey,
    utxo: { txid: string; vout: number; satoshis: number },
    contributionAmount: number
  ): Promise<Transaction> {
    // Clone transaction to avoid modifying original
    const contributorTx = Transaction.fromHex(tx.toHex())

    // Add contributor's input with ANYONECANPAY
    contributorTx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(
        contributorKey,
        'all',
        true  // ANYONECANPAY - allows others to add more inputs
      ),
      sequence: 0xffffffff
    })

    // Add change output for contributor
    const fee = 200
    const change = utxo.satoshis - contributionAmount - fee

    if (change > 546) {
      contributorTx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(contributorKey.toPublicKey().toHash())
      })
    }

    // Sign only the contributor's input
    await contributorTx.sign()

    return contributorTx
  }

  /**
   * Check total contributions
   */
  getTotalContributions(
    tx: Transaction,
    utxoValues: Map<string, number>
  ): number {
    let total = 0

    for (const input of tx.inputs) {
      const key = `${input.sourceTXID}:${input.sourceOutputIndex}`
      const value = utxoValues.get(key)
      if (value) {
        total += value
      }
    }

    return total
  }

  /**
   * Check if funding goal is reached
   */
  isFundingComplete(
    tx: Transaction,
    utxoValues: Map<string, number>,
    targetAmount: number
  ): boolean {
    const total = this.getTotalContributions(tx, utxoValues)
    return total >= targetAmount
  }

  /**
   * Complete example
   */
  async runCrowdfundingExample(
    targetAddress: string,
    targetAmount: number,
    contributors: Array<{
      privateKey: PrivateKey
      utxo: { txid: string; vout: number; satoshis: number }
      amount: number
    }>
  ): Promise<Transaction> {
    // Create initial transaction
    let tx = this.createCrowdfundingTransaction(targetAddress, targetAmount)

    // Each contributor adds their input
    for (const contributor of contributors) {
      tx = await this.addContribution(
        tx,
        contributor.privateKey,
        contributor.utxo,
        contributor.amount
      )
    }

    return tx
  }
}

/**
 * Usage Example
 */
async function crowdfundingExample() {
  const campaign = new CrowdfundingExample()

  // Campaign goal: 1 BSV to this address
  const targetAddress = '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa'
  const targetAmount = 100000000 // 1 BSV in satoshis

  // Three contributors
  const contributors = [
    {
      privateKey: PrivateKey.fromWif('contributor1-wif'),
      utxo: { txid: 'abc...', vout: 0, satoshis: 40000000 },
      amount: 40000000
    },
    {
      privateKey: PrivateKey.fromWif('contributor2-wif'),
      utxo: { txid: 'def...', vout: 0, satoshis: 35000000 },
      amount: 35000000
    },
    {
      privateKey: PrivateKey.fromWif('contributor3-wif'),
      utxo: { txid: 'ghi...', vout: 0, satoshis: 30000000 },
      amount: 25000000
    }
  ]

  const fundedTx = await campaign.runCrowdfundingExample(
    targetAddress,
    targetAmount,
    contributors
  )

  console.log('Crowdfunding transaction:', fundedTx.id('hex'))
}
```

## Related Examples

- [Transaction Building](../transaction-building/README.md)
- [Transaction Signing](../transaction-signing/README.md)
- [Multi-Signature](../multi-signature/README.md)
- [Custom Scripts](../custom-scripts/README.md)

## See Also

**SDK Components:**
- [P2PKH Template](../../sdk-components/p2pkh-template/README.md) - P2PKH template implementation
- [Script Templates](../../sdk-components/script-templates/README.md) - Template system overview
- [Transaction](../../sdk-components/transaction/README.md) - Transaction construction
- [Signatures](../../sdk-components/signatures/README.md) - Signature types and SIGHASH

**Learning Paths:**
- [First Transaction](../../learning-paths/beginner/first-transaction/README.md)
- [Script Templates](../../learning-paths/intermediate/script-templates/README.md)
- [Advanced Scripting](../../learning-paths/advanced/custom-scripts/README.md)
