# BSV MessageBox Platform - Developer Guide

## Table of Contents
1. [Overview](#overview)
2. [BSV Key Patterns](#bsv-key-patterns)
3. [System Architecture](#system-architecture)
4. [Data Flow & User Journeys](#data-flow--user-journeys)
5. [Why BSV?](#why-bsv)

---

## Overview

### What is the MessageBox Platform?

The **MessageBox Platform** is a complete web application demonstrating production-ready BSV blockchain integration patterns. It enables:

- **Identity Certification**: Register your identity on the BSV blockchain via the MessageBox network
- **Peer-to-Peer Payments**: Send satoshis directly between users with BRC-29 compliant key derivation
- **Session Management**: Persistent wallet connections that survive page reloads
- **Message Handling**: Encrypted payment notifications with automatic inbox polling

### Tech Stack

```
Backend:  Node.js + Express + TypeScript + MongoDB
Frontend: Next.js 15 + React 19 + Tailwind CSS
BSV:      @bsv/sdk ^1.1.58 + @bsv/message-box-client ^1.4.5
```

### Key BSV Concepts Used

1. **Identity Keys**: BSV public keys as user identities (no usernames/passwords)
2. **PeerPay Protocol**: Standardized payment token structure with derivation instructions
3. **MessageBox Network**: Encrypted, ephemeral message delivery between identities
4. **BRC-29**: Key derivation protocol for payment addressing
5. **WalletClient**: Browser-based signing without exposing private keys
6. **Transaction Internalization**: Adding received payments to wallet balance

---

## BSV Key Patterns

### Pattern 1: Identity Management with Public Keys

**Concept**: In BSV, your identity IS your public key. No separate authentication system needed.

**Implementation** (`frontend-next/lib/walletClient.ts`):

```typescript
/**
 * Browser-based WalletClient wrapper
 * Handles BSV wallet connection in the browser
 */
let walletClientInstance: any = null
let identityKey: string | null = null

export async function connectWallet(): Promise<{ identityKey: string }> {
  if (typeof window === 'undefined') {
    throw new Error('WalletClient can only be used in the browser')
  }

  // Dynamically import WalletClient only in browser
  const { WalletClient } = await import('@bsv/sdk')

  // Create wallet client in browser
  if (!walletClientInstance) {
    walletClientInstance = new WalletClient()
  }

  // Get identity key from wallet (prompts user's wallet app)
  const result = await walletClientInstance.getPublicKey({ identityKey: true })

  if (!result.publicKey) {
    throw new Error('Wallet returned no identity key')
  }

  identityKey = result.publicKey
  console.log('✓ Wallet connected:', identityKey.substring(0, 20) + '...')

  return { identityKey }
}
```

**Key Points**:
- Identity key is a 33-byte compressed public key (hex string)
- User approves once via their BSV wallet app (like MetaMask on Ethereum)
- No private keys ever sent to server
- Same identity key used across all BSV apps (interoperable)

**Benefits vs Traditional Auth**:
- ✅ No password management
- ✅ No database of credentials to protect
- ✅ Cryptographically provable identity
- ✅ Works across all BSV applications

---

### Pattern 2: Blockchain Identity Certification

**Concept**: Create a blockchain transaction that certifies your identity with a specific host/service.

**Implementation** (`frontend-next/lib/certificationClient.ts`):

```typescript
export async function certifyIdentity(alias?: string): Promise<{
  success: boolean
  identityKey: string
  txid?: string
  alias?: string
  error?: string
}> {
  try {
    // 1. Get wallet client instance
    const walletClient = await ensureWalletClient()

    // 2. Create MessageBoxClient with wallet
    const { MessageBoxClient } = await import('@bsv/message-box-client')
    const messageBoxHost = process.env.NEXT_PUBLIC_MESSAGEBOX_HOST ||
                          'https://messagebox.babbage.systems'

    const messageBoxClient = new MessageBoxClient({
      walletClient,
      host: messageBoxHost,
      enableLogging: true
    })

    await messageBoxClient.init()

    // 3. Anoint host (creates blockchain certification transaction)
    console.log('Certifying identity with host...')
    const result = await messageBoxClient.anointHost(messageBoxHost)
    console.log('✓ Certification complete. TXID:', result.txid)

    // 4. Get identity key
    const keyResult = await walletClient.getPublicKey({ identityKey: true })
    const identityKey = keyResult.publicKey

    // 5. Store certification on backend
    const response = await fetch(`${API_URL}/api/store-certification`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ identityKey, alias, txid: result.txid })
    })

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
      error: error.message
    }
  }
}
```

**What `anointHost()` Does**:
1. Creates a BSV transaction with certification data
2. Signs transaction with user's identity key
3. Broadcasts to BSV network
4. Returns transaction ID (TXID) - permanent blockchain proof
5. Cost: ~1 satoshi for transaction fee

**Benefits**:
- ✅ Permanent, immutable proof of registration
- ✅ Timestamped by blockchain
- ✅ Can verify certification independently
- ✅ No central authority can revoke

---

### Pattern 3: PeerPay - BRC-29 Payment Tokens

**Concept**: Create a self-contained payment token that includes transaction + derivation instructions.

**Implementation** (`frontend-next/lib/paymentClient.ts`):

```typescript
export async function sendPayment(
  recipient: string,
  amount: number
): Promise<{
  success: boolean
  messageId?: string
  error?: string
}> {
  try {
    // 1. Import PeerPayClient
    const { PeerPayClient } = await import('@bsv/message-box-client')
    const walletClient = await ensureWalletClient()

    const messageBoxHost = process.env.NEXT_PUBLIC_MESSAGEBOX_HOST ||
                          'https://messagebox.babbage.systems'

    // 2. Create PeerPay client
    const peerPayClient = new PeerPayClient({
      walletClient,
      messageBoxHost,
      enableLogging: true
    })

    console.log(`Sending payment: ${amount} satoshis to ${recipient}`)

    // 3. Create payment token (builds + broadcasts transaction)
    // randomizeOutputs: false means:
    //   output[0] = payment to recipient
    //   output[1+] = change back to sender
    const paymentToken = await peerPayClient.createPaymentToken({
      recipient,
      amount
    })

    console.log('✓ Payment token created:', paymentToken.transaction)

    // Note: Change outputs are automatically tracked by sender's wallet
    // Only recipient needs to internalize their payment

    // 4. Send payment token to recipient via MessageBox
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

**Payment Token Structure**:

```typescript
{
  transaction: string,           // Serialized BSV transaction (BEEF format)
  customInstructions: {
    derivationPrefix: string,    // Base64 encoded prefix
    derivationSuffix: string     // Base64 encoded suffix
  },
  amount: number,                // Payment amount in satoshis
  outputIndex: number            // Usually 0 (recipient output)
}
```

**Transaction Output Layout** (with `randomizeOutputs: false`):

```
Transaction:
├─ Inputs: Sender's UTXOs
├─ Output[0]: Payment to recipient
│             Address: derived(recipientKey, derivationPrefix, derivationSuffix)
│             Amount: specified amount in satoshis
└─ Output[1+]: Change back to sender
              Address: derived from sender's keys
              Amount: input - payment - fees
```

**Key Points**:
- Transaction is **broadcast immediately** by `createPaymentToken()`
- Payment is on-chain before message is sent
- Recipient gets both transaction + derivation instructions
- Enables recipient to derive the correct private key for spending

---

### Pattern 4: Message Box Protocol

**Concept**: Encrypted, ephemeral message delivery between BSV identities.

**Sending Messages**:

```typescript
// Already shown in Pattern 3
await peerPayClient.sendMessage({
  recipient: recipientIdentityKey,
  messageBox: 'payment_inbox',
  body: JSON.stringify(paymentToken)
})
```

**What Happens**:
1. Message encrypted with recipient's public key (ECIES)
2. Stored on MessageBox server under `payment_inbox`
3. Only recipient can decrypt (has private key)
4. Ephemeral storage - deleted when acknowledged

**Receiving Messages** (`frontend-next/lib/messageClient.ts`):

```typescript
export async function checkInbox(
  messageBox: string = 'payment_inbox'
): Promise<ReceivedMessage[]> {
  try {
    const { PeerPayClient } = await import('@bsv/message-box-client')
    const walletClient = await ensureWalletClient()

    const messageBoxHost = process.env.NEXT_PUBLIC_MESSAGEBOX_HOST ||
                          'https://messagebox.babbage.systems'

    // 1. Create PeerPay client
    const peerPayClient = new PeerPayClient({
      walletClient,
      messageBoxHost,
      enableLogging: true
    })

    await peerPayClient.init()

    // 2. List messages from inbox (auto-decrypts)
    const messages = await peerPayClient.listMessages({ messageBox })

    if (!messages || !Array.isArray(messages)) {
      return []
    }

    // 3. Process each payment message
    const processedMessages = []
    for (const msg of messages) {
      try {
        // Parse payment token from message body
        const token = parsePaymentToken(msg.body)

        if (token && token.transaction) {
          console.log('Internalizing payment from:', msg.sender)

          // 4. Internalize payment (add to wallet balance)
          await walletClient.internalizeAction({
            tx: token.transaction,
            outputs: [{
              outputIndex: token.outputIndex ?? 0,
              protocol: 'wallet payment',
              paymentRemittance: {
                senderIdentityKey: msg.sender,
                derivationPrefix: token.customInstructions.derivationPrefix,
                derivationSuffix: token.customInstructions.derivationSuffix
              }
            }],
            description: `Payment from ${msg.sender.substring(0, 20)}...`,
            labels: ['received_payment']
          })

          console.log('Transaction internalized successfully')
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
```

**Acknowledging Messages** (removes from server):

```typescript
export async function acknowledgeMessage(
  messageId: string,
  _messageBox: string = 'payment_inbox'
): Promise<void> {
  const { PeerPayClient } = await import('@bsv/message-box-client')
  const walletClient = await ensureWalletClient()

  const peerPayClient = new PeerPayClient({
    walletClient,
    messageBoxHost: process.env.NEXT_PUBLIC_MESSAGEBOX_HOST,
    enableLogging: true
  })

  await peerPayClient.init()
  await peerPayClient.acknowledgeMessage({ messageIds: [messageId] })

  console.log('Message acknowledged successfully')
}
```

**Key Points**:
- End-to-end encryption (only recipient can read)
- Server never sees message contents
- Ephemeral by design (privacy-focused)
- No blockchain storage needed for messages

---

### Pattern 5: Transaction Internalization

**Concept**: When receiving a BRC-29 payment, tell your wallet about it so it can derive the spending key.

**Why Internalization is Needed**:

1. **Sender** creates transaction with output locked to: `derived_key(recipientPubKey, prefix, suffix)`
2. **Recipient's wallet** doesn't automatically know about this UTXO
3. **Internalization** provides wallet with derivation instructions
4. **Wallet** can now derive the private key to spend this UTXO

**Implementation Details**:

```typescript
await walletClient.internalizeAction({
  // The transaction containing the payment
  tx: token.transaction,  // BEEF format

  // Which outputs belong to me
  outputs: [{
    outputIndex: 0,  // First output in transaction

    protocol: 'wallet payment',  // Standard protocol identifier

    // BRC-29 payment remittance information
    paymentRemittance: {
      senderIdentityKey: msg.sender,  // Who sent this payment

      // Derivation instructions from payment token
      derivationPrefix: token.customInstructions.derivationPrefix,
      derivationSuffix: token.customInstructions.derivationSuffix
    }
  }],

  // Metadata for wallet tracking
  description: `Payment from ${msg.sender.substring(0, 20)}...`,
  labels: ['received_payment']
})
```

**What Happens Inside Wallet**:

1. Wallet receives transaction + derivation instructions
2. Derives private key: `derive(myPrivateKey, senderPubKey, prefix, suffix)`
3. Verifies this key can unlock output[0]
4. Adds UTXO to wallet's spendable balance
5. Can now spend in future transactions

**Important Distinction**:

```typescript
// ❌ WRONG: Change outputs (sender's wallet)
// Sender's wallet auto-tracks change outputs from createPaymentToken()
// Do NOT internalize change - it causes BRC-29 validation errors

// ✅ CORRECT: Payment output (recipient's wallet)
// Recipient MUST internalize to add payment to balance
```

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         BROWSER (CLIENT)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐      │
│  │   Certify     │  │  Send Payment │  │    Receive    │      │
│  │   Page (/)    │  │  (/payments)  │  │  (/receive)   │      │
│  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘      │
│          │                  │                  │               │
│          └──────────────────┴──────────────────┘               │
│                             │                                  │
│                  ┌──────────▼──────────┐                       │
│                  │   WalletProvider    │                       │
│                  │   (React Context)   │                       │
│                  └──────────┬──────────┘                       │
│                             │                                  │
│          ┌──────────────────┼──────────────────┐               │
│          │                  │                  │               │
│   ┌──────▼──────┐  ┌────────▼────────┐  ┌─────▼──────┐        │
│   │certifyClient│  │ paymentClient   │  │messageClient│       │
│   └──────┬──────┘  └────────┬────────┘  └─────┬──────┘        │
│          │                  │                  │               │
│   ┌──────▼──────────────────▼──────────────────▼──────┐        │
│   │           walletClient.ts                         │        │
│   │     (Browser WalletClient Instance)               │        │
│   └──────┬────────────────────────────────────────────┘        │
│          │                                                     │
└──────────┼─────────────────────────────────────────────────────┘
           │
           │ HTTPS API Calls
           │
┌──────────▼─────────────────────────────────────────────────────┐
│                    EXPRESS SERVER (Node.js)                    │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                    API Routes                           │  │
│  ├─────────────────────────────────────────────────────────┤  │
│  │  POST   /api/wallet/connect                            │  │
│  │  GET    /api/wallet/status                             │  │
│  │  POST   /api/wallet/disconnect                         │  │
│  │  POST   /api/certify                                   │  │
│  │  POST   /api/store-certification                       │  │
│  │  GET    /api/certified-users                           │  │
│  │  GET    /api/certified-users/search?q=query            │  │
│  │  POST   /api/initiate-payment                          │  │
│  └─────────────────┬───────────────────────────────────────┘  │
│                    │                                          │
│  ┌─────────────────▼───────────────────────────────────────┐  │
│  │         WalletSessionManager (In-Memory)               │  │
│  │  • 30-minute session timeout                           │  │
│  │  • Auto-cleanup every 5 minutes                        │  │
│  │  • Stores: { sessionId, walletClient, identityKey }   │  │
│  └─────────────────┬───────────────────────────────────────┘  │
│                    │                                          │
│  ┌─────────────────▼───────────────────────────────────────┐  │
│  │         CertificationStorage (MongoDB)                  │  │
│  │  Collection: certified_users                           │  │
│  │  Schema: { identityKey, alias, txid, date, host }     │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
           │
           │ BSV SDK Calls
           │
┌──────────▼─────────────────────────────────────────────────────┐
│                    BSV INFRASTRUCTURE                          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌───────────────────────────────────────────────────────┐    │
│  │           MessageBox Network                          │    │
│  │   (messagebox.babbage.systems)                        │    │
│  │                                                       │    │
│  │  • Identity certification (anointHost)               │    │
│  │  • Encrypted message delivery                        │    │
│  │  • Payment inbox management                          │    │
│  │  • Ephemeral storage                                 │    │
│  └───────────────────────────────────────────────────────┘    │
│                                                                │
│  ┌───────────────────────────────────────────────────────┐    │
│  │            BSV Blockchain Network                     │    │
│  │                                                       │    │
│  │  • Transaction broadcasting                          │    │
│  │  • Transaction confirmation                          │    │
│  │  • UTXO management                                   │    │
│  │  • Permanent data storage                            │    │
│  └───────────────────────────────────────────────────────┘    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Component Interaction Flow

```
┌─────────────┐                                    ┌─────────────┐
│   Browser   │                                    │   Server    │
│  Wallet App │                                    │  (Express)  │
└──────┬──────┘                                    └──────┬──────┘
       │                                                  │
       │  1. User clicks "Connect Wallet"                │
       │◄────────────────────────────────────────────────┤
       │                                                  │
       │  2. Wallet prompts for approval                 │
       │     (Show public key?)                          │
       │                                                  │
       │  3. User approves                               │
       │──────────────────────────────────────────────►  │
       │     Returns: identityKey (public key)           │
       │                                                  │
       │  4. POST /api/wallet/connect                    │
       │     { identityKey }                             │
       ├─────────────────────────────────────────────────►
       │                                                  │
       │  5. Server creates session                      │
       │     sessionId = randomBytes(32)                 │
       │◄─────────────────────────────────────────────────┤
       │     Returns: { sessionId }                      │
       │                                                  │
       │  6. Store in localStorage                       │
       │     wallet_session_id = sessionId               │
       │     wallet_identity_key = identityKey           │
       │                                                  │
       │  7. User sends payment (later)                  │
       │     PeerPayClient.createPaymentToken()          │
       │                                                  │
       │  8. Wallet signs transaction                    │
       │     (No new approval needed)                    │
       │──────────────────────────────────────────────►  │
       │     Transaction broadcast to BSV network        │
       │                                                  │
       │  9. Send payment token via MessageBox           │
       │     Encrypted message to recipient              │
       │                                                  │
       │  10. Recipient checks inbox                     │
       │      PeerPayClient.listMessages()               │
       │◄─────────────────────────────────────────────────┤
       │      Returns: [{ messageId, body, sender }]     │
       │                                                  │
       │  11. Internalize payment                        │
       │      walletClient.internalizeAction()           │
       │      Balance updated                            │
       │                                                  │
       │  12. Acknowledge message                        │
       │      PeerPayClient.acknowledgeMessage()         │
       ├─────────────────────────────────────────────────►
       │      Message deleted from server                │
       │                                                  │
└──────┴──────┘                                    └──────┴──────┘
```

---

## Data Flow & User Journeys

### Journey 1: Identity Certification

```
Step 1: User Opens Certification Page (/)
├─ Component: app/page.tsx
├─ UI: Optional alias input + "Certify My Identity" button
└─ State: Not yet connected to wallet

Step 2: User Clicks "Certify My Identity"
├─ Calls: certificationClient.certifyIdentity(alias)
├─ Browser: ensureWalletClient()
│   └─ Creates new WalletClient() instance
│   └─ No wallet approval yet (deferred)
└─ Creates MessageBoxClient with walletClient

Step 3: Wallet App Prompts User
├─ MessageBoxClient.anointHost() called
├─ Wallet app shows: "Allow access to identity key?"
├─ User clicks "Allow"
└─ Wallet returns public key (identityKey)

Step 4: Transaction Created & Broadcast
├─ MessageBoxClient creates certification transaction
├─ Transaction signed with identity private key
├─ Broadcast to BSV network
├─ Returns: { txid: "abc123..." }
└─ Cost: ~1 satoshi

Step 5: Store Certification
├─ Frontend calls: POST /api/store-certification
├─ Body: { identityKey, alias, txid }
├─ Server stores in MongoDB:
│   {
│     identityKey: "03abc...",
│     alias: "Alice",
│     txid: "abc123...",
│     certificationDate: new Date(),
│     host: "messagebox.babbage.systems"
│   }
└─ Returns: { success: true }

Step 6: Display Confirmation
├─ UI shows success message
├─ Displays:
│   • Identity Key: 03abc...
│   • Alias: Alice
│   • TXID: abc123... (link to blockchain explorer)
└─ User can now send/receive payments
```

### Journey 2: Sending a Payment

```
Step 1: User Opens Send Payment Page (/payments)
├─ Component: app/payments/page.tsx
├─ Auto-loads certified users: GET /api/certified-users
└─ Displays user cards in grid

Step 2: User Searches for Recipient
├─ Types in search bar: "Alice"
├─ Calls: GET /api/certified-users/search?q=Alice
├─ Filters list (case-insensitive regex)
└─ Updates grid display

Step 3: User Selects Recipient
├─ Clicks on user card
├─ Card highlights in purple
├─ Shows payment form:
│   • Amount input (satoshis)
│   • "Send Payment" button
└─ State: { selectedUser: { identityKey, alias } }

Step 4: User Enters Amount
├─ Types: "10000" (satoshis)
├─ Validation: amount > 0
└─ State: { amount: 10000 }

Step 5: User Clicks "Send Payment"
├─ Calls: paymentClient.sendPayment(identityKey, amount)
├─ Creates PeerPayClient with walletClient
└─ ensureWalletClient() creates instance

Step 6: Create Payment Token
├─ PeerPayClient.createPaymentToken({ recipient, amount })
├─ Wallet builds transaction:
│   Inputs:  Sender's UTXOs (enough to cover amount + fees)
│   Output[0]: 10000 satoshis → recipient derived address
│   Output[1]: Change → sender address
│   Fees: ~1 satoshi
├─ Wallet signs transaction
├─ Broadcasts to BSV network
└─ Returns payment token:
    {
      transaction: "beef_encoded_tx",
      customInstructions: {
        derivationPrefix: "a2V5SUQ=",
        derivationSuffix: "c29tZXRoaW5n"
      },
      amount: 10000,
      outputIndex: 0
    }

Step 7: Send Token via MessageBox
├─ PeerPayClient.sendMessage()
├─ Encrypts token with recipient's public key
├─ Sends to MessageBox server
├─ Stored in recipient's "payment_inbox"
└─ Returns: { status: "ok", messageId: "msg_123" }

Step 8: Display Success
├─ UI shows green success message
├─ "Payment sent successfully to Alice"
├─ Form resets
└─ User can send another payment (no re-approval)
```

### Journey 3: Receiving a Payment

```
Step 1: User Opens Receive Page (/receive)
├─ Component: app/receive/page.tsx
├─ Auto-starts polling (useEffect on mount)
└─ Interval: 45 seconds

Step 2: Auto-Check Inbox
├─ Calls: messageClient.checkInbox('payment_inbox')
├─ Creates PeerPayClient
├─ Calls: peerPayClient.listMessages({ messageBox: 'payment_inbox' })
├─ MessageBox server:
│   • Decrypts messages with recipient's private key
│   • Returns array of messages
└─ Browser receives:
    [{
      messageId: "msg_123",
      sender: "03def...",
      body: { transaction, customInstructions, amount },
      created_at: "2025-02-02T10:30:00Z"
    }]

Step 3: Process Payment Messages
├─ For each message:
│   ├─ Parse body as payment token
│   ├─ Extract: transaction, customInstructions
│   └─ Internalize payment:
│       walletClient.internalizeAction({
│         tx: token.transaction,
│         outputs: [{
│           outputIndex: 0,
│           protocol: 'wallet payment',
│           paymentRemittance: {
│             senderIdentityKey: msg.sender,
│             derivationPrefix: token.customInstructions.derivationPrefix,
│             derivationSuffix: token.customInstructions.derivationSuffix
│           }
│         }],
│         description: "Payment from 03def...",
│         labels: ['received_payment']
│       })
│   └─ Wallet derives spending key
│   └─ Adds UTXO to spendable balance
└─ Returns processed messages array

Step 4: Display Messages in UI
├─ Shows list of payments:
│   • Sender: 03def... (first 20 chars)
│   • Amount: 10000 satoshis
│   • Timestamp: 2 minutes ago
│   • "Show Details" button
│   • "Acknowledge & Remove" button
└─ Auto-refreshes every 45 seconds

Step 5: User Expands Message Details
├─ Clicks "Show Details"
├─ Shows expanded view:
│   • Full sender identity key
│   • Message ID
│   • Payment token JSON (formatted)
│   • Transaction data
└─ Can collapse with "Hide Details"

Step 6: User Acknowledges Message
├─ Clicks "Acknowledge & Remove"
├─ Calls: messageClient.acknowledgeMessage(messageId)
├─ PeerPayClient.acknowledgeMessage({ messageIds: [messageId] })
├─ MessageBox server deletes message (ephemeral)
├─ Frontend removes from local state
└─ UI updates (message disappears from list)

Step 7: Balance Updated
├─ Payment now in wallet as spendable UTXO
├─ Can be spent in future transactions
├─ Wallet tracks as "received_payment" label
└─ User can verify in wallet app
```

### Journey 4: Session Management (Page Reload)

```
Step 1: User Reloads Page
├─ Browser refreshes
├─ WalletProvider mounts (app/layout.tsx)
└─ Calls: walletManager.loadSession()

Step 2: Load from localStorage
├─ Reads: wallet_session_id
├─ Reads: wallet_identity_key
├─ If found:
│   └─ Calls: walletManager.checkStatus()
└─ If not found:
    └─ State: disconnected

Step 3: Validate Session with Server
├─ GET /api/wallet/status
├─ Headers: { X-Session-Id: sessionId }
├─ Server checks:
│   • Session exists in WalletSessionManager?
│   • Not expired? (< 30 minutes since last access)
└─ Returns:
    {
      success: true,
      connected: true,
      identityKey: "03abc...",
      connectedAt: "2025-02-02T10:00:00Z"
    }

Step 4: Restore Connection State
├─ WalletProvider updates state:
│   {
│     isConnected: true,
│     sessionId: "existing_session_id",
│     identityKey: "03abc..."
│   }
├─ UI shows green "Connected" status
├─ Navigation shows all pages accessible
└─ User can interact without reconnecting

Step 5: Session Expiration Handling
├─ If 30 minutes pass with no activity:
│   • Server auto-deletes session (WalletSessionManager cleanup)
│   • Next API call returns 401 Unauthorized
│   • Frontend detects in apiRequest() helper
│   • Clears localStorage
│   • Shows "Session expired, please reconnect"
│   • User clicks "Connect Wallet" to restart
└─ Session refresh:
    • Each API call updates lastAccessedAt
    • Keeps session alive during active use
```

---

## Why BSV?

### Key Advantages Over Other Blockchains

#### 1. **Micropayments & Low Fees**

**BSV**:
- Transaction fee: ~0.05 satoshis per byte
- Average transaction: ~250 bytes = **0.0125 satoshis (~$0.000006 USD)**
- This platform's payments: **~1 satoshi per transaction**

**Bitcoin (BTC)**:
- Average fee: $5-50 USD depending on network congestion
- NOT viable for micropayments

**Ethereum (ETH)**:
- Gas fees: $1-100 USD depending on network congestion
- ERC-20 token transfers: $10-200 USD
- NOT viable for small payments

**Use Case Enabled**:
- Send 10 satoshis ($0.005 USD) and only pay 1 satoshi fee
- Viable for streaming payments, pay-per-use APIs, microtipping

---

#### 2. **Unlimited Scalability**

**BSV**:
- Block size: **Unlimited** (current: 4GB blocks tested)
- Throughput: **50,000+ transactions per second**
- Scales with hardware improvements

**Bitcoin (BTC)**:
- Block size: 1MB cap (SegWit ~4MB)
- Throughput: ~7 transactions per second
- Requires Layer 2 (Lightning Network)

**Ethereum (ETH)**:
- Current: ~15-30 transactions per second
- Post-merge: ~30 TPS
- Requires Layer 2 (Optimistic Rollups, ZK-Rollups)

**Use Case Enabled**:
- Global payment system without Layer 2 complexity
- All transactions on-chain (no trust assumptions)

---

#### 3. **Data Storage on Blockchain**

**BSV**:
- OP_RETURN: **No size limit**
- Can store files, documents, images on-chain
- Permanent, immutable storage
- Cost: ~$0.01 USD per MB

**Bitcoin (BTC)**:
- OP_RETURN: 80 bytes max
- NOT designed for data storage

**Ethereum (ETH)**:
- Storage gas costs: **$100-1000 USD per MB**
- Prohibitively expensive for data

**Use Case Enabled**:
- Store certification proofs on-chain
- NFTs with on-chain media (not just pointers)
- Permanent document timestamping

---

#### 4. **Stable Protocol**

**BSV**:
- Protocol **frozen** (no more hard forks)
- Applications built today work forever
- Focuses on scaling, not protocol changes

**Bitcoin (BTC)**:
- Regular soft forks (SegWit, Taproot, etc.)
- Breaking changes require application updates

**Ethereum (ETH)**:
- Major upgrades every 6-12 months
- EVM changes break contracts
- Solidity version incompatibilities

**Use Case Enabled**:
- Build once, run forever
- No maintenance for protocol changes
- Long-term application viability

---

#### 5. **Simplified Smart Contracts**

**BSV**:
- **sCrypt**: TypeScript-like language
- Compiles to Bitcoin Script
- Turing complete with loops
- No gas limit for computation

```typescript
// sCrypt example (BSV smart contract)
class Payment extends SmartContract {
  @prop()
  recipient: PubKey

  @prop()
  amount: bigint

  @method()
  public unlock(sig: Sig) {
    assert(this.checkSig(sig, this.recipient))
    assert(this.ctx.utxo.value >= this.amount)
  }
}
```

**Ethereum**:
- **Solidity**: Unique language with security pitfalls
- Gas limits restrict computation
- Re-entrancy attacks, overflow bugs
- High deployment costs

**Use Case Enabled**:
- JavaScript/TypeScript developers can write smart contracts
- No gas limit for complex logic
- Lower barrier to entry

---

#### 6. **Identity-Based Payments (BRC-29)**

**BSV**:
- Send to **public keys** (identity)
- No addresses needed
- Key derivation protocol
- Receiver can use same identity across apps

```typescript
// Send to identity (no address)
await peerPayClient.createPaymentToken({
  recipient: "03abc123...",  // Public key
  amount: 10000
})
```

**Bitcoin (BTC)**:
- Must send to **address**
- Address generated per transaction
- Receiver must share new address each time

**Ethereum (ETH)**:
- Send to **account address**
- Address reuse (privacy concerns)
- No built-in key derivation

**Use Case Enabled**:
- Universal identity across applications
- Privacy-preserving payments (new derived address each time)
- No address management needed

---

#### 7. **SPV (Simplified Payment Verification)**

**BSV**:
- Full **SPV** support in protocol
- Clients verify payments without full node
- Merkle proofs for transaction verification
- Scales to billions of users

**Bitcoin (BTC)**:
- SPV exists but limited adoption
- Most users rely on custodial wallets
- RBF (Replace-By-Fee) breaks SPV security

**Ethereum (ETH)**:
- Light clients exist but complex
- Most users rely on Infura/Alchemy (centralized)
- State proofs large and complex

**Use Case Enabled**:
- Truly decentralized mobile wallets
- No need to trust third parties
- Instant payment verification

---

### Real-World Comparison Table

| Feature | BSV | BTC | ETH |
|---------|-----|-----|-----|
| **Avg Fee** | $0.000006 | $5-50 | $1-100 |
| **TPS** | 50,000+ | 7 | 15-30 |
| **Block Size** | Unlimited | 1-4MB | N/A (gas limit) |
| **Data Storage** | Unlimited | 80 bytes | Expensive |
| **Protocol Stability** | Frozen | Evolving | Evolving |
| **Smart Contracts** | sCrypt (TS-like) | Limited | Solidity |
| **Identity Payments** | ✅ BRC-29 | ❌ | ❌ |
| **SPV** | ✅ Full support | ⚠️ Limited | ⚠️ Complex |
| **Micropayments** | ✅ Viable | ❌ Too expensive | ❌ Too expensive |

---

### Use Cases Best Suited for BSV

1. **Micropayment Platforms**
   - Pay-per-article news sites
   - API usage billing (pay per request)
   - Gaming (in-game purchases)
   - Streaming tips

2. **Data Anchoring & Certification**
   - Document timestamping
   - Identity certification (this platform)
   - Supply chain tracking
   - Academic credentials

3. **Peer-to-Peer Applications**
   - Direct payments between users
   - No intermediaries needed
   - Social media with monetization
   - Content creator platforms

4. **Enterprise Applications**
   - High-volume transaction processing
   - IoT device payments
   - Supply chain management
   - Audit trails

5. **Global Payment Systems**
   - Remittances
   - Cross-border transactions
   - Unbanked population access
   - Financial inclusion

---

### Developer Benefits

#### **Familiar Tools**
```typescript
// BSV development uses standard JavaScript/TypeScript
import { WalletClient } from '@bsv/sdk'
import { PeerPayClient } from '@bsv/message-box-client'

// No new languages to learn (unlike Solidity)
const walletClient = new WalletClient()
const payment = await peerPayClient.createPaymentToken({
  recipient: publicKey,
  amount: 10000
})
```

#### **Lower Complexity**
- No Layer 2 solutions needed
- No gas optimization required
- No mempool management
- No complex state management

#### **Better Economics**
- Build business models on micropayments
- Don't need venture capital for subsidized transactions
- Users can actually afford to use your app
- Transaction fees don't eat into profit margins

---

## Next Steps

📖 Continue to [Code Examples & Reference](./CODE_EXAMPLES.md)

📖 See [Deployment Guide](./DEPLOYMENT.md) for setup instructions

🔗 BSV Resources:
- [BSV SDK Documentation](https://docs.bsvblockchain.org/)
- [sCrypt Documentation](https://scrypt.io/docs/)
- [MessageBox Protocol](https://github.com/bitcoin-sv/message-box)
- [BRC Standards](https://bsv.brc.dev/)
