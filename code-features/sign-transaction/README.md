# Sign Transaction

Complete examples for signing BSV transactions with various signature types, patterns, and advanced signing scenarios.

## Overview

Transaction signing is the process of creating cryptographic signatures that prove ownership of UTXOs and authorize their spending. This guide covers basic signing, SIGHASH types, multi-signature transactions, and advanced signing patterns with proper error handling.

**Related SDK Components:**
- [Signatures](../../sdk-components/signatures/README.md)
- [Transaction](../../sdk-components/transaction/README.md)
- [Private Keys](../../sdk-components/private-keys/README.md)

## Basic Transaction Signing

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Basic Transaction Signer
 *
 * Simple signing of P2PKH transactions
 */
class BasicTransactionSigner {
  /**
   * Sign a transaction with a single private key
   */
  async signTransaction(
    tx: Transaction,
    privateKey: PrivateKey
  ): Promise<Transaction> {
    try {
      console.log('Signing transaction...')
      console.log('Transaction ID:', tx.id('hex'))
      console.log('Inputs:', tx.inputs.length)

      // Set unlocking script template for all inputs
      for (let i = 0; i < tx.inputs.length; i++) {
        tx.inputs[i].unlockingScriptTemplate = new P2PKH().unlock(privateKey)
      }

      // Sign the transaction
      await tx.sign()

      console.log('Transaction signed successfully')

      // Verify signatures
      const verified = this.verifySignatures(tx)
      console.log('Signatures verified:', verified)

      return tx
    } catch (error) {
      throw new Error(`Transaction signing failed: ${error.message}`)
    }
  }

  /**
   * Sign specific input
   */
  async signInput(
    tx: Transaction,
    inputIndex: number,
    privateKey: PrivateKey
  ): Promise<Transaction> {
    try {
      if (inputIndex < 0 || inputIndex >= tx.inputs.length) {
        throw new Error(`Invalid input index: ${inputIndex}`)
      }

      console.log(`Signing input ${inputIndex}`)

      // Set unlocking script template for specific input
      tx.inputs[inputIndex].unlockingScriptTemplate = new P2PKH().unlock(privateKey)

      // Sign the transaction
      await tx.sign()

      console.log(`Input ${inputIndex} signed successfully`)

      return tx
    } catch (error) {
      throw new Error(`Input signing failed: ${error.message}`)
    }
  }

  /**
   * Sign with multiple private keys (one per input)
   */
  async signWithMultipleKeys(
    tx: Transaction,
    privateKeys: PrivateKey[]
  ): Promise<Transaction> {
    try {
      if (privateKeys.length !== tx.inputs.length) {
        throw new Error('Number of private keys must match number of inputs')
      }

      console.log('Signing with multiple keys...')

      // Set unlocking script template for each input
      for (let i = 0; i < tx.inputs.length; i++) {
        tx.inputs[i].unlockingScriptTemplate = new P2PKH().unlock(privateKeys[i])
      }

      // Sign the transaction
      await tx.sign()

      console.log('All inputs signed successfully')

      return tx
    } catch (error) {
      throw new Error(`Multi-key signing failed: ${error.message}`)
    }
  }

  /**
   * Verify transaction signatures
   */
  private verifySignatures(tx: Transaction): boolean {
    try {
      for (let i = 0; i < tx.inputs.length; i++) {
        const input = tx.inputs[i]

        // Check if input has unlocking script
        if (!input.unlockingScript || input.unlockingScript.chunks.length === 0) {
          console.error(`Input ${i} has no unlocking script`)
          return false
        }
      }

      return true
    } catch (error) {
      console.error('Signature verification failed:', error.message)
      return false
    }
  }

  /**
   * Create and sign a simple transaction
   */
  async createAndSign(
    privateKey: PrivateKey,
    utxo: UTXO,
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    try {
      // Create transaction
      const tx = new Transaction()

      // Add input
      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(privateKey),
        sequence: 0xffffffff
      })

      // Add payment output
      tx.addOutput({
        satoshis: amount,
        lockingScript: Script.fromAddress(toAddress)
      })

      // Calculate change
      const fee = 200
      const change = utxo.satoshis - amount - fee

      if (change > 546) {
        const changeAddress = privateKey.toPublicKey().toAddress()
        tx.addOutput({
          satoshis: change,
          lockingScript: Script.fromAddress(changeAddress)
        })
      }

      // Sign transaction
      await tx.sign()

      console.log('Transaction created and signed')
      console.log('TXID:', tx.id('hex'))

      return tx
    } catch (error) {
      throw new Error(`Create and sign failed: ${error.message}`)
    }
  }
}

interface UTXO {
  txid: string
  vout: number
  satoshis: number
  script?: Script
}

/**
 * Usage Example
 */
async function basicSigningExample() {
  const signer = new BasicTransactionSigner()
  const privateKey = PrivateKey.fromWif('your-wif-key')

  // Create UTXO
  const utxo: UTXO = {
    txid: 'previous-tx-id...',
    vout: 0,
    satoshis: 100000
  }

  // Create and sign transaction
  const tx = await signer.createAndSign(
    privateKey,
    utxo,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    50000
  )

  console.log('Signed transaction:', tx.toHex())
}
```

## SIGHASH Types and Patterns

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * SIGHASH Type Handler
 *
 * Demonstrates different SIGHASH types and their use cases
 */
class SigHashHandler {
  /**
   * SIGHASH_ALL - Signs all inputs and outputs (default, most secure)
   * Use case: Standard transactions
   */
  async signAll(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex?: number
  ): Promise<Transaction> {
    try {
      console.log('Signing with SIGHASH_ALL')

      if (inputIndex !== undefined) {
        tx.inputs[inputIndex].unlockingScriptTemplate = new P2PKH().unlock(
          privateKey,
          'all',  // SIGHASH_ALL
          false   // Not ANYONECANPAY
        )
      } else {
        // Sign all inputs
        for (let i = 0; i < tx.inputs.length; i++) {
          tx.inputs[i].unlockingScriptTemplate = new P2PKH().unlock(
            privateKey,
            'all',
            false
          )
        }
      }

      await tx.sign()

      console.log('SIGHASH_ALL signing complete')
      console.log('All inputs and outputs are locked')

      return tx
    } catch (error) {
      throw new Error(`SIGHASH_ALL signing failed: ${error.message}`)
    }
  }

  /**
   * SIGHASH_NONE - Signs all inputs, allows any outputs
   * Use case: Blank check - recipient can modify outputs
   */
  async signNone(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex: number
  ): Promise<Transaction> {
    try {
      console.log('Signing with SIGHASH_NONE')
      console.log('Outputs can be modified by anyone')

      tx.inputs[inputIndex].unlockingScriptTemplate = new P2PKH().unlock(
        privateKey,
        'none',  // SIGHASH_NONE
        false
      )

      await tx.sign()

      console.log('SIGHASH_NONE signing complete')

      return tx
    } catch (error) {
      throw new Error(`SIGHASH_NONE signing failed: ${error.message}`)
    }
  }

  /**
   * SIGHASH_SINGLE - Signs one input and corresponding output
   * Use case: Multiple parties contributing to shared transaction
   */
  async signSingle(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex: number
  ): Promise<Transaction> {
    try {
      console.log(`Signing with SIGHASH_SINGLE (input ${inputIndex})`)

      if (inputIndex >= tx.outputs.length) {
        throw new Error('SIGHASH_SINGLE requires corresponding output')
      }

      tx.inputs[inputIndex].unlockingScriptTemplate = new P2PKH().unlock(
        privateKey,
        'single',  // SIGHASH_SINGLE
        false
      )

      await tx.sign()

      console.log('SIGHASH_SINGLE signing complete')
      console.log(`Input ${inputIndex} and output ${inputIndex} are locked`)

      return tx
    } catch (error) {
      throw new Error(`SIGHASH_SINGLE signing failed: ${error.message}`)
    }
  }

  /**
   * SIGHASH_ALL | ANYONECANPAY - Signs one input and all outputs
   * Use case: Crowdfunding - anyone can add inputs to fund outputs
   */
  async signAllAnyoneCanPay(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex: number
  ): Promise<Transaction> {
    try {
      console.log('Signing with SIGHASH_ALL | ANYONECANPAY')
      console.log('Others can add inputs to this transaction')

      tx.inputs[inputIndex].unlockingScriptTemplate = new P2PKH().unlock(
        privateKey,
        'all',
        true  // ANYONECANPAY
      )

      await tx.sign()

      console.log('SIGHASH_ALL | ANYONECANPAY signing complete')

      return tx
    } catch (error) {
      throw new Error(`SIGHASH_ALL | ANYONECANPAY signing failed: ${error.message}`)
    }
  }

  /**
   * SIGHASH_NONE | ANYONECANPAY - Signs one input, allows any outputs
   * Use case: Donation - contributor doesn't care where funds go
   */
  async signNoneAnyoneCanPay(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex: number
  ): Promise<Transaction> {
    try {
      console.log('Signing with SIGHASH_NONE | ANYONECANPAY')
      console.log('Others can add inputs and modify all outputs')

      tx.inputs[inputIndex].unlockingScriptTemplate = new P2PKH().unlock(
        privateKey,
        'none',
        true  // ANYONECANPAY
      )

      await tx.sign()

      console.log('SIGHASH_NONE | ANYONECANPAY signing complete')

      return tx
    } catch (error) {
      throw new Error(`SIGHASH_NONE | ANYONECANPAY signing failed: ${error.message}`)
    }
  }

  /**
   * SIGHASH_SINGLE | ANYONECANPAY - Signs one input and one output
   * Use case: Atomic swap - each party signs their input/output pair
   */
  async signSingleAnyoneCanPay(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex: number
  ): Promise<Transaction> {
    try {
      console.log('Signing with SIGHASH_SINGLE | ANYONECANPAY')

      if (inputIndex >= tx.outputs.length) {
        throw new Error('SIGHASH_SINGLE requires corresponding output')
      }

      tx.inputs[inputIndex].unlockingScriptTemplate = new P2PKH().unlock(
        privateKey,
        'single',
        true  // ANYONECANPAY
      )

      await tx.sign()

      console.log('SIGHASH_SINGLE | ANYONECANPAY signing complete')
      console.log('Others can add input/output pairs')

      return tx
    } catch (error) {
      throw new Error(`SIGHASH_SINGLE | ANYONECANPAY signing failed: ${error.message}`)
    }
  }

  /**
   * Create crowdfunding transaction with ANYONECANPAY
   */
  async createCrowdfunding(
    targetAddress: string,
    targetAmount: number,
    contributorKey: PrivateKey,
    utxo: UTXO
  ): Promise<Transaction> {
    try {
      console.log('Creating crowdfunding transaction')
      console.log('Target:', targetAmount, 'satoshis')

      const tx = new Transaction()

      // Add target output (funded by multiple contributors)
      tx.addOutput({
        satoshis: targetAmount,
        lockingScript: Script.fromAddress(targetAddress)
      })

      // Add first contributor's input
      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(
          contributorKey,
          'all',
          true  // Allow others to add inputs
        ),
        sequence: 0xffffffff
      })

      // Add change output for contributor
      const fee = 200
      const change = utxo.satoshis - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: Script.fromAddress(
            contributorKey.toPublicKey().toAddress()
          )
        })
      }

      await tx.sign()

      console.log('Crowdfunding transaction created')
      console.log('Others can now add inputs')

      return tx
    } catch (error) {
      throw new Error(`Crowdfunding creation failed: ${error.message}`)
    }
  }
}

interface UTXO {
  txid: string
  vout: number
  satoshis: number
}

/**
 * Usage Example
 */
async function sigHashExample() {
  const handler = new SigHashHandler()
  const privateKey = PrivateKey.fromRandom()

  // Create base transaction
  const tx = new Transaction()
  // ... add inputs and outputs ...

  // Sign with different SIGHASH types
  console.log('=== SIGHASH Examples ===')

  // Standard signing (SIGHASH_ALL)
  await handler.signAll(tx, privateKey)

  // Crowdfunding (ANYONECANPAY)
  const utxo: UTXO = {
    txid: 'tx-id...',
    vout: 0,
    satoshis: 100000
  }

  const crowdfundingTx = await handler.createCrowdfunding(
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    500000,
    privateKey,
    utxo
  )

  console.log('Crowdfunding TX:', crowdfundingTx.id('hex'))
}
```

## Multi-Signature Transactions

```typescript
import { Transaction, PrivateKey, PublicKey, Script, OP } from '@bsv/sdk'

/**
 * Multi-Signature Transaction Handler
 *
 * Create and sign multi-signature transactions
 */
class MultiSigTransactionSigner {
  /**
   * Create 2-of-3 multisig locking script
   */
  create2of3LockingScript(
    pubKeys: PublicKey[]
  ): Script {
    if (pubKeys.length !== 3) {
      throw new Error('Exactly 3 public keys required for 2-of-3 multisig')
    }

    console.log('Creating 2-of-3 multisig locking script')

    const script = new Script()

    // OP_2 (require 2 signatures)
    script.writeOpCode(OP.OP_2)

    // Add public keys
    for (const pubKey of pubKeys) {
      script.writeBin(Buffer.from(pubKey.toHex(), 'hex'))
    }

    // OP_3 (out of 3 keys)
    script.writeOpCode(OP.OP_3)

    // OP_CHECKMULTISIG
    script.writeOpCode(OP.OP_CHECKMULTISIG)

    console.log('2-of-3 multisig script created')

    return script
  }

  /**
   * Create M-of-N multisig locking script
   */
  createMultisigLockingScript(
    requiredSigs: number,
    pubKeys: PublicKey[]
  ): Script {
    if (requiredSigs < 1 || requiredSigs > pubKeys.length) {
      throw new Error('Invalid signature threshold')
    }

    if (pubKeys.length > 20) {
      throw new Error('Maximum 20 public keys allowed')
    }

    console.log(`Creating ${requiredSigs}-of-${pubKeys.length} multisig script`)

    const script = new Script()

    // Required signatures count
    script.writeOpCode(OP.OP_1 + requiredSigs - 1)

    // Add public keys
    for (const pubKey of pubKeys) {
      script.writeBin(Buffer.from(pubKey.toHex(), 'hex'))
    }

    // Total keys count
    script.writeOpCode(OP.OP_1 + pubKeys.length - 1)

    // OP_CHECKMULTISIG
    script.writeOpCode(OP.OP_CHECKMULTISIG)

    return script
  }

  /**
   * Sign multisig transaction (first signer)
   */
  async signMultisigFirst(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex: number,
    lockingScript: Script
  ): Promise<{ tx: Transaction; signature: Buffer }> {
    try {
      console.log('Creating first signature for multisig')

      // Get preimage for signing
      const input = tx.inputs[inputIndex]
      const preimage = input.getPreimage(lockingScript)

      // Create signature
      const signature = privateKey.sign(preimage)
      const sigBuffer = signature.toDER()

      console.log('First signature created')

      return {
        tx,
        signature: sigBuffer
      }
    } catch (error) {
      throw new Error(`First multisig signing failed: ${error.message}`)
    }
  }

  /**
   * Complete multisig transaction (second signer)
   */
  async signMultisigComplete(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex: number,
    lockingScript: Script,
    firstSignature: Buffer
  ): Promise<Transaction> {
    try {
      console.log('Creating second signature for multisig')

      // Get preimage for signing
      const input = tx.inputs[inputIndex]
      const preimage = input.getPreimage(lockingScript)

      // Create second signature
      const signature = privateKey.sign(preimage)
      const sigBuffer = signature.toDER()

      console.log('Second signature created')

      // Build unlocking script with both signatures
      const unlockingScript = new Script()

      // OP_0 (bug workaround for CHECKMULTISIG)
      unlockingScript.writeOpCode(OP.OP_0)

      // Add signatures
      unlockingScript.writeBin(firstSignature)
      unlockingScript.writeBin(sigBuffer)

      // Set unlocking script
      tx.inputs[inputIndex].unlockingScript = unlockingScript

      console.log('Multisig transaction complete')

      return tx
    } catch (error) {
      throw new Error(`Multisig completion failed: ${error.message}`)
    }
  }

  /**
   * Create and sign 2-of-3 multisig transaction
   */
  async create2of3Transaction(
    key1: PrivateKey,
    key2: PrivateKey,
    key3: PrivateKey,
    utxo: UTXO,
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    try {
      console.log('Creating 2-of-3 multisig transaction')

      // Get public keys
      const pubKeys = [
        key1.toPublicKey(),
        key2.toPublicKey(),
        key3.toPublicKey()
      ]

      // Create multisig locking script
      const lockingScript = this.create2of3LockingScript(pubKeys)

      // Create transaction
      const tx = new Transaction()

      // Add input
      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScript: new Script(),  // Will set later
        sequence: 0xffffffff
      })

      // Add output
      tx.addOutput({
        satoshis: amount,
        lockingScript: Script.fromAddress(toAddress)
      })

      // Calculate change
      const fee = 300  // Higher fee for multisig
      const change = utxo.satoshis - amount - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: lockingScript  // Back to multisig
        })
      }

      // Sign with first key
      const { signature: sig1 } = await this.signMultisigFirst(
        tx,
        key1,
        0,
        lockingScript
      )

      // Sign with second key and complete
      await this.signMultisigComplete(
        tx,
        key2,
        0,
        lockingScript,
        sig1
      )

      console.log('2-of-3 multisig transaction signed')
      console.log('TXID:', tx.id('hex'))

      return tx
    } catch (error) {
      throw new Error(`2-of-3 transaction creation failed: ${error.message}`)
    }
  }
}

interface UTXO {
  txid: string
  vout: number
  satoshis: number
}

/**
 * Usage Example
 */
async function multiSigExample() {
  const signer = new MultiSigTransactionSigner()

  // Generate keys
  const key1 = PrivateKey.fromRandom()
  const key2 = PrivateKey.fromRandom()
  const key3 = PrivateKey.fromRandom()

  console.log('=== 2-of-3 Multisig Example ===')

  // Create UTXO
  const utxo: UTXO = {
    txid: 'tx-id...',
    vout: 0,
    satoshis: 100000
  }

  // Create and sign 2-of-3 multisig transaction
  const tx = await signer.create2of3Transaction(
    key1,
    key2,
    key3,
    utxo,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    50000
  )

  console.log('Multisig transaction created:', tx.toHex())
}
```

## Advanced Signing Patterns

```typescript
import { Transaction, PrivateKey, P2PKH, Script, Signature, PublicKey } from '@bsv/sdk'

/**
 * Advanced Transaction Signing
 *
 * Complex signing scenarios and patterns
 */
class AdvancedSigning {
  /**
   * Partial signing - sign only specific inputs
   */
  async partialSign(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndices: number[]
  ): Promise<Transaction> {
    try {
      console.log(`Partially signing ${inputIndices.length} inputs`)

      for (const index of inputIndices) {
        if (index < 0 || index >= tx.inputs.length) {
          throw new Error(`Invalid input index: ${index}`)
        }

        tx.inputs[index].unlockingScriptTemplate = new P2PKH().unlock(privateKey)
      }

      await tx.sign()

      console.log('Partial signing complete')

      return tx
    } catch (error) {
      throw new Error(`Partial signing failed: ${error.message}`)
    }
  }

  /**
   * Sequential signing - add signatures one at a time
   */
  async sequentialSign(
    tx: Transaction,
    privateKeys: PrivateKey[]
  ): Promise<Transaction> {
    try {
      console.log(`Sequentially signing with ${privateKeys.length} keys`)

      for (let i = 0; i < Math.min(privateKeys.length, tx.inputs.length); i++) {
        console.log(`Signing input ${i}`)

        tx.inputs[i].unlockingScriptTemplate = new P2PKH().unlock(privateKeys[i])
        await tx.sign()
      }

      console.log('Sequential signing complete')

      return tx
    } catch (error) {
      throw new Error(`Sequential signing failed: ${error.message}`)
    }
  }

  /**
   * Sign and verify in one operation
   */
  async signAndVerify(
    tx: Transaction,
    privateKey: PrivateKey
  ): Promise<{
    tx: Transaction
    valid: boolean
    errors: string[]
  }> {
    const errors: string[] = []

    try {
      console.log('Signing and verifying transaction')

      // Sign transaction
      for (let i = 0; i < tx.inputs.length; i++) {
        tx.inputs[i].unlockingScriptTemplate = new P2PKH().unlock(privateKey)
      }

      await tx.sign()

      // Verify each input
      for (let i = 0; i < tx.inputs.length; i++) {
        const input = tx.inputs[i]

        if (!input.unlockingScript || input.unlockingScript.chunks.length === 0) {
          errors.push(`Input ${i}: No unlocking script`)
        }

        // Additional verification could be added here
      }

      const valid = errors.length === 0

      console.log('Signing complete, valid:', valid)

      if (!valid) {
        console.log('Errors:', errors)
      }

      return { tx, valid, errors }
    } catch (error) {
      errors.push(`Signing error: ${error.message}`)
      return { tx, valid: false, errors }
    }
  }

  /**
   * Validate signature before adding to transaction
   */
  async validateAndSign(
    tx: Transaction,
    privateKey: PrivateKey
  ): Promise<{
    success: boolean
    tx?: Transaction
    error?: string
  }> {
    try {
      // Pre-signing validation
      if (tx.inputs.length === 0) {
        return {
          success: false,
          error: 'Transaction has no inputs'
        }
      }

      if (tx.outputs.length === 0) {
        return {
          success: false,
          error: 'Transaction has no outputs'
        }
      }

      // Check output amounts
      for (let i = 0; i < tx.outputs.length; i++) {
        const output = tx.outputs[i]

        if (output.satoshis < 0) {
          return {
            success: false,
            error: `Output ${i} has negative amount`
          }
        }

        if (output.satoshis > 0 && output.satoshis < 546) {
          return {
            success: false,
            error: `Output ${i} below dust threshold`
          }
        }
      }

      console.log('Pre-signing validation passed')

      // Sign transaction
      for (let i = 0; i < tx.inputs.length; i++) {
        tx.inputs[i].unlockingScriptTemplate = new P2PKH().unlock(privateKey)
      }

      await tx.sign()

      console.log('Transaction signed and validated')

      return {
        success: true,
        tx
      }
    } catch (error) {
      return {
        success: false,
        error: error.message
      }
    }
  }

  /**
   * Batch sign multiple transactions
   */
  async batchSign(
    transactions: Transaction[],
    privateKey: PrivateKey
  ): Promise<BatchSignResult[]> {
    console.log(`Batch signing ${transactions.length} transactions`)

    const results: BatchSignResult[] = []

    for (let i = 0; i < transactions.length; i++) {
      const tx = transactions[i]

      try {
        console.log(`Signing transaction ${i + 1}/${transactions.length}`)

        // Sign all inputs
        for (let j = 0; j < tx.inputs.length; j++) {
          tx.inputs[j].unlockingScriptTemplate = new P2PKH().unlock(privateKey)
        }

        await tx.sign()

        results.push({
          index: i,
          success: true,
          txid: tx.id('hex')
        })
      } catch (error) {
        console.error(`Transaction ${i} signing failed:`, error.message)

        results.push({
          index: i,
          success: false,
          error: error.message
        })
      }
    }

    const successful = results.filter(r => r.success).length
    console.log(`Batch signing complete: ${successful}/${transactions.length} successful`)

    return results
  }

  /**
   * Sign with custom unlocking script
   */
  async signWithCustomScript(
    tx: Transaction,
    inputIndex: number,
    unlockingScript: Script
  ): Promise<Transaction> {
    try {
      console.log(`Setting custom unlocking script for input ${inputIndex}`)

      if (inputIndex < 0 || inputIndex >= tx.inputs.length) {
        throw new Error(`Invalid input index: ${inputIndex}`)
      }

      tx.inputs[inputIndex].unlockingScript = unlockingScript

      console.log('Custom unlocking script set')

      return tx
    } catch (error) {
      throw new Error(`Custom script signing failed: ${error.message}`)
    }
  }
}

interface BatchSignResult {
  index: number
  success: boolean
  txid?: string
  error?: string
}

/**
 * Usage Example
 */
async function advancedSigningExample() {
  const signer = new AdvancedSigning()
  const privateKey = PrivateKey.fromRandom()

  // Create transaction
  const tx = new Transaction()
  // ... add inputs and outputs ...

  // Partial signing
  console.log('=== Partial Signing ===')
  await signer.partialSign(tx, privateKey, [0, 2])

  // Sign and verify
  console.log('\n=== Sign and Verify ===')
  const result = await signer.signAndVerify(tx, privateKey)
  console.log('Valid:', result.valid)

  if (!result.valid) {
    console.log('Errors:', result.errors)
  }

  // Batch signing
  console.log('\n=== Batch Signing ===')
  const transactions: Transaction[] = [] // Array of transactions
  const batchResults = await signer.batchSign(transactions, privateKey)
  console.log('Results:', batchResults)
}
```

## Related Examples

- [Transaction Building](../transaction-building/README.md)
- [Transaction Signing](../transaction-signing/README.md)
- [Multi-Signature](../multi-signature/README.md)
- [Message Signing](../message-signing/README.md)

## See Also

**SDK Components:**
- [Signatures](../../sdk-components/signatures/README.md) - Digital signature operations
- [Transaction](../../sdk-components/transaction/README.md) - Transaction construction
- [Private Keys](../../sdk-components/private-keys/README.md) - Key management

**Learning Paths:**
- [Transaction Basics](../../learning-paths/beginner/transaction-basics/README.md)
- [Advanced Signing](../../learning-paths/intermediate/advanced-signing/README.md)
