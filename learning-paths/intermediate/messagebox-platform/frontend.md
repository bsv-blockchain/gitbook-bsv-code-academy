# MessageBox Platform - Frontend Guide

**Building Browser-Based MessageBox Applications with BSV SDK**

This guide covers frontend patterns for implementing MessageBox functionality in browser-based applications using WalletClient, MessageBoxClient, and PeerPayClient.

> **📦 Complete Code Repository**: [https://github.com/bsv-blockchain-demos/messagebox-platform](https://github.com/bsv-blockchain-demos/messagebox-platform)
>
> All code examples in this guide are taken from the working implementation in the `frontend-next/` directory.

---

## Table of Contents

1. [Setup](#setup)
2. [Wallet Integration](#wallet-integration)
3. [Identity Certification](#identity-certification)
4. [Sending Payments](#sending-payments)
5. [Receiving Payments](#receiving-payments)
6. [Complete Integration](#complete-integration)

---

## Setup

### Installation

```bash
npm install @bsv/sdk @bsv/message-box-client
```

### Environment Configuration

Create `.env.local`:

```env
NEXT_PUBLIC_MESSAGEBOX_HOST=https://messagebox.babbage.systems
NEXT_PUBLIC_API_URL=http://localhost:3000
```

> **Note**: For Next.js apps, use `NEXT_PUBLIC_` prefix for client-side environment variables.

---

## Wallet Integration

### Pattern 1: Browser WalletClient

WalletClient connects to the user's BSV wallet (like MetaMask on Ethereum):

> **📁 See full implementation**: [`frontend-next/lib/walletClient.ts`](https://github.com/bsv-blockchain-demos/messagebox-platform/blob/main/frontend-next/lib/walletClient.ts)

```typescript
// lib/walletClient.ts

let walletClientInstance: any = null
let identityKey: string | null = null

/**
 * Connect to user's BSV wallet and get identity key
 * Prompts user via their wallet app
 */
export async function connectWallet(): Promise<{ identityKey: string }> {
  // Ensure we're in the browser (not SSR)
  if (typeof window === 'undefined') {
    throw new Error('WalletClient can only be used in the browser')
  }

  try {
    // Dynamically import to prevent SSR issues
    const { WalletClient } = await import('@bsv/sdk')

    // Create wallet client instance
    if (!walletClientInstance) {
      walletClientInstance = new WalletClient()
    }

    // Get identity key from wallet (prompts user)
    const result = await walletClientInstance.getPublicKey({
      identityKey: true
    })

    if (!result.publicKey) {
      throw new Error('Wallet returned no identity key')
    }

    identityKey = result.publicKey
    console.log('✓ Wallet connected:', identityKey.substring(0, 20) + '...')

    return { identityKey }
  } catch (error: any) {
    console.error('Failed to connect wallet:', error)
    throw new Error(`Failed to connect to wallet: ${error.message}`)
  }
}

/**
 * Get or create wallet client instance
 */
export async function ensureWalletClient(): Promise<any> {
  if (typeof window === 'undefined') {
    throw new Error('WalletClient can only be used in the browser')
  }

  if (walletClientInstance) {
    return walletClientInstance
  }

  const { WalletClient } = await import('@bsv/sdk')
  walletClientInstance = new WalletClient()

  return walletClientInstance
}

/**
 * Get stored identity key
 */
export function getIdentityKey(): string | null {
  return identityKey
}

/**
 * Clear wallet connection
 */
export function clearWallet(): void {
  walletClientInstance = null
  identityKey = null
}
```

**Key Points:**
- Identity key is a 33-byte compressed public key (hex string)
- User approves once via their BSV wallet app
- No private keys ever sent to server
- Same identity key works across all BSV apps

**Usage in Component:**

```typescript
import { connectWallet, getIdentityKey } from '@/lib/walletClient'

function ConnectButton() {
  const [connected, setConnected] = useState(false)
  const [identity, setIdentity] = useState<string | null>(null)

  const handleConnect = async () => {
    const { identityKey } = await connectWallet()
    setIdentity(identityKey)
    setConnected(true)
  }

  return (
    <button onClick={handleConnect}>
      {connected ? `Connected: ${identity?.substring(0, 20)}...` : 'Connect Wallet'}
    </button>
  )
}
```

> **Reference**: [WalletClient Integration](../../beginner/wallet-client-integration/README.md)

---

## Identity Certification

### Pattern 2: Blockchain Identity Certification

Create a blockchain transaction that certifies your identity with the MessageBox host:

> **📁 See full implementation**: [`frontend-next/lib/certificationClient.ts`](https://github.com/bsv-blockchain-demos/messagebox-platform/blob/main/frontend-next/lib/certificationClient.ts)

```typescript
// lib/certificationClient.ts

import { ensureWalletClient } from './walletClient'

const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000'

/**
 * Certify user identity on BSV blockchain via MessageBox
 * Creates an on-chain transaction proving identity registration
 */
export async function certifyIdentity(alias?: string): Promise<{
  success: boolean
  identityKey: string
  txid?: string
  alias?: string
  error?: string
}> {
  if (typeof window === 'undefined') {
    throw new Error('Certification can only be done in the browser')
  }

  try {
    console.log('Getting wallet client...')
    const walletClient = await ensureWalletClient()

    // Dynamically import MessageBoxClient
    const { MessageBoxClient } = await import('@bsv/message-box-client')

    const messageBoxHost = process.env.NEXT_PUBLIC_MESSAGEBOX_HOST ||
                          'https://messagebox.babbage.systems'

    console.log('Creating MessageBox client...')

    // Create MessageBoxClient with wallet
    const messageBoxClient = new MessageBoxClient({
      walletClient,
      host: messageBoxHost,
      enableLogging: true
    })

    await messageBoxClient.init()

    // Anoint host - creates certification transaction
    console.log('Certifying identity with host...')
    const result = await messageBoxClient.anointHost(messageBoxHost)

    console.log('✓ Certification complete. TXID:', result.txid)

    // Get identity key from wallet
    const keyResult = await walletClient.getPublicKey({ identityKey: true })
    const identityKey = keyResult.publicKey

    if (!identityKey) {
      throw new Error('Failed to retrieve identity key from wallet')
    }

    // Store certification on backend
    console.log('Storing certification on backend...')
    const response = await fetch(`${API_URL}/api/store-certification`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        identityKey,
        alias,
        txid: result.txid
      })
    })

    if (!response.ok) {
      const errorData = await response.json()
      throw new Error(errorData.error || 'Failed to store certification')
    }

    const data = await response.json()
    console.log('✓ Certification stored:', data)

    return {
      success: true,
      identityKey,
      txid: result.txid,
      alias
    }
  } catch (error: any) {
    console.error('Certification failed:', error)
    return {
      success: false,
      identityKey: '',
      error: error.message
    }
  }
}
```

**What `anointHost()` Does:**

1. Creates a BSV transaction with certification data
2. Signs transaction with user's identity key
3. Broadcasts to BSV network
4. Returns transaction ID (TXID) - permanent blockchain proof
5. Cost: ~1 satoshi for transaction fee

**Benefits:**
- ✅ Permanent, immutable proof of registration
- ✅ Timestamped by blockchain
- ✅ Can verify certification independently
- ✅ No central authority can revoke

**Usage in Component:**

```typescript
import { certifyIdentity } from '@/lib/certificationClient'

function CertificationPage() {
  const [alias, setAlias] = useState('')
  const [result, setResult] = useState<any>(null)

  const handleCertify = async () => {
    const result = await certifyIdentity(alias)
    if (result.success) {
      console.log('Certified! TXID:', result.txid)
      setResult(result)
    } else {
      console.error('Certification failed:', result.error)
    }
  }

  return (
    <div>
      <input
        value={alias}
        onChange={(e) => setAlias(e.target.value)}
        placeholder="Enter alias (optional)"
      />
      <button onClick={handleCertify}>Certify My Identity</button>

      {result?.success && (
        <div>
          <p>✓ Certified!</p>
          <p>Identity: {result.identityKey.substring(0, 20)}...</p>
          <p>TXID: {result.txid}</p>
          <p>Alias: {result.alias}</p>
        </div>
      )}
    </div>
  )
}
```

---

## Sending Payments

### Pattern 3: PeerPay - BRC-29 Payment Tokens

Create self-contained payment tokens with transaction + derivation instructions:

> **📁 See full implementation**: [`frontend-next/lib/paymentClient.ts`](https://github.com/bsv-blockchain-demos/messagebox-platform/blob/main/frontend-next/lib/paymentClient.ts)

```typescript
// lib/paymentClient.ts

import { ensureWalletClient } from './walletClient'

/**
 * Send a BSV payment to another user using PeerPay protocol
 *
 * @param recipient - Recipient's identity key (public key)
 * @param amount - Amount in satoshis
 */
export async function sendPayment(
  recipient: string,
  amount: number
): Promise<{
  success: boolean
  messageId?: string
  error?: string
}> {
  if (typeof window === 'undefined') {
    throw new Error('Payments can only be sent from the browser')
  }

  try {
    // Import PeerPayClient
    const { PeerPayClient } = await import('@bsv/message-box-client')

    // Get wallet client
    const walletClient = await ensureWalletClient()

    const messageBoxHost = process.env.NEXT_PUBLIC_MESSAGEBOX_HOST ||
                          'https://messagebox.babbage.systems'

    console.log('Creating PeerPay client...')

    // Create PeerPay client
    const peerPayClient = new PeerPayClient({
      walletClient,
      messageBoxHost,
      enableLogging: true
    })

    console.log(`Sending payment: ${amount} satoshis to ${recipient}`)

    // Create payment token (builds + broadcasts transaction)
    const paymentToken = await peerPayClient.createPaymentToken({
      recipient,
      amount
    })

    console.log('✓ Payment token created:', paymentToken.transaction)

    // Note: Sender's wallet automatically tracks change outputs
    // No manual internalization needed for sender

    // Send payment token to recipient via MessageBox
    await peerPayClient.sendMessage({
      recipient,
      messageBox: 'payment_inbox',
      body: JSON.stringify(paymentToken)
    })

    console.log('Payment sent successfully!')

    return {
      success: true,
      messageId: 'Payment sent successfully'
    }
  } catch (error: any) {
    console.error('Failed to send payment:', error)
    throw new Error(`Failed to send payment: ${error.message}`)
  }
}
```

**Payment Token Structure:**

```typescript
interface PaymentToken {
  // Serialized transaction in BEEF format
  transaction: string

  // BRC-29 key derivation instructions
  customInstructions: {
    derivationPrefix: string  // Base64 encoded
    derivationSuffix: string  // Base64 encoded
  }

  // Payment amount in satoshis
  amount: number

  // Which output is the payment (usually 0)
  outputIndex: number
}
```

**Transaction Output Layout:**

```
Transaction:
├─ Inputs: Sender's UTXOs (e.g., 15000 satoshis)
├─ Output[0]: Payment to recipient (10000 satoshis)
│             Address: derived(recipientKey, derivationPrefix, derivationSuffix)
└─ Output[1]: Change back to sender (4999 satoshis)
              Address: derived from sender's keys
```

**Key Points:**
- Transaction is broadcast immediately by `createPaymentToken()`
- Payment is on-chain before message is sent
- Recipient gets both transaction + derivation instructions
- Enables recipient to derive the correct private key for spending

**Usage in Component:**

```typescript
import { sendPayment } from '@/lib/paymentClient'

function SendPaymentForm() {
  const [recipient, setRecipient] = useState('')
  const [amount, setAmount] = useState('')

  const handleSend = async () => {
    try {
      const result = await sendPayment(recipient, parseInt(amount))
      if (result.success) {
        console.log('Payment sent!')
      }
    } catch (error) {
      console.error('Payment failed:', error)
    }
  }

  return (
    <div>
      <input
        value={recipient}
        onChange={(e) => setRecipient(e.target.value)}
        placeholder="Recipient identity key"
      />
      <input
        type="number"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        placeholder="Amount (satoshis)"
      />
      <button onClick={handleSend}>Send Payment</button>
    </div>
  )
}
```

---

## Receiving Payments

### Pattern 4: Message Box Protocol & Internalization

Check inbox for encrypted payment messages and internalize them to wallet:

> **📁 See full implementation**: [`frontend-next/lib/messageClient.ts`](https://github.com/bsv-blockchain-demos/messagebox-platform/blob/main/frontend-next/lib/messageClient.ts)

```typescript
// lib/messageClient.ts

import { ensureWalletClient } from './walletClient'

export interface ReceivedMessage {
  messageId: string
  sender: string
  messageBox: string
  body: any
  timestamp: Date
}

/**
 * Safely parse a message body into a payment token
 */
function parsePaymentToken(body: string | Record<string, any>): Record<string, any> | null {
  try {
    const parsed = typeof body === 'string' ? JSON.parse(body) : body
    if (parsed && parsed.transaction && parsed.customInstructions) {
      return parsed
    }
    return null
  } catch {
    return null
  }
}

/**
 * Check inbox for incoming payments and internalize them
 */
export async function checkInbox(
  messageBox: string = 'payment_inbox'
): Promise<ReceivedMessage[]> {
  if (typeof window === 'undefined') {
    throw new Error('Message retrieval can only be done in the browser')
  }

  try {
    // Import PeerPayClient
    const { PeerPayClient } = await import('@bsv/message-box-client')

    // Get wallet client
    const walletClient = await ensureWalletClient()

    const messageBoxHost = process.env.NEXT_PUBLIC_MESSAGEBOX_HOST ||
                          'https://messagebox.babbage.systems'

    console.log(`Checking inbox: ${messageBox}`)

    // Create PeerPay client
    const peerPayClient = new PeerPayClient({
      walletClient,
      messageBoxHost,
      enableLogging: true
    })

    await peerPayClient.init()

    // List messages from message box (auto-decrypts)
    console.log('Calling listMessages...')
    const messages = await peerPayClient.listMessages({ messageBox })

    if (!messages || !Array.isArray(messages)) {
      console.log('No messages found')
      return []
    }

    console.log(`Found ${messages.length} message(s)`)

    // Process and internalize each payment
    const processedMessages = []

    for (const msg of messages) {
      try {
        // Parse message body as payment token
        const token = parsePaymentToken(msg.body)

        if (token && token.transaction) {
          console.log('Internalizing payment from:', msg.sender)

          // CRITICAL: Internalize the payment
          // This tells wallet how to derive the spending key
          await walletClient.internalizeAction({
            tx: token.transaction,
            outputs: [{
              outputIndex: token.outputIndex ?? 0,
              protocol: 'wallet payment',

              // BRC-29 payment remittance
              paymentRemittance: {
                senderIdentityKey: msg.sender,
                derivationPrefix: token.customInstructions.derivationPrefix,
                derivationSuffix: token.customInstructions.derivationSuffix
              }
            }],
            description: `Payment from ${msg.sender.substring(0, 20)}...`,
            labels: ['received_payment']
          })

          console.log('✓ Transaction internalized successfully')
        }
      } catch (internError: any) {
        console.error('Failed to internalize transaction:', internError)
        // Continue processing other messages
      }

      processedMessages.push({
        messageId: msg.messageId || 'unknown',
        sender: msg.sender || 'unknown',
        messageBox: messageBox,
        body: msg.body || msg,
        timestamp: msg.created_at ? new Date(msg.created_at) : new Date()
      })
    }

    return processedMessages
  } catch (error: any) {
    console.error('Failed to check inbox:', error)
    throw new Error(`Failed to check inbox: ${error.message}`)
  }
}

/**
 * Acknowledge and remove a message from the server
 */
export async function acknowledgeMessage(
  messageId: string,
  _messageBox: string = 'payment_inbox'
): Promise<void> {
  if (typeof window === 'undefined') {
    throw new Error('Message acknowledgment can only be done in the browser')
  }

  try {
    const { PeerPayClient } = await import('@bsv/message-box-client')
    const walletClient = await ensureWalletClient()

    const messageBoxHost = process.env.NEXT_PUBLIC_MESSAGEBOX_HOST ||
                          'https://messagebox.babbage.systems'

    console.log(`Acknowledging message: ${messageId}`)

    const peerPayClient = new PeerPayClient({
      walletClient,
      messageBoxHost,
      enableLogging: true
    })

    await peerPayClient.init()

    // Acknowledge message (removes from server)
    await peerPayClient.acknowledgeMessage({ messageIds: [messageId] })

    console.log('✓ Message acknowledged')
  } catch (error: any) {
    console.error('Failed to acknowledge message:', error)
    throw new Error(`Failed to acknowledge message: ${error.message}`)
  }
}
```

**Why Internalization is Required:**

```typescript
// Scenario: Alice sends 10000 satoshis to Bob

// 1. Alice creates transaction:
//    Output[0]: 10000 sats → derived_address(Bob's key, prefix, suffix)

// 2. Bob's wallet doesn't automatically know about this UTXO
//    Problem: Bob's wallet doesn't have derivation instructions

// 3. Alice sends payment token via MessageBox with derivation instructions

// 4. Bob calls internalizeAction():
await walletClient.internalizeAction({
  tx: token.transaction,
  outputs: [{
    outputIndex: 0,
    protocol: 'wallet payment',
    paymentRemittance: {
      senderIdentityKey: Alice_public_key,
      derivationPrefix: prefix,
      derivationSuffix: suffix
    }
  }]
})

// 5. Bob's wallet now:
//    a. Derives private key: derive(Bob_private_key, Alice_public_key, prefix, suffix)
//    b. Verifies this key can unlock output[0]
//    c. Adds UTXO to Bob's spendable balance
//    d. Bob can now spend these satoshis
```

**Usage in Component:**

```typescript
import { checkInbox, acknowledgeMessage } from '@/lib/messageClient'

function InboxPage() {
  const [messages, setMessages] = useState<ReceivedMessage[]>([])

  // Auto-check inbox every 45 seconds
  useEffect(() => {
    const pollInbox = async () => {
      const msgs = await checkInbox('payment_inbox')
      setMessages(msgs)
    }

    pollInbox()
    const interval = setInterval(pollInbox, 45000)
    return () => clearInterval(interval)
  }, [])

  const handleAcknowledge = async (messageId: string) => {
    await acknowledgeMessage(messageId)
    setMessages(messages.filter(m => m.messageId !== messageId))
  }

  return (
    <div>
      <h2>Payment Inbox</h2>
      {messages.map(msg => (
        <div key={msg.messageId}>
          <p>From: {msg.sender.substring(0, 20)}...</p>
          <p>Amount: {msg.body.amount} satoshis</p>
          <p>Time: {msg.timestamp.toLocaleString()}</p>
          <button onClick={() => handleAcknowledge(msg.messageId)}>
            Acknowledge & Remove
          </button>
        </div>
      ))}
    </div>
  )
}
```

---

## Complete Integration

### Full Payment Flow Example

```typescript
// ============================================
// SENDER SIDE
// ============================================

// 1. User connects wallet (one-time)
import { connectWallet } from '@/lib/walletClient'

const { identityKey } = await connectWallet()
// → Wallet prompts user for approval
// → Returns identity key (public key)

// 2. User selects recipient and amount
const recipientKey = "03def456..."
const amount = 10000  // satoshis

// 3. Send payment
import { sendPayment } from '@/lib/paymentClient'

await sendPayment(recipientKey, amount)
// → Creates PeerPayClient
// → Builds transaction (sender UTXO → recipient + change)
// → Signs with wallet (no new approval needed)
// → Broadcasts to BSV network
// → Sends payment token via MessageBox

// ============================================
// RECIPIENT SIDE
// ============================================

// 1. Auto-check inbox (polling)
import { checkInbox, acknowledgeMessage } from '@/lib/messageClient'

const messages = await checkInbox('payment_inbox')
// → Creates PeerPayClient
// → Lists messages (auto-decrypts)
// → For each payment:
//   • Parses payment token
//   • Calls walletClient.internalizeAction()
//   • Wallet derives spending key
//   • Adds UTXO to balance

// 2. Display messages
messages.forEach(msg => {
  console.log(`Payment: ${msg.body.amount} sats from ${msg.sender}`)
})

// 3. Acknowledge message
await acknowledgeMessage(messages[0].messageId)
// → Message deleted from server (ephemeral)

// ============================================
// BLOCKCHAIN STATE
// ============================================

// Transaction confirmed on BSV:
// TXID: abc123...
// Inputs:  Sender UTXO (15000 sats)
// Output[0]: 10000 sats → Recipient (derived address)
// Output[1]: 4999 sats → Sender (change)
// Fee: 1 sat

// Recipient's wallet:
// • Has private key for output[0]
// • Can spend 10000 satoshis
// • Tracked as "received_payment"
```

---

## Summary

This frontend guide covered:

- **WalletClient Integration** - Connecting to user wallets in the browser
- **Identity Certification** - Creating blockchain proofs with `anointHost()`
- **PeerPay Payments** - Sending BRC-29 payment tokens with `createPaymentToken()`
- **Message Handling** - Receiving and decrypting messages with `listMessages()`
- **Transaction Internalization** - Adding payments to wallet with `internalizeAction()`

These patterns form the foundation for building peer-to-peer payment and messaging applications on BSV.

---

## Next Steps

- [Server Implementation Guide](./server.md) - Learn backend patterns
- [Overlay Services](../overlay-services/README.md) - Deploy custom MessageBox networks
- [BRC-29 Documentation](https://brc.dev/29) - Deep dive into payment addressing
- [MessageBox Protocol](https://github.com/bitcoin-sv/message-box) - Protocol specification
