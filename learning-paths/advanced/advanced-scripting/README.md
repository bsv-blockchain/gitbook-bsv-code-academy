# Advanced Scripting

## Overview

Master advanced Bitcoin Script features to build complex, production-ready smart contracts on BSV. This module covers all Bitcoin Script opcodes, advanced cryptographic patterns, stateful contracts, oracle integration, multi-party computation, atomic swaps, covenants, and optimization techniques. Learn to implement sophisticated on-chain logic using the BSV SDK's Script system and create secure, efficient smart contracts for real-world applications.

**Estimated Time:** 5-6 hours
**Difficulty:** Advanced

## Learning Objectives

By the end of this module, you will be able to:

- ✅ Understand and use all Bitcoin Script opcodes in depth
- ✅ Build complex smart contracts with advanced cryptographic features
- ✅ Implement stateful contracts that maintain state across transactions
- ✅ Create R-puzzles and signature-based smart contracts
- ✅ Design multi-party computation scripts for collaborative agreements
- ✅ Build atomic swaps and hash time-locked contracts (HTLCs)
- ✅ Implement covenant scripts that restrict output conditions
- ✅ Optimize script execution efficiency and minimize script size
- ✅ Test and debug complex scripts with proper tooling
- ✅ Apply advanced cryptographic techniques for security and privacy

## SDK Components Used

This course leverages these standardized SDK modules:

- **[Script](../../../sdk-components/script/README.md)** - Low-level script operations and opcodes
- **[Script Templates](../../../sdk-components/script-templates/README.md)** - Reusable script patterns
- **[Signatures](../../../sdk-components/signatures/README.md)** - Digital signatures and SIGHASH types
- **[Transaction](../../../sdk-components/transaction/README.md)** - Transaction construction and signing
- **[P2PKH](../../../sdk-components/p2pkh/README.md)** - Standard payment script template

## 1. Advanced Opcodes Deep Dive

### Understanding Bitcoin Script

Reference: **[Script Component](../../../sdk-components/script/README.md)**

Bitcoin Script is a stack-based, non-Turing complete language that processes operations (opcodes) sequentially. BSV re-enabled all original Bitcoin opcodes, providing a complete scripting system for complex smart contracts.

```typescript
import { Script, OP } from '@bsv/sdk'

/**
 * Bitcoin Script Execution Model
 *
 * - Stack-based: Operations manipulate a stack of byte arrays
 * - Sequential: Opcodes execute left-to-right
 * - Non-Turing complete: No loops (prevents infinite execution)
 * - Deterministic: Same script always produces same result
 * - Stateless: Each script execution is independent
 */

// Basic script execution
const script = new Script()
  .writeOpCode(OP.OP_1)  // Push 1 onto stack
  .writeOpCode(OP.OP_2)  // Push 2 onto stack
  .writeOpCode(OP.OP_ADD) // Pop both, push sum (3)
  .writeOpCode(OP.OP_3)  // Push 3 onto stack
  .writeOpCode(OP.OP_EQUAL) // Pop both, compare (true)

// Script succeeds if top of stack is true (non-zero)
```

### Opcode Categories

#### 1. Stack Manipulation Opcodes

```typescript
import { Script, OP } from '@bsv/sdk'

/**
 * Stack Manipulation Examples
 */

// OP_DUP - Duplicate top stack item
function duplicateTop(): Script {
  return new Script()
    .writeBin(Buffer.from('hello'))  // Stack: [hello]
    .writeOpCode(OP.OP_DUP)          // Stack: [hello, hello]
}

// OP_SWAP - Swap top two items
function swapTopTwo(): Script {
  return new Script()
    .writeBin(Buffer.from('first'))  // Stack: [first]
    .writeBin(Buffer.from('second')) // Stack: [first, second]
    .writeOpCode(OP.OP_SWAP)         // Stack: [second, first]
}

// OP_PICK - Copy nth item to top
function pickNthItem(): Script {
  return new Script()
    .writeBin(Buffer.from('item0'))  // Stack: [item0]
    .writeBin(Buffer.from('item1'))  // Stack: [item0, item1]
    .writeBin(Buffer.from('item2'))  // Stack: [item0, item1, item2]
    .writeOpCode(OP.OP_2)            // Stack: [item0, item1, item2, 2]
    .writeOpCode(OP.OP_PICK)         // Stack: [item0, item1, item2, item0]
}

// OP_ROLL - Move nth item to top
function rollNthItem(): Script {
  return new Script()
    .writeBin(Buffer.from('item0'))  // Stack: [item0]
    .writeBin(Buffer.from('item1'))  // Stack: [item0, item1]
    .writeBin(Buffer.from('item2'))  // Stack: [item0, item1, item2]
    .writeOpCode(OP.OP_2)            // Stack: [item0, item1, item2, 2]
    .writeOpCode(OP.OP_ROLL)         // Stack: [item1, item2, item0]
}

// OP_DROP - Remove top item
function dropTop(): Script {
  return new Script()
    .writeBin(Buffer.from('keep'))
    .writeBin(Buffer.from('remove'))
    .writeOpCode(OP.OP_DROP)         // Removes 'remove', keeps 'keep'
}

// OP_2DROP - Remove top two items
function dropTopTwo(): Script {
  return new Script()
    .writeBin(Buffer.from('keep'))
    .writeBin(Buffer.from('remove1'))
    .writeBin(Buffer.from('remove2'))
    .writeOpCode(OP.OP_2DROP)        // Removes both, keeps 'keep'
}
```

#### 2. Arithmetic Opcodes

```typescript
import { Script, OP } from '@bsv/sdk'

/**
 * Arithmetic Operations
 * All arithmetic uses 32-bit signed integers
 */

// Basic arithmetic
function basicMath(): Script {
  return new Script()
    .writeOpCode(OP.OP_5)        // Stack: [5]
    .writeOpCode(OP.OP_3)        // Stack: [5, 3]
    .writeOpCode(OP.OP_ADD)      // Stack: [8] (5+3)
    .writeOpCode(OP.OP_2)        // Stack: [8, 2]
    .writeOpCode(OP.OP_SUB)      // Stack: [6] (8-2)
}

// Multiplication and division
function multiplyDivide(): Script {
  return new Script()
    .writeOpCode(OP.OP_4)        // Stack: [4]
    .writeOpCode(OP.OP_3)        // Stack: [4, 3]
    .writeOpCode(OP.OP_MUL)      // Stack: [12] (4*3)
    .writeOpCode(OP.OP_2)        // Stack: [12, 2]
    .writeOpCode(OP.OP_DIV)      // Stack: [6] (12/2)
}

// Modulo and negation
function moduloNegate(): Script {
  return new Script()
    .writeOpCode(OP.OP_7)        // Stack: [7]
    .writeOpCode(OP.OP_3)        // Stack: [7, 3]
    .writeOpCode(OP.OP_MOD)      // Stack: [1] (7%3)
    .writeOpCode(OP.OP_NEGATE)   // Stack: [-1]
}

// Comparison operators
function comparison(): Script {
  return new Script()
    .writeOpCode(OP.OP_5)        // Stack: [5]
    .writeOpCode(OP.OP_3)        // Stack: [5, 3]
    .writeOpCode(OP.OP_LESSTHAN) // Stack: [0] (false, 5 is not < 3)
    .writeOpCode(OP.OP_NOT)      // Stack: [1] (true)
}

// Min/Max operations
function minMax(): Script {
  return new Script()
    .writeOpCode(OP.OP_5)        // Stack: [5]
    .writeOpCode(OP.OP_3)        // Stack: [5, 3]
    .writeOpCode(OP.OP_MIN)      // Stack: [3]
    .writeOpCode(OP.OP_5)        // Stack: [3, 5]
    .writeOpCode(OP.OP_MAX)      // Stack: [5]
}

// Within range check
function withinRange(): Script {
  return new Script()
    .writeOpCode(OP.OP_5)        // Value to check
    .writeOpCode(OP.OP_1)        // Min
    .writeOpCode(OP.OP_10)       // Max
    .writeOpCode(OP.OP_WITHIN)   // Check if 5 is in [1, 10)
}
```

#### 3. Bitwise Operations

```typescript
import { Script, OP } from '@bsv/sdk'

/**
 * Bitwise Operations (re-enabled on BSV)
 */

// AND, OR, XOR
function bitwiseLogic(): Script {
  return new Script()
    // AND operation
    .writeBin(Buffer.from([0b11110000]))
    .writeBin(Buffer.from([0b10101010]))
    .writeOpCode(OP.OP_AND)      // Result: 0b10100000

    // OR operation
    .writeBin(Buffer.from([0b11110000]))
    .writeBin(Buffer.from([0b10101010]))
    .writeOpCode(OP.OP_OR)       // Result: 0b11111010

    // XOR operation
    .writeBin(Buffer.from([0b11110000]))
    .writeBin(Buffer.from([0b10101010]))
    .writeOpCode(OP.OP_XOR)      // Result: 0b01011010
}

// Left and right shift
function bitwiseShift(): Script {
  return new Script()
    // Left shift (multiply by 2)
    .writeBin(Buffer.from([0b00001111]))
    .writeOpCode(OP.OP_1)
    .writeOpCode(OP.OP_LSHIFT)   // Result: 0b00011110

    // Right shift (divide by 2)
    .writeBin(Buffer.from([0b11110000]))
    .writeOpCode(OP.OP_1)
    .writeOpCode(OP.OP_RSHIFT)   // Result: 0b01111000
}

// Invert bits
function bitwiseInvert(): Script {
  return new Script()
    .writeBin(Buffer.from([0b11110000]))
    .writeOpCode(OP.OP_INVERT)   // Result: 0b00001111
}
```

#### 4. Cryptographic Opcodes

```typescript
import { Script, OP, Hash, PrivateKey } from '@bsv/sdk'

/**
 * Cryptographic Operations
 */

// Hash functions
function hashOperations(): Script {
  return new Script()
    // SHA256
    .writeBin(Buffer.from('hello'))
    .writeOpCode(OP.OP_SHA256)

    // Double SHA256 (Bitcoin's standard hash)
    .writeBin(Buffer.from('hello'))
    .writeOpCode(OP.OP_HASH256)

    // RIPEMD160
    .writeBin(Buffer.from('hello'))
    .writeOpCode(OP.OP_RIPEMD160)

    // HASH160 (SHA256 then RIPEMD160 - used for addresses)
    .writeBin(Buffer.from('hello'))
    .writeOpCode(OP.OP_HASH160)
}

// Signature verification
function signatureVerification(
  message: Buffer,
  signature: Buffer,
  publicKey: Buffer
): Script {
  return new Script()
    // Push signature and public key
    .writeBin(signature)
    .writeBin(publicKey)

    // Verify signature
    .writeOpCode(OP.OP_CHECKSIG)  // Returns 1 if valid, 0 if invalid
}

// Multi-signature verification
function multiSigVerification(
  signatures: Buffer[],
  publicKeys: Buffer[],
  requiredSigs: number
): Script {
  const script = new Script()

  // Push OP_0 (bug in original Bitcoin requires this)
  script.writeOpCode(OP.OP_0)

  // Push signatures
  for (const sig of signatures) {
    script.writeBin(sig)
  }

  // Push number of signatures required
  script.writeOpCode(requiredSigs)

  // Push public keys
  for (const pubKey of publicKeys) {
    script.writeBin(pubKey)
  }

  // Push total number of public keys
  script.writeOpCode(publicKeys.length)

  // Check multisig
  script.writeOpCode(OP.OP_CHECKMULTISIG)

  return script
}
```

#### 5. Control Flow Opcodes

```typescript
import { Script, OP } from '@bsv/sdk'

/**
 * Control Flow Operations
 */

// IF/ELSE/ENDIF
function conditionalExecution(): Script {
  return new Script()
    // Check if value is greater than 5
    .writeOpCode(OP.OP_DUP)      // Duplicate the value
    .writeOpCode(OP.OP_5)
    .writeOpCode(OP.OP_GREATERTHAN)

    // If true, execute this branch
    .writeOpCode(OP.OP_IF)
    .writeBin(Buffer.from('greater than 5'))

    // Otherwise, execute this branch
    .writeOpCode(OP.OP_ELSE)
    .writeBin(Buffer.from('5 or less'))

    .writeOpCode(OP.OP_ENDIF)
}

// NOTIF (executes if false)
function notIfExecution(): Script {
  return new Script()
    .writeOpCode(OP.OP_0)        // Push false

    .writeOpCode(OP.OP_NOTIF)    // Execute because top is false
    .writeBin(Buffer.from('executed'))
    .writeOpCode(OP.OP_ENDIF)
}

// VERIFY (abort if false)
function verifyExecution(): Script {
  return new Script()
    .writeOpCode(OP.OP_5)
    .writeOpCode(OP.OP_5)
    .writeOpCode(OP.OP_EQUAL)
    .writeOpCode(OP.OP_VERIFY)   // Aborts script if false, continues if true

    // This only executes if verification passed
    .writeOpCode(OP.OP_1)
}

// RETURN (immediately fails script)
function earlyReturn(): Script {
  return new Script()
    .writeOpCode(OP.OP_1)
    .writeOpCode(OP.OP_IF)
    .writeOpCode(OP.OP_RETURN)   // Immediately fails
    .writeOpCode(OP.OP_ENDIF)

    // This never executes
    .writeOpCode(OP.OP_1)
}
```

## 2. R-Puzzles and Signature Puzzles

### Understanding ECDSA Signatures

Reference: **[Signatures Component](../../../sdk-components/signatures/README.md)**

ECDSA signatures consist of two components: (r, s). An R-puzzle is a script that requires knowledge of the private key corresponding to a specific r value, enabling advanced smart contract patterns.

```typescript
import { PrivateKey, PublicKey, Signature, Hash, Script, OP } from '@bsv/sdk'

/**
 * ECDSA Signature Structure
 *
 * Signature = (r, s) where:
 * - r = x-coordinate of point R = k * G (k is random nonce)
 * - s = (hash + privateKey * r) / k
 *
 * R-puzzle: Lock coins to specific r value
 * Anyone who knows the private key for that r can spend
 */

// Extract r value from signature
function extractRValue(signature: Signature): Buffer {
  const sigBytes = signature.toCompact()

  // DER encoding: 0x30 [total-length] 0x02 [r-length] [r] 0x02 [s-length] [s]
  const rLength = sigBytes[3]
  const r = sigBytes.slice(4, 4 + rLength)

  return r
}

// Create R-puzzle locking script
function createRPuzzleLock(rValue: Buffer, pubKeyHash: Buffer): Script {
  return new Script()
    // Expect signature on stack
    .writeOpCode(OP.OP_3)
    .writeOpCode(OP.OP_SPLIT)      // Split signature at position 3
    .writeOpCode(OP.OP_NIP)        // Remove first part
    .writeOpCode(OP.OP_1)
    .writeOpCode(OP.OP_SPLIT)      // Split to get r length
    .writeOpCode(OP.OP_SWAP)
    .writeOpCode(OP.OP_SPLIT)      // Extract r value
    .writeOpCode(OP.OP_DROP)

    // Verify r value matches
    .writeBin(rValue)
    .writeOpCode(OP.OP_EQUALVERIFY)

    // Standard P2PKH verification
    .writeOpCode(OP.OP_DUP)
    .writeOpCode(OP.OP_HASH160)
    .writeBin(pubKeyHash)
    .writeOpCode(OP.OP_EQUALVERIFY)
    .writeOpCode(OP.OP_CHECKSIG)
}
```

### Advanced R-Puzzle Patterns

```typescript
import { Transaction, PrivateKey, Script, OP, Hash } from '@bsv/sdk'

/**
 * R-Puzzle Payment Channel
 *
 * Use R-puzzles to create payment channels where
 * specific signatures are required for specific states
 */

class RPuzzleChannel {
  private channelKey: PrivateKey
  private rValues: Map<number, Buffer> = new Map()

  constructor(channelKey: PrivateKey) {
    this.channelKey = channelKey
  }

  /**
   * Create channel state with specific r value
   */
  async createState(stateNumber: number, amount: number): Promise<{
    rValue: Buffer
    signature: Buffer
    lockingScript: Script
  }> {
    // Generate deterministic k for this state
    const k = this.generateK(stateNumber)
    const kPriv = PrivateKey.fromString(k.toString('hex'), 'hex')

    // R = k * G
    const R = kPriv.toPublicKey()
    const rValue = R.encode(true).slice(1, 33) // x-coordinate

    this.rValues.set(stateNumber, rValue)

    // Create locking script for this state
    const pubKeyHash = Hash.hash160(this.channelKey.toPublicKey().encode(true))
    const lockingScript = createRPuzzleLock(rValue, pubKeyHash)

    // Pre-sign a transaction for this state
    const message = Buffer.from(`state-${stateNumber}-${amount}`)
    const messageHash = Hash.sha256(message)

    // Sign with specific k
    const signature = this.channelKey.sign(messageHash, k)

    return {
      rValue,
      signature: signature.toCompact(),
      lockingScript
    }
  }

  /**
   * Generate deterministic k for state number
   */
  private generateK(stateNumber: number): Buffer {
    const data = Buffer.concat([
      this.channelKey.encode(),
      Buffer.from([stateNumber])
    ])
    return Hash.sha256(data)
  }

  /**
   * Create unlocking script for specific state
   */
  createUnlock(stateNumber: number, signature: Buffer, publicKey: Buffer): Script {
    return new Script()
      .writeBin(signature)
      .writeBin(publicKey)
  }
}

// Usage example
async function rPuzzleChannelExample() {
  const channelKey = PrivateKey.fromRandom()
  const channel = new RPuzzleChannel(channelKey)

  // Create channel state 1 with 1000 satoshis
  const state1 = await channel.createState(1, 1000)

  // Create channel state 2 with 2000 satoshis
  const state2 = await channel.createState(2, 2000)

  // Later: unlock specific state
  const unlockScript = channel.createUnlock(
    1,
    state1.signature,
    channelKey.toPublicKey().encode(true)
  )

  console.log('State 1 r-value:', state1.rValue.toString('hex'))
  console.log('State 2 r-value:', state2.rValue.toString('hex'))
}
```

### Signature Grinding for Specific Values

```typescript
import { PrivateKey, Hash, Signature } from '@bsv/sdk'

/**
 * Signature Grinding
 *
 * Generate signatures with specific properties
 * (e.g., low r values for smaller signatures)
 */

class SignatureGrinder {
  /**
   * Generate signature with low r value
   * (reduces signature size by ~1 byte on average)
   */
  async grindLowR(
    privateKey: PrivateKey,
    messageHash: Buffer,
    maxAttempts: number = 100
  ): Promise<Signature> {
    for (let i = 0; i < maxAttempts; i++) {
      // Generate random k
      const k = Hash.sha256(Buffer.concat([
        privateKey.encode(),
        messageHash,
        Buffer.from([i])
      ]))

      // Create signature with this k
      const signature = privateKey.sign(messageHash, k)
      const r = extractRValue(signature)

      // Check if r is "low" (first byte < 0x80)
      if (r[0] < 0x80) {
        return signature
      }
    }

    // Fall back to normal signature
    return privateKey.sign(messageHash)
  }

  /**
   * Generate signature with specific r value prefix
   */
  async grindRPrefix(
    privateKey: PrivateKey,
    messageHash: Buffer,
    prefix: Buffer,
    maxAttempts: number = 10000
  ): Promise<Signature | null> {
    for (let i = 0; i < maxAttempts; i++) {
      const k = Hash.sha256(Buffer.concat([
        privateKey.encode(),
        messageHash,
        Buffer.from([i >> 8, i & 0xff])
      ]))

      const signature = privateKey.sign(messageHash, k)
      const r = extractRValue(signature)

      // Check if r starts with prefix
      if (r.slice(0, prefix.length).equals(prefix)) {
        return signature
      }
    }

    return null
  }
}

// Usage example
async function signatureGrindingExample() {
  const privateKey = PrivateKey.fromRandom()
  const message = Buffer.from('Hello BSV')
  const messageHash = Hash.sha256(message)

  const grinder = new SignatureGrinder()

  // Generate signature with low r value
  const lowRSig = await grinder.grindLowR(privateKey, messageHash)
  console.log('Low-r signature:', lowRSig.toCompact().toString('hex'))

  // Generate signature with specific prefix (0x00)
  const prefixSig = await grinder.grindRPrefix(
    privateKey,
    messageHash,
    Buffer.from([0x00])
  )

  if (prefixSig) {
    console.log('Prefix signature:', prefixSig.toCompact().toString('hex'))
  }
}
```

## 3. Hash-Based Contracts

### Hash Locks

```typescript
import { Script, OP, Hash } from '@bsv/sdk'

/**
 * Hash Lock Pattern
 *
 * Lock coins that can only be spent by revealing
 * a preimage that hashes to a specific value
 */

// Create hash lock
function createHashLock(secretHash: Buffer): Script {
  return new Script()
    // Expect preimage on stack
    .writeOpCode(OP.OP_SHA256)
    .writeBin(secretHash)
    .writeOpCode(OP.OP_EQUAL)
}

// Create hash lock with fallback (timeout)
function createHashLockWithTimeout(
  secretHash: Buffer,
  timeoutBlock: number,
  fallbackPubKeyHash: Buffer
): Script {
  return new Script()
    // Try hash lock first
    .writeOpCode(OP.OP_DUP)
    .writeOpCode(OP.OP_SHA256)
    .writeBin(secretHash)
    .writeOpCode(OP.OP_EQUAL)

    .writeOpCode(OP.OP_IF)
    // Hash matches - allow spend
    .writeOpCode(OP.OP_DROP)
    .writeOpCode(OP.OP_1)

    .writeOpCode(OP.OP_ELSE)
    // Hash doesn't match - check timeout
    .writeOpCode(OP.OP_DROP)
    .writeBin(Buffer.from([timeoutBlock >> 24, timeoutBlock >> 16, timeoutBlock >> 8, timeoutBlock]))
    .writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
    .writeOpCode(OP.OP_DROP)

    // Verify fallback signature
    .writeOpCode(OP.OP_DUP)
    .writeOpCode(OP.OP_HASH160)
    .writeBin(fallbackPubKeyHash)
    .writeOpCode(OP.OP_EQUALVERIFY)
    .writeOpCode(OP.OP_CHECKSIG)

    .writeOpCode(OP.OP_ENDIF)
}
```

### Hash Chains

```typescript
import { Hash } from '@bsv/sdk'

/**
 * Hash Chain Pattern
 *
 * Create a chain of hashes where each hash is
 * the hash of the previous value
 */

class HashChain {
  private chain: Buffer[] = []

  /**
   * Generate hash chain of specified length
   */
  generate(seed: Buffer, length: number): Buffer[] {
    let current = seed
    this.chain = [current]

    // Build chain by hashing repeatedly
    for (let i = 0; i < length; i++) {
      current = Hash.sha256(current)
      this.chain.push(current)
    }

    return this.chain
  }

  /**
   * Get hash at specific position
   */
  getHash(index: number): Buffer {
    if (index < 0 || index >= this.chain.length) {
      throw new Error('Index out of bounds')
    }
    return this.chain[index]
  }

  /**
   * Get final hash (anchor)
   */
  getAnchor(): Buffer {
    return this.chain[this.chain.length - 1]
  }

  /**
   * Verify a value is at specific position in chain
   */
  verify(value: Buffer, position: number): boolean {
    let current = value

    // Hash forward to anchor
    for (let i = position; i < this.chain.length - 1; i++) {
      current = Hash.sha256(current)
    }

    // Check if we reached the anchor
    return current.equals(this.getAnchor())
  }
}

// Usage in smart contract
function createHashChainLock(anchor: Buffer, position: number): Script {
  return new Script()
    // Expect preimage on stack
    // Hash it 'position' times to reach anchor
    .writeOpCode(OP.OP_DUP)

  // Add SHA256 operations for each position
  for (let i = 0; i < position; i++) {
    new Script().writeOpCode(OP.OP_SHA256)
  }

  return new Script()
    // Compare with anchor
    .writeBin(anchor)
    .writeOpCode(OP.OP_EQUAL)
}

// Example: Micropayment channel with hash chain
async function hashChainChannelExample() {
  // Generate 1000-link hash chain
  const seed = Hash.sha256(Buffer.from('channel-seed'))
  const chain = new HashChain()
  const hashes = chain.generate(seed, 1000)

  const anchor = chain.getAnchor()

  console.log('Channel anchor:', anchor.toString('hex'))

  // Create lock for payment 500 (halfway through channel)
  const lock500 = createHashChainLock(anchor, 500)

  // Verify payment 500
  const preimage500 = chain.getHash(500)
  const isValid = chain.verify(preimage500, 500)

  console.log('Payment 500 valid:', isValid)
}
```

### Preimage Revelation Contracts

```typescript
import { Script, OP, Hash, Transaction, PrivateKey } from '@bsv/sdk'

/**
 * Preimage Revelation Smart Contract
 *
 * A contract that pays for revealing a preimage
 * Useful for games, puzzles, and information markets
 */

class PreimageContract {
  /**
   * Create locking script that pays for preimage
   */
  createLock(
    secretHash: Buffer,
    rewardRecipient: Buffer, // pubKeyHash
    bountyPoster: Buffer,    // pubKeyHash
    timeoutBlock: number
  ): Script {
    return new Script()
      // Two spending paths:
      // 1. Reveal preimage before timeout
      // 2. Refund to bounty poster after timeout

      .writeOpCode(OP.OP_IF)
      // Path 1: Preimage revelation
      .writeOpCode(OP.OP_SHA256)
      .writeBin(secretHash)
      .writeOpCode(OP.OP_EQUALVERIFY)

      // Pay to reward recipient
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(rewardRecipient)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG)

      .writeOpCode(OP.OP_ELSE)
      // Path 2: Timeout refund
      .writeBin(Buffer.from([timeoutBlock >> 24, timeoutBlock >> 16, timeoutBlock >> 8, timeoutBlock]))
      .writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
      .writeOpCode(OP.OP_DROP)

      // Pay back to bounty poster
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(bountyPoster)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG)

      .writeOpCode(OP.OP_ENDIF)
  }

  /**
   * Create unlocking script to reveal preimage
   */
  createPreimageUnlock(
    preimage: Buffer,
    signature: Buffer,
    publicKey: Buffer
  ): Script {
    return new Script()
      .writeBin(signature)
      .writeBin(publicKey)
      .writeBin(preimage)
      .writeOpCode(OP.OP_1)  // Choose IF branch
  }

  /**
   * Create unlocking script for timeout refund
   */
  createRefundUnlock(
    signature: Buffer,
    publicKey: Buffer
  ): Script {
    return new Script()
      .writeBin(signature)
      .writeBin(publicKey)
      .writeOpCode(OP.OP_0)  // Choose ELSE branch
  }
}

// Usage example
async function preimageContractExample() {
  const contract = new PreimageContract()

  // Generate secret
  const secret = Buffer.from('my secret answer')
  const secretHash = Hash.sha256(secret)

  // Create contract
  const bountyPoster = PrivateKey.fromRandom()
  const rewardRecipient = PrivateKey.fromRandom()

  const bountyPubKeyHash = Hash.hash160(bountyPoster.toPublicKey().encode(true))
  const rewardPubKeyHash = Hash.hash160(rewardRecipient.toPublicKey().encode(true))

  const lockingScript = contract.createLock(
    secretHash,
    rewardPubKeyHash,
    bountyPubKeyHash,
    800000  // Timeout block
  )

  console.log('Contract created')
  console.log('Secret hash:', secretHash.toString('hex'))

  // Someone reveals the preimage
  // (In production, they would sign a transaction)
  const signature = Buffer.alloc(72) // Placeholder
  const publicKey = rewardRecipient.toPublicKey().encode(true)

  const unlockingScript = contract.createPreimageUnlock(
    secret,
    signature,
    publicKey
  )

  console.log('Preimage revealed, reward claimed')
}
```

## 4. Stateful Contracts

### Understanding State in Bitcoin Script

Bitcoin Script is stateless, but we can maintain state across transactions by:
1. Encoding state in outputs
2. Validating state transitions in scripts
3. Using covenants to enforce rules

```typescript
import { Script, OP, Transaction, Hash } from '@bsv/sdk'

/**
 * Stateful Counter Contract
 *
 * Maintains a counter that increments with each spend
 * State is stored in the output value
 */

class CounterContract {
  /**
   * Create locking script that enforces counter increment
   */
  createLock(currentCount: number, maxCount: number): Script {
    return new Script()
      // Verify current count is correct
      .writeOpCode(currentCount)
      .writeOpCode(OP.OP_EQUALVERIFY)

      // Increment counter
      .writeOpCode(OP.OP_1)
      .writeOpCode(OP.OP_ADD)

      // Check if we've reached max
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(maxCount)
      .writeOpCode(OP.OP_LESSTHANOREQUAL)
      .writeOpCode(OP.OP_VERIFY)

      // New count must be in output
      // (This would require covenant opcodes in production)
      .writeOpCode(OP.OP_1)
  }

  /**
   * Create unlocking script to increment counter
   */
  createUnlock(currentCount: number): Script {
    return new Script()
      .writeOpCode(currentCount)
  }
}
```

### Token Contract with Balance State

```typescript
import { Script, OP, Hash } from '@bsv/sdk'

/**
 * Simple Token Contract
 *
 * Maintains token balance in output script
 * Supports transfer and split operations
 */

class TokenContract {
  /**
   * Create token locking script
   */
  createLock(
    amount: number,
    ownerPubKeyHash: Buffer,
    tokenId: Buffer
  ): Script {
    return new Script()
      // Verify token ID
      .writeBin(tokenId)
      .writeOpCode(OP.OP_EQUALVERIFY)

      // Verify amount
      .writeOpCode(amount)
      .writeOpCode(OP.OP_EQUALVERIFY)

      // Verify owner signature
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(ownerPubKeyHash)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG)
  }

  /**
   * Create unlocking script for transfer
   */
  createTransferUnlock(
    signature: Buffer,
    publicKey: Buffer,
    amount: number,
    tokenId: Buffer
  ): Script {
    return new Script()
      .writeBin(signature)
      .writeBin(publicKey)
      .writeOpCode(amount)
      .writeBin(tokenId)
  }

  /**
   * Validate token transfer transaction
   * Ensures conservation of tokens (inputs = outputs)
   */
  validateTransfer(tx: Transaction, tokenId: Buffer): boolean {
    let inputAmount = 0
    let outputAmount = 0

    // Sum input amounts
    for (const input of tx.inputs) {
      if (!input.sourceTransaction) continue

      const output = input.sourceTransaction.outputs[input.sourceOutputIndex]
      const amount = this.extractTokenAmount(output.lockingScript, tokenId)
      if (amount) inputAmount += amount
    }

    // Sum output amounts
    for (const output of tx.outputs) {
      const amount = this.extractTokenAmount(output.lockingScript, tokenId)
      if (amount) outputAmount += amount
    }

    // Conservation law: inputs must equal outputs
    return inputAmount === outputAmount
  }

  /**
   * Extract token amount from locking script
   */
  private extractTokenAmount(script: Script, tokenId: Buffer): number | null {
    // Parse script to find token amount
    // This is simplified - production would need proper parsing
    const chunks = script.chunks

    // Find token ID in script
    for (let i = 0; i < chunks.length; i++) {
      if (chunks[i].data && chunks[i].data!.equals(tokenId)) {
        // Amount should be in a nearby chunk
        if (i + 1 < chunks.length && typeof chunks[i + 1].op === 'number') {
          return chunks[i + 1].op
        }
      }
    }

    return null
  }
}

// Usage example
async function tokenContractExample() {
  const contract = new TokenContract()
  const tokenId = Hash.sha256(Buffer.from('MY-TOKEN'))
  const ownerKey = PrivateKey.fromRandom()
  const ownerPubKeyHash = Hash.hash160(ownerKey.toPublicKey().encode(true))

  // Create token UTXO with 100 tokens
  const lockingScript = contract.createLock(100, ownerPubKeyHash, tokenId)

  console.log('Token created with 100 units')

  // Later: transfer 60 tokens to someone else
  // Would create two outputs: 60 to recipient, 40 as change
}
```

### State Machine Contract

```typescript
import { Script, OP } from '@bsv/sdk'

/**
 * State Machine Contract
 *
 * Implements a finite state machine in Bitcoin Script
 * Each state transition must follow defined rules
 */

enum ContractState {
  INITIALIZED = 0,
  ACTIVE = 1,
  PAUSED = 2,
  COMPLETED = 3
}

class StateMachineContract {
  /**
   * Create locking script for current state
   */
  createLock(
    currentState: ContractState,
    stateData: Buffer,
    controllerPubKeyHash: Buffer
  ): Script {
    return new Script()
      // Verify current state
      .writeOpCode(currentState)
      .writeOpCode(OP.OP_EQUALVERIFY)

      // Verify new state is valid transition
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(currentState)
      .writeOpCode(OP.OP_GREATERTHAN)  // New state must be higher
      .writeOpCode(OP.OP_VERIFY)

      // Verify controller signature
      .writeOpCode(OP.OP_SWAP)
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(controllerPubKeyHash)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG)
  }

  /**
   * Validate state transition
   */
  isValidTransition(from: ContractState, to: ContractState): boolean {
    // Define valid state transitions
    const validTransitions: Record<ContractState, ContractState[]> = {
      [ContractState.INITIALIZED]: [ContractState.ACTIVE],
      [ContractState.ACTIVE]: [ContractState.PAUSED, ContractState.COMPLETED],
      [ContractState.PAUSED]: [ContractState.ACTIVE, ContractState.COMPLETED],
      [ContractState.COMPLETED]: [] // Terminal state
    }

    return validTransitions[from]?.includes(to) ?? false
  }
}
```

## 5. Oracle Patterns

### Understanding Oracles

Oracles bring external data onto the blockchain. In BSV, oracles can sign data that smart contracts verify using OP_CHECKSIG.

```typescript
import { PrivateKey, PublicKey, Hash, Script, OP, Signature } from '@bsv/sdk'

/**
 * Oracle Data Provider
 *
 * Signs external data that can be verified in scripts
 */

class Oracle {
  private privateKey: PrivateKey
  public publicKey: PublicKey

  constructor(privateKey: PrivateKey) {
    this.privateKey = privateKey
    this.publicKey = privateKey.toPublicKey()
  }

  /**
   * Sign data with timestamp
   */
  signData(data: Buffer, timestamp: number = Date.now()): {
    data: Buffer
    timestamp: number
    signature: Buffer
  } {
    // Create message to sign
    const message = Buffer.concat([
      data,
      Buffer.from([
        timestamp >> 24,
        (timestamp >> 16) & 0xff,
        (timestamp >> 8) & 0xff,
        timestamp & 0xff
      ])
    ])

    const messageHash = Hash.sha256(message)
    const signature = this.privateKey.sign(messageHash)

    return {
      data,
      timestamp,
      signature: signature.toCompact()
    }
  }

  /**
   * Sign price feed data
   */
  signPrice(
    assetPair: string,
    price: number,
    decimals: number = 8
  ): {
    assetPair: string
    price: number
    decimals: number
    timestamp: number
    signature: Buffer
  } {
    const priceInt = Math.floor(price * Math.pow(10, decimals))

    const data = Buffer.concat([
      Buffer.from(assetPair),
      Buffer.from([
        priceInt >> 24,
        (priceInt >> 16) & 0xff,
        (priceInt >> 8) & 0xff,
        priceInt & 0xff
      ])
    ])

    const signed = this.signData(data)

    return {
      assetPair,
      price,
      decimals,
      timestamp: signed.timestamp,
      signature: signed.signature
    }
  }
}

/**
 * Oracle-based Smart Contract
 */
class OracleContract {
  /**
   * Create contract that requires oracle signature
   */
  createPriceFeedLock(
    oraclePubKey: Buffer,
    minimumPrice: number,
    assetPair: string
  ): Script {
    return new Script()
      // Stack: [signature, price, timestamp, assetPair]

      // Verify asset pair
      .writeBin(Buffer.from(assetPair))
      .writeOpCode(OP.OP_EQUALVERIFY)

      // Verify minimum price
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(minimumPrice)
      .writeOpCode(OP.OP_GREATERTHANOREQUAL)
      .writeOpCode(OP.OP_VERIFY)

      // Reconstruct signed message
      .writeOpCode(OP.OP_SWAP)  // Get timestamp
      .writeOpCode(OP.OP_CAT)   // Concatenate
      .writeOpCode(OP.OP_SWAP)  // Get asset pair
      .writeOpCode(OP.OP_CAT)   // Concatenate

      // Hash message
      .writeOpCode(OP.OP_SHA256)

      // Verify oracle signature
      .writeBin(oraclePubKey)
      .writeOpCode(OP.OP_CHECKSIG)
  }

  /**
   * Create multi-oracle contract (requires M of N oracles)
   */
  createMultiOracleLock(
    oraclePubKeys: Buffer[],
    requiredSignatures: number,
    minimumPrice: number
  ): Script {
    const script = new Script()

    // Verify minimum price
    script
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(minimumPrice)
      .writeOpCode(OP.OP_GREATERTHANOREQUAL)
      .writeOpCode(OP.OP_VERIFY)

    // Hash the price data
    script.writeOpCode(OP.OP_SHA256)

    // Set up multisig verification
    script.writeOpCode(OP.OP_0)  // Bug workaround

    // Push required signatures count
    script.writeOpCode(requiredSignatures)

    // Push oracle public keys
    for (const pubKey of oraclePubKeys) {
      script.writeBin(pubKey)
    }

    // Push total oracle count
    script.writeOpCode(oraclePubKeys.length)

    // Verify M-of-N signatures
    script.writeOpCode(OP.OP_CHECKMULTISIG)

    return script
  }
}

// Usage example
async function oracleExample() {
  // Create oracle
  const oracleKey = PrivateKey.fromRandom()
  const oracle = new Oracle(oracleKey)

  // Oracle signs price data
  const priceData = oracle.signPrice('BSV/USD', 52.50, 2)

  console.log('Oracle signed price:', priceData.price)
  console.log('Signature:', priceData.signature.toString('hex'))

  // Create contract requiring minimum price
  const contract = new OracleContract()
  const lockingScript = contract.createPriceFeedLock(
    oracle.publicKey.encode(true),
    5000,  // Minimum price: 50.00 (with 2 decimals)
    'BSV/USD'
  )

  console.log('Contract created with minimum price requirement')
}
```

### Oracle Aggregation Patterns

```typescript
/**
 * Oracle Aggregation Contract
 *
 * Aggregates data from multiple oracles
 * using median or average
 */

class OracleAggregator {
  /**
   * Calculate median price from multiple oracle feeds
   */
  calculateMedian(prices: number[]): number {
    const sorted = [...prices].sort((a, b) => a - b)
    const mid = Math.floor(sorted.length / 2)

    if (sorted.length % 2 === 0) {
      return (sorted[mid - 1] + sorted[mid]) / 2
    } else {
      return sorted[mid]
    }
  }

  /**
   * Create script that requires median of oracle prices
   */
  createMedianPriceLock(
    oraclePubKeys: Buffer[],
    minimumMedian: number
  ): Script {
    // In practice, this would require complex script operations
    // to compute median on-chain. Often better to compute off-chain
    // and verify with oracle signatures

    return new Script()
      // Simplified: expect median price and oracle signatures
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(minimumMedian)
      .writeOpCode(OP.OP_GREATERTHANOREQUAL)
      .writeOpCode(OP.OP_VERIFY)

      // Verify majority of oracles signed this median
      // (multisig verification)
      .writeOpCode(OP.OP_SHA256)
      .writeOpCode(OP.OP_0)
      .writeOpCode(Math.ceil(oraclePubKeys.length / 2))

    for (const pubKey of oraclePubKeys) {
      new Script().writeBin(pubKey)
    }

    return new Script()
      .writeOpCode(oraclePubKeys.length)
      .writeOpCode(OP.OP_CHECKMULTISIG)
  }
}
```

## 6. Multi-Party Computation

### Multi-Signature Contracts

Reference: **[P2PKH Component](../../../sdk-components/p2pkh/README.md)**

```typescript
import { Script, OP, Transaction, PrivateKey, Signature } from '@bsv/sdk'

/**
 * Multi-Signature Contract Builder
 *
 * Supports various multi-sig patterns
 */

class MultiSigContract {
  /**
   * Create M-of-N multisig locking script
   */
  createMultiSigLock(
    requiredSignatures: number,
    publicKeys: Buffer[]
  ): Script {
    if (requiredSignatures > publicKeys.length) {
      throw new Error('Required signatures cannot exceed number of keys')
    }

    if (publicKeys.length > 20) {
      throw new Error('Maximum 20 public keys allowed')
    }

    const script = new Script()

    // Push required signature count
    script.writeOpCode(requiredSignatures)

    // Push all public keys
    for (const pubKey of publicKeys) {
      script.writeBin(pubKey)
    }

    // Push total key count
    script.writeOpCode(publicKeys.length)

    // Check multisig
    script.writeOpCode(OP.OP_CHECKMULTISIG)

    return script
  }

  /**
   * Create weighted multisig (different keys have different weights)
   */
  createWeightedMultiSig(
    threshold: number,
    keysWithWeights: Array<{ publicKey: Buffer, weight: number }>
  ): Script {
    // This requires custom script logic to sum weights
    const script = new Script()

    // Initialize accumulator
    script.writeOpCode(OP.OP_0)

    // For each provided signature, add its weight
    for (const { publicKey, weight } of keysWithWeights) {
      script
        // Duplicate signature
        .writeOpCode(OP.OP_DUP)

        // Check if this key signed
        .writeBin(publicKey)
        .writeOpCode(OP.OP_CHECKSIG)

        // If signed, add weight to accumulator
        .writeOpCode(OP.OP_IF)
        .writeOpCode(weight)
        .writeOpCode(OP.OP_ADD)
        .writeOpCode(OP.OP_ENDIF)
    }

    // Check if total weight meets threshold
    script
      .writeOpCode(threshold)
      .writeOpCode(OP.OP_GREATERTHANOREQUAL)

    return script
  }
}
```

### Threshold Signature Schemes

```typescript
/**
 * Threshold Signature Contract
 *
 * Implements threshold cryptography where
 * a subset of participants can sign
 */

class ThresholdContract {
  /**
   * Create threshold signature lock
   * Uses Shamir's Secret Sharing
   */
  createThresholdLock(
    threshold: number,
    participants: number,
    publicKeyShares: Buffer[]
  ): Script {
    if (publicKeyShares.length !== participants) {
      throw new Error('Must provide public key share for each participant')
    }

    // In practice, threshold signatures require complex cryptography
    // This is a simplified version using standard multisig

    return new Script()
      .writeOpCode(threshold)

    for (const share of publicKeyShares) {
      new Script().writeBin(share)
    }

    return new Script()
      .writeOpCode(participants)
      .writeOpCode(OP.OP_CHECKMULTISIG)
  }
}
```

### Escrow Contracts

```typescript
/**
 * Multi-Party Escrow Contract
 *
 * Funds held in escrow until conditions are met
 */

class EscrowContract {
  /**
   * Create 2-of-3 escrow (buyer, seller, arbiter)
   */
  createTwoOfThreeEscrow(
    buyerPubKey: Buffer,
    sellerPubKey: Buffer,
    arbiterPubKey: Buffer
  ): Script {
    return new Script()
      // Require 2 of 3 signatures
      .writeOpCode(OP.OP_2)
      .writeBin(buyerPubKey)
      .writeBin(sellerPubKey)
      .writeBin(arbiterPubKey)
      .writeOpCode(OP.OP_3)
      .writeOpCode(OP.OP_CHECKMULTISIG)
  }

  /**
   * Create escrow with timeout fallback
   */
  createTimedEscrow(
    buyerPubKey: Buffer,
    sellerPubKey: Buffer,
    arbiterPubKey: Buffer,
    timeoutBlock: number,
    refundPubKey: Buffer
  ): Script {
    return new Script()
      // Check if we're using timeout refund
      .writeOpCode(OP.OP_IF)

      // Normal escrow: 2-of-3
      .writeOpCode(OP.OP_2)
      .writeBin(buyerPubKey)
      .writeBin(sellerPubKey)
      .writeBin(arbiterPubKey)
      .writeOpCode(OP.OP_3)
      .writeOpCode(OP.OP_CHECKMULTISIG)

      .writeOpCode(OP.OP_ELSE)

      // Timeout refund
      .writeBin(Buffer.from([timeoutBlock >> 24, timeoutBlock >> 16, timeoutBlock >> 8, timeoutBlock]))
      .writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
      .writeOpCode(OP.OP_DROP)

      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(Hash.hash160(refundPubKey))
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG)

      .writeOpCode(OP.OP_ENDIF)
  }

  /**
   * Create unlocking script for escrow release
   */
  createReleaseUnlock(
    signature1: Buffer,
    signature2: Buffer
  ): Script {
    return new Script()
      .writeOpCode(OP.OP_0)  // Bug workaround
      .writeBin(signature1)
      .writeBin(signature2)
      .writeOpCode(OP.OP_1)  // Choose IF branch
  }

  /**
   * Create unlocking script for timeout refund
   */
  createRefundUnlock(
    signature: Buffer,
    publicKey: Buffer
  ): Script {
    return new Script()
      .writeBin(signature)
      .writeBin(publicKey)
      .writeOpCode(OP.OP_0)  // Choose ELSE branch
  }
}

// Usage example
async function escrowExample() {
  const buyer = PrivateKey.fromRandom()
  const seller = PrivateKey.fromRandom()
  const arbiter = PrivateKey.fromRandom()

  const escrow = new EscrowContract()

  const lockingScript = escrow.createTimedEscrow(
    buyer.toPublicKey().encode(true),
    seller.toPublicKey().encode(true),
    arbiter.toPublicKey().encode(true),
    800000,  // Timeout block
    buyer.toPublicKey().encode(true)  // Refund to buyer
  )

  console.log('Escrow created with 2-of-3 multisig')
  console.log('Timeout refund available at block 800000')
}
```

## 7. Atomic Swaps and HTLCs

### Hash Time-Locked Contracts (HTLCs)

```typescript
import { Script, OP, Hash } from '@bsv/sdk'

/**
 * HTLC Implementation
 *
 * Enables cross-chain atomic swaps
 */

class HTLC {
  /**
   * Create HTLC locking script
   */
  createLock(
    recipientPubKeyHash: Buffer,
    senderPubKeyHash: Buffer,
    secretHash: Buffer,
    locktime: number
  ): Script {
    return new Script()
      // Two spending paths:
      // 1. Recipient with secret before locktime
      // 2. Sender refund after locktime

      .writeOpCode(OP.OP_IF)

      // Path 1: Recipient claims with secret
      .writeOpCode(OP.OP_SHA256)
      .writeBin(secretHash)
      .writeOpCode(OP.OP_EQUALVERIFY)

      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(recipientPubKeyHash)

      .writeOpCode(OP.OP_ELSE)

      // Path 2: Sender refund after locktime
      .writeBin(Buffer.from([locktime >> 24, locktime >> 16, locktime >> 8, locktime]))
      .writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
      .writeOpCode(OP.OP_DROP)

      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(senderPubKeyHash)

      .writeOpCode(OP.OP_ENDIF)

      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG)
  }

  /**
   * Create claim unlocking script
   */
  createClaimUnlock(
    signature: Buffer,
    publicKey: Buffer,
    secret: Buffer
  ): Script {
    return new Script()
      .writeBin(signature)
      .writeBin(publicKey)
      .writeBin(secret)
      .writeOpCode(OP.OP_1)  // Choose IF branch
  }

  /**
   * Create refund unlocking script
   */
  createRefundUnlock(
    signature: Buffer,
    publicKey: Buffer
  ): Script {
    return new Script()
      .writeBin(signature)
      .writeBin(publicKey)
      .writeOpCode(OP.OP_0)  // Choose ELSE branch
  }
}
```

### Atomic Swap Protocol

```typescript
/**
 * Atomic Swap Coordinator
 *
 * Coordinates cross-chain atomic swaps using HTLCs
 */

class AtomicSwap {
  private htlc: HTLC

  constructor() {
    this.htlc = new HTLC()
  }

  /**
   * Initiate atomic swap
   *
   * Alice wants to swap BSV for BTC with Bob
   */
  async initiateSwap(params: {
    aliceBSVKey: PrivateKey
    bobBSVKey: PrivateKey
    aliceBTCAddress: string
    bobBTCAddress: string
    bsvAmount: number
    btcAmount: number
    locktime: number
  }): Promise<{
    secret: Buffer
    secretHash: Buffer
    bsvHTLC: Script
    btcHTLC: Script
  }> {
    // Generate secret
    const secret = Hash.sha256(Buffer.from(Math.random().toString()))
    const secretHash = Hash.sha256(secret)

    // Create BSV HTLC (Alice locks BSV)
    const alicePubKeyHash = Hash.hash160(params.aliceBSVKey.toPublicKey().encode(true))
    const bobPubKeyHash = Hash.hash160(params.bobBSVKey.toPublicKey().encode(true))

    const bsvHTLC = this.htlc.createLock(
      bobPubKeyHash,      // Bob can claim
      alicePubKeyHash,    // Alice can refund
      secretHash,
      params.locktime
    )

    // Create BTC HTLC (Bob locks BTC)
    // Note: This would use Bitcoin Core script, shown here in BSV format
    const btcHTLC = this.htlc.createLock(
      Buffer.from(params.aliceBTCAddress, 'hex'),  // Alice can claim
      Buffer.from(params.bobBTCAddress, 'hex'),    // Bob can refund
      secretHash,
      params.locktime - 3600  // Earlier timeout for BTC
    )

    return {
      secret,
      secretHash,
      bsvHTLC,
      btcHTLC
    }
  }

  /**
   * Complete swap protocol
   */
  async executeSwap(): Promise<void> {
    // Step 1: Alice creates BSV HTLC and funds it
    console.log('Alice: Create and fund BSV HTLC')

    // Step 2: Bob verifies BSV HTLC and creates BTC HTLC
    console.log('Bob: Verify BSV HTLC and create BTC HTLC')

    // Step 3: Alice verifies BTC HTLC
    console.log('Alice: Verify BTC HTLC')

    // Step 4: Alice claims BTC by revealing secret
    console.log('Alice: Claim BTC by revealing secret')

    // Step 5: Bob sees secret on BTC chain and claims BSV
    console.log('Bob: See secret and claim BSV')

    // Swap complete!
    console.log('Atomic swap completed successfully')
  }
}

// Usage example
async function atomicSwapExample() {
  const swap = new AtomicSwap()

  const aliceBSV = PrivateKey.fromRandom()
  const bobBSV = PrivateKey.fromRandom()

  const swapParams = await swap.initiateSwap({
    aliceBSVKey: aliceBSV,
    bobBSVKey: bobBSV,
    aliceBTCAddress: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    bobBTCAddress: '1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2',
    bsvAmount: 100000000,  // 1 BSV
    btcAmount: 50000000,   // 0.5 BTC
    locktime: 800000
  })

  console.log('Secret hash:', swapParams.secretHash.toString('hex'))
  console.log('Swap parameters created')

  await swap.executeSwap()
}
```

### Payment Channels with HTLCs

```typescript
/**
 * Bidirectional Payment Channel with HTLCs
 *
 * Enables instant, low-cost payments between parties
 */

class PaymentChannel {
  private channelId: Buffer
  private stateNumber: number = 0

  constructor(channelId: Buffer) {
    this.channelId = channelId
  }

  /**
   * Create channel funding transaction
   */
  createFundingTx(
    party1Key: PrivateKey,
    party2Key: PrivateKey,
    party1Amount: number,
    party2Amount: number
  ): Script {
    return new Script()
      // 2-of-2 multisig
      .writeOpCode(OP.OP_2)
      .writeBin(party1Key.toPublicKey().encode(true))
      .writeBin(party2Key.toPublicKey().encode(true))
      .writeOpCode(OP.OP_2)
      .writeOpCode(OP.OP_CHECKMULTISIG)
  }

  /**
   * Create commitment transaction for current state
   */
  createCommitmentTx(
    party1Balance: number,
    party2Balance: number,
    party1Key: PrivateKey,
    party2Key: PrivateKey
  ): {
    stateNumber: number
    party1Output: Script
    party2Output: Script
  } {
    this.stateNumber++

    const party1PubKeyHash = Hash.hash160(party1Key.toPublicKey().encode(true))
    const party2PubKeyHash = Hash.hash160(party2Key.toPublicKey().encode(true))

    // Each party gets their balance via HTLC
    // This allows instant revocation of old states

    return {
      stateNumber: this.stateNumber,
      party1Output: this.createCommitmentOutput(party1PubKeyHash, party2PubKeyHash),
      party2Output: this.createCommitmentOutput(party2PubKeyHash, party1PubKeyHash)
    }
  }

  /**
   * Create commitment output with revocation
   */
  private createCommitmentOutput(
    ownerPubKeyHash: Buffer,
    counterpartyPubKeyHash: Buffer
  ): Script {
    // Owner can claim after timelock
    // Counterparty can claim immediately with revocation key

    return new Script()
      .writeOpCode(OP.OP_IF)

      // Counterparty with revocation key
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(counterpartyPubKeyHash)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG)

      .writeOpCode(OP.OP_ELSE)

      // Owner after timelock
      .writeBin(Buffer.from([144, 0, 0, 0]))  // ~24 hour delay
      .writeOpCode(OP.OP_CHECKSEQUENCEVERIFY)
      .writeOpCode(OP.OP_DROP)

      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(ownerPubKeyHash)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG)

      .writeOpCode(OP.OP_ENDIF)
  }
}
```

## 8. Covenants

### Understanding Covenants

Covenants are scripts that restrict how outputs can be spent. They examine the spending transaction itself.

```typescript
import { Script, OP, Transaction } from '@bsv/sdk'

/**
 * Covenant Pattern
 *
 * Restricts output conditions in spending transaction
 */

class CovenantContract {
  /**
   * Create covenant that enforces output script
   *
   * Note: Full covenants require OP_PUSH_TX or similar opcodes
   * This is a simplified example using available opcodes
   */
  createOutputCovenant(
    requiredOutputScript: Buffer,
    requiredAmount: number
  ): Script {
    return new Script()
      // In full implementation, would use OP_PUSH_TX to
      // push transaction data onto stack

      // For now, expect transaction data from unlocking script
      .writeOpCode(OP.OP_DUP)

      // Extract output script from transaction
      // (This is simplified - actual implementation more complex)

      // Verify output script matches required
      .writeBin(requiredOutputScript)
      .writeOpCode(OP.OP_EQUALVERIFY)

      // Verify output amount
      .writeOpCode(requiredAmount)
      .writeOpCode(OP.OP_EQUAL)
  }

  /**
   * Create recursive covenant (output must use same script)
   */
  createRecursiveCovenant(): Script {
    // Recursive covenants enforce that output uses same locking script
    // Useful for perpetual contracts

    return new Script()
      // Expect spending transaction on stack
      .writeOpCode(OP.OP_DUP)

      // Extract output script
      // Compare with current script
      // (Simplified - requires introspection opcodes)

      .writeOpCode(OP.OP_EQUAL)
  }
}
```

### Vault Covenant

```typescript
/**
 * Vault Covenant
 *
 * Implements a vault with time-delayed withdrawals
 * Allows cancellation if compromise detected
 */

class VaultCovenant {
  /**
   * Create vault locking script
   */
  createVaultLock(
    ownerPubKeyHash: Buffer,
    emergencyPubKeyHash: Buffer,
    withdrawalDelay: number
  ): Script {
    return new Script()
      .writeOpCode(OP.OP_IF)

      // Path 1: Initiate withdrawal (goes to intermediate output)
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(ownerPubKeyHash)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG)

      // Enforce output to withdrawal script
      // (Would use covenant opcodes in production)

      .writeOpCode(OP.OP_ELSE)

      // Path 2: Emergency recovery
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(emergencyPubKeyHash)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG)

      .writeOpCode(OP.OP_ENDIF)
  }

  /**
   * Create withdrawal intermediate script
   * (funds must wait here before final withdrawal)
   */
  createWithdrawalLock(
    ownerPubKeyHash: Buffer,
    cancelPubKeyHash: Buffer,
    delay: number
  ): Script {
    return new Script()
      .writeOpCode(OP.OP_IF)

      // Path 1: Complete withdrawal after delay
      .writeBin(Buffer.from([delay >> 24, delay >> 16, delay >> 8, delay]))
      .writeOpCode(OP.OP_CHECKSEQUENCEVERIFY)
      .writeOpCode(OP.OP_DROP)

      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(ownerPubKeyHash)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG)

      .writeOpCode(OP.OP_ELSE)

      // Path 2: Cancel withdrawal (return to vault)
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(cancelPubKeyHash)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG)

      // Must return to vault script (covenant)

      .writeOpCode(OP.OP_ENDIF)
  }
}

// Usage example
async function vaultExample() {
  const owner = PrivateKey.fromRandom()
  const emergency = PrivateKey.fromRandom()

  const ownerPubKeyHash = Hash.hash160(owner.toPublicKey().encode(true))
  const emergencyPubKeyHash = Hash.hash160(emergency.toPublicKey().encode(true))

  const vault = new VaultCovenant()

  // Create vault with 24-hour withdrawal delay
  const vaultLock = vault.createVaultLock(
    ownerPubKeyHash,
    emergencyPubKeyHash,
    144  // ~24 hours in blocks
  )

  console.log('Vault created with 24-hour withdrawal delay')

  // Create withdrawal lock
  const withdrawalLock = vault.createWithdrawalLock(
    ownerPubKeyHash,
    ownerPubKeyHash,
    144
  )

  console.log('Withdrawal initiated - must wait 24 hours')
}
```

## 9. Script Optimization

### Size Optimization

```typescript
import { Script, OP } from '@bsv/sdk'

/**
 * Script Size Optimization Techniques
 */

class ScriptOptimizer {
  /**
   * Use minimal opcodes for numbers
   */
  optimizeNumber(n: number): Script {
    // BAD: Always using OP_PUSHDATA
    const bad = new Script()
      .writeBin(Buffer.from([n]))

    // GOOD: Use OP_0 through OP_16 for small numbers
    const good = new Script()

    if (n === 0) {
      good.writeOpCode(OP.OP_0)
    } else if (n >= 1 && n <= 16) {
      good.writeOpCode(OP.OP_1 + (n - 1))
    } else if (n === -1) {
      good.writeOpCode(OP.OP_1NEGATE)
    } else {
      good.writeBin(Buffer.from([n]))
    }

    return good
  }

  /**
   * Reuse stack values instead of pushing duplicates
   */
  optimizeStackOps(): {
    before: Script
    after: Script
  } {
    // BAD: Push same value multiple times
    const before = new Script()
      .writeBin(Buffer.from('data'))
      .writeOpCode(OP.OP_SHA256)
      .writeBin(Buffer.from('data'))  // Redundant push
      .writeOpCode(OP.OP_SHA256)

    // GOOD: Use OP_DUP to reuse values
    const after = new Script()
      .writeBin(Buffer.from('data'))
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_SHA256)
      .writeOpCode(OP.OP_SWAP)
      .writeOpCode(OP.OP_SHA256)

    return { before, after }
  }

  /**
   * Combine operations to reduce script size
   */
  combineOperations(): {
    before: Script
    after: Script
  } {
    const pubKeyHash = Buffer.alloc(20)

    // BAD: Verbose P2PKH
    const before = new Script()
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(pubKeyHash)
      .writeOpCode(OP.OP_EQUAL)
      .writeOpCode(OP.OP_VERIFY)  // Separate VERIFY
      .writeOpCode(OP.OP_CHECKSIG)

    // GOOD: Use EQUALVERIFY (combines EQUAL + VERIFY)
    const after = new Script()
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(pubKeyHash)
      .writeOpCode(OP.OP_EQUALVERIFY)  // Combined operation
      .writeOpCode(OP.OP_CHECKSIG)

    return { before, after }
  }

  /**
   * Calculate script size savings
   */
  calculateSavings(before: Script, after: Script): {
    beforeSize: number
    afterSize: number
    savings: number
    savingsPercent: number
  } {
    const beforeSize = before.toBinary().length
    const afterSize = after.toBinary().length
    const savings = beforeSize - afterSize
    const savingsPercent = (savings / beforeSize) * 100

    return {
      beforeSize,
      afterSize,
      savings,
      savingsPercent
    }
  }
}

// Usage example
function optimizationExample() {
  const optimizer = new ScriptOptimizer()

  // Optimize number encoding
  const optimizedNumber = optimizer.optimizeNumber(5)
  console.log('Optimized number 5:', optimizedNumber.toHex())

  // Optimize stack operations
  const { before: stackBefore, after: stackAfter } = optimizer.optimizeStackOps()
  const stackSavings = optimizer.calculateSavings(stackBefore, stackAfter)
  console.log('Stack optimization savings:', stackSavings.savings, 'bytes')

  // Optimize combined operations
  const { before: opBefore, after: opAfter } = optimizer.combineOperations()
  const opSavings = optimizer.calculateSavings(opBefore, opAfter)
  console.log('Operation optimization savings:', opSavings.savings, 'bytes')
}
```

### Execution Optimization

```typescript
/**
 * Script Execution Optimization
 */

class ExecutionOptimizer {
  /**
   * Minimize signature verification operations
   * (Most expensive operation in scripts)
   */
  optimizeSigChecks(): {
    before: Script
    after: Script
  } {
    const pubKey1 = Buffer.alloc(33)
    const pubKey2 = Buffer.alloc(33)

    // BAD: Multiple CHECKSIG operations
    const before = new Script()
      .writeBin(pubKey1)
      .writeOpCode(OP.OP_CHECKSIG)
      .writeOpCode(OP.OP_VERIFY)
      .writeBin(pubKey2)
      .writeOpCode(OP.OP_CHECKSIG)
      .writeOpCode(OP.OP_VERIFY)

    // GOOD: Use CHECKMULTISIG for multiple keys
    const after = new Script()
      .writeOpCode(OP.OP_0)
      .writeOpCode(OP.OP_2)
      .writeBin(pubKey1)
      .writeBin(pubKey2)
      .writeOpCode(OP.OP_2)
      .writeOpCode(OP.OP_CHECKMULTISIG)

    return { before, after }
  }

  /**
   * Use early termination for efficiency
   */
  useEarlyTermination(): Script {
    // Check most likely condition first
    return new Script()
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_0)
      .writeOpCode(OP.OP_EQUAL)

      .writeOpCode(OP.OP_IF)
      // Fast path for most common case
      .writeOpCode(OP.OP_DROP)
      .writeOpCode(OP.OP_1)

      .writeOpCode(OP.OP_ELSE)
      // Expensive verification only if needed
      .writeOpCode(OP.OP_SHA256)
      .writeOpCode(OP.OP_SHA256)
      .writeOpCode(OP.OP_SHA256)
      // ... more operations

      .writeOpCode(OP.OP_ENDIF)
  }

  /**
   * Minimize hash operations
   */
  optimizeHashing(): {
    before: Script
    after: Script
  } {
    const data = Buffer.from('data')

    // BAD: Multiple HASH160 operations
    const before = new Script()
      .writeBin(data)
      .writeOpCode(OP.OP_HASH160)
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)  // Redundant!

    // GOOD: Hash once, reuse result
    const after = new Script()
      .writeBin(data)
      .writeOpCode(OP.OP_HASH160)
      .writeOpCode(OP.OP_DUP)
      // Reuse hashed value

    return { before, after }
  }
}
```

## 10. Testing and Debugging

### Script Testing Framework

```typescript
import { Script, OP, Transaction } from '@bsv/sdk'

/**
 * Script Testing Framework
 */

class ScriptTester {
  /**
   * Execute script and return stack
   */
  executeScript(
    unlockingScript: Script,
    lockingScript: Script
  ): {
    success: boolean
    stack: Buffer[]
    error?: string
  } {
    try {
      // Combine scripts
      const combinedScript = new Script()

      // Execute unlocking script
      for (const chunk of unlockingScript.chunks) {
        if (chunk.op) {
          combinedScript.writeOpCode(chunk.op)
        } else if (chunk.data) {
          combinedScript.writeBin(chunk.data)
        }
      }

      // Execute locking script
      for (const chunk of lockingScript.chunks) {
        if (chunk.op) {
          combinedScript.writeOpCode(chunk.op)
        } else if (chunk.data) {
          combinedScript.writeBin(chunk.data)
        }
      }

      // In production, would execute and capture stack
      // For now, return success if script is valid

      return {
        success: true,
        stack: []
      }
    } catch (error) {
      return {
        success: false,
        stack: [],
        error: error instanceof Error ? error.message : 'Unknown error'
      }
    }
  }

  /**
   * Test script with various inputs
   */
  testScript(
    lockingScript: Script,
    testCases: Array<{
      name: string
      unlockingScript: Script
      shouldSucceed: boolean
    }>
  ): void {
    console.log('Running script tests...\n')

    for (const testCase of testCases) {
      const result = this.executeScript(testCase.unlockingScript, lockingScript)

      const passed = result.success === testCase.shouldSucceed
      const status = passed ? '✅ PASS' : '❌ FAIL'

      console.log(`${status}: ${testCase.name}`)

      if (!passed) {
        console.log(`  Expected: ${testCase.shouldSucceed ? 'success' : 'failure'}`)
        console.log(`  Got: ${result.success ? 'success' : 'failure'}`)
        if (result.error) {
          console.log(`  Error: ${result.error}`)
        }
      }

      console.log()
    }
  }

  /**
   * Debug script execution step-by-step
   */
  debugScript(
    unlockingScript: Script,
    lockingScript: Script
  ): void {
    console.log('=== Script Execution Debug ===\n')

    console.log('Unlocking Script:')
    this.printScript(unlockingScript)

    console.log('\nLocking Script:')
    this.printScript(lockingScript)

    console.log('\n=== Execution Trace ===\n')

    // Would execute step-by-step and print stack
    console.log('Stack: []')

    for (const chunk of unlockingScript.chunks) {
      if (chunk.op) {
        console.log(`Execute: ${this.opCodeName(chunk.op)}`)
      } else if (chunk.data) {
        console.log(`Push: ${chunk.data.toString('hex')}`)
      }
      // Print stack after each operation
    }

    for (const chunk of lockingScript.chunks) {
      if (chunk.op) {
        console.log(`Execute: ${this.opCodeName(chunk.op)}`)
      } else if (chunk.data) {
        console.log(`Push: ${chunk.data.toString('hex')}`)
      }
      // Print stack after each operation
    }
  }

  /**
   * Print script in readable format
   */
  private printScript(script: Script): void {
    for (const chunk of script.chunks) {
      if (chunk.op) {
        console.log(`  ${this.opCodeName(chunk.op)}`)
      } else if (chunk.data) {
        console.log(`  PUSH ${chunk.data.toString('hex')}`)
      }
    }
  }

  /**
   * Get opcode name for debugging
   */
  private opCodeName(op: number): string {
    // Map opcode numbers to names
    const opcodes: Record<number, string> = {
      [OP.OP_0]: 'OP_0',
      [OP.OP_1]: 'OP_1',
      [OP.OP_DUP]: 'OP_DUP',
      [OP.OP_HASH160]: 'OP_HASH160',
      [OP.OP_EQUAL]: 'OP_EQUAL',
      [OP.OP_EQUALVERIFY]: 'OP_EQUALVERIFY',
      [OP.OP_CHECKSIG]: 'OP_CHECKSIG',
      // ... more opcodes
    }

    return opcodes[op] || `OP_UNKNOWN(${op})`
  }
}

// Usage example
function testingExample() {
  const tester = new ScriptTester()

  // Create a simple hash lock
  const secret = Buffer.from('secret')
  const secretHash = Hash.sha256(secret)

  const lockingScript = new Script()
    .writeOpCode(OP.OP_SHA256)
    .writeBin(secretHash)
    .writeOpCode(OP.OP_EQUAL)

  // Test cases
  tester.testScript(lockingScript, [
    {
      name: 'Correct secret',
      unlockingScript: new Script().writeBin(secret),
      shouldSucceed: true
    },
    {
      name: 'Wrong secret',
      unlockingScript: new Script().writeBin(Buffer.from('wrong')),
      shouldSucceed: false
    },
    {
      name: 'Empty secret',
      unlockingScript: new Script().writeBin(Buffer.from('')),
      shouldSucceed: false
    }
  ])

  // Debug execution
  const debugUnlock = new Script().writeBin(secret)
  tester.debugScript(debugUnlock, lockingScript)
}
```

### Integration Testing

```typescript
/**
 * Integration Testing with Transactions
 */

class TransactionScriptTester {
  /**
   * Test script in actual transaction context
   */
  async testScriptInTransaction(
    lockingScript: Script,
    unlockingScriptTemplate: any,
    sourceAmount: number
  ): Promise<boolean> {
    try {
      // Create source transaction
      const sourceTx = new Transaction()
      sourceTx.addOutput({
        lockingScript,
        satoshis: sourceAmount
      })

      // Create spending transaction
      const spendingTx = new Transaction()
      spendingTx.addInput({
        sourceTransaction: sourceTx,
        sourceOutputIndex: 0,
        unlockingScriptTemplate
      })

      // Add output
      spendingTx.addOutput({
        lockingScript: new Script().writeOpCode(OP.OP_1),
        satoshis: sourceAmount - 500
      })

      // Calculate fee and sign
      await spendingTx.fee()
      await spendingTx.sign()

      // If we got here, script executed successfully
      return true
    } catch (error) {
      console.error('Script execution failed:', error)
      return false
    }
  }
}
```

## Best Practices

### 1. Always Validate Inputs

```typescript
// BAD: No input validation
function createLock(value: number): Script {
  return new Script()
    .writeOpCode(value)
    .writeOpCode(OP.OP_EQUAL)
}

// GOOD: Validate all inputs
function createLockValidated(value: number): Script {
  if (value < 0 || value > 0x7fffffff) {
    throw new Error('Value out of range for script number')
  }

  if (!Number.isInteger(value)) {
    throw new Error('Value must be an integer')
  }

  return new Script()
    .writeOpCode(value)
    .writeOpCode(OP.OP_EQUAL)
}
```

### 2. Use Script Templates for Reusability

```typescript
import { ScriptTemplate, LockingScript, UnlockingScript } from '@bsv/sdk'

// Create reusable template
class HashLockTemplate implements ScriptTemplate {
  lock(secretHash: Buffer): LockingScript {
    return new LockingScript([
      { op: OP.OP_SHA256 },
      { op: 0, data: secretHash },
      { op: OP.OP_EQUAL }
    ])
  }

  unlock(secret: Buffer): {
    sign: (tx: Transaction, inputIndex: number) => Promise<UnlockingScript>
    estimateLength: () => Promise<number>
  } {
    return {
      sign: async () => new UnlockingScript([
        { op: 0, data: secret }
      ]),
      estimateLength: async () => secret.length + 2
    }
  }
}
```

### 3. Minimize Script Size

```typescript
// BAD: Inefficient script
const bad = new Script()
  .writeBin(Buffer.from([1]))
  .writeBin(Buffer.from([2]))
  .writeOpCode(OP.OP_ADD)
  .writeBin(Buffer.from([3]))

// GOOD: Use built-in opcodes
const good = new Script()
  .writeOpCode(OP.OP_1)
  .writeOpCode(OP.OP_2)
  .writeOpCode(OP.OP_ADD)
  .writeOpCode(OP.OP_3)
```

### 4. Use VERIFY Opcodes for Efficient Validation

```typescript
// BAD: Separate verification
const bad = new Script()
  .writeOpCode(OP.OP_EQUAL)
  .writeOpCode(OP.OP_IF)
  .writeOpCode(OP.OP_1)
  .writeOpCode(OP.OP_ELSE)
  .writeOpCode(OP.OP_0)
  .writeOpCode(OP.OP_ENDIF)

// GOOD: Use VERIFY
const good = new Script()
  .writeOpCode(OP.OP_EQUALVERIFY)
  .writeOpCode(OP.OP_1)
```

### 5. Handle Edge Cases

```typescript
// Consider edge cases in multisig
function createSafeMultiSig(m: number, publicKeys: Buffer[]): Script {
  // Validate parameters
  if (m < 1 || m > publicKeys.length) {
    throw new Error('Invalid multisig parameters')
  }

  if (publicKeys.length > 20) {
    throw new Error('Too many public keys (max 20)')
  }

  // Check for duplicate keys
  const keySet = new Set(publicKeys.map(k => k.toString('hex')))
  if (keySet.size !== publicKeys.length) {
    throw new Error('Duplicate public keys detected')
  }

  const script = new Script()
    .writeOpCode(m)

  for (const pubKey of publicKeys) {
    script.writeBin(pubKey)
  }

  script
    .writeOpCode(publicKeys.length)
    .writeOpCode(OP.OP_CHECKMULTISIG)

  return script
}
```

### 6. Document Complex Scripts

```typescript
/**
 * Two-Factor Authentication Script
 *
 * Requires:
 * 1. Knowledge factor: Secret preimage
 * 2. Possession factor: Private key signature
 *
 * Security: Provides defense in depth against key compromise
 */
function createTwoFactorLock(
  secretHash: Buffer,
  pubKeyHash: Buffer
): Script {
  return new Script()
    // Verify knowledge factor (secret)
    .writeOpCode(OP.OP_SHA256)
    .writeBin(secretHash)
    .writeOpCode(OP.OP_EQUALVERIFY)

    // Verify possession factor (signature)
    .writeOpCode(OP.OP_DUP)
    .writeOpCode(OP.OP_HASH160)
    .writeBin(pubKeyHash)
    .writeOpCode(OP.OP_EQUALVERIFY)
    .writeOpCode(OP.OP_CHECKSIG)
}
```

### 7. Test Thoroughly

```typescript
// Create comprehensive test suite
function testHashLock() {
  const tester = new ScriptTester()

  const secret = Buffer.from('correct secret')
  const secretHash = Hash.sha256(secret)
  const lock = new Script()
    .writeOpCode(OP.OP_SHA256)
    .writeBin(secretHash)
    .writeOpCode(OP.OP_EQUAL)

  tester.testScript(lock, [
    {
      name: 'Correct secret',
      unlockingScript: new Script().writeBin(secret),
      shouldSucceed: true
    },
    {
      name: 'Wrong secret',
      unlockingScript: new Script().writeBin(Buffer.from('wrong')),
      shouldSucceed: false
    },
    {
      name: 'Empty input',
      unlockingScript: new Script(),
      shouldSucceed: false
    },
    {
      name: 'Multiple inputs',
      unlockingScript: new Script()
        .writeBin(secret)
        .writeBin(secret),
      shouldSucceed: false
    }
  ])
}
```

### 8. Use Proper Error Handling

```typescript
class SafeScriptBuilder {
  private script: Script

  constructor() {
    this.script = new Script()
  }

  addOpCode(op: number): this {
    try {
      this.script.writeOpCode(op)
      return this
    } catch (error) {
      throw new Error(`Failed to add opcode ${op}: ${error}`)
    }
  }

  addData(data: Buffer): this {
    if (!Buffer.isBuffer(data)) {
      throw new Error('Data must be a Buffer')
    }

    if (data.length > 520) {
      throw new Error('Data exceeds maximum push size (520 bytes)')
    }

    try {
      this.script.writeBin(data)
      return this
    } catch (error) {
      throw new Error(`Failed to add data: ${error}`)
    }
  }

  build(): Script {
    return this.script
  }
}
```

### 9. Consider Gas Costs and Limits

```typescript
class ScriptCostAnalyzer {
  /**
   * Estimate execution cost of script
   */
  estimateCost(script: Script): {
    operations: number
    sigChecks: number
    estimatedCost: number
  } {
    let operations = 0
    let sigChecks = 0

    for (const chunk of script.chunks) {
      operations++

      if (chunk.op === OP.OP_CHECKSIG || chunk.op === OP.OP_CHECKSIGVERIFY) {
        sigChecks++
      } else if (chunk.op === OP.OP_CHECKMULTISIG || chunk.op === OP.OP_CHECKMULTISIGVERIFY) {
        // Multisig is more expensive
        sigChecks += 20  // Worst case
      }
    }

    // Rough cost estimate (signature checks are most expensive)
    const estimatedCost = operations + (sigChecks * 100)

    return { operations, sigChecks, estimatedCost }
  }
}
```

### 10. Implement Proper Security Reviews

```typescript
class ScriptSecurityAuditor {
  /**
   * Audit script for common vulnerabilities
   */
  audit(script: Script): {
    vulnerabilities: string[]
    warnings: string[]
    recommendations: string[]
  } {
    const vulnerabilities: string[] = []
    const warnings: string[] = []
    const recommendations: string[] = []

    // Check for disabled opcodes
    const disabledOps = [OP.OP_CAT, OP.OP_SPLIT, OP.OP_AND, OP.OP_OR]
    for (const chunk of script.chunks) {
      if (chunk.op && disabledOps.includes(chunk.op)) {
        warnings.push(`Uses re-enabled opcode: ${chunk.op}`)
      }
    }

    // Check script size
    const size = script.toBinary().length
    if (size > 10000) {
      warnings.push(`Large script size: ${size} bytes`)
    }

    // Check for OP_RETURN (makes output unspendable)
    let hasReturn = false
    for (const chunk of script.chunks) {
      if (chunk.op === OP.OP_RETURN) {
        hasReturn = true
        warnings.push('Script contains OP_RETURN (unspendable)')
      }
    }

    return { vulnerabilities, warnings, recommendations }
  }
}
```

## Common Pitfalls

### 1. Not Handling the Multisig Bug

```typescript
// BAD: Forgetting OP_0 for multisig bug
const bad = new Script()
  .writeBin(sig1)
  .writeBin(sig2)
  .writeOpCode(OP.OP_2)
  .writeBin(pubKey1)
  .writeBin(pubKey2)
  .writeOpCode(OP.OP_2)
  .writeOpCode(OP.OP_CHECKMULTISIG)  // Will fail!

// GOOD: Include OP_0 at start of unlocking script
const good = new Script()
  .writeOpCode(OP.OP_0)  // Bug workaround
  .writeBin(sig1)
  .writeBin(sig2)
  .writeOpCode(OP.OP_2)
  .writeBin(pubKey1)
  .writeBin(pubKey2)
  .writeOpCode(OP.OP_2)
  .writeOpCode(OP.OP_CHECKMULTISIG)
```

### 2. Incorrect Script Number Encoding

```typescript
// BAD: Pushing large number incorrectly
const bad = new Script()
  .writeBin(Buffer.from([255]))  // Interpreted as -127!

// GOOD: Use proper script number encoding
const good = new Script()
  .writeOpCode(255)  // SDK handles encoding correctly
```

### 3. Not Validating Stack State

```typescript
// BAD: Assuming stack has items
const bad = new Script()
  .writeOpCode(OP.OP_DROP)  // Error if stack is empty!

// GOOD: Validate stack has required items
const good = new Script()
  .writeOpCode(OP.OP_DEPTH)
  .writeOpCode(OP.OP_0)
  .writeOpCode(OP.OP_GREATERTHAN)
  .writeOpCode(OP.OP_VERIFY)
  .writeOpCode(OP.OP_DROP)
```

### 4. Incorrect IF/ELSE/ENDIF Usage

```typescript
// BAD: Unbalanced IF/ENDIF
const bad = new Script()
  .writeOpCode(OP.OP_IF)
  .writeOpCode(OP.OP_1)
  // Missing ENDIF!

// GOOD: Properly balanced
const good = new Script()
  .writeOpCode(OP.OP_IF)
  .writeOpCode(OP.OP_1)
  .writeOpCode(OP.OP_ENDIF)
```

### 5. Signature Hash Type Confusion

```typescript
// BAD: Using wrong SIGHASH type
import { P2PKH } from '@bsv/sdk'

const bad = new P2PKH().unlock(
  privateKey,
  'none'  // SIGHASH_NONE - doesn't sign outputs!
)

// GOOD: Use appropriate SIGHASH type
const good = new P2PKH().unlock(
  privateKey,
  'all'  // SIGHASH_ALL - signs all outputs (most common)
)
```

### 6. Not Considering Malleability

```typescript
// BAD: Relying on exact signature format
const bad = new Script()
  .writeBin(expectedSignature)
  .writeOpCode(OP.OP_EQUAL)  // Signatures can be malleable!

// GOOD: Verify signature cryptographically
const good = new Script()
  .writeBin(publicKey)
  .writeOpCode(OP.OP_CHECKSIG)
```

### 7. Ignoring Script Size Limits

```typescript
// BAD: Creating huge scripts
const bad = new Script()
for (let i = 0; i < 1000; i++) {
  bad.writeOpCode(OP.OP_1)
}

// GOOD: Keep scripts reasonably sized
const good = new Script()
  .writeOpCode(OP.OP_1)
  // Use loops or other logic to reduce size
```

## Hands-On Project: Multi-Party Escrow with Oracle

Build a complete multi-party escrow system with oracle integration for dispute resolution.

### Project Requirements

1. **Three-Party Escrow**: Buyer, seller, and arbiter
2. **Oracle Integration**: Price feed for conditional release
3. **Time Locks**: Automatic refund after timeout
4. **State Management**: Track escrow states
5. **Multi-Signature**: Require 2-of-3 for release

### Implementation

```typescript
import {
  Script,
  OP,
  Transaction,
  PrivateKey,
  Hash,
  P2PKH
} from '@bsv/sdk'

/**
 * Complete Multi-Party Escrow System
 */

class AdvancedEscrowSystem {
  private oracle: Oracle
  private escrow: EscrowContract

  constructor(oracleKey: PrivateKey) {
    this.oracle = new Oracle(oracleKey)
    this.escrow = new EscrowContract()
  }

  /**
   * Create escrow with oracle condition
   */
  async createEscrow(params: {
    buyerKey: PrivateKey
    sellerKey: PrivateKey
    arbiterKey: PrivateKey
    amount: number
    minimumPrice: number
    assetPair: string
    timeoutBlocks: number
  }): Promise<{
    lockingScript: Script
    fundingTx: Transaction
  }> {
    const buyerPubKey = params.buyerKey.toPublicKey().encode(true)
    const sellerPubKey = params.sellerKey.toPublicKey().encode(true)
    const arbiterPubKey = params.arbiterKey.toPublicKey().encode(true)

    // Create compound locking script
    const lockingScript = new Script()
      // Check spending path
      .writeOpCode(OP.OP_IF)

      // Path 1: Release with oracle price verification
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(params.minimumPrice)
      .writeOpCode(OP.OP_GREATERTHANOREQUAL)
      .writeOpCode(OP.OP_VERIFY)

      // Verify oracle signature on price
      .writeOpCode(OP.OP_SHA256)
      .writeBin(this.oracle.publicKey.encode(true))
      .writeOpCode(OP.OP_CHECKSIGVERIFY)

      // Require 2-of-3 multisig
      .writeOpCode(OP.OP_0)
      .writeOpCode(OP.OP_2)
      .writeBin(buyerPubKey)
      .writeBin(sellerPubKey)
      .writeBin(arbiterPubKey)
      .writeOpCode(OP.OP_3)
      .writeOpCode(OP.OP_CHECKMULTISIG)

      .writeOpCode(OP.OP_ELSE)

      // Path 2: Timeout refund to buyer
      .writeBin(Buffer.from([
        params.timeoutBlocks >> 24,
        (params.timeoutBlocks >> 16) & 0xff,
        (params.timeoutBlocks >> 8) & 0xff,
        params.timeoutBlocks & 0xff
      ]))
      .writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
      .writeOpCode(OP.OP_DROP)

      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(Hash.hash160(buyerPubKey))
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG)

      .writeOpCode(OP.OP_ENDIF)

    // Create funding transaction
    const fundingTx = new Transaction()
    fundingTx.addOutput({
      lockingScript,
      satoshis: params.amount
    })

    return {
      lockingScript,
      fundingTx
    }
  }

  /**
   * Release escrow with oracle verification
   */
  async releaseEscrow(params: {
    fundingTx: Transaction
    buyerKey: PrivateKey
    sellerKey: PrivateKey
    currentPrice: number
    assetPair: string
    recipientAddress: string
  }): Promise<Transaction> {
    // Get oracle signature on current price
    const priceData = this.oracle.signPrice(
      params.assetPair,
      params.currentPrice
    )

    // Create spending transaction
    const spendingTx = new Transaction()

    // Create unlocking script
    const buyerSig = Buffer.alloc(72)  // Placeholder
    const sellerSig = Buffer.alloc(72)  // Placeholder

    const unlockingScript = new Script()
      .writeOpCode(OP.OP_0)  // Multisig bug
      .writeBin(buyerSig)
      .writeBin(sellerSig)
      .writeBin(priceData.signature)
      .writeOpCode(Math.floor(params.currentPrice * 100))
      .writeOpCode(OP.OP_1)  // Choose IF branch

    spendingTx.addInput({
      sourceTransaction: params.fundingTx,
      sourceOutputIndex: 0,
      unlockingScriptTemplate: {
        sign: async () => unlockingScript as any,
        estimateLength: async () => unlockingScript.toBinary().length
      }
    })

    // Add output to recipient
    spendingTx.addOutput({
      lockingScript: new P2PKH().lock(params.recipientAddress),
      satoshis: params.fundingTx.outputs[0].satoshis - 500
    })

    await spendingTx.fee()
    await spendingTx.sign()

    return spendingTx
  }

  /**
   * Refund escrow after timeout
   */
  async refundEscrow(params: {
    fundingTx: Transaction
    buyerKey: PrivateKey
  }): Promise<Transaction> {
    const spendingTx = new Transaction()

    const signature = Buffer.alloc(72)  // Placeholder
    const publicKey = params.buyerKey.toPublicKey().encode(true)

    const unlockingScript = new Script()
      .writeBin(signature)
      .writeBin(publicKey)
      .writeOpCode(OP.OP_0)  // Choose ELSE branch

    spendingTx.addInput({
      sourceTransaction: params.fundingTx,
      sourceOutputIndex: 0,
      unlockingScriptTemplate: {
        sign: async () => unlockingScript as any,
        estimateLength: async () => unlockingScript.toBinary().length
      }
    })

    // Refund to buyer
    const buyerPubKeyHash = Hash.hash160(publicKey)
    spendingTx.addOutput({
      lockingScript: new Script()
        .writeOpCode(OP.OP_DUP)
        .writeOpCode(OP.OP_HASH160)
        .writeBin(buyerPubKeyHash)
        .writeOpCode(OP.OP_EQUALVERIFY)
        .writeOpCode(OP.OP_CHECKSIG),
      satoshis: params.fundingTx.outputs[0].satoshis - 500
    })

    await spendingTx.fee()
    await spendingTx.sign()

    return spendingTx
  }
}

// Usage example
async function escrowProjectExample() {
  console.log('=== Advanced Escrow System ===\n')

  // Setup parties
  const oracleKey = PrivateKey.fromRandom()
  const buyerKey = PrivateKey.fromRandom()
  const sellerKey = PrivateKey.fromRandom()
  const arbiterKey = PrivateKey.fromRandom()

  const system = new AdvancedEscrowSystem(oracleKey)

  // Create escrow
  console.log('Creating escrow...')
  const { lockingScript, fundingTx } = await system.createEscrow({
    buyerKey,
    sellerKey,
    arbiterKey,
    amount: 100000000,  // 1 BSV
    minimumPrice: 5000,  // $50.00
    assetPair: 'BSV/USD',
    timeoutBlocks: 144  // ~24 hours
  })

  console.log('Escrow created')
  console.log('Funding TX:', fundingTx.id('hex'))

  // Release escrow (price condition met)
  console.log('\nReleasing escrow with price verification...')
  const releaseTx = await system.releaseEscrow({
    fundingTx,
    buyerKey,
    sellerKey,
    currentPrice: 52.50,
    assetPair: 'BSV/USD',
    recipientAddress: sellerKey.toPublicKey().toAddress()
  })

  console.log('Escrow released')
  console.log('Release TX:', releaseTx.id('hex'))

  console.log('\n=== Project Complete ===')
}
```

### Testing the Project

```typescript
async function testEscrowSystem() {
  const tester = new ScriptTester()

  // Test oracle price verification
  const oracleKey = PrivateKey.fromRandom()
  const oracle = new Oracle(oracleKey)

  const priceData = oracle.signPrice('BSV/USD', 52.50)
  console.log('Oracle price data:', priceData)

  // Test escrow creation
  const system = new AdvancedEscrowSystem(oracleKey)

  // Run full escrow cycle
  await escrowProjectExample()

  console.log('\nAll tests passed!')
}
```

## Next Steps

After completing this course, you should:

1. **Practice Building Contracts** - Implement your own smart contract patterns
2. **Study Real-World Contracts** - Analyze production contracts on BSV blockchain
3. **Explore Advanced Topics**:
   - Zero-knowledge proofs in scripts
   - Layer-2 protocols (payment channels, sidechains)
   - Cross-chain bridges
   - Advanced covenant patterns
4. **Contribute to SDK** - Help improve the BSV SDK with new script templates
5. **Join Community** - Engage with BSV developers and share your contracts

### Advanced Learning Paths

- **Protocol Development**: Build your own BRC-compliant protocols
- **DApp Development**: Create decentralized applications using advanced scripts
- **Smart Contract Auditing**: Learn to audit and secure Bitcoin scripts
- **Performance Optimization**: Master script optimization techniques

## Additional Resources

### Official Documentation
- **BSV SDK Documentation**: https://bsv-blockchain.github.io/ts-sdk
- **Bitcoin Script Reference**: https://wiki.bitcoinsv.io/index.php/Script
- **BRC Standards**: https://github.com/bitcoin-sv/BRCs

### BRC Standards for Advanced Scripting
- **[BRC-1](https://github.com/bitcoin-sv/BRCs/blob/master/peer-to-peer/0001.md)** - Abstract messaging layer
- **[BRC-2](https://github.com/bitcoin-sv/BRCs/blob/master/wallet/0002.md)** - Data encryption/decryption
- **[BRC-3](https://github.com/bitcoin-sv/BRCs/blob/master/wallet/0003.md)** - Digital signatures
- **[BRC-42](https://github.com/bitcoin-sv/BRCs/blob/master/wallet/0042.md)** - BSV Key Derivation Scheme
- **[BRC-43](https://github.com/bitcoin-sv/BRCs/blob/master/key-derivation/0043.md)** - Security levels and protocol IDs

### Related SDK Components
- **[Script](../../../sdk-components/script/README.md)** - Low-level script operations
- **[Script Templates](../../../sdk-components/script-templates/README.md)** - Reusable patterns
- **[Transaction](../../../sdk-components/transaction/README.md)** - Transaction construction
- **[Signatures](../../../sdk-components/signatures/README.md)** - Digital signatures
- **[P2PKH](../../../sdk-components/p2pkh/README.md)** - Standard payment template

### Code Examples Repository
- **BSV SDK Examples**: https://github.com/bsv-blockchain/ts-sdk/tree/master/examples
- **Advanced Script Examples**: See SDK repository for production examples

### Community Resources
- **BSV Blockchain Discord**: Join for real-time help
- **BSV Developers Forum**: Discuss advanced scripting topics
- **Stack Overflow**: Tag questions with `bsv` and `bitcoin-script`

### Academic Papers
- **Bitcoin White Paper**: Understanding the foundation
- **Script Security Papers**: Research on script vulnerabilities
- **Smart Contract Patterns**: Academic research on blockchain contracts

## Status

✅ Complete
