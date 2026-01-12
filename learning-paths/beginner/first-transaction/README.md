# Your First Transaction

## Overview

In this module, you'll create, sign, and broadcast your first BSV transaction. We'll show you two approaches: backend development using the SDK's Transaction class, and frontend development using the WalletClient. Both leverage the SDK's built-in features to handle complexity automatically.

**Important Paradigm Note**: The BSV SDK handles most transaction complexity for you - fee calculation, change outputs, UTXO selection, and broadcasting are all automated. You focus on what you want to accomplish, not the low-level mechanics.

## Learning Objectives

By the end of this module, you will be able to:
- Understand transaction structure conceptually
- Build transactions using SDK's simplified methods
- Use backend Transaction class for server-side applications
- Use frontend WalletClient for browser-based applications
- Broadcast transactions using SDK methods
- Monitor transaction confirmations

## Prerequisites

Before starting, ensure you have:
- Completed [Your First Wallet](../first-wallet/README.md)
- A wallet with testnet BSV (get from MetaNet Desktop Wallet's faucet or [BSV Discord](https://discord.gg/bsv))
- Basic understanding of UTXOs and transactions

## Transaction Structure (Conceptual)

Understanding transaction structure helps you reason about Bitcoin, even though the SDK handles the details.

### What Happens Under the Hood

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
    |
    v
Transaction Logic
- Validates inputs
- Creates outputs
- Calculates fees
- Generates change
    |
    v
New UTXOs (Outputs)
```

### What the SDK Handles Automatically

The SDK takes care of:
- **Fee Calculation**: Analyzes transaction size and applies fee rate
- **Change Outputs**: Calculates and creates change back to your address
- **UTXO Selection**: Chooses appropriate UTXOs for your transaction
- **Dust Limits**: Ensures outputs meet minimum values
- **Script Templates**: Manages locking and unlocking scripts

You just specify: **what to send, how much, and to whom**.

## Backend Approach: Transaction Class

Use this approach for server-side applications, backend services, or when you need direct control over wallet management.

### Simple Payment Transaction

```typescript
import { Transaction, PrivateKey, P2PKH } from '@bsv/sdk'

async function sendPayment(
  privateKeyWif: string,
  recipientAddress: string,
  amountSatoshis: number
): Promise<string> {
  console.log('=== Creating Payment Transaction ===')
  console.log('Recipient:', recipientAddress)
  console.log('Amount:', amountSatoshis, 'satoshis')

  // Initialize your private key
  const privateKey = PrivateKey.fromWif(privateKeyWif)
  const myAddress = privateKey.toPublicKey().toAddress()
  console.log('Sending from:', myAddress)

  // Create a transaction
  const tx = new Transaction()

  // Add input (UTXO you're spending)
  // Note: In practice, you get this from a UTXO manager or wallet service
  await tx.addInput({
    sourceTransaction: previousTxid,  // Your UTXO's transaction ID
    sourceOutputIndex: outputIndex,    // Which output in that transaction
    unlockingScriptTemplate: new P2PKH().unlock(privateKey)
  })

  // Add payment output
  tx.addOutput({
    satoshis: amountSatoshis,
    lockingScript: new P2PKH().lock(recipientAddress)
  })

  console.log('\n⚙️  SDK automatically handles:')
  console.log('   - Fee calculation based on transaction size')
  console.log('   - Change output back to your address')
  console.log('   - Dust limit validation')

  // Calculate fees and change amounts
  await tx.fee()
  console.log('\n✅ Fees calculated')

  // Sign the transaction
  await tx.sign()
  console.log('✅ Transaction signed')

  // Broadcast using SDK method
  const result = await tx.broadcast()
  console.log('✅ Transaction broadcast!')
  console.log('TXID:', result.txid)
  console.log('\n🔗 View on explorer:')
  console.log('   https://test.whatsonchain.com/tx/' + result.txid)

  return result.txid
}

// Run this example: npx ts-node send-payment.ts
//
// ⚠️  Note: You need testnet BSV first!
// Get it from BSV Desktop: https://desktop.bsvb.tech/
// Or request from BSV Discord: https://discord.gg/bsv
```

### What You Write vs What Happens

**What You Write**:
```typescript
tx.addInput({ sourceTransaction, sourceOutputIndex, unlockingScriptTemplate })
tx.addOutput({ satoshis, lockingScript })
await tx.fee()
await tx.sign()
await tx.broadcast()
```

**What the SDK Does**:
1. Validates the input UTXO
2. Calculates transaction size
3. Computes appropriate fee
4. Determines change amount
5. Creates change output (if needed)
6. Checks dust limits
7. Signs all inputs
8. Serializes transaction
9. Broadcasts to network
10. Returns transaction ID

### Complete Backend Example

```typescript
import { Transaction, PrivateKey, P2PKH, ARC } from '@bsv/sdk'

interface PaymentRequest {
  privateKeyWif: string
  recipientAddress: string
  amountSatoshis: number
}

async function createAndSendPayment(
  request: PaymentRequest
): Promise<{ txid: string; fee: number }> {
  const { privateKeyWif, recipientAddress, amountSatoshis } = request

  // Setup
  const privateKey = PrivateKey.fromWif(privateKeyWif)
  const myAddress = privateKey.toPublicKey().toAddress()

  console.log('Sending from:', myAddress)
  console.log('Sending to:', recipientAddress)
  console.log('Amount:', amountSatoshis, 'satoshis')

  // Create transaction
  const tx = new Transaction()

  // Add your UTXO as input
  // In production: get UTXOs from your wallet manager or overlay services
  await tx.addInput({
    sourceTransaction: yourUTXO.txid,
    sourceOutputIndex: yourUTXO.vout,
    unlockingScriptTemplate: new P2PKH().unlock(privateKey),
    sequence: 0xffffffff
  })

  // Add payment output
  tx.addOutput({
    satoshis: amountSatoshis,
    lockingScript: new P2PKH().lock(recipientAddress)
  })

  // SDK handles:
  // - Fee calculation
  // - Change output creation
  // - Dust limit checking

  // Calculate fees and change
  await tx.fee()

  // Sign transaction
  await tx.sign()

  console.log('Transaction signed')
  console.log('Transaction ID:', tx.id('hex'))

  // Broadcast using SDK's broadcast method
  const broadcastResult = await tx.broadcast()

  console.log('Transaction broadcast successfully')
  console.log('Status:', broadcastResult.status)
  console.log('TXID:', broadcastResult.txid)

  // Get fee information (SDK calculated this)
  const fee = await tx.getFee()

  return {
    txid: broadcastResult.txid,
    fee
  }
}

// Usage Example
async function main() {
  try {
    // ⚠️  Replace with your testnet private key!
    // Generate one with: npx ts-node -e "import {PrivateKey} from '@bsv/sdk'; console.log(PrivateKey.fromRandom().toWif())"
    const myPrivateKey = 'YOUR-TESTNET-PRIVATE-KEY-WIF-HERE'

    // ⚠️  Replace with recipient address
    const recipient = '1RecipientAddressGoesHere...'

    console.log('💡 Before running: Fund your address with testnet BSV')
    console.log('   1. Get BSV Desktop: https://desktop.bsvb.tech/')
    console.log('   2. Switch to testnet mode')
    console.log('   3. Use the built-in faucet to get free testnet BSV')
    console.log('   4. Your address:', PrivateKey.fromWif(myPrivateKey).toPublicKey().toAddress())
    console.log('\n Press Ctrl+C to exit if you need to fund your address first\n')

    const result = await createAndSendPayment({
      privateKeyWif: myPrivateKey,
      recipientAddress: recipient,
      amountSatoshis: 10000  // 0.0001 BSV
    })

    console.log('\n🎉 Success!')
    console.log('Transaction ID:', result.txid)
    console.log('Fee paid:', result.fee, 'satoshis')
    console.log('\n🔗 View on testnet explorer:')
    console.log(`   https://test.whatsonchain.com/tx/${result.txid}`)

  } catch (error: any) {
    console.error('\n❌ Transaction failed:', error.message)
    console.log('\n💡 Common issues:')
    console.log('   - No testnet BSV? Get from BSV Desktop faucet')
    console.log('   - Invalid address? Check recipient address format')
    console.log('   - Insufficient funds? Need more than amount + fees (~500 sats)')
  }
}

main()

// Run this complete example: npx ts-node complete-transaction.ts
```

### Backend: Multiple Recipients

Sending to multiple recipients is simple - just add multiple outputs:

```typescript
async function sendToMultipleRecipients(
  privateKeyWif: string,
  payments: Array<{ address: string; amount: number }>
): Promise<string> {
  const privateKey = PrivateKey.fromWif(privateKeyWif)
  const tx = new Transaction()

  // Add input(s)
  await tx.addInput({
    sourceTransaction: yourUTXO.txid,
    sourceOutputIndex: yourUTXO.vout,
    unlockingScriptTemplate: new P2PKH().unlock(privateKey)
  })

  // Add multiple payment outputs
  for (const payment of payments) {
    tx.addOutput({
      satoshis: payment.amount,
      lockingScript: new P2PKH().lock(payment.address)
    })
  }

  // SDK automatically handles:
  // - Fee calculation for larger transaction
  // - Change output creation

  await tx.fee()
  await tx.sign()
  const result = await tx.broadcast()

  return result.txid
}

// Usage: send to 3 recipients in one transaction
async function runBatchPayment() {
  console.log('=== Batch Payment Example ===')
  console.log('Sending to 3 recipients in a single transaction\n')

  try {
    const txid = await sendToMultipleRecipients(
      privateKeyWif,
      [
        { address: 'recipient1...', amount: 5000 },
        { address: 'recipient2...', amount: 10000 },
        { address: 'recipient3...', amount: 15000 }
      ]
    )

    console.log('\n✅ Batch payment successful!')
    console.log('TXID:', txid)
    console.log('Total sent: 30,000 satoshis to 3 recipients')
    console.log('\nView: https://test.whatsonchain.com/tx/' + txid)
  } catch (error: any) {
    console.error('❌ Batch payment failed:', error.message)
  }
}

runBatchPayment()

// Run this: npx ts-node batch-payment.ts
```

## Frontend Approach: WalletClient

Use this approach for browser-based applications where the user's wallet (like Panda Wallet) handles all the complexity.

### How WalletClient Works

WalletClient is a standardized interface that connects to user wallets. The wallet:
- Manages private keys securely
- Selects and manages UTXOs
- Calculates fees
- Signs transactions
- Broadcasts to network

You just specify what you want to accomplish.

### Simple Payment with WalletClient

```typescript
import { WalletClient } from '@bsv/sdk'

async function sendPaymentViaWallet(
  recipientAddress: string,
  amountSatoshis: number
): Promise<string> {
  // Connect to user's wallet
  const wallet = await WalletClient.connect()

  // Request payment - wallet handles everything
  const result = await wallet.createAction({
    outputs: [
      {
        satoshis: amountSatoshis,
        script: new P2PKH().lock(recipientAddress).toHex()
      }
    ],
    description: 'Payment transaction'
  })

  // Wallet has:
  // - Selected UTXOs
  // - Calculated fees
  // - Created change output
  // - Signed transaction
  // - Broadcast to network

  return result.txid
}

// Usage
try {
  const txid = await sendPaymentViaWallet(
    '1RecipientAddressGoesHere...',
    10000  // 0.0001 BSV
  )

  console.log('Payment sent!')
  console.log('Transaction ID:', txid)
  console.log('View:', `https://whatsonchain.com/tx/${txid}`)
} catch (error) {
  console.error('Payment failed:', error.message)
}
```

### Frontend: Complete Example with UI

```typescript
import { WalletClient, P2PKH } from '@bsv/sdk'

class PaymentApp {
  private wallet: WalletClient | null = null

  async connectWallet(): Promise<void> {
    try {
      this.wallet = await WalletClient.connect()
      console.log('Wallet connected successfully')

      // Get user's identity
      const identity = await this.wallet.getPublicKey()
      console.log('Connected as:', identity)
    } catch (error) {
      throw new Error('Failed to connect wallet: ' + error.message)
    }
  }

  async sendPayment(
    recipientAddress: string,
    amountSatoshis: number,
    description: string = 'Payment'
  ): Promise<string> {
    if (!this.wallet) {
      throw new Error('Wallet not connected')
    }

    console.log('Creating payment...')
    console.log('To:', recipientAddress)
    console.log('Amount:', amountSatoshis, 'satoshis')

    // Create payment action
    const result = await this.wallet.createAction({
      outputs: [
        {
          satoshis: amountSatoshis,
          script: new P2PKH().lock(recipientAddress).toHex()
        }
      ],
      description
    })

    console.log('Payment successful!')
    console.log('TXID:', result.txid)

    return result.txid
  }

  async sendToMultiple(
    payments: Array<{ address: string; amount: number }>
  ): Promise<string> {
    if (!this.wallet) {
      throw new Error('Wallet not connected')
    }

    // Create multiple outputs
    const outputs = payments.map(p => ({
      satoshis: p.amount,
      script: new P2PKH().lock(p.address).toHex()
    }))

    const result = await this.wallet.createAction({
      outputs,
      description: `Payment to ${payments.length} recipients`
    })

    return result.txid
  }
}

// Usage in your app
const app = new PaymentApp()

async function handlePayment() {
  try {
    // Connect wallet
    await app.connectWallet()

    // Send payment
    const txid = await app.sendPayment(
      '1RecipientAddress...',
      10000,
      'Payment for goods'
    )

    // Show success
    alert(`Payment sent! TXID: ${txid}`)
  } catch (error) {
    alert(`Payment failed: ${error.message}`)
  }
}
```

### Frontend: Request Payment from User

You can also request a payment from the user with a specific amount:

```typescript
async function requestPayment(
  recipientAddress: string,
  amountSatoshis: number,
  description: string
): Promise<string> {
  const wallet = await WalletClient.connect()

  // Request specific payment
  const result = await wallet.createAction({
    outputs: [
      {
        satoshis: amountSatoshis,
        script: new P2PKH().lock(recipientAddress).toHex()
      }
    ],
    description
  })

  return result.txid
}

// Usage: request payment for a service
const txid = await requestPayment(
  'your-business-address',
  50000,  // 0.0005 BSV
  'Payment for Premium Subscription'
)
```

## Broadcasting Transactions

### Using SDK's Built-in Broadcast

The simplest way to broadcast is using the Transaction class's built-in method:

```typescript
// Backend approach
const tx = new Transaction()
// ... add inputs and outputs ...
await tx.fee()
await tx.sign()

// Broadcast using SDK method
const result = await tx.broadcast()

console.log('Transaction ID:', result.txid)
console.log('Status:', result.status)
```

**What `tx.broadcast()` does**:
- Serializes the transaction to hex
- Connects to ARC (miner broadcasting service)
- Submits transaction to the network
- Returns confirmation with TXID

### WalletClient Auto-Broadcast

With WalletClient, broadcasting is automatic:

```typescript
// Frontend approach
const wallet = await WalletClient.connect()

// This broadcasts automatically
const result = await wallet.createAction({
  outputs: [{ satoshis: 10000, script: lockingScript }],
  description: 'Payment'
})

// Transaction is already broadcast
console.log('Broadcast TXID:', result.txid)
```

### Broadcast Options (Advanced)

If you need custom broadcast behavior:

```typescript
import { Transaction, ARC } from '@bsv/sdk'

const tx = new Transaction()
// ... build transaction ...
await tx.fee()
await tx.sign()

// Option 1: Use default ARC
await tx.broadcast()

// Option 2: Specify custom ARC endpoint
const customARC = new ARC({
  apiKey: 'your-api-key',
  deploymentId: 'your-deployment'
})

await tx.broadcast(customARC)

// Option 3: Get raw hex for manual broadcasting
const rawHex = tx.toHex()
// ... use your own broadcast mechanism ...
```

## Monitoring Transactions

### Check Transaction Status

After broadcasting, you can check the transaction status:

```typescript
async function checkTransactionStatus(txid: string): Promise<void> {
  // In production, use overlay services or block explorers
  // This is a simplified example

  console.log('Transaction ID:', txid)
  console.log('View on explorer:', `https://test.whatsonchain.com/tx/${txid}`)
  console.log('Status: Broadcast to network')
  console.log('Waiting for confirmation...')
}

// Usage
const result = await tx.broadcast()
await checkTransactionStatus(result.txid)
```

### Understanding Confirmations

**Confirmations** = number of blocks mined after your transaction's block

```
Your transaction broadcast
    |
    v
Included in block (1 confirmation)
    |
    v
Next block mined (2 confirmations)
    |
    v
Next block mined (3 confirmations)
... and so on
```

**Confirmation Guidelines**:
- **0 confirmations**: In mempool, waiting to be mined
- **1 confirmation**: Included in a block, generally safe
- **6+ confirmations**: Very secure, standard for larger amounts

### Polling for Confirmations

```typescript
async function waitForFirstConfirmation(
  txid: string,
  maxWaitMinutes: number = 30
): Promise<boolean> {
  const startTime = Date.now()
  const maxWaitMs = maxWaitMinutes * 60 * 1000

  console.log(`Waiting for transaction ${txid} to be confirmed...`)

  while (Date.now() - startTime < maxWaitMs) {
    // Check if transaction is in a block
    // In production: use overlay services for this
    const confirmed = await isTransactionConfirmed(txid)

    if (confirmed) {
      console.log('Transaction confirmed!')
      return true
    }

    // Wait 30 seconds before checking again
    await new Promise(resolve => setTimeout(resolve, 30000))
  }

  console.log('Timeout waiting for confirmation')
  return false
}

// Helper function (implement based on your infrastructure)
async function isTransactionConfirmed(txid: string): Promise<boolean> {
  // Use overlay services or block explorer APIs
  // Return true if transaction has at least 1 confirmation
  // This is application-specific
  return false // Placeholder
}
```

## Common Patterns

### Pattern 1: Simple Payment

**Backend**:
```typescript
const tx = new Transaction()
await tx.addInput({ sourceTransaction, sourceOutputIndex, unlockingScriptTemplate })
tx.addOutput({ satoshis, lockingScript })
await tx.fee()
await tx.sign()
await tx.broadcast()
```

**Frontend**:
```typescript
const wallet = await WalletClient.connect()
await wallet.createAction({
  outputs: [{ satoshis, script }],
  description: 'Payment'
})
```

### Pattern 2: Batch Payments

**Backend**:
```typescript
const tx = new Transaction()
await tx.addInput({ /* input */ })

for (const recipient of recipients) {
  tx.addOutput({
    satoshis: recipient.amount,
    lockingScript: new P2PKH().lock(recipient.address)
  })
}

await tx.fee()
await tx.sign()
await tx.broadcast()
```

**Frontend**:
```typescript
const wallet = await WalletClient.connect()
const outputs = recipients.map(r => ({
  satoshis: r.amount,
  script: new P2PKH().lock(r.address).toHex()
}))

await wallet.createAction({ outputs, description: 'Batch payment' })
```

### Pattern 3: Transaction with Data

You can embed data in transactions using OP_RETURN:

```typescript
import { Transaction, PrivateKey, P2PKH, OpReturn } from '@bsv/sdk'

// Backend: Transaction with embedded data
const tx = new Transaction()

await tx.addInput({ /* input */ })

// Payment output
tx.addOutput({
  satoshis: 10000,
  lockingScript: new P2PKH().lock(recipientAddress)
})

// Data output (OP_RETURN)
tx.addOutput({
  satoshis: 0,
  lockingScript: new OpReturn().lock(['Hello', 'BSV', 'Blockchain'])
})

await tx.fee()
await tx.sign()
await tx.broadcast()
```

## Error Handling

### Common Errors and Solutions

#### Error: "Insufficient funds"

**Cause**: Not enough satoshis to cover payment + fees

**Solution**:
```typescript
try {
  await tx.broadcast()
} catch (error) {
  if (error.message.includes('insufficient')) {
    console.error('Not enough funds. Need more BSV.')
    console.log('Get testnet BSV from MetaNet Desktop Wallet faucet or BSV Discord')
  }
}
```

#### Error: "Transaction broadcast failed"

**Cause**: Invalid transaction or network issue

**Solution**:
```typescript
try {
  const result = await tx.broadcast()
} catch (error) {
  if (error.message.includes('broadcast')) {
    // Verify transaction
    const isValid = await tx.verify()
    if (!isValid) {
      console.error('Transaction is invalid')
    } else {
      console.error('Network issue, retry broadcast')
    }
  }
}
```

#### Error: "UTXO already spent"

**Cause**: Trying to spend a UTXO that's already been used

**Solution**:
```typescript
// Always get fresh UTXOs before creating transaction
// Don't reuse UTXO references from previous transactions
```

### Best Practices for Error Handling

```typescript
async function sendPaymentWithErrorHandling(
  privateKeyWif: string,
  recipientAddress: string,
  amountSatoshis: number
): Promise<{ success: boolean; txid?: string; error?: string }> {
  try {
    const privateKey = PrivateKey.fromWif(privateKeyWif)
    const tx = new Transaction()

    // Build transaction
    await tx.addInput({
      sourceTransaction: utxo.txid,
      sourceOutputIndex: utxo.vout,
      unlockingScriptTemplate: new P2PKH().unlock(privateKey)
    })

    tx.addOutput({
      satoshis: amountSatoshis,
      lockingScript: new P2PKH().lock(recipientAddress)
    })

    // Calculate fee and sign
    await tx.fee()
    await tx.sign()

    // Verify before broadcasting
    const isValid = await tx.verify()
    if (!isValid) {
      return { success: false, error: 'Transaction validation failed' }
    }

    // Broadcast
    const result = await tx.broadcast()

    return { success: true, txid: result.txid }

  } catch (error) {
    return { success: false, error: error.message }
  }
}

// Usage with error handling
const result = await sendPaymentWithErrorHandling(wif, address, amount)

if (result.success) {
  console.log('Success! TXID:', result.txid)
} else {
  console.error('Failed:', result.error)
}
```

## Testing on Testnet

### Always Test First

```typescript
// Use testnet for development
const NETWORK = 'test'
// Get testnet BSV from MetaNet Desktop Wallet's built-in faucet
// or request from BSV Discord: https://discord.gg/bsv

console.log('Testing on testnet')
console.log('Get testnet BSV from MetaNet Desktop Wallet faucet or BSV Discord')
console.log('View transactions:', 'https://test.whatsonchain.com/')
```

### Testnet Checklist

Before moving to mainnet:
- [ ] Successfully send payment transaction
- [ ] Successfully send batch payment
- [ ] Handle insufficient funds error
- [ ] Handle invalid address error
- [ ] Verify transaction confirmation
- [ ] Test with different amounts
- [ ] Test change output behavior

## Practice Exercises

### Exercise 1: Simple Payment

Send 0.0001 BSV (10,000 satoshis) to another testnet address.

**Backend approach**:
```typescript
// TODO: Implement using Transaction class
// - Create transaction
// - Add input
// - Add output
// - Sign and broadcast
```

**Frontend approach**:
```typescript
// TODO: Implement using WalletClient
// - Connect wallet
// - Create action with output
// - Display result
```

### Exercise 2: Batch Payment

Send different amounts to 3 recipients in a single transaction.

```typescript
const recipients = [
  { address: 'address1...', amount: 5000 },
  { address: 'address2...', amount: 10000 },
  { address: 'address3...', amount: 15000 }
]

// TODO: Implement batch payment
```

### Exercise 3: Transaction Monitor

Create a function that monitors a transaction until it receives 3 confirmations.

```typescript
async function monitorTransaction(txid: string): Promise<void> {
  // TODO: Poll for confirmations
  // Display progress: 0, 1, 2, 3 confirmations
  // Complete when 3 confirmations reached
}
```

### Exercise 4: Error Recovery

Implement a payment function that handles common errors gracefully.

```typescript
async function robustPayment(
  wif: string,
  address: string,
  amount: number
): Promise<{ success: boolean; txid?: string; error?: string }> {
  // TODO: Implement with error handling
  // - Validate inputs
  // - Handle insufficient funds
  // - Handle broadcast failures
  // - Return meaningful error messages
}
```

## Key Takeaways

### What the SDK Does for You

- **Fee Calculation**: Automatically computes correct fees based on transaction size
- **Change Management**: Creates change outputs when needed
- **UTXO Selection**: Backend can select appropriate UTXOs (or use wallet for frontend)
- **Dust Limits**: Ensures all outputs meet minimum values
- **Broadcasting**: Handles network communication with miners
- **Verification**: Validates transactions before broadcast

### What You Focus On

- **Business Logic**: What payments to make and when
- **User Experience**: How users interact with transactions
- **Error Handling**: How to handle edge cases gracefully
- **Application Flow**: Integration with your app's workflow

### Backend vs Frontend

**Backend (Transaction class)**:
- Direct control over transaction building
- Manage your own UTXOs and keys
- Server-side or programmatic access
- Full flexibility

**Frontend (WalletClient)**:
- User's wallet handles everything
- Better security (keys stay in wallet)
- Simpler integration
- Standard user experience

## Related Components

- [Transaction](../../../sdk-components/transaction/README.md)
- [Transaction Input](../../../sdk-components/transaction-input/README.md)
- [Transaction Output](../../../sdk-components/transaction-output/README.md)
- [P2PKH](../../../sdk-components/p2pkh/README.md)
- <!-- [WalletClient](../../../sdk-components/wallet-client/README.md) (Component pending) -->

## Related Code Features

- [Transaction Creation](../../../code-features/transaction-creation/README.md)
- [Sign Transaction](../../../code-features/sign-transaction/README.md)
- [Broadcast with ARC](../../../code-features/broadcast-arc/README.md)

## Next Steps

Congratulations! You've completed the Beginner Learning Path. You now know how to:
- Set up a BSV development environment
- Understand BSV blockchain fundamentals
- Create and manage wallets
- Build, sign, and broadcast transactions using SDK methods

**Ready for more?** Continue to the [Intermediate Learning Path](../../intermediate/README.md) to learn about:
- Complex transaction patterns
- Custom Bitcoin Scripts
- SPV verification
- BRC standards implementation
- Overlay services integration

## Additional Resources

- [WhatsOnChain Testnet](https://test.whatsonchain.com/) - View your transactions
- [BSV Discord](https://discord.gg/bsv) - Get free testnet BSV and community support
- [MetaNet Desktop Wallet](https://desktop.bsvb.tech/) - Has built-in testnet faucet
- [Transaction Format](https://wiki.bitcoinsv.io/index.php/Transaction) - Technical details
- [BSV SDK Documentation](https://bsv-blockchain.github.io/ts-sdk/) - Complete SDK reference
