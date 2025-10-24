# Your First Transaction

## Overview

In this module, you'll create, sign, and broadcast your first BSV transaction. This is where blockchain development becomes real - you'll be moving actual value on the BSV blockchain!

## Learning Objectives

By the end of this module, you will be able to:
- Build a complete BSV transaction
- Add inputs and outputs correctly
- Calculate and manage fees
- Sign transactions with private keys
- Broadcast transactions to the network
- Monitor transaction confirmations

## Prerequisites

Before starting, ensure you have:
- ✅ Completed [Your First Wallet](../first-wallet/README.md)
- ✅ A wallet with testnet BSV (get from [faucet](https://faucet.bsvblockchain.org/))
- ✅ Basic understanding of UTXOs and transactions

## Transaction Anatomy

A BSV transaction consists of:

```typescript
interface Transaction {
  version: number              // Protocol version (usually 1)
  inputs: TransactionInput[]   // UTXOs being spent
  outputs: TransactionOutput[] // New UTXOs being created
  lockTime: number            // When tx can be mined (usually 0)
}
```

### Transaction Flow

```
Your UTXOs (Inputs)
    ↓
Transaction
    ↓
Recipient UTXOs (Outputs)
+ Your Change (Output)
```

## Step-by-Step: Creating Your First Transaction

### Step 1: Set Up

```typescript
import {
  Transaction,
  PrivateKey,
  P2PKH,
  SatoshisPerKilobyte
} from '@bsv/sdk'

// Your wallet private key
const privateKey = PrivateKey.fromWif('your-testnet-private-key-wif')
const myAddress = privateKey.toPublicKey().toAddress()

// Recipient address
const recipientAddress = '1RecipientAddressGoesHere...'

// Amount to send (in satoshis)
const amountToSend = 10000 // 0.0001 BSV
```

### Step 2: Fetch Your UTXOs

```typescript
interface UTXO {
  txid: string
  vout: number
  satoshis: number
  script: string
}

async function getUTXOs(address: string): Promise<UTXO[]> {
  const response = await fetch(
    `https://api.whatsonchain.com/v1/bsv/test/address/${address}/unspent`
  )

  if (!response.ok) {
    throw new Error('Failed to fetch UTXOs')
  }

  return await response.json()
}

// Fetch your UTXOs
const myUTXOs = await getUTXOs(myAddress)
console.log('Available UTXOs:', myUTXOs)
```

### Step 3: Select UTXOs for Input

```typescript
// Simple UTXO selection: use first UTXO with enough funds
function selectUTXOs(utxos: UTXO[], targetAmount: number): UTXO[] {
  const selected: UTXO[] = []
  let total = 0

  for (const utxo of utxos) {
    selected.push(utxo)
    total += utxo.satoshis

    // Include estimated fee (500 sats for simple tx)
    if (total >= targetAmount + 500) {
      break
    }
  }

  if (total < targetAmount + 500) {
    throw new Error('Insufficient funds')
  }

  return selected
}

const selectedUTXOs = selectUTXOs(myUTXOs, amountToSend)
console.log('Selected UTXOs:', selectedUTXOs)
```

### Step 4: Create Transaction

```typescript
// Create new transaction
const tx = new Transaction()

// Calculate total input amount
const inputTotal = selectedUTXOs.reduce((sum, utxo) => sum + utxo.satoshis, 0)

console.log('Input total:', inputTotal, 'satoshis')
```

### Step 5: Add Inputs

```typescript
// Add each UTXO as an input
for (const utxo of selectedUTXOs) {
  await tx.addInput({
    sourceTransaction: utxo.txid,
    sourceOutputIndex: utxo.vout,
    unlockingScriptTemplate: new P2PKH().unlock(privateKey),
    sequence: 0xffffffff // Standard sequence number
  })
}

console.log('Added', tx.inputs.length, 'inputs')
```

### Step 6: Add Payment Output

```typescript
// Create output paying recipient
tx.addOutput({
  satoshis: amountToSend,
  lockingScript: new P2PKH().lock(recipientAddress)
})

console.log('Added payment output:', amountToSend, 'satoshis')
```

### Step 7: Calculate Fee and Add Change

```typescript
// Create fee model (50 satoshis per kilobyte)
const feeModel = new SatoshisPerKilobyte(50)

// Calculate fee for this transaction
const estimatedFee = await tx.getFee(feeModel)
console.log('Estimated fee:', estimatedFee, 'satoshis')

// Calculate change
const changeAmount = inputTotal - amountToSend - estimatedFee

if (changeAmount > 0) {
  // Add change output back to yourself
  tx.addOutput({
    satoshis: changeAmount,
    lockingScript: new P2PKH().lock(myAddress)
  })

  console.log('Added change output:', changeAmount, 'satoshis')
} else if (changeAmount < 0) {
  throw new Error('Insufficient funds for fee')
}
```

### Step 8: Sign Transaction

```typescript
// Sign all inputs
await tx.sign()

console.log('Transaction signed successfully')

// Get transaction ID
const txid = tx.id('hex')
console.log('Transaction ID:', txid)

// Get raw transaction hex
const rawTx = tx.toHex()
console.log('Raw transaction:', rawTx)
```

### Step 9: Broadcast Transaction

```typescript
async function broadcastTransaction(txHex: string): Promise<string> {
  const response = await fetch(
    'https://api.whatsonchain.com/v1/bsv/test/tx/raw',
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ txhex: txHex })
    }
  )

  if (!response.ok) {
    const error = await response.text()
    throw new Error(`Broadcast failed: ${error}`)
  }

  const result = await response.text()
  return result.replace(/"/g, '') // Remove quotes from txid
}

// Broadcast the transaction
try {
  const broadcastedTxid = await broadcastTransaction(rawTx)
  console.log('✅ Transaction broadcast successfully!')
  console.log('Transaction ID:', broadcastedTxid)
  console.log('View on explorer:', `https://test.whatsonchain.com/tx/${broadcastedTxid}`)
} catch (error) {
  console.error('❌ Broadcast failed:', error.message)
}
```

## Complete Example Function

```typescript
import {
  Transaction,
  PrivateKey,
  P2PKH,
  SatoshisPerKilobyte
} from '@bsv/sdk'

async function sendBSV(
  privateKeyWif: string,
  recipientAddress: string,
  amountSatoshis: number,
  network: 'main' | 'test' = 'test'
): Promise<string> {
  // 1. Setup
  const privateKey = PrivateKey.fromWif(privateKeyWif)
  const myAddress = privateKey.toPublicKey().toAddress()

  console.log('Sending from:', myAddress)
  console.log('Sending to:', recipientAddress)
  console.log('Amount:', amountSatoshis, 'satoshis')

  // 2. Fetch UTXOs
  const apiBase = network === 'main'
    ? 'https://api.whatsonchain.com/v1/bsv/main'
    : 'https://api.whatsonchain.com/v1/bsv/test'

  const utxosResponse = await fetch(`${apiBase}/address/${myAddress}/unspent`)
  const utxos: UTXO[] = await utxosResponse.json()

  if (utxos.length === 0) {
    throw new Error('No UTXOs available. Fund your wallet first.')
  }

  // 3. Select UTXOs
  let inputTotal = 0
  const selectedUTXOs: UTXO[] = []

  for (const utxo of utxos) {
    selectedUTXOs.push(utxo)
    inputTotal += utxo.satoshis

    if (inputTotal >= amountSatoshis + 1000) { // +1000 for fee buffer
      break
    }
  }

  if (inputTotal < amountSatoshis + 500) {
    throw new Error(`Insufficient funds. Have: ${inputTotal}, Need: ${amountSatoshis + 500}`)
  }

  // 4. Build transaction
  const tx = new Transaction()

  // Add inputs
  for (const utxo of selectedUTXOs) {
    await tx.addInput({
      sourceTransaction: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(privateKey)
    })
  }

  // Add payment output
  tx.addOutput({
    satoshis: amountSatoshis,
    lockingScript: new P2PKH().lock(recipientAddress)
  })

  // Calculate fee
  const feeModel = new SatoshisPerKilobyte(50)
  const fee = await tx.getFee(feeModel)

  // Add change output
  const change = inputTotal - amountSatoshis - fee

  if (change > 546) { // Dust limit
    tx.addOutput({
      satoshis: change,
      lockingScript: new P2PKH().lock(myAddress)
    })
  } else if (change < 0) {
    throw new Error('Insufficient funds for transaction fee')
  }

  // 5. Sign transaction
  await tx.sign()

  // 6. Broadcast
  const rawTx = tx.toHex()
  const broadcastResponse = await fetch(`${apiBase}/tx/raw`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ txhex: rawTx })
  })

  if (!broadcastResponse.ok) {
    const error = await broadcastResponse.text()
    throw new Error(`Broadcast failed: ${error}`)
  }

  const txid = await broadcastResponse.text()
  return txid.replace(/"/g, '')
}

// Usage
try {
  const txid = await sendBSV(
    'your-private-key-wif',
    'recipient-address',
    10000, // 0.0001 BSV
    'test'
  )

  console.log('✅ Success! Transaction ID:', txid)
  console.log('View:', `https://test.whatsonchain.com/tx/${txid}`)
} catch (error) {
  console.error('❌ Error:', error.message)
}
```

## Monitoring Confirmations

After broadcasting, monitor confirmations:

```typescript
async function getConfirmations(
  txid: string,
  network: 'main' | 'test' = 'test'
): Promise<number> {
  const apiBase = network === 'main'
    ? 'https://api.whatsonchain.com/v1/bsv/main'
    : 'https://api.whatsonchain.com/v1/bsv/test'

  const response = await fetch(`${apiBase}/tx/hash/${txid}`)

  if (!response.ok) {
    return 0 // Not found or not confirmed
  }

  const txInfo = await response.json()

  if (!txInfo.blockheight) {
    return 0 // In mempool, not in block
  }

  // Get current block height
  const chainInfoResponse = await fetch(`${apiBase}/chain/info`)
  const chainInfo = await chainInfoResponse.json()

  const confirmations = chainInfo.blocks - txInfo.blockheight + 1
  return confirmations
}

// Monitor transaction
async function waitForConfirmations(
  txid: string,
  targetConfirmations: number = 6,
  network: 'main' | 'test' = 'test'
): Promise<void> {
  console.log(`Waiting for ${targetConfirmations} confirmations...`)

  while (true) {
    const confirmations = await getConfirmations(txid, network)
    console.log(`Confirmations: ${confirmations}/${targetConfirmations}`)

    if (confirmations >= targetConfirmations) {
      console.log('✅ Transaction confirmed!')
      break
    }

    // Wait 30 seconds before checking again
    await new Promise(resolve => setTimeout(resolve, 30000))
  }
}

// Usage
await waitForConfirmations(txid, 6, 'test')
```

## Common Issues and Solutions

### Issue 1: "Insufficient funds"

**Problem**: Not enough satoshis to cover amount + fee

**Solution**:
```typescript
// Check balance first
const utxos = await getUTXOs(myAddress)
const balance = utxos.reduce((sum, utxo) => sum + utxo.satoshis, 0)

console.log('Balance:', balance)
console.log('Needed:', amountToSend + 500)

if (balance < amountToSend + 500) {
  throw new Error('Insufficient balance. Get testnet BSV from faucet.')
}
```

### Issue 2: "Transaction broadcast rejected"

**Problem**: Invalid transaction or double-spend

**Solutions**:
- Ensure UTXOs haven't been spent
- Verify signatures are correct
- Check transaction format

```typescript
// Verify transaction before broadcasting
const isValid = await tx.verify()
if (!isValid) {
  throw new Error('Transaction validation failed')
}
```

### Issue 3: "Dust output"

**Problem**: Change output too small (< 546 satoshis)

**Solution**:
```typescript
const DUST_LIMIT = 546

if (change > DUST_LIMIT) {
  // Add change output
  tx.addOutput({
    satoshis: change,
    lockingScript: new P2PKH().lock(myAddress)
  })
} else {
  // Add to fee instead
  console.log('Change below dust limit, adding to fee')
}
```

## Understanding Transaction Fees

### Fee Calculation

Fees are based on transaction size:
```
Fee = (Transaction Size in Bytes) × (Fee Rate in sats/byte)
```

**Fee Rate**: Typically 0.5 - 1 satoshi per byte on BSV

**Transaction Size**: Depends on inputs and outputs
- Each input: ~148 bytes
- Each output: ~34 bytes
- Fixed overhead: ~10 bytes

### Example Calculation

```typescript
// 1 input, 2 outputs transaction
const size = 10 + (1 * 148) + (2 * 34) // = 226 bytes

// At 0.5 sats/byte
const fee = 226 * 0.5 // = 113 satoshis

// At 1 sat/byte
const fee2 = 226 * 1 // = 226 satoshis
```

### Fee Models in SDK

```typescript
import { SatoshisPerKilobyte } from '@bsv/sdk'

// 50 satoshis per kilobyte = 0.05 sats/byte
const feeModel = new SatoshisPerKilobyte(50)

const fee = await tx.getFee(feeModel)
```

## Best Practices

### 1. Always Test on Testnet

```typescript
const NETWORK = 'test' // Change to 'main' only after testing

// Use testnet faucet
console.log('Get testnet BSV:', 'https://faucet.bsvblockchain.org/')
```

### 2. Verify Before Broadcasting

```typescript
// Check transaction is valid
if (await tx.verify()) {
  console.log('✅ Transaction is valid')
} else {
  console.error('❌ Transaction validation failed')
  return
}
```

### 3. Handle Errors Gracefully

```typescript
try {
  const txid = await sendBSV(...)
  console.log('Success:', txid)
} catch (error) {
  if (error.message.includes('Insufficient')) {
    console.error('Not enough funds. Get more from faucet.')
  } else if (error.message.includes('double spend')) {
    console.error('UTXO already spent. Fetch fresh UTXOs.')
  } else {
    console.error('Error:', error.message)
  }
}
```

### 4. Wait for Confirmations

```typescript
// For small amounts: 1-3 confirmations
// For medium amounts: 6 confirmations
// For large amounts: 6+ confirmations

const REQUIRED_CONFIRMATIONS = 6
await waitForConfirmations(txid, REQUIRED_CONFIRMATIONS)
```

## Practice Exercises

1. **Send Testnet BSV**: Send 0.0001 BSV to another address
2. **Batch Payment**: Send to multiple recipients in one transaction
3. **Monitor Confirmations**: Track a transaction from broadcast to 6 confirmations
4. **Calculate Fees**: Estimate fees for different transaction sizes
5. **Handle Errors**: Implement proper error handling for all cases

## Related Components

- [Transaction](../../../sdk-components/transaction/README.md)
- [Transaction Input](../../../sdk-components/transaction-input/README.md)
- [Transaction Output](../../../sdk-components/transaction-output/README.md)
- [P2PKH](../../../sdk-components/p2pkh/README.md)

## Related Code Features

- [Transaction Creation](../../../code-features/transaction-creation/README.md)
- [Sign Transaction](../../../code-features/sign-transaction/README.md)
- [Broadcast with ARC](../../../code-features/broadcast-arc/README.md)

## Next Steps

Congratulations! You've completed the Beginner Learning Path. You now know how to:
- ✅ Set up a BSV development environment
- ✅ Understand BSV blockchain fundamentals
- ✅ Create and manage wallets
- ✅ Build, sign, and broadcast transactions

**Ready for more?** Continue to the [Intermediate Learning Path](../../intermediate/README.md) to learn about:
- Complex transaction building
- Custom Bitcoin Scripts
- SPV verification
- BRC standards implementation

## Additional Resources

- [WhatsOnChain Testnet](https://test.whatsonchain.com/) - View your transactions
- [BSV Testnet Faucet](https://faucet.bsvblockchain.org/) - Get free testnet BSV
- [Transaction Format](https://wiki.bitcoinsv.io/index.php/Transaction) - Technical details
