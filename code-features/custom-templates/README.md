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

## Related Examples

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
