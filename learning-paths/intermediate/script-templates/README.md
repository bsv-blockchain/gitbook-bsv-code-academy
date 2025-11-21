# Script Templates

## Overview

Master the art of creating reusable, secure script templates for BSV blockchain applications. This module teaches you how to build custom locking and unlocking scripts using the Script Template system, enabling you to create sophisticated payment conditions, time locks, multi-signature schemes, and custom protocols.

**Estimated Time:** 3-4 hours
**Difficulty:** Intermediate
**Prerequisites:** Complete [BSV Fundamentals](../../beginner/bsv-fundamentals/README.md) and [Transaction Building](../transaction-building/README.md)

## Learning Objectives

By the end of this module, you will be able to:

- ✅ Understand the Script Template architecture
- ✅ Create custom locking and unlocking script templates
- ✅ Implement P2PKH templates with advanced features
- ✅ Build time-locked transactions with CHECKLOCKTIMEVERIFY
- ✅ Create multi-signature (M-of-N) templates
- ✅ Design hash puzzle and preimage templates
- ✅ Compose complex templates from simpler ones
- ✅ Build protocol-specific templates

## SDK Components Used

This course leverages these standardized SDK modules:

- **[Script Templates](../../../sdk-components/script-templates/README.md)** - Template system architecture
- **[Script](../../../sdk-components/script/README.md)** - Bitcoin Script operations
- **[P2PKH](../../../sdk-components/p2pkh/README.md)** - Standard payment template
- **[Transaction](../../../sdk-components/transaction/README.md)** - Using templates in transactions
- **[Signatures](../../../sdk-components/signatures/README.md)** - Signing with templates
- **[Private Keys](../../../sdk-components/private-keys/README.md)** - Key management for templates

## 1. Understanding Script Templates

### What is a Script Template?

Reference: **[Script Templates Component](../../../sdk-components/script-templates/README.md)**

A script template is a reusable pattern that generates locking scripts (controls how funds are spent) and corresponding unlocking scripts (provides the data to spend funds). Templates separate the "logic" from the "data."

```typescript
import { LockingScript, UnlockingScript } from '@bsv/sdk'

// Template interface
interface ScriptTemplate {
  /**
   * Create a locking script (scriptPubKey)
   * This script defines the spending conditions
   */
  lock(...args: any[]): LockingScript

  /**
   * Create an unlocking script (scriptSig)
   * This script satisfies the spending conditions
   */
  unlock(...args: any[]): UnlockingScript
}
```

### Template Architecture

```
┌─────────────────────────────────────────┐
│         Script Template                 │
├─────────────────────────────────────────┤
│                                         │
│  lock()  ──► Locking Script (Output)   │
│              "How can this be spent?"   │
│                                         │
│  unlock() ──► Unlocking Script (Input) │
│               "Here's the proof"        │
│                                         │
└─────────────────────────────────────────┘
```

### Benefits of Templates

1. **Reusability** - Write once, use everywhere
2. **Type Safety** - Compile-time checking
3. **Abstraction** - Hide complexity
4. **Composability** - Combine templates
5. **Testing** - Unit test templates independently

## 2. P2PKH Template Deep Dive

### Basic P2PKH Template

Reference: **[P2PKH Component](../../../sdk-components/p2pkh/README.md)**

```typescript
import { P2PKH, PrivateKey, Transaction } from '@bsv/sdk'

// Create P2PKH template
const p2pkh = new P2PKH()

// Locking: Pay to this address
const address = '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa'
const lockingScript = p2pkh.lock(address)

// Unlocking: Prove you have the private key
const privateKey = PrivateKey.fromWif('L1aW4aubDFB7...')
const unlockingScriptTemplate = p2pkh.unlock(privateKey)

// Use in transaction
const tx = new Transaction()
tx.addInput({
  sourceTXID: 'abc123...',
  sourceOutputIndex: 0,
  unlockingScriptTemplate
})
```

### P2PKH with SIGHASH Types

Reference: **[P2PKH - SIGHASH Types](../../../sdk-components/p2pkh/README.md#key-features)**

```typescript
import { P2PKH, PrivateKey, TransactionSignature } from '@bsv/sdk'

const privateKey = PrivateKey.fromRandom()
const p2pkh = new P2PKH()

// SIGHASH_ALL (default) - Signs entire transaction
const unlockAll = p2pkh.unlock(privateKey, 'all')

// SIGHASH_NONE - Allows anyone to change outputs
const unlockNone = p2pkh.unlock(privateKey, 'none')

// SIGHASH_SINGLE - Only signs corresponding output
const unlockSingle = p2pkh.unlock(privateKey, 'single')

// SIGHASH_ANYONECANPAY - Allows adding more inputs
const unlockAnyoneCanPay = p2pkh.unlock(
  privateKey,
  'all',
  true // anyoneCanPay flag
)

// Use case: Crowdfunding with ANYONECANPAY
async function addCrowdfundingContribution(
  fundingTx: Transaction,
  privateKey: PrivateKey,
  contributionUTXO: UTXO
): Promise<Transaction> {
  // Add contribution as input
  fundingTx.addInput({
    sourceTXID: contributionUTXO.txid,
    sourceOutputIndex: contributionUTXO.vout,
    unlockingScriptTemplate: new P2PKH().unlock(
      privateKey,
      'all',
      true // anyoneCanPay - others can add inputs too
    )
  })

  return fundingTx
}
```

### Estimated vs Actual Unlocking

```typescript
import { P2PKH, PrivateKey, Transaction } from '@bsv/sdk'

const p2pkh = new P2PKH()
const privateKey = PrivateKey.fromRandom()

// Estimated unlocking (for fee calculation)
const estimated = p2pkh.unlock(privateKey)

// Create transaction with estimated script
const tx = new Transaction()
tx.addInput({
  sourceTXID: 'abc123...',
  sourceOutputIndex: 0,
  unlockingScriptTemplate: estimated
})

// Calculate fees using estimated size
await tx.fee()

// Sign to get actual unlocking script
await tx.sign()

// Now unlocking script is actual, not estimated
```

## 3. Custom Script Templates

### Creating a Custom Template

Reference: **[Script Templates - Custom Templates](../../../sdk-components/script-templates/README.md#common-patterns)**

```typescript
import {
  LockingScript,
  UnlockingScript,
  ScriptTemplate,
  Script,
  PrivateKey,
  Hash
} from '@bsv/sdk'

/**
 * Simple Password Template
 * Locks funds with a password hash
 */
class PasswordTemplate implements ScriptTemplate {
  /**
   * Create locking script
   * OP_HASH256 <passwordHash> OP_EQUAL
   */
  lock(passwordHash: string): LockingScript {
    const script = new Script()

    // OP_HASH256 <hash> OP_EQUAL
    script.writeOpCode(Script.OP_HASH256)
    script.writeBytes(Buffer.from(passwordHash, 'hex'))
    script.writeOpCode(Script.OP_EQUAL)

    return script
  }

  /**
   * Create unlocking script
   * <password>
   */
  unlock(password: string): UnlockingScript {
    const script = new Script()

    // Push password as data
    script.writeBytes(Buffer.from(password, 'utf8'))

    return {
      script,
      estimatedLength: script.toBinary().length,
      sign: async () => {}, // No signing needed
      estimateLength: async () => script.toBinary().length
    }
  }
}

// Usage
const passwordTemplate = new PasswordTemplate()

// Lock with password hash
const password = 'secret123'
const passwordHash = Hash.hash256(Buffer.from(password, 'utf8')).toString('hex')
const lockingScript = passwordTemplate.lock(passwordHash)

// Unlock with password
const unlockingScriptTemplate = passwordTemplate.unlock(password)

// In transaction
const tx = new Transaction()
tx.addInput({
  sourceTXID: 'abc123...',
  sourceOutputIndex: 0,
  unlockingScriptTemplate
})
tx.addOutput({
  lockingScript,
  satoshis: 10000
})
```

### Hash Puzzle Template

```typescript
import { Script, Hash } from '@bsv/sdk'

/**
 * SHA-256 Hash Puzzle Template
 * Anyone who knows the preimage can unlock
 */
class HashPuzzleTemplate implements ScriptTemplate {
  lock(hash: Buffer): LockingScript {
    const script = new Script()

    // OP_HASH256 <hash> OP_EQUALVERIFY
    script.writeOpCode(Script.OP_HASH256)
    script.writeBytes(hash)
    script.writeOpCode(Script.OP_EQUALVERIFY)

    return script
  }

  unlock(preimage: Buffer): UnlockingScript {
    const script = new Script()
    script.writeBytes(preimage)

    return {
      script,
      estimatedLength: preimage.length + 3,
      sign: async () => {},
      estimateLength: async () => preimage.length + 3
    }
  }
}

// Usage - Create a hash puzzle anyone can solve
const preimage = Buffer.from('secret_data')
const hash = Hash.hash256(preimage)

const puzzle = new HashPuzzleTemplate()
const lockingScript = puzzle.lock(hash)

// Later, someone who knows the preimage can unlock
const unlockingScript = puzzle.unlock(preimage)
```

## 4. Time-Locked Templates

### CHECKLOCKTIMEVERIFY Template

Reference: **[Script Component - Time Locks](../../../sdk-components/script/README.md#common-patterns)**

```typescript
import { Script, P2PKH, PrivateKey, PublicKey, Utils } from '@bsv/sdk'

/**
 * Time-locked P2PKH Template
 * Funds locked until specific time/block height
 */
class TimeLockP2PKH implements ScriptTemplate {
  /**
   * Lock until timestamp or block height
   * @param address - Payment address
   * @param locktime - Unix timestamp or block height
   */
  lock(address: string, locktime: number): LockingScript {
    const script = new Script()

    // <locktime> OP_CHECKLOCKTIMEVERIFY OP_DROP
    script.writeBigNum(locktime)
    script.writeOpCode(Script.OP_CHECKLOCKTIMEVERIFY)
    script.writeOpCode(Script.OP_DROP)

    // Then standard P2PKH
    script.writeOpCode(Script.OP_DUP)
    script.writeOpCode(Script.OP_HASH160)

    const decoded = Utils.fromBase58Check(address)
    const pubKeyHash = Buffer.from(decoded.data)
    script.writeBytes(pubKeyHash)

    script.writeOpCode(Script.OP_EQUALVERIFY)
    script.writeOpCode(Script.OP_CHECKSIG)

    return script
  }

  /**
   * Unlock with signature after locktime expires
   */
  unlock(privateKey: PrivateKey, locktime: number): UnlockingScript {
    const publicKey = privateKey.toPublicKey()

    return {
      script: new Script(), // Will be filled during signing
      estimatedLength: 139, // Typical sig + pubkey size

      sign: async (tx: Transaction, inputIndex: number) => {
        const script = new Script()

        // Create signature
        const signature = tx.sign(
          inputIndex,
          privateKey,
          publicKey,
          locktime // Use locktime in signature
        )

        // <signature> <publicKey>
        script.writeBytes(signature.toDER())
        script.writeBytes(publicKey.toDER())

        return script
      },

      estimateLength: async () => 139
    }
  }
}

// Usage - Lock funds for 1 week
const timeLock = new TimeLockP2PKH()
const oneWeekFromNow = Math.floor(Date.now() / 1000) + (7 * 24 * 60 * 60)

const lockingScript = timeLock.lock(recipientAddress, oneWeekFromNow)

// Create time-locked payment
const tx = new Transaction()
tx.addOutput({
  lockingScript,
  satoshis: 100000
})

// Set transaction locktime
tx.lockTime = oneWeekFromNow

// Later, recipient unlocks after time expires
const unlockTx = new Transaction()
unlockTx.lockTime = oneWeekFromNow
unlockTx.addInput({
  sourceTXID: tx.id('hex'),
  sourceOutputIndex: 0,
  unlockingScriptTemplate: timeLock.unlock(privateKey, oneWeekFromNow),
  sequence: 0xfffffffe // Required for CHECKLOCKTIMEVERIFY
})
```

### Relative Time Lock (CHECKSEQUENCEVERIFY)

```typescript
/**
 * Relative time-lock template
 * Funds locked for N blocks after confirmation
 */
class RelativeTimeLockP2PKH implements ScriptTemplate {
  /**
   * @param address - Payment address
   * @param blocks - Number of blocks to wait
   */
  lock(address: string, blocks: number): LockingScript {
    const script = new Script()

    // <blocks> OP_CHECKSEQUENCEVERIFY OP_DROP
    script.writeBigNum(blocks)
    script.writeOpCode(Script.OP_CHECKSEQUENCEVERIFY)
    script.writeOpCode(Script.OP_DROP)

    // Then P2PKH
    script.writeOpCode(Script.OP_DUP)
    script.writeOpCode(Script.OP_HASH160)

    const decoded = Utils.fromBase58Check(address)
    const pubKeyHash = Buffer.from(decoded.data)
    script.writeBytes(pubKeyHash)

    script.writeOpCode(Script.OP_EQUALVERIFY)
    script.writeOpCode(Script.OP_CHECKSIG)

    return script
  }

  unlock(privateKey: PrivateKey, blocks: number): UnlockingScript {
    const publicKey = privateKey.toPublicKey()

    return {
      script: new Script(),
      estimatedLength: 139,

      sign: async (tx: Transaction, inputIndex: number) => {
        const script = new Script()

        const signature = tx.sign(inputIndex, privateKey, publicKey)

        script.writeBytes(signature.toDER())
        script.writeBytes(publicKey.toDER())

        return script
      },

      estimateLength: async () => 139
    }
  }
}

// Usage - Lock for 10 blocks after confirmation
const relTimeLock = new RelativeTimeLockP2PKH()
const lockingScript = relTimeLock.lock(address, 10)

// When spending, set sequence number
unlockTx.addInput({
  sourceTXID: txid,
  sourceOutputIndex: 0,
  unlockingScriptTemplate: relTimeLock.unlock(privateKey, 10),
  sequence: 10 // Must wait 10 blocks
})
```

## 5. Multi-Signature Templates

### M-of-N Multisig Template

Reference: **[Script Templates - Multi-Signature](../../../sdk-components/script-templates/README.md#common-patterns)**

```typescript
import { Script, PublicKey, Signature } from '@bsv/sdk'

/**
 * M-of-N Multi-signature Template
 * Requires M signatures from N public keys
 */
class MultisigTemplate implements ScriptTemplate {
  /**
   * Create M-of-N multisig locking script
   * @param m - Required signatures
   * @param publicKeys - Array of public keys
   */
  lock(m: number, publicKeys: PublicKey[]): LockingScript {
    const n = publicKeys.length

    if (m > n) {
      throw new Error('M cannot be greater than N')
    }

    if (n > 16) {
      throw new Error('Maximum 16 public keys allowed')
    }

    const script = new Script()

    // OP_M
    script.writeOpCode(Script.OP_1 + m - 1)

    // Public keys
    for (const pubKey of publicKeys) {
      script.writeBytes(pubKey.toDER())
    }

    // OP_N OP_CHECKMULTISIG
    script.writeOpCode(Script.OP_1 + n - 1)
    script.writeOpCode(Script.OP_CHECKMULTISIG)

    return script
  }

  /**
   * Create unlocking script with M signatures
   */
  unlock(signatures: Signature[]): UnlockingScript {
    return {
      script: new Script(),
      estimatedLength: signatures.length * 73 + 1,

      sign: async () => {
        const script = new Script()

        // OP_0 (due to CHECKMULTISIG bug)
        script.writeOpCode(Script.OP_0)

        // Signatures
        for (const sig of signatures) {
          script.writeBytes(sig.toDER())
        }

        return script
      },

      estimateLength: async () => signatures.length * 73 + 1
    }
  }
}

// Usage - 2-of-3 multisig
const multisig = new MultisigTemplate()

const pubKey1 = privateKey1.toPublicKey()
const pubKey2 = privateKey2.toPublicKey()
const pubKey3 = privateKey3.toPublicKey()

// Create 2-of-3 multisig output
const lockingScript = multisig.lock(2, [pubKey1, pubKey2, pubKey3])

const tx = new Transaction()
tx.addOutput({
  lockingScript,
  satoshis: 100000
})

// Later, spend with 2 signatures
const sig1 = privateKey1.sign(txHash)
const sig2 = privateKey2.sign(txHash)

const unlockingScript = multisig.unlock([sig1, sig2])
```

### Threshold Signature Template

```typescript
/**
 * 2-of-2 Multisig with named parties
 * Common for escrow or joint accounts
 */
class TwoPartyEscrow implements ScriptTemplate {
  lock(buyerPubKey: PublicKey, sellerPubKey: PublicKey): LockingScript {
    const script = new Script()

    // OP_2
    script.writeOpCode(Script.OP_2)

    // Buyer public key
    script.writeBytes(buyerPubKey.toDER())

    // Seller public key
    script.writeBytes(sellerPubKey.toDER())

    // OP_2 OP_CHECKMULTISIG
    script.writeOpCode(Script.OP_2)
    script.writeOpCode(Script.OP_CHECKMULTISIG)

    return script
  }

  unlock(buyerSig: Signature, sellerSig: Signature): UnlockingScript {
    return {
      script: new Script(),
      estimatedLength: 147, // Two signatures

      sign: async () => {
        const script = new Script()

        script.writeOpCode(Script.OP_0) // CHECKMULTISIG bug
        script.writeBytes(buyerSig.toDER())
        script.writeBytes(sellerSig.toDER())

        return script
      },

      estimateLength: async () => 147
    }
  }
}

// Usage - Escrow between buyer and seller
const escrow = new TwoPartyEscrow()

const buyerKey = buyerPrivateKey.toPublicKey()
const sellerKey = sellerPrivateKey.toPublicKey()

const lockingScript = escrow.lock(buyerKey, sellerKey)

// Both parties must sign to release funds
const buyerSig = buyerPrivateKey.sign(txHash)
const sellerSig = sellerPrivateKey.sign(txHash)

const unlockingScript = escrow.unlock(buyerSig, sellerSig)
```

## 6. Template Composition

### Combining Multiple Templates

```typescript
/**
 * Composite Template
 * Combines time lock + multisig
 */
class TimeLockMultisig implements ScriptTemplate {
  private timeLock: TimeLockP2PKH
  private multisig: MultisigTemplate

  constructor() {
    this.timeLock = new TimeLockP2PKH()
    this.multisig = new MultisigTemplate()
  }

  /**
   * Lock with time lock OR multisig
   * Either wait until locktime with 1 signature,
   * or spend immediately with M signatures
   */
  lock(
    address: string,
    locktime: number,
    m: number,
    publicKeys: PublicKey[]
  ): LockingScript {
    const script = new Script()

    // IF
    script.writeOpCode(Script.OP_IF)

    // Path 1: Time lock + single sig
    const timeLockScript = this.timeLock.lock(address, locktime)
    script.writeBytes(timeLockScript.toBinary())

    // ELSE
    script.writeOpCode(Script.OP_ELSE)

    // Path 2: Multisig (immediate)
    const multisigScript = this.multisig.lock(m, publicKeys)
    script.writeBytes(multisigScript.toBinary())

    // ENDIF
    script.writeOpCode(Script.OP_ENDIF)

    return script
  }

  unlockWithTimeLock(privateKey: PrivateKey, locktime: number): UnlockingScript {
    return {
      script: new Script(),
      estimatedLength: 140,

      sign: async (tx: Transaction, inputIndex: number) => {
        const script = new Script()

        // Signature for time lock path
        const sig = tx.sign(inputIndex, privateKey, privateKey.toPublicKey())

        script.writeBytes(sig.toDER())
        script.writeBytes(privateKey.toPublicKey().toDER())
        script.writeOpCode(Script.OP_1) // Choose IF branch

        return script
      },

      estimateLength: async () => 140
    }
  }

  unlockWithMultisig(signatures: Signature[]): UnlockingScript {
    return {
      script: new Script(),
      estimatedLength: signatures.length * 73 + 2,

      sign: async () => {
        const script = new Script()

        script.writeOpCode(Script.OP_0) // CHECKMULTISIG bug

        for (const sig of signatures) {
          script.writeBytes(sig.toDER())
        }

        script.writeOpCode(Script.OP_0) // Choose ELSE branch

        return script
      },

      estimateLength: async () => signatures.length * 73 + 2
    }
  }
}

// Usage
const composite = new TimeLockMultisig()

// Lock: Can spend after 1 week with 1 sig, OR immediately with 2-of-3 multisig
const lockingScript = composite.lock(
  address,
  oneWeekFromNow,
  2,
  [pubKey1, pubKey2, pubKey3]
)

// Option 1: Wait and spend with single signature
const unlockSingle = composite.unlockWithTimeLock(privateKey, oneWeekFromNow)

// Option 2: Spend immediately with multisig
const unlockMulti = composite.unlockWithMultisig([sig1, sig2])
```

## 7. Protocol-Specific Templates

### Token Transfer Template

```typescript
/**
 * Simple Token Protocol Template
 * Embeds token data in locking script
 */
class TokenTemplate implements ScriptTemplate {
  /**
   * Lock with token data
   */
  lock(
    address: string,
    tokenId: string,
    amount: number
  ): LockingScript {
    const script = new Script()

    // Token protocol marker
    script.writeBytes(Buffer.from('TOKEN', 'utf8'))

    // Token ID
    script.writeBytes(Buffer.from(tokenId, 'hex'))

    // Amount
    script.writeBigNum(amount)

    // OP_DROP to remove token data
    script.writeOpCode(Script.OP_2DROP)
    script.writeOpCode(Script.OP_DROP)

    // Standard P2PKH for spending
    script.writeOpCode(Script.OP_DUP)
    script.writeOpCode(Script.OP_HASH160)

    const decoded = Utils.fromBase58Check(address)
    const pubKeyHash = Buffer.from(decoded.data)
    script.writeBytes(pubKeyHash)

    script.writeOpCode(Script.OP_EQUALVERIFY)
    script.writeOpCode(Script.OP_CHECKSIG)

    return script
  }

  unlock(privateKey: PrivateKey): UnlockingScript {
    return new P2PKH().unlock(privateKey)
  }

  /**
   * Parse token data from locking script
   */
  parseTokenData(lockingScript: Script): {
    tokenId: string
    amount: number
  } {
    const chunks = lockingScript.chunks

    return {
      tokenId: chunks[1].data.toString('hex'),
      amount: Script.fromBinary(chunks[2].data).readBigNum()
    }
  }
}

// Usage
const token = new TokenTemplate()

// Create token output
const lockingScript = token.lock(
  recipientAddress,
  'token123', // Token ID
  1000 // Amount
)

// Parse token data
const tokenData = token.parseTokenData(lockingScript)
console.log('Token:', tokenData.tokenId, 'Amount:', tokenData.amount)
```

### HTLC (Hashed Timelock Contract) Template

```typescript
/**
 * HTLC Template for Atomic Swaps
 * Combines hash lock + time lock
 */
class HTLCTemplate implements ScriptTemplate {
  /**
   * Lock with hash and timelock
   * Recipient can claim with preimage before timeout
   * Sender can refund after timeout
   */
  lock(
    recipientPubKey: PublicKey,
    senderPubKey: PublicKey,
    hash: Buffer,
    locktime: number
  ): LockingScript {
    const script = new Script()

    // IF recipient path (hash + signature)
    script.writeOpCode(Script.OP_IF)

    // Verify hash
    script.writeOpCode(Script.OP_HASH256)
    script.writeBytes(hash)
    script.writeOpCode(Script.OP_EQUALVERIFY)

    // Verify recipient signature
    script.writeBytes(recipientPubKey.toDER())

    // ELSE sender refund path (time + signature)
    script.writeOpCode(Script.OP_ELSE)

    // Check locktime
    script.writeBigNum(locktime)
    script.writeOpCode(Script.OP_CHECKLOCKTIMEVERIFY)
    script.writeOpCode(Script.OP_DROP)

    // Verify sender signature
    script.writeBytes(senderPubKey.toDER())

    // ENDIF
    script.writeOpCode(Script.OP_ENDIF)

    // Check signature
    script.writeOpCode(Script.OP_CHECKSIG)

    return script
  }

  unlockWithPreimage(
    preimage: Buffer,
    recipientPrivateKey: PrivateKey
  ): UnlockingScript {
    return {
      script: new Script(),
      estimatedLength: preimage.length + 73 + 1,

      sign: async (tx: Transaction, inputIndex: number) => {
        const script = new Script()

        const sig = tx.sign(
          inputIndex,
          recipientPrivateKey,
          recipientPrivateKey.toPublicKey()
        )

        script.writeBytes(sig.toDER())
        script.writeBytes(preimage)
        script.writeOpCode(Script.OP_1) // IF branch

        return script
      },

      estimateLength: async () => preimage.length + 73 + 1
    }
  }

  unlockWithTimeout(
    senderPrivateKey: PrivateKey,
    locktime: number
  ): UnlockingScript {
    return {
      script: new Script(),
      estimatedLength: 73 + 1,

      sign: async (tx: Transaction, inputIndex: number) => {
        const script = new Script()

        const sig = tx.sign(
          inputIndex,
          senderPrivateKey,
          senderPrivateKey.toPublicKey(),
          locktime
        )

        script.writeBytes(sig.toDER())
        script.writeOpCode(Script.OP_0) // ELSE branch

        return script
      },

      estimateLength: async () => 73 + 1
    }
  }
}

// Usage - Atomic swap
const htlc = new HTLCTemplate()

const preimage = Buffer.from('secret_value')
const hash = Hash.hash256(preimage)
const timeoutIn24Hours = Math.floor(Date.now() / 1000) + 86400

// Create HTLC output
const lockingScript = htlc.lock(
  recipientPubKey,
  senderPubKey,
  hash,
  timeoutIn24Hours
)

// Recipient claims with preimage
const unlockRecipient = htlc.unlockWithPreimage(preimage, recipientPrivateKey)

// OR sender refunds after timeout
const unlockRefund = htlc.unlockWithTimeout(senderPrivateKey, timeoutIn24Hours)
```

## 8. Testing Script Templates

### Unit Testing Templates

```typescript
import { Transaction, PrivateKey } from '@bsv/sdk'

describe('PasswordTemplate', () => {
  it('should lock and unlock with correct password', async () => {
    const template = new PasswordTemplate()
    const password = 'secret123'
    const passwordHash = Hash.hash256(Buffer.from(password, 'utf8'))
      .toString('hex')

    // Create locking script
    const lockingScript = template.lock(passwordHash)

    // Create unlocking script
    const unlockingScript = template.unlock(password)

    // Create transaction
    const tx = new Transaction()
    tx.addInput({
      sourceTXID: 'a'.repeat(64),
      sourceOutputIndex: 0,
      unlockingScriptTemplate: unlockingScript,
      sourceOutput: {
        lockingScript,
        satoshis: 10000
      }
    })

    tx.addOutput({
      lockingScript: new P2PKH().lock(recipientAddress),
      satoshis: 9500
    })

    // Verify transaction is valid
    const result = tx.verify()
    expect(result).toBe(true)
  })

  it('should fail with incorrect password', async () => {
    const template = new PasswordTemplate()
    const password = 'secret123'
    const wrongPassword = 'wrong'
    const passwordHash = Hash.hash256(Buffer.from(password, 'utf8'))
      .toString('hex')

    const lockingScript = template.lock(passwordHash)
    const unlockingScript = template.unlock(wrongPassword)

    const tx = new Transaction()
    tx.addInput({
      sourceTXID: 'a'.repeat(64),
      sourceOutputIndex: 0,
      unlockingScriptTemplate: unlockingScript,
      sourceOutput: {
        lockingScript,
        satoshis: 10000
      }
    })

    // Should fail verification
    expect(() => tx.verify()).toThrow()
  })
})
```

## 9. Best Practices

1. **Always estimate unlocking script size** accurately for fee calculation
2. **Test templates thoroughly** with both valid and invalid unlocking attempts
3. **Document template requirements** clearly (locktime, sequence numbers, etc.)
4. **Use appropriate sighash types** for your use case
5. **Validate template parameters** before creating scripts
6. **Consider script size limits** (10,000 bytes per script)
7. **Implement proper error handling** for template creation
8. **Cache compiled templates** for performance
9. **Use type-safe interfaces** for template parameters
10. **Follow BRC standards** for interoperability

## 10. Common Pitfalls

1. **Forgetting OP_0 for CHECKMULTISIG** - Off-by-one bug requires extra OP_0
2. **Wrong sequence numbers** for CHECKLOCKTIMEVERIFY/CHECKSEQUENCEVERIFY
3. **Incorrect locktime format** - Block height vs timestamp
4. **Not handling estimated vs actual** unlocking scripts properly
5. **Script size exceeding limits** - Monitor total script size
6. **Incorrect sighash type** - Using wrong type for the use case
7. **Not setting transaction locktime** when using time locks

## Hands-On Project: Multi-Party Payment Channel

Build a complete payment channel system using custom templates:

```typescript
class PaymentChannelTemplate {
  // Create channel funding output (2-of-2 multisig)
  createFundingOutput(
    party1PubKey: PublicKey,
    party2PubKey: PublicKey,
    amount: number
  ): Transaction {
    // Implementation using multisig template
  }

  // Create commitment transaction (with time lock)
  createCommitment(
    party1Balance: number,
    party2Balance: number,
    sequence: number
  ): Transaction {
    // Implementation using relative time lock
  }

  // Close channel cooperatively
  closeChannel(
    party1Sig: Signature,
    party2Sig: Signature
  ): Transaction {
    // Implementation using immediate multisig spend
  }
}
```

## Next Steps

Continue to:
- **[SPV Verification](../spv-verification/README.md)** - Verify transactions without full blockchain


## Additional Resources

- [Script Templates SDK Component](../../../sdk-components/script-templates/README.md)
- [Script SDK Component](../../../sdk-components/script/README.md)
- [P2PKH SDK Component](../../../sdk-components/p2pkh/README.md)
- [Bitcoin Script Opcodes Reference](https://wiki.bitcoinsv.io/index.php/Opcodes_used_in_Bitcoin_Script)
- [BRC-3: Digital Signatures](https://github.com/bitcoin-sv/BRCs/blob/master/key-derivation/0003.md)

---

**Status:** ✅ Complete - Ready for learning
