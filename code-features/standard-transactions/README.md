# Standard Transaction Types

Complete examples for creating standard BSV transaction types including P2PKH, P2PK, and their variations.

## Overview

BSV supports several standard transaction types that define how bitcoins are locked and unlocked. The most common is Pay-to-Public-Key-Hash (P2PKH), but understanding all standard types is essential for building robust applications. This guide covers P2PKH, P2PK (Pay-to-Public-Key), and multi-output patterns.

**Related SDK Components:**
- [P2PKH](../../sdk-components/p2pkh/README.md)
- [Transaction](../../sdk-components/transaction/README.md)
- [Script](../../sdk-components/script/README.md)
- [Script Templates](../../sdk-components/script-templates/README.md)

## P2PKH Transactions

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Pay-to-Public-Key-Hash (P2PKH) Transactions
 *
 * The most common transaction type on BSV
 */
class P2PKHTransactions {
  /**
   * Create a standard P2PKH payment transaction
   */
  async createP2PKHPayment(
    senderKey: PrivateKey,
    recipientAddress: string,
    amount: number,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<Transaction> {
    try {
      if (amount < 546) {
        throw new Error('Amount below dust threshold (546 satoshis)')
      }

      if (amount > utxo.satoshis) {
        throw new Error('Insufficient funds in UTXO')
      }

      const tx = new Transaction()

      // Add input with P2PKH unlocking script
      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(senderKey),
        sequence: 0xffffffff
      })

      // Add payment output with P2PKH locking script
      tx.addOutput({
        satoshis: amount,
        lockingScript: Script.fromAddress(recipientAddress)
      })

      // Calculate and add change output
      const fee = 500
      const change = utxo.satoshis - amount - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('P2PKH payment created')
      console.log('Transaction ID:', tx.id('hex'))
      console.log('Amount:', amount)
      console.log('Recipient:', recipientAddress)

      return tx
    } catch (error) {
      throw new Error(`P2PKH payment failed: ${error.message}`)
    }
  }

  /**
   * Create P2PKH transaction with multiple recipients
   */
  async createMultiRecipientPayment(
    senderKey: PrivateKey,
    recipients: Array<{ address: string; amount: number }>,
    utxos: Array<{
      txid: string
      vout: number
      satoshis: number
      script: Script
    }>
  ): Promise<Transaction> {
    try {
      // Validate recipients
      for (const recipient of recipients) {
        if (recipient.amount < 546) {
          throw new Error(`Amount for ${recipient.address} below dust threshold`)
        }
      }

      const totalAmount = recipients.reduce((sum, r) => sum + r.amount, 0)
      const totalInput = utxos.reduce((sum, u) => sum + u.satoshis, 0)
      const fee = 500 + (utxos.length * 150) + (recipients.length * 50)

      if (totalInput < totalAmount + fee) {
        throw new Error('Insufficient funds')
      }

      const tx = new Transaction()

      // Add all inputs
      for (const utxo of utxos) {
        tx.addInput({
          sourceTXID: utxo.txid,
          sourceOutputIndex: utxo.vout,
          unlockingScriptTemplate: new P2PKH().unlock(senderKey),
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

      // Add change output
      const change = totalInput - totalAmount - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('Multi-recipient payment created')
      console.log('Recipients:', recipients.length)
      console.log('Total amount:', totalAmount)
      console.log('Transaction ID:', tx.id('hex'))

      return tx
    } catch (error) {
      throw new Error(`Multi-recipient payment failed: ${error.message}`)
    }
  }

  /**
   * Create P2PKH transaction with custom fee rate
   */
  async createWithCustomFee(
    senderKey: PrivateKey,
    recipientAddress: string,
    amount: number,
    feeRate: number, // satoshis per byte
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<Transaction> {
    try {
      // Estimate transaction size
      const estimatedSize =
        10 + // version, locktime
        148 + // input (typical P2PKH)
        34 + // payment output
        34 // change output

      const fee = Math.ceil(estimatedSize * feeRate)

      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(senderKey),
        sequence: 0xffffffff
      })

      tx.addOutput({
        satoshis: amount,
        lockingScript: Script.fromAddress(recipientAddress)
      })

      const change = utxo.satoshis - amount - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('P2PKH payment with custom fee')
      console.log('Fee rate:', feeRate, 'sat/byte')
      console.log('Total fee:', fee, 'satoshis')
      console.log('Transaction ID:', tx.id('hex'))

      return tx
    } catch (error) {
      throw new Error(`Custom fee payment failed: ${error.message}`)
    }
  }
}

/**
 * Usage Example
 */
async function p2pkhExample() {
  const p2pkh = new P2PKHTransactions()
  const privateKey = PrivateKey.fromRandom()

  const utxo = {
    txid: 'funding-tx...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(privateKey.toPublicKey().toHash())
  }

  // Simple payment
  const tx1 = await p2pkh.createP2PKHPayment(
    privateKey,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    50000,
    utxo
  )

  // Multi-recipient payment
  const tx2 = await p2pkh.createMultiRecipientPayment(
    privateKey,
    [
      { address: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa', amount: 10000 },
      { address: '1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2', amount: 20000 }
    ],
    [utxo]
  )
}
```

## P2PK Transactions

```typescript
import { Transaction, PrivateKey, PublicKey, Script, OP } from '@bsv/sdk'

/**
 * Pay-to-Public-Key (P2PK) Transactions
 *
 * Original transaction type where coins are sent directly to a public key
 */
class P2PKTransactions {
  /**
   * Create P2PK locking script
   */
  createP2PKLockingScript(publicKey: PublicKey): Script {
    const script = new Script()

    // Format: <pubKey> OP_CHECKSIG
    script.writeBin(publicKey.encode())
    script.writeOpCode(OP.OP_CHECKSIG)

    return script
  }

  /**
   * Create P2PK unlocking script
   */
  createP2PKUnlockingScript(signature: Buffer): Script {
    const script = new Script()

    // Format: <signature>
    script.writeBin(signature)

    return script
  }

  /**
   * Create a P2PK payment transaction
   */
  async createP2PKPayment(
    senderKey: PrivateKey,
    recipientPubKey: PublicKey,
    amount: number,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<Transaction> {
    try {
      const tx = new Transaction()

      // Add input (assuming input is P2PKH)
      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(senderKey),
        sequence: 0xffffffff
      })

      // Add P2PK output
      tx.addOutput({
        satoshis: amount,
        lockingScript: this.createP2PKLockingScript(recipientPubKey)
      })

      // Add change output (P2PKH)
      const fee = 500
      const change = utxo.satoshis - amount - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('P2PK payment created')
      console.log('Transaction ID:', tx.id('hex'))
      console.log('Recipient pubkey:', recipientPubKey.toHex())

      return tx
    } catch (error) {
      throw new Error(`P2PK payment failed: ${error.message}`)
    }
  }

  /**
   * Spend from P2PK output
   */
  async spendP2PKOutput(
    recipientKey: PrivateKey,
    p2pkUTXO: {
      txid: string
      vout: number
      satoshis: number
      lockingScript: Script
    },
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    try {
      const tx = new Transaction()

      // Add P2PK input (initially without unlocking script)
      tx.addInput({
        sourceTXID: p2pkUTXO.txid,
        sourceOutputIndex: p2pkUTXO.vout,
        unlockingScript: new Script(),
        sequence: 0xffffffff
      })

      // Add payment output
      tx.addOutput({
        satoshis: amount,
        lockingScript: Script.fromAddress(toAddress)
      })

      // Add change
      const fee = 500
      const change = p2pkUTXO.satoshis - amount - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(recipientKey.toPublicKey().toHash())
        })
      }

      // Create signature for P2PK input
      const preimage = tx.inputs[0].getPreimage(p2pkUTXO.lockingScript)
      const signature = recipientKey.sign(preimage)

      // Set unlocking script
      tx.inputs[0].unlockingScript = this.createP2PKUnlockingScript(
        signature.toDER()
      )

      console.log('P2PK output spent')
      console.log('Transaction ID:', tx.id('hex'))

      return tx
    } catch (error) {
      throw new Error(`P2PK spend failed: ${error.message}`)
    }
  }

  /**
   * Convert P2PK to P2PKH
   *
   * Spend a P2PK output and send to a P2PKH address
   */
  async convertP2PKtoP2PKH(
    ownerKey: PrivateKey,
    p2pkUTXO: {
      txid: string
      vout: number
      satoshis: number
      lockingScript: Script
    },
    targetAddress: string
  ): Promise<Transaction> {
    try {
      const amount = p2pkUTXO.satoshis - 500 // subtract fee

      const tx = await this.spendP2PKOutput(
        ownerKey,
        p2pkUTXO,
        targetAddress,
        amount
      )

      console.log('Converted P2PK to P2PKH')
      console.log('Target address:', targetAddress)

      return tx
    } catch (error) {
      throw new Error(`Conversion failed: ${error.message}`)
    }
  }
}

/**
 * Usage Example
 */
async function p2pkExample() {
  const p2pk = new P2PKTransactions()

  const sender = PrivateKey.fromRandom()
  const recipient = PrivateKey.fromRandom()

  const utxo = {
    txid: 'funding-tx...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(sender.toPublicKey().toHash())
  }

  // Create P2PK payment
  const tx = await p2pk.createP2PKPayment(
    sender,
    recipient.toPublicKey(),
    50000,
    utxo
  )

  // Later, spend the P2PK output
  const p2pkUTXO = {
    txid: tx.id('hex'),
    vout: 0,
    satoshis: 50000,
    lockingScript: p2pk.createP2PKLockingScript(recipient.toPublicKey())
  }

  await p2pk.spendP2PKOutput(
    recipient,
    p2pkUTXO,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    49000
  )
}
```

## Transaction Templates

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Standard Transaction Templates
 *
 * Reusable templates for common transaction patterns
 */
class TransactionTemplates {
  /**
   * Simple send template
   */
  async simpleSend(params: {
    senderKey: PrivateKey
    recipientAddress: string
    amount: number
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  }): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: params.utxo.txid,
      sourceOutputIndex: params.utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(params.senderKey),
      sequence: 0xffffffff
    })

    tx.addOutput({
      satoshis: params.amount,
      lockingScript: Script.fromAddress(params.recipientAddress)
    })

    const fee = 500
    const change = params.utxo.satoshis - params.amount - fee

    if (change >= 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(params.senderKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * Consolidation template
   *
   * Combine multiple UTXOs into one
   */
  async consolidate(params: {
    ownerKey: PrivateKey
    utxos: Array<{
      txid: string
      vout: number
      satoshis: number
      script: Script
    }>
    targetAddress?: string
  }): Promise<Transaction> {
    const tx = new Transaction()

    // Add all UTXOs as inputs
    for (const utxo of params.utxos) {
      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.ownerKey),
        sequence: 0xffffffff
      })
    }

    // Calculate total
    const total = params.utxos.reduce((sum, u) => sum + u.satoshis, 0)
    const fee = 500 + (params.utxos.length * 150)
    const outputAmount = total - fee

    // Create single output
    const targetScript = params.targetAddress
      ? Script.fromAddress(params.targetAddress)
      : new P2PKH().lock(params.ownerKey.toPublicKey().toHash())

    tx.addOutput({
      satoshis: outputAmount,
      lockingScript: targetScript
    })

    await tx.sign()

    console.log('Consolidated', params.utxos.length, 'UTXOs')
    console.log('Output amount:', outputAmount)

    return tx
  }

  /**
   * Split template
   *
   * Split one UTXO into multiple equal outputs
   */
  async split(params: {
    ownerKey: PrivateKey
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
    numOutputs: number
  }): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: params.utxo.txid,
      sourceOutputIndex: params.utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(params.ownerKey),
      sequence: 0xffffffff
    })

    // Calculate amount per output
    const fee = 500 + (params.numOutputs * 50)
    const amountPerOutput = Math.floor(
      (params.utxo.satoshis - fee) / params.numOutputs
    )

    if (amountPerOutput < 546) {
      throw new Error('Split would create dust outputs')
    }

    // Create equal outputs
    const lockingScript = new P2PKH().lock(params.ownerKey.toPublicKey().toHash())

    for (let i = 0; i < params.numOutputs; i++) {
      tx.addOutput({
        satoshis: amountPerOutput,
        lockingScript
      })
    }

    await tx.sign()

    console.log('Split into', params.numOutputs, 'outputs')
    console.log('Amount per output:', amountPerOutput)

    return tx
  }

  /**
   * Sweep template
   *
   * Send all funds from multiple UTXOs to a destination
   */
  async sweep(params: {
    sourceKey: PrivateKey
    destinationAddress: string
    utxos: Array<{
      txid: string
      vout: number
      satoshis: number
      script: Script
    }>
  }): Promise<Transaction> {
    const tx = new Transaction()

    // Add all inputs
    for (const utxo of params.utxos) {
      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.sourceKey),
        sequence: 0xffffffff
      })
    }

    // Calculate total and send everything
    const total = params.utxos.reduce((sum, u) => sum + u.satoshis, 0)
    const fee = 500 + (params.utxos.length * 150)
    const sweepAmount = total - fee

    tx.addOutput({
      satoshis: sweepAmount,
      lockingScript: Script.fromAddress(params.destinationAddress)
    })

    await tx.sign()

    console.log('Swept', params.utxos.length, 'UTXOs')
    console.log('Total swept:', sweepAmount)
    console.log('Destination:', params.destinationAddress)

    return tx
  }
}

/**
 * Usage Example
 */
async function templatesExample() {
  const templates = new TransactionTemplates()
  const privateKey = PrivateKey.fromRandom()

  const utxo = {
    txid: 'funding-tx...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(privateKey.toPublicKey().toHash())
  }

  // Simple send
  await templates.simpleSend({
    senderKey: privateKey,
    recipientAddress: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    amount: 50000,
    utxo
  })

  // Split UTXO
  await templates.split({
    ownerKey: privateKey,
    utxo,
    numOutputs: 5
  })
}
```

## Address Validation and Conversion

```typescript
import { PrivateKey, PublicKey, Script, P2PKH } from '@bsv/sdk'

/**
 * Address Utilities
 *
 * Validate and convert between different address formats
 */
class AddressUtilities {
  /**
   * Validate BSV address
   */
  validateAddress(address: string): {
    valid: boolean
    network: 'mainnet' | 'testnet' | 'unknown'
    type: 'P2PKH' | 'P2SH' | 'unknown'
    error?: string
  } {
    try {
      // Try to create a script from the address
      Script.fromAddress(address)

      // Check address prefix
      const prefix = address.charAt(0)
      let network: 'mainnet' | 'testnet' | 'unknown' = 'unknown'
      let type: 'P2PKH' | 'P2SH' | 'unknown' = 'unknown'

      if (prefix === '1') {
        network = 'mainnet'
        type = 'P2PKH'
      } else if (prefix === '3') {
        network = 'mainnet'
        type = 'P2SH'
      } else if (prefix === 'm' || prefix === 'n') {
        network = 'testnet'
        type = 'P2PKH'
      } else if (prefix === '2') {
        network = 'testnet'
        type = 'P2SH'
      }

      return {
        valid: true,
        network,
        type
      }
    } catch (error) {
      return {
        valid: false,
        network: 'unknown',
        type: 'unknown',
        error: error.message
      }
    }
  }

  /**
   * Get address from private key
   */
  getAddressFromPrivateKey(privateKey: PrivateKey): string {
    return privateKey.toPublicKey().toAddress()
  }

  /**
   * Get address from public key
   */
  getAddressFromPublicKey(publicKey: PublicKey): string {
    return publicKey.toAddress()
  }

  /**
   * Get script from address
   */
  getScriptFromAddress(address: string): Script {
    return Script.fromAddress(address)
  }

  /**
   * Compare two addresses
   */
  compareAddresses(address1: string, address2: string): boolean {
    try {
      const script1 = Script.fromAddress(address1)
      const script2 = Script.fromAddress(address2)
      return script1.toHex() === script2.toHex()
    } catch {
      return false
    }
  }
}

/**
 * Usage Example
 */
function addressUtilitiesExample() {
  const utils = new AddressUtilities()

  // Validate addresses
  const validation1 = utils.validateAddress('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa')
  console.log('Validation:', validation1)

  // Generate address
  const privateKey = PrivateKey.fromRandom()
  const address = utils.getAddressFromPrivateKey(privateKey)
  console.log('Generated address:', address)

  // Compare addresses
  const same = utils.compareAddresses(address, address)
  console.log('Addresses match:', same)
}
```

## Related Examples

- [P2PKH Template](../p2pkh-template/README.md)
- [Transaction Building](../transaction-building/README.md)
- [UTXO Management](../utxo-management/README.md)

## See Also

**SDK Components:**
- [P2PKH](../../sdk-components/p2pkh/README.md) - P2PKH script template
- [Transaction](../../sdk-components/transaction/README.md) - Transaction building
- [Script](../../sdk-components/script/README.md) - Script operations
- [Script Templates](../../sdk-components/script-templates/README.md) - Template system

**Learning Paths:**
- [First Transaction](../../learning-paths/beginner/first-transaction/README.md)
- [Transaction Types](../../learning-paths/intermediate/transaction-types/README.md)
