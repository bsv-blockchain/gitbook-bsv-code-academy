# BSV Primitives

## Overview

This module provides a deep dive into the core primitives of the BSV blockchain. Understanding these building blocks is essential for creating sophisticated applications and optimizing transaction efficiency.

## Learning Objectives

By the end of this module, you will understand:
- Transaction structure and serialization formats
- Input and output types and their uses
- Script opcodes and operations in detail
- Signature hash types (SIGHASH) and their applications
- Locktime and sequence numbers
- Advanced UTXO management strategies

## Transaction Structure Deep Dive

### Binary Format

Transactions are serialized in a specific binary format:

```
[Version (4 bytes)]
[Input Count (VarInt)]
[Inputs...]
[Output Count (VarInt)]
[Outputs...]
[Locktime (4 bytes)]
```

### Serialization Example

```typescript
import { Transaction } from '@bsv/sdk'

const tx = new Transaction()
// ... add inputs and outputs

// Serialize to different formats
const hex = tx.toHex()           // Hexadecimal string
const buffer = tx.toBuffer()     // Binary buffer
const binary = tx.toBinary()     // Binary array

console.log('Hex:', hex)
console.log('Size:', buffer.length, 'bytes')
```

### Transaction ID Calculation

The transaction ID (txid) is the double SHA-256 hash of the serialized transaction:

```typescript
function calculateTxid(tx: Transaction): string {
  const txBuffer = tx.toBuffer()

  // First SHA-256
  const hash1 = crypto.createHash('sha256')
    .update(txBuffer)
    .digest()

  // Second SHA-256
  const hash2 = crypto.createHash('sha256')
    .update(hash1)
    .digest()

  // Reverse for display (little-endian)
  return hash2.reverse().toString('hex')
}

// SDK method
const txid = tx.id('hex')
```

## Transaction Inputs

### Input Structure

```typescript
interface TransactionInput {
  sourceTransaction: string    // Previous txid
  sourceOutputIndex: number    // Output index (vout)
  unlockingScript: Script      // scriptSig
  sequence: number             // Sequence number
}
```

### Sequence Numbers

Sequence numbers enable advanced features:

```typescript
// Maximum sequence (default, final)
const SEQUENCE_FINAL = 0xffffffff

// Enable replace-by-fee
const SEQUENCE_RBF = 0xfffffffd

// Relative locktime (512 seconds)
const SEQUENCE_LOCKTIME = 512

tx.addInput({
  sourceTransaction: utxo.txid,
  sourceOutputIndex: utxo.vout,
  unlockingScript: unlockScript,
  sequence: SEQUENCE_FINAL
})
```

### Input Weight and Size

Each input adds ~148 bytes to transaction size:
- Previous txid: 32 bytes
- Output index: 4 bytes
- Script length: 1 byte
- Unlocking script: ~107 bytes (P2PKH signature + pubkey)
- Sequence: 4 bytes

## Transaction Outputs

### Output Structure

```typescript
interface TransactionOutput {
  satoshis: number            // Amount in satoshis
  lockingScript: Script       // scriptPubKey
}
```

### Output Types

#### 1. P2PKH (Pay-to-Public-Key-Hash)

Standard payment output:
```typescript
import { P2PKH } from '@bsv/sdk'

const output = {
  satoshis: 10000,
  lockingScript: new P2PKH().lock(address)
}

// Script: OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
```

#### 2. P2PK (Pay-to-Public-Key)

Direct public key payment:
```typescript
const output = {
  satoshis: 10000,
  lockingScript: Script.fromASM(`${publicKey.toHex()} OP_CHECKSIG`)
}
```

#### 3. OP_RETURN (Data Output)

Data storage on-chain:
```typescript
const data = 'Hello, BSV!'
const output = {
  satoshis: 0,
  lockingScript: Script.fromASM(`OP_FALSE OP_RETURN ${Buffer.from(data).toString('hex')}`)
}
```

#### 4. Multi-Signature

M-of-N signature requirement:
```typescript
// 2-of-3 multisig
const output = {
  satoshis: 10000,
  lockingScript: Script.fromASM(`
    OP_2
    ${pubKey1.toHex()}
    ${pubKey2.toHex()}
    ${pubKey3.toHex()}
    OP_3
    OP_CHECKMULTISIG
  `)
}
```

### Output Weight and Size

Each output adds ~34 bytes:
- Amount: 8 bytes
- Script length: 1 byte
- Locking script: ~25 bytes (P2PKH)

### Dust Limit

Outputs must be above the dust threshold:

```typescript
const DUST_LIMIT = 546 // satoshis

function isDust(satoshis: number): boolean {
  return satoshis < DUST_LIMIT
}

// Don't create dust outputs
if (changeAmount > DUST_LIMIT) {
  tx.addOutput({
    satoshis: changeAmount,
    lockingScript: new P2PKH().lock(myAddress)
  })
} else {
  // Add to fee instead
  console.log('Change below dust limit, added to fee')
}
```

## Script Opcodes

### Categories of Opcodes

#### 1. Constants
```
OP_0, OP_1, OP_2, ... OP_16
OP_FALSE = OP_0
OP_TRUE = OP_1
```

#### 2. Flow Control
```
OP_IF, OP_ELSE, OP_ENDIF
OP_VERIFY
OP_RETURN
```

#### 3. Stack Operations
```
OP_DUP        // Duplicate top item
OP_DROP       // Remove top item
OP_SWAP       // Swap top two items
OP_ROT        // Rotate top three items
```

#### 4. Arithmetic
```
OP_ADD, OP_SUB
OP_MUL, OP_DIV
OP_MOD
```

#### 5. Cryptographic
```
OP_SHA256      // SHA-256 hash
OP_HASH160     // SHA-256 then RIPEMD-160
OP_CHECKSIG    // Verify signature
OP_CHECKMULTISIG  // Verify M-of-N signatures
```

#### 6. Comparison
```
OP_EQUAL       // Check equality
OP_EQUALVERIFY // Equal + verify
OP_GREATERTHAN, OP_LESSTHAN
```

### Script Example Walkthrough

P2PKH script execution:

```
Unlocking: <signature> <publicKey>
Locking:   OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG

Combined: <sig> <pubKey> OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG

Execution:
Stack: []
  Push <sig>
Stack: [sig]
  Push <pubKey>
Stack: [sig, pubKey]
  OP_DUP
Stack: [sig, pubKey, pubKey]
  OP_HASH160
Stack: [sig, pubKey, hash(pubKey)]
  Push <pubKeyHash>
Stack: [sig, pubKey, hash(pubKey), pubKeyHash]
  OP_EQUALVERIFY (checks hashes match)
Stack: [sig, pubKey]
  OP_CHECKSIG (verifies signature)
Stack: [true]

Result: true (valid)
```

## Signature Hash Types (SIGHASH)

SIGHASH flags control what parts of a transaction are signed:

### SIGHASH Types

```typescript
enum SIGHASH {
  ALL = 0x01,           // Sign all inputs and outputs
  NONE = 0x02,          // Sign all inputs, no outputs
  SINGLE = 0x03,        // Sign all inputs, one output
  ANYONECANPAY = 0x80,  // Sign one input (can combine with above)
  FORKID = 0x40         // BSV fork identifier
}
```

### Common Combinations

#### SIGHASH_ALL (0x41)
Default. Signs entire transaction:
```typescript
// Most common - signs everything
const sighash = SIGHASH.ALL | SIGHASH.FORKID // 0x41
```

#### SIGHASH_NONE | ANYONECANPAY (0xC2)
Signs only one input, allows any outputs:
```typescript
// "Blank check" - anyone can add outputs
const sighash = SIGHASH.NONE | SIGHASH.ANYONECANPAY | SIGHASH.FORKID
```

#### SIGHASH_SINGLE | ANYONECANPAY (0xC3)
Signs one input to one output:
```typescript
// Crowdfunding pattern
const sighash = SIGHASH.SINGLE | SIGHASH.ANYONECANPAY | SIGHASH.FORKID
```

### Use Cases

**SIGHASH_ALL**: Standard payments (signs everything)
**SIGHASH_NONE**: Let others decide outputs
**SIGHASH_SINGLE**: Pair specific input with specific output
**ANYONECANPAY**: Allow others to add inputs (crowdfunding)

## Locktime and Timelocks

### Transaction Locktime

Controls when transaction can be mined:

```typescript
const tx = new Transaction()
tx.lockTime = 0  // Can mine immediately (default)

// Locktime by block height
tx.lockTime = 800000  // Can't mine until block 800,000

// Locktime by timestamp (if >= 500000000)
tx.lockTime = Math.floor(Date.now() / 1000) + 3600  // 1 hour from now
```

### Sequence-based Relative Locktime

```typescript
// Wait 100 blocks after input was created
const input = {
  sourceTransaction: utxo.txid,
  sourceOutputIndex: utxo.vout,
  unlockingScript: script,
  sequence: 100  // Relative locktime: 100 blocks
}
```

### Use Cases

- **Escrow**: Funds locked until specific time
- **Payment Channels**: Update states with locktime
- **Delayed Transactions**: Schedule future payments

## Advanced UTXO Management

### UTXO Selection Strategies

#### 1. First-Fit Strategy
```typescript
function selectUTXOsFirstFit(utxos: UTXO[], target: number): UTXO[] {
  const selected: UTXO[] = []
  let total = 0

  for (const utxo of utxos) {
    selected.push(utxo)
    total += utxo.satoshis

    if (total >= target) break
  }

  return selected
}
```

#### 2. Largest-First Strategy
```typescript
function selectUTXOsLargest(utxos: UTXO[], target: number): UTXO[] {
  const sorted = [...utxos].sort((a, b) => b.satoshis - a.satoshis)
  return selectUTXOsFirstFit(sorted, target)
}
```

#### 3. Smallest-First Strategy
```typescript
function selectUTXOsSmallest(utxos: UTXO[], target: number): UTXO[] {
  const sorted = [...utxos].sort((a, b) => a.satoshis - b.satoshis)
  return selectUTXOsFirstFit(sorted, target)
}
```

#### 4. Branch-and-Bound (Optimal)
```typescript
function selectUTXOsOptimal(utxos: UTXO[], target: number): UTXO[] {
  // Find exact match or minimize change
  // Implementation of branch-and-bound algorithm
  // Used by Bitcoin Core
}
```

### UTXO Consolidation

Combine many small UTXOs into fewer larger ones:

```typescript
async function consolidateUTXOs(
  privateKey: PrivateKey,
  utxos: UTXO[]
): Promise<Transaction> {
  const tx = new Transaction()
  const address = privateKey.toPublicKey().toAddress()

  // Add all UTXOs as inputs
  for (const utxo of utxos) {
    await tx.addInput({
      sourceTransaction: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(privateKey)
    })
  }

  // Calculate total
  const total = utxos.reduce((sum, u) => sum + u.satoshis, 0)

  // Estimate fee
  const fee = await tx.getFee(new SatoshisPerKilobyte(50))

  // Single output with all funds minus fee
  tx.addOutput({
    satoshis: total - fee,
    lockingScript: new P2PKH().lock(address)
  })

  await tx.sign()
  return tx
}
```

### UTXO Fragmentation

Split one UTXO into many:

```typescript
async function fragmentUTXO(
  privateKey: PrivateKey,
  utxo: UTXO,
  count: number
): Promise<Transaction> {
  const tx = new Transaction()
  const address = privateKey.toPublicKey().toAddress()

  // Add input
  await tx.addInput({
    sourceTransaction: utxo.txid,
    sourceOutputIndex: utxo.vout,
    unlockingScriptTemplate: new P2PKH().unlock(privateKey)
  })

  // Estimate fee
  const fee = 1000 // Approximate
  const perOutput = Math.floor((utxo.satoshis - fee) / count)

  // Create multiple outputs
  for (let i = 0; i < count; i++) {
    tx.addOutput({
      satoshis: perOutput,
      lockingScript: new P2PKH().lock(address)
    })
  }

  await tx.sign()
  return tx
}
```

## Transaction Size Optimization

### Calculating Transaction Size

```typescript
function estimateTransactionSize(
  inputCount: number,
  outputCount: number
): number {
  const overhead = 10  // Version + locktime
  const inputSize = 148  // Average P2PKH input
  const outputSize = 34  // Average P2PKH output

  return overhead + (inputCount * inputSize) + (outputCount * outputSize)
}

// Example: 2 inputs, 2 outputs
const size = estimateTransactionSize(2, 2)
// = 10 + (2 * 148) + (2 * 34) = 374 bytes
```

### Size Optimization Techniques

1. **Use fewer inputs** - Consolidate UTXOs in advance
2. **Use P2PK instead of P2PKH** - Saves ~25 bytes per output
3. **Batch payments** - Multiple outputs in one transaction
4. **Avoid dust** - Don't create tiny outputs

## Best Practices

✅ **UTXO Management**
- Consolidate small UTXOs during low-fee periods
- Maintain mix of UTXO sizes for flexibility
- Monitor UTXO count to avoid excessive inputs

✅ **Script Usage**
- Use standard scripts when possible (P2PKH)
- Test custom scripts thoroughly
- Minimize script size for efficiency

✅ **Signatures**
- Use SIGHASH_ALL for standard payments
- Understand implications of other SIGHASH types
- Always include FORKID for BSV

✅ **Locktime**
- Use locktime for time-based conditions
- Consider sequence numbers for relative time
- Test timelocked transactions on testnet

## Practice Exercises

1. **Analyze Transaction**: Deserialize a raw transaction and examine its structure
2. **Custom Script**: Create a script requiring a specific number and signature
3. **UTXO Selection**: Implement an optimal UTXO selection algorithm
4. **Multi-sig**: Create a 2-of-3 multisignature transaction
5. **Timelocked Payment**: Create a transaction locked until a future block

## Related Components

- [Transaction](../../../sdk-components/transaction/README.md)
- [Transaction Input](../../../sdk-components/transaction-input/README.md)
- [Transaction Output](../../../sdk-components/transaction-output/README.md)
- [Script](../../../sdk-components/script/README.md)
- [UTXO Management](../../../sdk-components/utxo-management/README.md)

## Next Module

Continue to: [Transaction Building](../transaction-building/README.md) to apply these primitives in complex transactions.

## Additional Resources

- [Transaction Format](https://wiki.bitcoinsv.io/index.php/Transaction)
- [Script Opcodes](https://wiki.bitcoinsv.io/index.php/Opcodes_used_in_Bitcoin_Script)
- [SIGHASH Types](https://wiki.bitcoinsv.io/index.php/SIGHASH_flags)
