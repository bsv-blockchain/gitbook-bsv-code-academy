# BSV Fundamentals

**Module 2: Core BSV Blockchain Concepts**

This module covers the core concepts and building blocks of the BSV blockchain. Understanding these fundamentals is essential for all BSV development.

> **Note**: These concepts apply to **both Backend and Frontend development paradigms**. After completing this module, you'll choose your path:
> - **Backend Development**: Server-side wallet management → [First Wallet (Backend)](../first-wallet/)
> - **Frontend Development**: User wallet integration → [WalletClient Integration](../wallet-client-integration/)

## Learning Objectives

By the end of this module, you will understand:
- How transactions and UTXOs work
- Public/private key cryptography
- Addresses and their derivation
- Bitcoin Script basics
- Satoshis and denominations
- Block structure and confirmations
- SPV and blockchain verification

## Core Concepts

### 1. Transactions

Transactions are the fundamental unit of activity on the BSV blockchain.

#### Structure
```typescript
interface Transaction {
  version: number
  inputs: TransactionInput[]
  outputs: TransactionOutput[]
  lockTime: number
}
```

**Key Points**:
- Transactions move value from inputs to outputs
- Inputs reference previous outputs (UTXOs)
- Outputs create new spendable UTXOs
- Transaction ID (txid) is double SHA-256 hash

#### Example
```typescript
import { Transaction } from '@bsv/sdk'

// Create a transaction
const tx = new Transaction()

// Add inputs (spending previous outputs)
tx.addInput(/* ... */)

// Add outputs (creating new spendable outputs)
tx.addOutput(/* ... */)
```

### 2. UTXO Model

BSV uses the Unspent Transaction Output (UTXO) model, not an account model.

#### What is a UTXO?

A UTXO is:
- An **unspent output** from a previous transaction
- A discrete chunk of satoshis
- Locked by a script (locking script)
- Can only be spent once

#### UTXO vs Account Model

**UTXO Model (BSV)**:
```
Alice's UTXOs:
- UTXO1: 50,000 sats (from tx abc...)
- UTXO2: 30,000 sats (from tx def...)
- UTXO3: 20,000 sats (from tx ghi...)
Total: 100,000 sats
```

**Account Model (Ethereum)**:
```
Alice's Account:
- Balance: 100,000 sats
```

#### Benefits of UTXO Model
- **Parallel Processing**: UTXOs can be processed independently
- **Clear Ownership**: Each UTXO has explicit owner
- **Atomic Transactions**: All inputs spent or none
- **Efficient Verification**: Can verify without full state

#### Working with UTXOs
```typescript
// A UTXO reference
interface UTXO {
  txid: string        // Transaction ID
  vout: number        // Output index in that transaction
  satoshis: number    // Amount in satoshis
  script: Script      // Locking script
}

// To spend a UTXO, reference it as an input
const input = {
  sourceTransaction: utxo.txid,
  sourceOutputIndex: utxo.vout,
  unlockingScript: /* script to unlock */
}
```

### 3. Public/Private Key Cryptography

BSV uses elliptic curve cryptography (specifically secp256k1).

#### Private Key
- **256-bit random number**
- Must be kept secret
- Used to sign transactions
- Controls access to funds

```typescript
import { PrivateKey } from '@bsv/sdk'

// Generate random private key
const privateKey = PrivateKey.fromRandom()

// From WIF (Wallet Import Format)
const privateKey2 = PrivateKey.fromWif('L1234...')

// Never share your private key!
```

#### Public Key
- **Derived from private key** (one-way)
- Can be shared publicly
- Used to verify signatures
- 33 bytes (compressed) or 65 bytes (uncompressed)

```typescript
// Derive public key from private key
const publicKey = privateKey.toPublicKey()

console.log(publicKey.toString()) // 02a1b2c3...
```

#### Key Properties
- **One-way derivation**: Private → Public (easy)
- **No reverse**: Public → Private (impossible)
- **Signatures**: Only private key can sign
- **Verification**: Anyone with public key can verify

### 4. Addresses

Addresses are human-readable representations of public key hashes.

#### Address Derivation
```
Private Key
    ↓ (ECDSA)
Public Key
    ↓ (SHA-256)
SHA-256 Hash
    ↓ (RIPEMD-160)
Public Key Hash
    ↓ (Base58Check)
Address
```

#### Creating Addresses
```typescript
const privateKey = PrivateKey.fromRandom()
const publicKey = privateKey.toPublicKey()
const address = publicKey.toAddress()

console.log(address) // 1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa
```

#### Address Formats
- **Legacy (P2PKH)**: Starts with `1` (e.g., `1A1zP1eP...`)
- **Testnet**: Starts with `m` or `n`

#### Important Notes
- Addresses are **not** stored on blockchain
- Only public key hashes are in locking scripts
- Addresses are for human convenience
- Multiple addresses can come from one public key

### 5. Satoshis and Denominations

#### Satoshi
The smallest unit of BSV:
- **1 BSV = 100,000,000 satoshis**
- Named after Satoshi Nakamoto
- All amounts are integers (no decimals on blockchain)

#### Denominations
```
1 BSV       = 100,000,000 satoshis
0.01 BSV    = 1,000,000 satoshis (1 million)
0.0001 BSV  = 10,000 satoshis
0.00000001  = 1 satoshi (smallest unit)
```

#### In Code
```typescript
// Always use satoshis in code
const amount = 100000000  // 1 BSV
const payment = 50000     // 0.0005 BSV

// Helper for conversion
function bsvToSatoshis(bsv: number): number {
  return Math.round(bsv * 100000000)
}

function satoshisToBsv(sats: number): number {
  return sats / 100000000
}
```

#### Why Satoshis?
- **No floating point errors**: Integer math is precise
- **Micropayments**: Enable sub-cent transactions
- **Protocol native**: Blockchain uses satoshis

### 6. Bitcoin Script

Bitcoin Script is a stack-based scripting language that defines spending conditions.

#### Locking Script (scriptPubKey)
Defines conditions to spend an output:
```
OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
```

#### Unlocking Script (scriptSig)
Provides data to satisfy locking script:
```
<signature> <publicKey>
```

#### Script Execution
Scripts are concatenated and executed:
```
<signature> <publicKey> OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
```

**Execution Steps**:
1. Push signature to stack
2. Push publicKey to stack
3. OP_DUP duplicates publicKey
4. OP_HASH160 hashes publicKey
5. Push expected pubKeyHash
6. OP_EQUALVERIFY checks hash matches
7. OP_CHECKSIG verifies signature

If script completes without errors and top of stack is TRUE, output can be spent.

#### Script Features
- **Turing complete** (with some limitations)
- **Stack-based**: Uses a stack for operations
- **Deterministic**: Same input = same output
- **Flexible**: Can create complex conditions

```typescript
import { Script } from '@bsv/sdk'

// Simple P2PKH locking script
const lockingScript = Script.fromASM(`
  OP_DUP
  OP_HASH160
  ${pubKeyHash}
  OP_EQUALVERIFY
  OP_CHECKSIG
`)
```

### 7. Blocks and Confirmations

#### Block Structure
```typescript
interface Block {
  header: BlockHeader
  transactions: Transaction[]
}

interface BlockHeader {
  version: number
  previousBlockHash: string
  merkleRoot: string
  timestamp: number
  bits: number  // Difficulty target
  nonce: number // Proof of work
}
```

#### Block Creation
1. Miners collect transactions from mempool
2. Build merkle tree of transaction IDs
3. Find nonce that satisfies difficulty
4. Broadcast block to network
5. Other nodes validate and extend chain

#### Confirmations
- **0 confirmations**: In mempool, not in block (unconfirmed)
- **1 confirmation**: Included in latest block
- **2 confirmations**: One block built on top
- **6+ confirmations**: Generally considered final

#### Confirmation Time
- **Average block time**: ~10 minutes
- **1 confirmation**: ~10 minutes
- **6 confirmations**: ~1 hour

```typescript
// Check confirmations
async function getConfirmations(txid: string): Promise<number> {
  const txInfo = await fetchTransactionInfo(txid)

  if (!txInfo.blockHeight) {
    return 0 // Unconfirmed
  }

  const currentHeight = await getCurrentBlockHeight()
  return currentHeight - txInfo.blockHeight + 1
}
```

### 8. SPV (Simplified Payment Verification)

SPV allows lightweight clients to verify transactions without downloading the full blockchain.

#### How SPV Works
1. Download block headers only (~80 bytes each)
2. Request merkle proofs for specific transactions
3. Verify transaction is in block using merkle proof
4. Trust the chain with most proof-of-work

#### Merkle Proofs
```
Block Header
    ↓
Merkle Root
    ↓ (verify path)
Your Transaction
```

#### Benefits
- **Lightweight**: Only headers needed (~80 MB for 1M blocks)
- **Fast**: Quick verification
- **Secure**: Cryptographic proof of inclusion
- **Mobile-friendly**: Works on resource-constrained devices

#### SPV vs Full Node
**SPV Client**:
- Downloads headers only
- Requests proofs as needed
- Verifies specific transactions
- Suitable for wallets

**Full Node**:
- Downloads all blocks
- Validates all transactions
- Maintains full UTXO set
- Suitable for miners, services

The example below demonstrates how SPV verification enables you to cryptographically prove that a transaction is included in a block without downloading the entire block. You only need the block header and a Merkle proof.

```typescript
import { MerkleProof } from '@bsv/sdk'

// Verify a transaction is in a block using its merkle proof
// merkleProof: cryptographic proof from SPV server showing tx is in block
// blockHeader: 80-byte header containing merkle root to verify against
function verifyTxInBlock(
  txid: string,
  merkleProof: MerkleProof,
  blockHeader: BlockHeader
): boolean {
  // Calculate merkle root from the proof path
  const calculatedRoot = merkleProof.calculateRoot(txid)

  // Compare with the merkle root in the block header
  // If they match, the transaction is proven to be in the block
  return calculatedRoot === blockHeader.merkleRoot
}
```

## Putting It All Together

### Transaction Lifecycle

1. **Create Transaction**
   - Select UTXOs to spend (inputs)
   - Create outputs for recipients
   - Create change output if needed

2. **Sign Transaction**
   - Create unlocking scripts with private key
   - Sign each input

3. **Broadcast Transaction**
   - Send to network via ARC or nodes
   - Transaction enters mempool

4. **Mining**
   - Miner includes transaction in block
   - Finds proof-of-work
   - Broadcasts block

5. **Confirmation**
   - Block added to longest chain
   - Each new block adds confirmation
   - After 6+ confirmations, considered final

### Example: Complete Payment Flow

**Important Note**: This example shows the conceptual flow to understand how transactions work internally. In practice:
- **Frontend apps**: Use `WalletClient` which handles all of this automatically
- **Backend apps**: Use SDK's built-in methods for UTXO management, fees, and broadcasting

#### Conceptual Example (Understanding the Flow)

```typescript
import { PrivateKey, Transaction, P2PKH } from '@bsv/sdk'

// This shows what happens "under the hood"
// You typically won't write this code yourself
async function sendPaymentConceptual(
  privateKey: PrivateKey,
  recipientAddress: string,
  amount: number
) {
  // 1. Get UTXOs (in practice, SDK or wallet handles this)
  const myAddress = privateKey.toPublicKey().toAddress()
  const utxos = await getUTXOs(myAddress)

  // 2. Create transaction
  const tx = new Transaction()

  // 3. Add inputs (SDK can select UTXOs automatically)
  let inputTotal = 0
  for (const utxo of utxos) {
    if (inputTotal >= amount + 1000) break // +fee

    await tx.addInput({
      sourceTransaction: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(privateKey)
    })

    inputTotal += utxo.satoshis
  }

  // 4. Add payment output
  tx.addOutput({
    satoshis: amount,
    lockingScript: new P2PKH().lock(recipientAddress)
  })

  // 5. Add change output (SDK can calculate this automatically)
  const fee = 500
  const change = inputTotal - amount - fee
  if (change > 0) {
    tx.addOutput({
      satoshis: change,
      lockingScript: new P2PKH().lock(myAddress)
    })
  }

  // 6. Sign (SDK handles)
  await tx.sign()

  // 7. Broadcast (SDK/wallet handles)
  const txid = await broadcast(tx)

  return txid
}
```

#### In Practice: Frontend with WalletClient

```typescript
import { WalletClient } from '@bsv/sdk'

// What you actually write for frontend apps
async function sendPaymentFrontend(
  wallet: WalletClient,
  recipientAddress: string,
  amount: number
) {
  // Wallet handles: UTXO selection, fees, change, signing, broadcasting
  const result = await wallet.createAction({
    description: 'Send payment',
    outputs: [{
      lockingScript: new P2PKH().lock(recipientAddress).toHex(),
      satoshis: amount,
      outputDescription: 'Payment'
    }]
  })

  return result.txid
}
```

#### In Practice: Backend with SDK

```typescript
import { PrivateKey, Transaction, P2PKH } from '@bsv/sdk'

// What you write for backend services (simplified)
async function sendPaymentBackend(
  privateKey: PrivateKey,
  recipientAddress: string,
  amount: number
) {
  const tx = new Transaction()

  // Add payment output
  tx.addOutput({
    satoshis: amount,
    lockingScript: new P2PKH().lock(recipientAddress)
  })

  // SDK handles: UTXO selection, fee calculation, change outputs
  await tx.sign(privateKey)
  const txid = await tx.broadcast()

  return txid
}
```

**Key Point**: The detailed manual code above is for **understanding the concepts**. In real applications, the SDK and WalletClient handle UTXO selection, fee calculation, change outputs, and broadcasting automatically.

## Key Takeaways

- ✅ **Transactions** move value via inputs and outputs
- ✅ **UTXO Model** enables parallel processing and clear ownership
- ✅ **Private Keys** must be kept secret and control funds
- ✅ **Public Keys** are derived from private keys and can be shared
- ✅ **Addresses** are Base58-encoded public key hashes
- ✅ **Satoshis** are the base unit (100M sats = 1 BSV)
- ✅ **Bitcoin Script** defines spending conditions
- ✅ **Blocks** contain transactions and build on previous blocks
- ✅ **Confirmations** indicate transaction finality
- ✅ **SPV** enables lightweight verification

## Practice Exercises

1. **Generate Keys**: Create a private key, derive public key and address
2. **Calculate Amounts**: Convert between BSV and satoshis
3. **Understand UTXOs**: Trace how UTXOs are created and spent
4. **Read Scripts**: Interpret a P2PKH locking script
5. **Track Confirmations**: Monitor a transaction from broadcast to 6 confirmations

## Related Components

- [Transaction](../../../sdk-components/transaction/README.md)
- [Private Keys](../../../sdk-components/private-keys/README.md)
- [Script](../../../sdk-components/script/README.md)
- [SPV](../../../sdk-components/spv/README.md)

## Next Steps

Now that you understand BSV fundamentals, choose your development path:

### Backend Development (Custodial)
**You control private keys server-side**

Continue to: [Your First Wallet (Backend)](../first-wallet/) - Learn server-side wallet management, UTXO tracking, and programmatic transaction creation.

### Frontend Development (Non-Custodial)
**Users control their own keys via wallet**

Continue to: [WalletClient Integration](../wallet-client-integration/) - Learn to connect to MetaNet Desktop Wallet and request user signatures.

---

**Not sure which path?** Review [Development Paradigms](../development-paradigms/) to understand the differences.

## Additional Resources

- [Bitcoin Whitepaper](https://bitcoinsv.io/bitcoin.pdf)
- [BSV Wiki - Script](https://wiki.bitcoinsv.io/index.php/Script)
- [BSV Wiki - Transaction](https://wiki.bitcoinsv.io/index.php/Transaction)
- [Mastering Bitcoin](https://github.com/bitcoinbook/bitcoinbook) - General Bitcoin concepts
