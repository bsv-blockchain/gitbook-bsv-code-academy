# Transaction Signing

Complete examples for signing BSV transactions with various signature types and patterns.

## Overview

This code feature demonstrates practical transaction signing patterns, including single and multi-signature transactions, different SIGHASH types, and advanced signing scenarios.

**Related SDK Components:**
- [Signatures](../../sdk-components/signatures/README.md)
- [Transaction](../../sdk-components/transaction/README.md)
- [Private Keys](../../sdk-components/private-keys/README.md)

## Basic Transaction Signing

```typescript
import { Transaction, PrivateKey, P2PKH, SigHash } from '@bsv/sdk'

/**
 * Basic Transaction Signing
 *
 * Sign a simple P2PKH transaction
 */
class BasicSigner {
  /**
   * Sign a transaction with a single key
   */
  async signTransaction(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex: number = 0
  ): Promise<Transaction> {
    // Set the unlocking script template for the input
    tx.inputs[inputIndex].unlockingScriptTemplate = new P2PKH().unlock(
      privateKey,
      'all',  // SIGHASH type
      true    // Anyone can pay
    )

    // Sign the transaction
    await tx.sign()

    return tx
  }

  /**
   * Sign transaction with default SIGHASH_ALL
   */
  async signWithSigHashAll(
    tx: Transaction,
    privateKey: PrivateKey
  ): Promise<Transaction> {
    for (let i = 0; i < tx.inputs.length; i++) {
      tx.inputs[i].unlockingScriptTemplate = new P2PKH().unlock(privateKey)
    }

    await tx.sign()
    return tx
  }
}

/**
 * Usage Example
 */
async function example() {
  const signer = new BasicSigner()
  const privateKey = PrivateKey.fromWif('your-wif-key')

  const tx = new Transaction()
  // ... add inputs and outputs ...

  const signedTx = await signer.signTransaction(tx, privateKey)
  console.log('Signed transaction:', signedTx.toHex())
}
```

## SIGHASH Types

```typescript
import { Transaction, PrivateKey, P2PKH, Signature } from '@bsv/sdk'

/**
 * SIGHASH Type Examples
 *
 * Demonstrates different signature hash types and their use cases
 */
class SigHashExamples {
  /**
   * SIGHASH_ALL - Signs all inputs and outputs (default, most secure)
   */
  async signAll(tx: Transaction, privateKey: PrivateKey): Promise<Transaction> {
    for (let i = 0; i < tx.inputs.length; i++) {
      tx.inputs[i].unlockingScriptTemplate = new P2PKH().unlock(
        privateKey,
        'all', // SIGHASH_ALL
        false
      )
    }

    await tx.sign()
    return tx
  }

  /**
   * SIGHASH_NONE - Signs all inputs, allows any outputs
   * Use case: Blank check - recipient can add their own outputs
   */
  async signNone(tx: Transaction, privateKey: PrivateKey): Promise<Transaction> {
    for (let i = 0; i < tx.inputs.length; i++) {
      tx.inputs[i].unlockingScriptTemplate = new P2PKH().unlock(
        privateKey,
        'none', // SIGHASH_NONE
        false
      )
    }

    await tx.sign()
    return tx
  }

  /**
   * SIGHASH_SINGLE - Signs one input and corresponding output
   * Use case: Multiple parties contribute inputs to a shared transaction
   */
  async signSingle(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex: number
  ): Promise<Transaction> {
    tx.inputs[inputIndex].unlockingScriptTemplate = new P2PKH().unlock(
      privateKey,
      'single', // SIGHASH_SINGLE
      false
    )

    await tx.sign()
    return tx
  }

  /**
   * SIGHASH_ALL | ANYONECANPAY - Signs one input and all outputs
   * Use case: Crowdfunding - anyone can add their input to fund outputs
   */
  async signAllAnyoneCanPay(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex: number
  ): Promise<Transaction> {
    tx.inputs[inputIndex].unlockingScriptTemplate = new P2PKH().unlock(
      privateKey,
      'all',
      true  // ANYONECANPAY
    )

    await tx.sign()
    return tx
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
    tx.inputs[inputIndex].unlockingScriptTemplate = new P2PKH().unlock(
      privateKey,
      'none',
      true  // ANYONECANPAY
    )

    await tx.sign()
    return tx
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
    tx.inputs[inputIndex].unlockingScriptTemplate = new P2PKH().unlock(
      privateKey,
      'single',
      true  // ANYONECANPAY
    )

    await tx.sign()
    return tx
  }
}
```

## Multi-Signature Signing

```typescript
import { Transaction, PrivateKey, P2PKH, Script, OP } from '@bsv/sdk'

/**
 * Multi-Signature Signing
 *
 * Sign transactions with multiple required signatures
 */
class MultiSigSigner {
  /**
   * Create 2-of-3 multisig locking script
   */
  create2of3LockingScript(
    pubKey1: string,
    pubKey2: string,
    pubKey3: string
  ): Script {
    const script = new Script()

    script.writeOpCode(OP.OP_2)  // Require 2 signatures
    script.writeBin(Buffer.from(pubKey1, 'hex'))
    script.writeBin(Buffer.from(pubKey2, 'hex'))
    script.writeBin(Buffer.from(pubKey3, 'hex'))
    script.writeOpCode(OP.OP_3)  // Out of 3 keys
    script.writeOpCode(OP.OP_CHECKMULTISIG)

    return script
  }

  /**
   * Sign multisig transaction (first signature)
   */
  async signMultisigFirst(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex: number,
    lockingScript: Script
  ): Promise<{ tx: Transaction; signature: Buffer }> {
    // Create signature
    const preimage = tx.inputs[inputIndex].getPreimage(lockingScript)
    const signature = privateKey.sign(preimage)

    return {
      tx,
      signature: signature.toDER()
    }
  }

  /**
   * Add second signature to complete multisig
   */
  async signMultisigSecond(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex: number,
    lockingScript: Script,
    firstSignature: Buffer
  ): Promise<Transaction> {
    // Create second signature
    const preimage = tx.inputs[inputIndex].getPreimage(lockingScript)
    const signature = privateKey.sign(preimage)

    // Build unlocking script with both signatures
    const unlockingScript = new Script()
    unlockingScript.writeOpCode(OP.OP_0)  // Bug workaround
    unlockingScript.writeBin(firstSignature)
    unlockingScript.writeBin(signature.toDER())

    tx.inputs[inputIndex].unlockingScript = unlockingScript

    return tx
  }

  /**
   * Complete example: 2-of-3 multisig transaction
   */
  async create2of3MultisigTx(
    key1: PrivateKey,
    key2: PrivateKey,
    key3: PrivateKey,
    utxo: UTXO,
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    // Create multisig locking script
    const lockingScript = this.create2of3LockingScript(
      key1.toPublicKey().toString(),
      key2.toPublicKey().toString(),
      key3.toPublicKey().toString()
    )

    // Build transaction
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScript: new Script(),
      sequence: 0xffffffff
    })

    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    // Sign with first key
    const { signature: sig1 } = await this.signMultisigFirst(
      tx,
      key1,
      0,
      lockingScript
    )

    // Sign with second key and complete
    await this.signMultisigSecond(tx, key2, 0, lockingScript, sig1)

    return tx
  }
}
```

## Advanced Signing Patterns

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Advanced Signing Patterns
 */
class AdvancedSigning {
  /**
   * Sign transaction with multiple keys (different inputs)
   */
  async signWithMultipleKeys(
    tx: Transaction,
    privateKeys: PrivateKey[]
  ): Promise<Transaction> {
    if (privateKeys.length !== tx.inputs.length) {
      throw new Error('Number of keys must match number of inputs')
    }

    for (let i = 0; i < tx.inputs.length; i++) {
      tx.inputs[i].unlockingScriptTemplate = new P2PKH().unlock(privateKeys[i])
    }

    await tx.sign()
    return tx
  }

  /**
   * Partial signing - sign only specific inputs
   */
  async partialSign(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndices: number[]
  ): Promise<Transaction> {
    for (const index of inputIndices) {
      if (index >= tx.inputs.length) {
        throw new Error(`Input index ${index} out of bounds`)
      }

      tx.inputs[index].unlockingScriptTemplate = new P2PKH().unlock(privateKey)
    }

    await tx.sign()
    return tx
  }

  /**
   * Sign with custom unlocking script
   */
  async signWithCustomScript(
    tx: Transaction,
    inputIndex: number,
    unlockingScript: Script
  ): Promise<Transaction> {
    tx.inputs[inputIndex].unlockingScript = unlockingScript
    return tx
  }

  /**
   * Verify signature before broadcasting
   */
  verifySignature(
    tx: Transaction,
    inputIndex: number,
    lockingScript: Script
  ): boolean {
    try {
      const input = tx.inputs[inputIndex]
      const preimage = input.getPreimage(lockingScript)

      // Extract signature from unlocking script
      const unlockingScript = input.unlockingScript
      const chunks = unlockingScript.chunks

      if (chunks.length < 2) return false

      const sigBuffer = chunks[0].buf
      const pubKeyBuffer = chunks[1].buf

      if (!sigBuffer || !pubKeyBuffer) return false

      // Verify signature
      const signature = Signature.fromDER(sigBuffer)
      const publicKey = PublicKey.fromString(pubKeyBuffer.toString('hex'))

      return publicKey.verify(preimage, signature)
    } catch (error) {
      console.error('Signature verification failed:', error)
      return false
    }
  }

  /**
   * Sign and verify in one operation
   */
  async signAndVerify(
    tx: Transaction,
    privateKey: PrivateKey,
    inputIndex: number
  ): Promise<{ tx: Transaction; valid: boolean }> {
    // Sign transaction
    tx.inputs[inputIndex].unlockingScriptTemplate = new P2PKH().unlock(privateKey)
    await tx.sign()

    // Verify signature
    const lockingScript = new P2PKH().lock(privateKey.toPublicKey().toHash())
    const valid = this.verifySignature(tx, inputIndex, lockingScript)

    return { tx, valid }
  }
}
```

## Crowdfunding Example

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Crowdfunding Transaction Pattern
 *
 * Multiple contributors add inputs to fund a target output
 */
class CrowdfundingTransaction {
  /**
   * Create crowdfunding transaction template
   */
  createCrowdfundingTemplate(
    targetAddress: string,
    targetAmount: number
  ): Transaction {
    const tx = new Transaction()

    // Add target output (locked to recipient)
    tx.addOutput({
      satoshis: targetAmount,
      lockingScript: Script.fromAddress(targetAddress)
    })

    return tx
  }

  /**
   * Contributor adds their input to crowdfunding transaction
   */
  async contributeToTransaction(
    tx: Transaction,
    contributorKey: PrivateKey,
    utxo: UTXO
  ): Promise<Transaction> {
    // Add contributor's input
    const inputIndex = tx.inputs.length

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(
        contributorKey,
        'all',
        true  // ANYONECANPAY - allows others to add inputs
      ),
      sequence: 0xffffffff
    })

    // Add change output for contributor if needed
    const fee = 200 // Estimated fee per input
    const change = utxo.satoshis - fee

    if (change > 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(contributorKey.toPublicKey().toHash())
      })
    }

    // Sign only this input
    await tx.sign()

    return tx
  }

  /**
   * Check if crowdfunding goal is met
   */
  isFundingComplete(tx: Transaction, targetAmount: number): boolean {
    const totalInput = tx.inputs.reduce((sum, input) => {
      // Would need to look up UTXO values
      return sum
    }, 0)

    return totalInput >= targetAmount
  }
}
```

## Related Examples

- [Transaction Building](../transaction-building/README.md)
- [Multi-Signature](../multi-signature/README.md)
- [Message Signing](../message-signing/README.md)
- [Custom Scripts](../custom-scripts/README.md)

## See Also

**SDK Components:**
- [Signatures](../../sdk-components/signatures/README.md) - Digital signature creation and verification
- [Transaction](../../sdk-components/transaction/README.md) - Transaction construction
- [Private Keys](../../sdk-components/private-keys/README.md) - Key management

**Learning Paths:**
- [First Transaction](../../learning-paths/beginner/first-transaction/README.md)
- [Script Templates](../../learning-paths/intermediate/script-templates/README.md)
