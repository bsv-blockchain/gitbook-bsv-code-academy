# Custom Script Templates

Complete examples for creating custom script templates for reusable locking and unlocking patterns.

## Overview

The BSV SDK provides a template system that allows you to define custom locking and unlocking script patterns. Templates make it easy to create reusable script patterns without manually constructing Bitcoin Script each time. This is essential for applications that use custom scripts like multisig, time locks, or complex spending conditions.

**Related SDK Components:**
- [Script Templates](../../sdk-components/script-templates/README.md)
- [Script](../../sdk-components/script/README.md)
- [Transaction](../../sdk-components/transaction/README.md)
- [P2PKH](../../sdk-components/p2pkh/README.md)

## Basic Template Creation

```typescript
import {
  Transaction,
  PrivateKey,
  PublicKey,
  Script,
  OP,
  LockingScript,
  UnlockingScript
} from '@bsv/sdk'

/**
 * Basic Custom Template
 *
 * Simple template for a password-locked output
 */
class PasswordLockTemplate {
  private password: string

  constructor(password: string) {
    this.password = password
  }

  /**
   * Create locking script
   *
   * Requires: <password> <signature> <pubkey>
   */
  lock(publicKeyHash: number[]): LockingScript {
    return {
      script: (): Script => {
        const script = new Script()

        // Password verification
        const passwordHash = Buffer.from(this.hashPassword(this.password), 'hex')
        script.writeOpCode(OP.OP_HASH256)
        script.writeBin(passwordHash)
        script.writeOpCode(OP.OP_EQUALVERIFY)

        // Standard P2PKH verification
        script.writeOpCode(OP.OP_DUP)
        script.writeOpCode(OP.OP_HASH160)
        script.writeBin(Buffer.from(publicKeyHash))
        script.writeOpCode(OP.OP_EQUALVERIFY)
        script.writeOpCode(OP.OP_CHECKSIG)

        return script
      }
    }
  }

  /**
   * Create unlocking script
   *
   * Provides: <password> <signature> <pubkey>
   */
  unlock(privateKey: PrivateKey, password: string): UnlockingScript {
    return {
      sign: async (tx: Transaction, inputIndex: number) => {
        const script = new Script()

        // Get locking script for this input
        const input = tx.inputs[inputIndex]
        const lockingScript = input.sourceTransaction?.outputs[input.sourceOutputIndex]?.lockingScript

        if (!lockingScript) {
          throw new Error('Cannot find locking script for input')
        }

        // Create signature
        const preimage = input.getPreimage(lockingScript)
        const signature = privateKey.sign(preimage)

        // Build unlocking script: <sig> <pubkey> <password>
        script.writeBin(signature.toDER())
        script.writeBin(privateKey.toPublicKey().encode())
        script.writeBin(Buffer.from(password, 'utf8'))

        return script
      },
      estimateLength: async () => {
        return 150 // Approximate length
      }
    }
  }

  /**
   * Hash password for script
   */
  private hashPassword(password: string): string {
    const hash = require('crypto')
      .createHash('sha256')
      .update(password)
      .digest()

    return require('crypto')
      .createHash('sha256')
      .update(hash)
      .digest('hex')
  }
}

/**
 * Usage Example
 */
async function basicTemplateExample() {
  const template = new PasswordLockTemplate('my-secret-password')
  const privateKey = PrivateKey.fromRandom()

  // Create a transaction with password-locked output
  const tx = new Transaction()

  tx.addInput({
    sourceTXID: 'funding-tx...',
    sourceOutputIndex: 0,
    unlockingScriptTemplate: template.unlock(privateKey, 'my-secret-password'),
    sequence: 0xffffffff
  })

  tx.addOutput({
    satoshis: 50000,
    lockingScript: template.lock(privateKey.toPublicKey().toHash()).script()
  })

  await tx.sign()

  console.log('Password-locked transaction created')
  console.log('Transaction ID:', tx.id('hex'))
}
```

## Timelock Template

```typescript
import {
  Transaction,
  PrivateKey,
  Script,
  OP,
  LockingScript,
  UnlockingScript
} from '@bsv/sdk'

/**
 * Timelock Template
 *
 * Funds locked until a specific time or block height
 */
class TimelockTemplate {
  private lockTime: number

  constructor(lockTime: number) {
    this.lockTime = lockTime
  }

  /**
   * Create time-locked locking script
   */
  lock(publicKeyHash: number[]): LockingScript {
    return {
      script: (): Script => {
        const script = new Script()

        // Time lock verification
        script.writeBin(this.numberToBuffer(this.lockTime))
        script.writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
        script.writeOpCode(OP.OP_DROP)

        // Standard P2PKH
        script.writeOpCode(OP.OP_DUP)
        script.writeOpCode(OP.OP_HASH160)
        script.writeBin(Buffer.from(publicKeyHash))
        script.writeOpCode(OP.OP_EQUALVERIFY)
        script.writeOpCode(OP.OP_CHECKSIG)

        return script
      }
    }
  }

  /**
   * Create unlocking script for time-locked output
   */
  unlock(privateKey: PrivateKey): UnlockingScript {
    return {
      sign: async (tx: Transaction, inputIndex: number) => {
        const script = new Script()
        const input = tx.inputs[inputIndex]

        // Set transaction locktime
        tx.lockTime = this.lockTime

        // Set sequence to enable locktime
        input.sequence = 0xfffffffe

        const lockingScript = input.sourceTransaction?.outputs[input.sourceOutputIndex]?.lockingScript

        if (!lockingScript) {
          throw new Error('Cannot find locking script')
        }

        // Create signature
        const preimage = input.getPreimage(lockingScript)
        const signature = privateKey.sign(preimage)

        // Build unlocking script: <sig> <pubkey>
        script.writeBin(signature.toDER())
        script.writeBin(privateKey.toPublicKey().encode())

        return script
      },
      estimateLength: async () => {
        return 140
      }
    }
  }

  /**
   * Convert number to script buffer
   */
  private numberToBuffer(num: number): Buffer {
    if (num === 0) return Buffer.from([])

    const isNegative = num < 0
    const absNum = Math.abs(num)
    const bytes: number[] = []

    let n = absNum
    while (n > 0) {
      bytes.push(n & 0xff)
      n >>= 8
    }

    if (bytes[bytes.length - 1] & 0x80) {
      bytes.push(isNegative ? 0x80 : 0x00)
    } else if (isNegative) {
      bytes[bytes.length - 1] |= 0x80
    }

    return Buffer.from(bytes)
  }
}

/**
 * Usage Example
 */
async function timelockTemplateExample() {
  const privateKey = PrivateKey.fromRandom()

  // Lock until 1 hour from now
  const unlockTime = Math.floor(Date.now() / 1000) + 3600
  const template = new TimelockTemplate(unlockTime)

  const fundingTx = new Transaction()

  // Create funding transaction with time-locked output
  fundingTx.addOutput({
    satoshis: 100000,
    lockingScript: template.lock(privateKey.toPublicKey().toHash()).script()
  })

  console.log('Time-locked output created')
  console.log('Unlock time:', new Date(unlockTime * 1000).toISOString())

  // Later, after the lock time...
  const spendTx = new Transaction()

  spendTx.addInput({
    sourceTXID: fundingTx.id('hex'),
    sourceOutputIndex: 0,
    sourceTransaction: fundingTx,
    unlockingScriptTemplate: template.unlock(privateKey),
    sequence: 0xfffffffe
  })

  spendTx.addOutput({
    satoshis: 99500,
    lockingScript: Script.fromAddress('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa')
  })

  await spendTx.sign()

  console.log('Time-locked funds spent')
  console.log('Spend transaction:', spendTx.id('hex'))
}
```

## Multisig Template

```typescript
import {
  Transaction,
  PrivateKey,
  PublicKey,
  Script,
  OP,
  LockingScript,
  UnlockingScript,
  Signature
} from '@bsv/sdk'

/**
 * M-of-N Multisig Template
 *
 * Requires M signatures from N public keys
 */
class MultisigTemplate {
  private requiredSigs: number
  private publicKeys: PublicKey[]

  constructor(requiredSigs: number, publicKeys: PublicKey[]) {
    if (requiredSigs < 1 || requiredSigs > publicKeys.length) {
      throw new Error('Invalid required signatures count')
    }

    if (publicKeys.length > 16) {
      throw new Error('Maximum 16 public keys allowed')
    }

    this.requiredSigs = requiredSigs
    this.publicKeys = publicKeys
  }

  /**
   * Create multisig locking script
   */
  lock(): LockingScript {
    return {
      script: (): Script => {
        const script = new Script()

        // Format: <M> <pubkey1> <pubkey2> ... <pubkeyN> <N> OP_CHECKMULTISIG

        // Push M (required signatures)
        script.writeOpCode(this.numToOpCode(this.requiredSigs))

        // Push all public keys
        for (const pubKey of this.publicKeys) {
          script.writeBin(pubKey.encode())
        }

        // Push N (total keys)
        script.writeOpCode(this.numToOpCode(this.publicKeys.length))

        // Add OP_CHECKMULTISIG
        script.writeOpCode(OP.OP_CHECKMULTISIG)

        return script
      }
    }
  }

  /**
   * Create unlocking script with multiple signatures
   */
  unlock(privateKeys: PrivateKey[]): UnlockingScript {
    if (privateKeys.length < this.requiredSigs) {
      throw new Error(`Need at least ${this.requiredSigs} private keys`)
    }

    return {
      sign: async (tx: Transaction, inputIndex: number) => {
        const script = new Script()
        const input = tx.inputs[inputIndex]

        const lockingScript = input.sourceTransaction?.outputs[input.sourceOutputIndex]?.lockingScript

        if (!lockingScript) {
          throw new Error('Cannot find locking script')
        }

        // OP_CHECKMULTISIG bug requires extra OP_0
        script.writeOpCode(OP.OP_0)

        // Create signatures
        const signatures: Buffer[] = []
        for (let i = 0; i < this.requiredSigs && i < privateKeys.length; i++) {
          const preimage = input.getPreimage(lockingScript)
          const signature = privateKeys[i].sign(preimage)
          signatures.push(signature.toDER())
        }

        // Push signatures
        for (const sig of signatures) {
          script.writeBin(sig)
        }

        return script
      },
      estimateLength: async () => {
        return 10 + this.requiredSigs * 75 // Approximate
      }
    }
  }

  /**
   * Convert number to OP_N opcode
   */
  private numToOpCode(n: number): number {
    if (n === 0) return OP.OP_0
    if (n >= 1 && n <= 16) return OP.OP_1 + (n - 1)
    throw new Error('Number must be between 0 and 16')
  }
}

/**
 * Usage Example
 */
async function multisigTemplateExample() {
  // Create 2-of-3 multisig
  const key1 = PrivateKey.fromRandom()
  const key2 = PrivateKey.fromRandom()
  const key3 = PrivateKey.fromRandom()

  const template = new MultisigTemplate(2, [
    key1.toPublicKey(),
    key2.toPublicKey(),
    key3.toPublicKey()
  ])

  // Create funding transaction
  const fundingTx = new Transaction()

  fundingTx.addOutput({
    satoshis: 100000,
    lockingScript: template.lock().script()
  })

  console.log('2-of-3 multisig output created')

  // Spend using 2 of 3 keys
  const spendTx = new Transaction()

  spendTx.addInput({
    sourceTXID: fundingTx.id('hex'),
    sourceOutputIndex: 0,
    sourceTransaction: fundingTx,
    unlockingScriptTemplate: template.unlock([key1, key2]), // Use first 2 keys
    sequence: 0xffffffff
  })

  spendTx.addOutput({
    satoshis: 99500,
    lockingScript: Script.fromAddress('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa')
  })

  await spendTx.sign()

  console.log('Multisig spent with 2 signatures')
  console.log('Transaction:', spendTx.id('hex'))
}
```

## Conditional Template

```typescript
import {
  Transaction,
  PrivateKey,
  PublicKey,
  Script,
  OP,
  LockingScript,
  UnlockingScript
} from '@bsv/sdk'

/**
 * Conditional Template
 *
 * Different spending conditions based on a flag
 */
class ConditionalTemplate {
  private recipientPubKey: PublicKey
  private refundPubKey: PublicKey
  private timelock: number

  constructor(
    recipientPubKey: PublicKey,
    refundPubKey: PublicKey,
    timelock: number
  ) {
    this.recipientPubKey = recipientPubKey
    this.refundPubKey = refundPubKey
    this.timelock = timelock
  }

  /**
   * Create conditional locking script
   *
   * IF
   *   <recipientPubKey> OP_CHECKSIG
   * ELSE
   *   <timelock> OP_CHECKLOCKTIMEVERIFY OP_DROP
   *   <refundPubKey> OP_CHECKSIG
   * ENDIF
   */
  lock(): LockingScript {
    return {
      script: (): Script => {
        const script = new Script()

        script.writeOpCode(OP.OP_IF)

        // Recipient path
        script.writeBin(this.recipientPubKey.encode())
        script.writeOpCode(OP.OP_CHECKSIG)

        script.writeOpCode(OP.OP_ELSE)

        // Refund path (after timelock)
        script.writeBin(this.numberToBuffer(this.timelock))
        script.writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
        script.writeOpCode(OP.OP_DROP)
        script.writeBin(this.refundPubKey.encode())
        script.writeOpCode(OP.OP_CHECKSIG)

        script.writeOpCode(OP.OP_ENDIF)

        return script
      }
    }
  }

  /**
   * Unlock via recipient path
   */
  unlockRecipient(recipientKey: PrivateKey): UnlockingScript {
    return {
      sign: async (tx: Transaction, inputIndex: number) => {
        const script = new Script()
        const input = tx.inputs[inputIndex]

        const lockingScript = input.sourceTransaction?.outputs[input.sourceOutputIndex]?.lockingScript

        if (!lockingScript) {
          throw new Error('Cannot find locking script')
        }

        // Create signature
        const preimage = input.getPreimage(lockingScript)
        const signature = recipientKey.sign(preimage)

        // Unlocking script: <sig> OP_TRUE
        script.writeBin(signature.toDER())
        script.writeOpCode(OP.OP_TRUE)

        return script
      },
      estimateLength: async () => 80
    }
  }

  /**
   * Unlock via refund path (after timelock)
   */
  unlockRefund(refundKey: PrivateKey): UnlockingScript {
    return {
      sign: async (tx: Transaction, inputIndex: number) => {
        const script = new Script()
        const input = tx.inputs[inputIndex]

        // Set transaction locktime
        tx.lockTime = this.timelock
        input.sequence = 0xfffffffe

        const lockingScript = input.sourceTransaction?.outputs[input.sourceOutputIndex]?.lockingScript

        if (!lockingScript) {
          throw new Error('Cannot find locking script')
        }

        // Create signature
        const preimage = input.getPreimage(lockingScript)
        const signature = refundKey.sign(preimage)

        // Unlocking script: <sig> OP_FALSE
        script.writeBin(signature.toDER())
        script.writeOpCode(OP.OP_FALSE)

        return script
      },
      estimateLength: async () => 80
    }
  }

  /**
   * Convert number to buffer
   */
  private numberToBuffer(num: number): Buffer {
    if (num === 0) return Buffer.from([])

    const bytes: number[] = []
    let n = num

    while (n > 0) {
      bytes.push(n & 0xff)
      n >>= 8
    }

    return Buffer.from(bytes)
  }
}

/**
 * Usage Example
 */
async function conditionalTemplateExample() {
  const recipient = PrivateKey.fromRandom()
  const refunder = PrivateKey.fromRandom()

  // Lock for 1 hour
  const unlockTime = Math.floor(Date.now() / 1000) + 3600

  const template = new ConditionalTemplate(
    recipient.toPublicKey(),
    refunder.toPublicKey(),
    unlockTime
  )

  // Create funding transaction
  const fundingTx = new Transaction()

  fundingTx.addOutput({
    satoshis: 100000,
    lockingScript: template.lock().script()
  })

  console.log('Conditional output created')

  // Recipient can spend immediately
  const recipientSpend = new Transaction()

  recipientSpend.addInput({
    sourceTXID: fundingTx.id('hex'),
    sourceOutputIndex: 0,
    sourceTransaction: fundingTx,
    unlockingScriptTemplate: template.unlockRecipient(recipient),
    sequence: 0xffffffff
  })

  recipientSpend.addOutput({
    satoshis: 99500,
    lockingScript: Script.fromAddress('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa')
  })

  await recipientSpend.sign()

  console.log('Recipient spent funds')

  // Or refunder can claim after timelock
  const refundSpend = new Transaction()

  refundSpend.addInput({
    sourceTXID: fundingTx.id('hex'),
    sourceOutputIndex: 0,
    sourceTransaction: fundingTx,
    unlockingScriptTemplate: template.unlockRefund(refunder),
    sequence: 0xfffffffe
  })

  refundSpend.addOutput({
    satoshis: 99500,
    lockingScript: Script.fromAddress('1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2')
  })

  await refundSpend.sign()

  console.log('Refund claimed after timeout')
}
```

## Related Examples

- [Custom Scripts](../custom-scripts/README.md)
- [P2PKH Template](../p2pkh-template/README.md)
- [Multi-Signature](../multi-signature/README.md)
- [Smart Contracts](../smart-contracts/README.md)

## See Also

**SDK Components:**
- [Script Templates](../../sdk-components/script-templates/README.md) - Template system documentation
- [Script](../../sdk-components/script/README.md) - Script operations
- [Transaction](../../sdk-components/transaction/README.md) - Transaction building
- [P2PKH](../../sdk-components/p2pkh/README.md) - P2PKH template reference

**Learning Paths:**
- [Script Templates](../../learning-paths/intermediate/script-templates/README.md)
- [Advanced Scripting](../../learning-paths/advanced/custom-scripts/README.md)
- [Template Patterns](../../learning-paths/advanced/template-patterns/README.md)
