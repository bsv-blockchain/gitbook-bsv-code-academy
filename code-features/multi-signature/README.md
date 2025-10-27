# Multi-Signature Transactions

Complete examples for creating and spending multi-signature (multisig) transactions in various configurations.

## Overview

This code feature demonstrates multi-signature transactions, which require multiple signatures to spend funds. Common use cases include corporate wallets, escrow services, and enhanced security setups. This guide covers 2-of-2, 2-of-3, 3-of-5, and custom multisig configurations.

**Related SDK Components:**
- [Script](../../sdk-components/script/README.md)
- [Transaction](../../sdk-components/transaction/README.md)
- [Signatures](../../sdk-components/signatures/README.md)
- [Script Templates](../../sdk-components/script-templates/README.md)

## 2-of-2 Multisig

```typescript
import { Transaction, PrivateKey, PublicKey, Script, OP } from '@bsv/sdk'

/**
 * 2-of-2 Multisig
 *
 * Both parties must sign to spend the funds
 */
class TwoOfTwoMultisig {
  /**
   * Create 2-of-2 multisig locking script
   */
  createLockingScript(pubKey1: PublicKey, pubKey2: PublicKey): Script {
    const script = new Script()

    script.writeOpCode(OP.OP_2)  // Require 2 signatures
    script.writeBin(pubKey1.encode())
    script.writeBin(pubKey2.encode())
    script.writeOpCode(OP.OP_2)  // Out of 2 keys
    script.writeOpCode(OP.OP_CHECKMULTISIG)

    return script
  }

  /**
   * Create transaction with 2-of-2 multisig output
   */
  async createMultisigOutput(
    senderKey: PrivateKey,
    recipientKey1: PublicKey,
    recipientKey2: PublicKey,
    amount: number,
    utxo: { txid: string; vout: number; satoshis: number; script: Script }
  ): Promise<Transaction> {
    const tx = new Transaction()

    // Add input
    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(senderKey),
      sequence: 0xffffffff
    })

    // Add 2-of-2 multisig output
    tx.addOutput({
      satoshis: amount,
      lockingScript: this.createLockingScript(recipientKey1, recipientKey2)
    })

    // Add change output
    const fee = 500
    const change = utxo.satoshis - amount - fee

    if (change > 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * Spend from 2-of-2 multisig (both signatures required)
   */
  async spendMultisigOutput(
    key1: PrivateKey,
    key2: PrivateKey,
    multisigUtxo: { txid: string; vout: number; satoshis: number; script: Script },
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    const tx = new Transaction()

    // Add multisig input
    tx.addInput({
      sourceTXID: multisigUtxo.txid,
      sourceOutputIndex: multisigUtxo.vout,
      unlockingScript: new Script(), // Will be filled after signing
      sequence: 0xffffffff
    })

    // Add output
    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    // Create signatures
    const preimage = tx.inputs[0].getPreimage(multisigUtxo.script)
    const sig1 = key1.sign(preimage)
    const sig2 = key2.sign(preimage)

    // Build unlocking script
    const unlockingScript = new Script()
    unlockingScript.writeOpCode(OP.OP_0)  // Bug workaround for CHECKMULTISIG
    unlockingScript.writeBin(sig1.toDER())
    unlockingScript.writeBin(sig2.toDER())

    tx.inputs[0].unlockingScript = unlockingScript

    return tx
  }
}

/**
 * Usage Example
 */
async function twoOfTwoExample() {
  const multisig = new TwoOfTwoMultisig()

  // Generate keys
  const sender = PrivateKey.fromRandom()
  const key1 = PrivateKey.fromRandom()
  const key2 = PrivateKey.fromRandom()

  // Create multisig output
  const utxo = {
    txid: 'abc123...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(sender.toPublicKey().toHash())
  }

  const createTx = await multisig.createMultisigOutput(
    sender,
    key1.toPublicKey(),
    key2.toPublicKey(),
    50000,
    utxo
  )

  console.log('Created multisig output:', createTx.id('hex'))

  // Spend from multisig (requires both keys)
  const multisigUtxo = {
    txid: createTx.id('hex'),
    vout: 0,
    satoshis: 50000,
    script: multisig.createLockingScript(
      key1.toPublicKey(),
      key2.toPublicKey()
    )
  }

  const spendTx = await multisig.spendMultisigOutput(
    key1,
    key2,
    multisigUtxo,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    49000
  )

  console.log('Spent from multisig:', spendTx.id('hex'))
}
```

## 2-of-3 Multisig

```typescript
import { Transaction, PrivateKey, PublicKey, Script, OP } from '@bsv/sdk'

/**
 * 2-of-3 Multisig
 *
 * Any 2 of 3 parties can sign to spend the funds
 */
class TwoOfThreeMultisig {
  /**
   * Create 2-of-3 multisig locking script
   */
  createLockingScript(
    pubKey1: PublicKey,
    pubKey2: PublicKey,
    pubKey3: PublicKey
  ): Script {
    const script = new Script()

    script.writeOpCode(OP.OP_2)  // Require 2 signatures
    script.writeBin(pubKey1.encode())
    script.writeBin(pubKey2.encode())
    script.writeBin(pubKey3.encode())
    script.writeOpCode(OP.OP_3)  // Out of 3 keys
    script.writeOpCode(OP.OP_CHECKMULTISIG)

    return script
  }

  /**
   * Create 2-of-3 multisig address
   */
  createMultisigAddress(
    pubKey1: PublicKey,
    pubKey2: PublicKey,
    pubKey3: PublicKey
  ): string {
    const lockingScript = this.createLockingScript(pubKey1, pubKey2, pubKey3)
    return lockingScript.toAddress()
  }

  /**
   * Create transaction with 2-of-3 multisig output
   */
  async lockToMultisig(
    senderKey: PrivateKey,
    pubKey1: PublicKey,
    pubKey2: PublicKey,
    pubKey3: PublicKey,
    amount: number,
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(senderKey),
      sequence: 0xffffffff
    })

    // Add 2-of-3 multisig output
    tx.addOutput({
      satoshis: amount,
      lockingScript: this.createLockingScript(pubKey1, pubKey2, pubKey3)
    })

    // Add change
    const fee = 500
    const change = utxo.satoshis - amount - fee

    if (change > 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * Spend from 2-of-3 multisig (using keys 1 and 2)
   */
  async spendWithKeys1And2(
    key1: PrivateKey,
    key2: PrivateKey,
    pubKey3: PublicKey,
    multisigUtxo: { txid: string; vout: number; satoshis: number },
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    const lockingScript = this.createLockingScript(
      key1.toPublicKey(),
      key2.toPublicKey(),
      pubKey3
    )

    return this.spendMultisig(
      [key1, key2],
      lockingScript,
      multisigUtxo,
      toAddress,
      amount
    )
  }

  /**
   * Spend from 2-of-3 multisig (using keys 1 and 3)
   */
  async spendWithKeys1And3(
    key1: PrivateKey,
    pubKey2: PublicKey,
    key3: PrivateKey,
    multisigUtxo: { txid: string; vout: number; satoshis: number },
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    const lockingScript = this.createLockingScript(
      key1.toPublicKey(),
      pubKey2,
      key3.toPublicKey()
    )

    return this.spendMultisig(
      [key1, key3],
      lockingScript,
      multisigUtxo,
      toAddress,
      amount
    )
  }

  /**
   * Spend from 2-of-3 multisig (using keys 2 and 3)
   */
  async spendWithKeys2And3(
    pubKey1: PublicKey,
    key2: PrivateKey,
    key3: PrivateKey,
    multisigUtxo: { txid: string; vout: number; satoshis: number },
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    const lockingScript = this.createLockingScript(
      pubKey1,
      key2.toPublicKey(),
      key3.toPublicKey()
    )

    return this.spendMultisig(
      [key2, key3],
      lockingScript,
      multisigUtxo,
      toAddress,
      amount
    )
  }

  /**
   * Generic spend function
   */
  private async spendMultisig(
    signingKeys: PrivateKey[],
    lockingScript: Script,
    multisigUtxo: { txid: string; vout: number; satoshis: number },
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: multisigUtxo.txid,
      sourceOutputIndex: multisigUtxo.vout,
      unlockingScript: new Script(),
      sequence: 0xffffffff
    })

    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    // Create signatures
    const preimage = tx.inputs[0].getPreimage(lockingScript)
    const signatures = signingKeys.map(key => key.sign(preimage))

    // Build unlocking script
    const unlockingScript = new Script()
    unlockingScript.writeOpCode(OP.OP_0)  // Bug workaround

    for (const sig of signatures) {
      unlockingScript.writeBin(sig.toDER())
    }

    tx.inputs[0].unlockingScript = unlockingScript

    return tx
  }
}

/**
 * Usage Example
 */
async function twoOfThreeExample() {
  const multisig = new TwoOfThreeMultisig()

  // Three parties
  const key1 = PrivateKey.fromWif('key1-wif')
  const key2 = PrivateKey.fromWif('key2-wif')
  const key3 = PrivateKey.fromWif('key3-wif')

  // Create multisig address
  const multisigAddress = multisig.createMultisigAddress(
    key1.toPublicKey(),
    key2.toPublicKey(),
    key3.toPublicKey()
  )

  console.log('2-of-3 Multisig Address:', multisigAddress)

  // Later, spend with any 2 keys (e.g., keys 1 and 3)
  const multisigUtxo = {
    txid: 'abc123...',
    vout: 0,
    satoshis: 100000
  }

  const spendTx = await multisig.spendWithKeys1And3(
    key1,
    key2.toPublicKey(),
    key3,
    multisigUtxo,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    99000
  )

  console.log('Spent with keys 1 and 3:', spendTx.id('hex'))
}
```

## 3-of-5 Multisig

```typescript
import { Transaction, PrivateKey, PublicKey, Script, OP } from '@bsv/sdk'

/**
 * 3-of-5 Multisig
 *
 * Any 3 of 5 parties can sign to spend the funds
 */
class ThreeOfFiveMultisig {
  /**
   * Create 3-of-5 multisig locking script
   */
  createLockingScript(publicKeys: PublicKey[]): Script {
    if (publicKeys.length !== 5) {
      throw new Error('Exactly 5 public keys required')
    }

    const script = new Script()

    script.writeOpCode(OP.OP_3)  // Require 3 signatures

    for (const pubKey of publicKeys) {
      script.writeBin(pubKey.encode())
    }

    script.writeOpCode(OP.OP_5)  // Out of 5 keys
    script.writeOpCode(OP.OP_CHECKMULTISIG)

    return script
  }

  /**
   * Create transaction with 3-of-5 multisig output
   */
  async lockToMultisig(
    senderKey: PrivateKey,
    publicKeys: PublicKey[],
    amount: number,
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(senderKey),
      sequence: 0xffffffff
    })

    tx.addOutput({
      satoshis: amount,
      lockingScript: this.createLockingScript(publicKeys)
    })

    const fee = 500
    const change = utxo.satoshis - amount - fee

    if (change > 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * Spend from 3-of-5 multisig
   */
  async spendMultisig(
    allPublicKeys: PublicKey[],
    signingKeys: PrivateKey[],
    multisigUtxo: { txid: string; vout: number; satoshis: number },
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    if (allPublicKeys.length !== 5) {
      throw new Error('Exactly 5 public keys required')
    }

    if (signingKeys.length !== 3) {
      throw new Error('Exactly 3 signing keys required for 3-of-5')
    }

    const tx = new Transaction()
    const lockingScript = this.createLockingScript(allPublicKeys)

    tx.addInput({
      sourceTXID: multisigUtxo.txid,
      sourceOutputIndex: multisigUtxo.vout,
      unlockingScript: new Script(),
      sequence: 0xffffffff
    })

    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    // Create signatures
    const preimage = tx.inputs[0].getPreimage(lockingScript)
    const signatures = signingKeys.map(key => key.sign(preimage))

    // Build unlocking script
    const unlockingScript = new Script()
    unlockingScript.writeOpCode(OP.OP_0)  // Bug workaround

    for (const sig of signatures) {
      unlockingScript.writeBin(sig.toDER())
    }

    tx.inputs[0].unlockingScript = unlockingScript

    return tx
  }

  /**
   * Validate signing keys match public keys
   */
  validateSigningKeys(
    allPublicKeys: PublicKey[],
    signingKeys: PrivateKey[]
  ): boolean {
    const signingPubKeys = signingKeys.map(key =>
      key.toPublicKey().toString()
    )

    const allPubKeyStrings = allPublicKeys.map(key => key.toString())

    // Check each signing key has corresponding public key
    return signingPubKeys.every(pubKey =>
      allPubKeyStrings.includes(pubKey)
    )
  }
}

/**
 * Usage Example
 */
async function threeOfFiveExample() {
  const multisig = new ThreeOfFiveMultisig()

  // Five parties
  const keys = [
    PrivateKey.fromRandom(),
    PrivateKey.fromRandom(),
    PrivateKey.fromRandom(),
    PrivateKey.fromRandom(),
    PrivateKey.fromRandom()
  ]

  const publicKeys = keys.map(key => key.toPublicKey())

  // Create multisig output
  const sender = PrivateKey.fromRandom()
  const utxo = {
    txid: 'abc123...',
    vout: 0,
    satoshis: 100000
  }

  const lockTx = await multisig.lockToMultisig(
    sender,
    publicKeys,
    50000,
    utxo
  )

  console.log('Created 3-of-5 multisig:', lockTx.id('hex'))

  // Spend using any 3 keys (e.g., keys 0, 2, 4)
  const signingKeys = [keys[0], keys[2], keys[4]]

  const multisigUtxo = {
    txid: lockTx.id('hex'),
    vout: 0,
    satoshis: 50000
  }

  const spendTx = await multisig.spendMultisig(
    publicKeys,
    signingKeys,
    multisigUtxo,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    49000
  )

  console.log('Spent with 3 of 5 keys:', spendTx.id('hex'))
}
```

## Custom M-of-N Multisig

```typescript
import { Transaction, PrivateKey, PublicKey, Script, OP } from '@bsv/sdk'

/**
 * Custom M-of-N Multisig
 *
 * Flexible multisig implementation for any M-of-N configuration
 */
class CustomMultisig {
  /**
   * Create M-of-N multisig locking script
   */
  createLockingScript(
    requiredSignatures: number,
    publicKeys: PublicKey[]
  ): Script {
    const totalKeys = publicKeys.length

    // Validate parameters
    if (requiredSignatures < 1 || requiredSignatures > 16) {
      throw new Error('Required signatures must be between 1 and 16')
    }

    if (totalKeys < requiredSignatures || totalKeys > 16) {
      throw new Error('Total keys must be between required signatures and 16')
    }

    const script = new Script()

    // Push required signature count (OP_1 to OP_16)
    script.writeOpCode(OP.OP_1 - 1 + requiredSignatures)

    // Push all public keys
    for (const pubKey of publicKeys) {
      script.writeBin(pubKey.encode())
    }

    // Push total key count (OP_1 to OP_16)
    script.writeOpCode(OP.OP_1 - 1 + totalKeys)

    // Add CHECKMULTISIG
    script.writeOpCode(OP.OP_CHECKMULTISIG)

    return script
  }

  /**
   * Create multisig address
   */
  createMultisigAddress(
    requiredSignatures: number,
    publicKeys: PublicKey[]
  ): string {
    const lockingScript = this.createLockingScript(requiredSignatures, publicKeys)
    return lockingScript.toAddress()
  }

  /**
   * Lock funds to M-of-N multisig
   */
  async lockToMultisig(
    senderKey: PrivateKey,
    requiredSignatures: number,
    publicKeys: PublicKey[],
    amount: number,
    utxo: { txid: string; vout: number; satoshis: number }
  ): Promise<Transaction> {
    const tx = new Transaction()

    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(senderKey),
      sequence: 0xffffffff
    })

    tx.addOutput({
      satoshis: amount,
      lockingScript: this.createLockingScript(requiredSignatures, publicKeys)
    })

    const fee = 500
    const change = utxo.satoshis - amount - fee

    if (change > 546) {
      tx.addOutput({
        satoshis: change,
        lockingScript: new P2PKH().lock(senderKey.toPublicKey().toHash())
      })
    }

    await tx.sign()
    return tx
  }

  /**
   * Spend from M-of-N multisig
   */
  async spendMultisig(
    requiredSignatures: number,
    allPublicKeys: PublicKey[],
    signingKeys: PrivateKey[],
    multisigUtxo: { txid: string; vout: number; satoshis: number },
    toAddress: string,
    amount: number
  ): Promise<Transaction> {
    // Validate signing keys
    if (signingKeys.length !== requiredSignatures) {
      throw new Error(
        `Exactly ${requiredSignatures} signing keys required`
      )
    }

    const tx = new Transaction()
    const lockingScript = this.createLockingScript(
      requiredSignatures,
      allPublicKeys
    )

    tx.addInput({
      sourceTXID: multisigUtxo.txid,
      sourceOutputIndex: multisigUtxo.vout,
      unlockingScript: new Script(),
      sequence: 0xffffffff
    })

    tx.addOutput({
      satoshis: amount,
      lockingScript: Script.fromAddress(toAddress)
    })

    // Create signatures
    const preimage = tx.inputs[0].getPreimage(lockingScript)
    const signatures = signingKeys.map(key => key.sign(preimage))

    // Build unlocking script
    const unlockingScript = new Script()
    unlockingScript.writeOpCode(OP.OP_0)  // CHECKMULTISIG bug workaround

    for (const sig of signatures) {
      unlockingScript.writeBin(sig.toDER())
    }

    tx.inputs[0].unlockingScript = unlockingScript

    return tx
  }

  /**
   * Parse multisig script to get configuration
   */
  parseMultisigScript(script: Script): {
    requiredSignatures: number
    totalKeys: number
    publicKeys: PublicKey[]
  } | null {
    try {
      const chunks = script.chunks

      if (chunks.length < 4) return null

      // Last chunk should be OP_CHECKMULTISIG
      if (chunks[chunks.length - 1].opCodeNum !== OP.OP_CHECKMULTISIG) {
        return null
      }

      // Extract required signatures (first chunk)
      const requiredSigs = chunks[0].opCodeNum - (OP.OP_1 - 1)

      // Extract total keys (second to last chunk)
      const totalKeys = chunks[chunks.length - 2].opCodeNum - (OP.OP_1 - 1)

      // Extract public keys (middle chunks)
      const publicKeys: PublicKey[] = []
      for (let i = 1; i < chunks.length - 2; i++) {
        if (chunks[i].buf) {
          publicKeys.push(PublicKey.fromString(chunks[i].buf.toString('hex')))
        }
      }

      return {
        requiredSignatures: requiredSigs,
        totalKeys,
        publicKeys
      }
    } catch (error) {
      return null
    }
  }
}

/**
 * Usage Examples
 */
async function customMultisigExamples() {
  const multisig = new CustomMultisig()

  // Example 1: 4-of-7 multisig
  console.log('--- 4-of-7 Multisig ---')

  const keys7 = Array.from({ length: 7 }, () => PrivateKey.fromRandom())
  const pubKeys7 = keys7.map(k => k.toPublicKey())

  const address4of7 = multisig.createMultisigAddress(4, pubKeys7)
  console.log('4-of-7 Address:', address4of7)

  // Example 2: 1-of-2 multisig (either party can spend)
  console.log('\n--- 1-of-2 Multisig ---')

  const key1 = PrivateKey.fromRandom()
  const key2 = PrivateKey.fromRandom()

  const address1of2 = multisig.createMultisigAddress(
    1,
    [key1.toPublicKey(), key2.toPublicKey()]
  )
  console.log('1-of-2 Address:', address1of2)

  // Example 3: 5-of-9 multisig
  console.log('\n--- 5-of-9 Multisig ---')

  const keys9 = Array.from({ length: 9 }, () => PrivateKey.fromRandom())
  const pubKeys9 = keys9.map(k => k.toPublicKey())

  const address5of9 = multisig.createMultisigAddress(5, pubKeys9)
  console.log('5-of-9 Address:', address5of9)
}
```

## Related Examples

- [Transaction Signing](../transaction-signing/README.md)
- [P2PKH Template](../p2pkh-template/README.md)
- [Custom Scripts](../custom-scripts/README.md)
- [Transaction Building](../transaction-building/README.md)

## See Also

**SDK Components:**
- [Script](../../sdk-components/script/README.md) - Script creation and manipulation
- [Transaction](../../sdk-components/transaction/README.md) - Transaction construction
- [Signatures](../../sdk-components/signatures/README.md) - Digital signatures
- [Script Templates](../../sdk-components/script-templates/README.md) - Template system

**Learning Paths:**
- [Script Templates](../../learning-paths/intermediate/script-templates/README.md)
- [Advanced Scripting](../../learning-paths/advanced/custom-scripts/README.md)
- [Multi-Party Transactions](../../learning-paths/advanced/multi-party-transactions/README.md)
