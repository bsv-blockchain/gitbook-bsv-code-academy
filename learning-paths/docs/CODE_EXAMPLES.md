# Code Examples & Technical Reference

This document provides detailed code examples and technical explanations of key patterns in the MessageBox Platform.

## Table of Contents
1. [Wallet Integration](#wallet-integration)
2. [Certification Implementation](#certification-implementation)
3. [Payment Flow](#payment-flow)
4. [Message Handling](#message-handling)
5. [Session Management](#session-management)
6. [Server API Implementation](#server-api-implementation)

---

## Wallet Integration

### Browser-Side WalletClient

**File**: `frontend-next/lib/walletClient.ts`

```typescript
/**
 * Browser-based WalletClient wrapper
 * Handles BSV wallet connection in the browser
 * IMPORTANT: This must only run in the browser, not during SSR
 */

let walletClientInstance: any = null
let identityKey: string | null = null

/**
 * Connect to user's BSV wallet and get identity key
 * Prompts user via their wallet app (like MetaMask on Ethereum)
 */
export async function connectWallet(): Promise<{ identityKey: string }> {
  // Ensure we're in the browser
  if (typeof window === 'undefined') {
    throw new Error('WalletClient can only be used in the browser')
  }

  try {
    // Dynamically import WalletClient only in browser
    // This prevents SSR issues with Next.js
    const { WalletClient } = await import('@bsv/sdk')

    // Create wallet client in browser
    if (!walletClientInstance) {
      walletClientInstance = new WalletClient()
    }

    // Get identity key from wallet (this prompts the user's wallet)
    const result = await walletClientInstance.getPublicKey({
      identityKey: true
    })

    if (!result.publicKey) {
      throw new Error('Wallet returned no identity key')
    }

    identityKey = result.publicKey

    if (!identityKey) {
      throw new Error('Failed to obtain identity key from wallet')
    }

    console.log('✓ Wallet connected:', identityKey.substring(0, 20) + '...')

    return { identityKey }
  } catch (error: any) {
    console.error('Failed to connect wallet:', error)
    throw new Error(`Failed to connect to wallet: ${error.message}`)
  }
}

/**
 * Get or create wallet client instance
 * Used by payment/message clients that need wallet access
 */
export async function ensureWalletClient(): Promise<any> {
  // Ensure we're in the browser
  if (typeof window === 'undefined') {
    throw new Error('WalletClient can only be used in the browser')
  }

  // If wallet client already exists, return it
  if (walletClientInstance) {
    return walletClientInstance
  }

  // Create new wallet client instance
  const { WalletClient } = await import('@bsv/sdk')
  walletClientInstance = new WalletClient()

  return walletClientInstance
}

/**
 * Get current wallet client instance
 */
export function getWalletClient(): any {
  return walletClientInstance
}

/**
 * Get stored identity key
 */
export function getIdentityKey(): string | null {
  return identityKey
}

/**
 * Clear wallet connection (logout)
 */
export function clearWallet(): void {
  walletClientInstance = null
  identityKey = null
}
```

**Key Points**:
- **SSR Safety**: Dynamic imports prevent server-side execution
- **Singleton Pattern**: Only one wallet instance per browser session
- **Identity Key**: User's public key serves as universal identifier
- **No Private Keys**: Private keys never leave the user's wallet app

---

## Certification Implementation

### Browser Certification

**File**: `frontend-next/lib/certificationClient.ts`

```typescript
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
    // Step 1: Get or create wallet client
    console.log('Getting wallet client...')
    const walletClient = await ensureWalletClient()

    // Step 2: Dynamically import MessageBoxClient
    const { MessageBoxClient } = await import('@bsv/message-box-client')

    // Step 3: Get MessageBox host from environment
    const messageBoxHost = process.env.NEXT_PUBLIC_MESSAGEBOX_HOST ||
                          'https://messagebox.babbage.systems'

    console.log('Creating MessageBox client...')

    // Step 4: Create MessageBoxClient with wallet
    const messageBoxClient = new MessageBoxClient({
      walletClient,
      host: messageBoxHost,
      enableLogging: true  // Enable console logging for debugging
    })

    // Step 5: Initialize the client
    await messageBoxClient.init()

    // Step 6: Anoint host (create certification transaction)
    console.log('Certifying identity with host...')
    const result = await messageBoxClient.anointHost(messageBoxHost)

    console.log('✓ Certification complete. TXID:', result.txid)

    // Step 7: Get identity key from wallet
    const keyResult = await walletClient.getPublicKey({ identityKey: true })
    const identityKey = keyResult.publicKey

    if (!identityKey) {
      throw new Error('Failed to retrieve identity key from wallet')
    }

    // Step 8: Store certification on backend
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

**What `anointHost()` Does Internally**:

```typescript
// Pseudo-code of anointHost() internals
async function anointHost(host: string): Promise<{ txid: string }> {
  // 1. Create transaction with certification data
  const tx = new Transaction()

  // 2. Add OP_RETURN output with host certification
  tx.addOutput({
    script: Script.buildPublicKeyHashOut(publicKey),
    satoshis: 1
  })

  // 3. Add OP_RETURN with certification proof
  tx.addOutput({
    script: Script.buildSafeDataOut([
      'messagebox',
      'anoint',
      host,
      identityKey,
      Date.now()
    ]),
    satoshis: 0
  })

  // 4. Sign transaction with wallet
  await walletClient.signTransaction(tx)

  // 5. Broadcast to BSV network
  const txid = await broadcast(tx)

  return { txid }
}
```

### Server Certification Endpoint

**File**: `server/src/routes/certify.ts`

```typescript
import { Router } from 'express'
import { WalletClient } from '@bsv/sdk'
import { MessageBoxClient } from '@bsv/message-box-client'
import { WalletSessionManager } from '../wallet/WalletSessionManager'
import { CertificationStorage } from '../storage/CertificationStorage'

const router = Router()

/**
 * POST /api/certify
 * Server-side certification (alternative to client-side)
 * Uses session wallet if available, creates new wallet otherwise
 */
router.post('/certify', async (req, res) => {
  try {
    const { alias } = req.body
    const sessionId = req.headers['x-session-id'] as string

    let walletClient: WalletClient

    // Try to get wallet from session, or create new
    if (sessionId) {
      const sessionWallet = WalletSessionManager.getInstance().getWalletClient(sessionId)
      if (sessionWallet) {
        walletClient = sessionWallet
      } else {
        // Session expired or invalid
        return res.status(401).json({
          success: false,
          error: 'Invalid or expired session'
        })
      }
    } else {
      // Create new wallet client for this request
      walletClient = new WalletClient()
    }

    // Create MessageBoxClient
    const messageBoxHost = process.env.MESSAGEBOX_HOST ||
                          'https://messagebox.babbage.systems'

    const messageBoxClient = new MessageBoxClient({
      walletClient,
      host: messageBoxHost,
      enableLogging: true
    })

    await messageBoxClient.init()

    // Anoint host (create certification transaction)
    console.log('Certifying identity...')
    const result = await messageBoxClient.anointHost(messageBoxHost)
    console.log('Certification TXID:', result.txid)

    // Get identity key
    const keyResult = await walletClient.getPublicKey({ identityKey: true })
    const identityKey = keyResult.publicKey

    // Store in database
    const storage = CertificationStorage.getInstance()
    await storage.storeCertification({
      identityKey,
      alias,
      txid: result.txid,
      host: messageBoxHost
    })

    res.json({
      success: true,
      identityKey,
      txid: result.txid,
      alias,
      message: 'Identity certified successfully'
    })
  } catch (error: any) {
    console.error('Certification error:', error)
    res.status(500).json({
      success: false,
      error: error.message
    })
  }
})

export default router
```

---

## Payment Flow

### Creating and Sending Payments

**File**: `frontend-next/lib/paymentClient.ts`

```typescript
import { ensureWalletClient } from './walletClient'

/**
 * Send a BSV payment to another user using PeerPay protocol
 *
 * @param recipient - Recipient's identity key (public key)
 * @param amount - Amount in satoshis
 * @returns Payment result with messageId
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
    // Step 1: Dynamically import PeerPayClient
    const { PeerPayClient } = await import('@bsv/message-box-client')

    // Step 2: Get or create wallet client
    const walletClient = await ensureWalletClient()

    // Step 3: Get MessageBox host from env
    const messageBoxHost = process.env.NEXT_PUBLIC_MESSAGEBOX_HOST ||
                          'https://messagebox.babbage.systems'

    console.log('Creating PeerPay client...')

    // Step 4: Create PeerPay client
    const peerPayClient = new PeerPayClient({
      walletClient,
      messageBoxHost,
      enableLogging: true
    })

    console.log(`Sending payment: ${amount} satoshis to ${recipient}`)

    // Step 5: Create payment token
    // This builds the transaction, signs it, and broadcasts to network
    // randomizeOutputs: false means:
    //   output[0] = payment to recipient
    //   output[1+] = change back to sender
    const paymentToken = await peerPayClient.createPaymentToken({
      recipient,
      amount
    })

    console.log('✓ Payment token created:', paymentToken.transaction)

    // Note: Sender's wallet automatically tracks change outputs
    // We don't need to manually internalize them

    // Step 6: Send payment token to recipient via MessageBox
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

**Example Payment Token**:

```json
{
  "transaction": "0100beef...",
  "customInstructions": {
    "derivationPrefix": "a2V5SUQ=",
    "derivationSuffix": "c29tZXRoaW5nSXdvbnRmb3JnZXQ="
  },
  "amount": 10000,
  "outputIndex": 0
}
```

### Understanding Transaction Outputs

```typescript
// After createPaymentToken() broadcasts this transaction:

Transaction {
  inputs: [
    {
      // Sender's UTXO (e.g., 15000 satoshis)
      prevTxId: "abc123...",
      outputIndex: 0,
      script: <unlock_script>,
      sequence: 0xffffffff
    }
  ],
  outputs: [
    {
      // Output[0]: Payment to recipient
      value: 10000,  // Amount specified
      script: P2PKH(derivedAddress)
      // derivedAddress = derive(recipientKey, prefix, suffix)
    },
    {
      // Output[1]: Change back to sender
      value: 4999,  // 15000 - 10000 - 1 (fee)
      script: P2PKH(senderAddress)
      // Sender's wallet auto-tracks this
    }
  ],
  lockTime: 0
}
```

**Key Insight**:
- Output[0] is locked to a **derived address** using recipient's key + derivation instructions
- Recipient must internalize with those instructions to spend it
- Sender's change is automatically tracked (no internalization needed)

---

## Message Handling

### Checking Inbox and Internalizing Payments

**File**: `frontend-next/lib/messageClient.ts`

```typescript
import { ensureWalletClient } from './walletClient'

export interface ReceivedMessage {
  messageId: string
  sender: string
  messageBox: string
  body: any
  timestamp: Date
}

/**
 * Safely parse a message body into a payment token object
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
 *
 * @param messageBox - Message box name (default: 'payment_inbox')
 * @returns Array of received messages
 */
export async function checkInbox(
  messageBox: string = 'payment_inbox'
): Promise<ReceivedMessage[]> {
  if (typeof window === 'undefined') {
    throw new Error('Message retrieval can only be done in the browser')
  }

  try {
    // Step 1: Import PeerPayClient
    const { PeerPayClient } = await import('@bsv/message-box-client')

    // Step 2: Get wallet client
    const walletClient = await ensureWalletClient()

    // Step 3: Get MessageBox host
    const messageBoxHost = process.env.NEXT_PUBLIC_MESSAGEBOX_HOST ||
                          'https://messagebox.babbage.systems'

    console.log(`Checking inbox: ${messageBox}`)

    // Step 4: Create PeerPay client
    const peerPayClient = new PeerPayClient({
      walletClient,
      messageBoxHost,
      enableLogging: true
    })

    await peerPayClient.init()

    // Step 5: List messages from message box
    console.log('Calling listMessages...')
    const messages = await peerPayClient.listMessages({ messageBox })

    console.log('Raw messages response:', messages)

    if (!messages || !Array.isArray(messages)) {
      console.log('No messages or unexpected format, returning empty array')
      return []
    }

    console.log(`Found ${messages.length} message(s) in inbox`)

    // Step 6: Process and internalize each payment
    const processedMessages = []

    for (const msg of messages) {
      try {
        // Parse message body as payment token
        const token = parsePaymentToken(msg.body)

        if (token && token.transaction) {
          console.log('Internalizing payment from:', msg.sender)

          // CRITICAL: Internalize the payment
          // This tells wallet how to derive the spending key
          const args = {
            tx: token.transaction,
            outputs: [{
              outputIndex: token.outputIndex ?? 0,  // Usually output[0]
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
          }

          await walletClient.internalizeAction(args)
          console.log('Transaction internalized successfully')
        }
      } catch (internError: any) {
        console.error('Failed to internalize transaction:', internError)
        // Continue processing other messages even if one fails
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
 *
 * @param messageId - Message ID to acknowledge
 * @param _messageBox - Message box name (not currently used)
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

    console.log('Message acknowledged successfully')
  } catch (error: any) {
    console.error('Failed to acknowledge message:', error)
    throw new Error(`Failed to acknowledge message: ${error.message}`)
  }
}
```

### Understanding Internalization

**Why Internalization is Required**:

```typescript
// Scenario: Alice sends 10000 satoshis to Bob

// 1. Alice creates transaction:
//    Output[0]: 10000 sats → derived_address(Bob's key, prefix, suffix)

// 2. Bob's wallet DOESN'T automatically know about this UTXO
//    Problem: Bob's wallet doesn't have derivation instructions

// 3. Alice sends payment token via MessageBox:
//    {
//      transaction: <tx>,
//      customInstructions: { prefix, suffix }
//    }

// 4. Bob receives message and calls internalizeAction():
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
//    d. Bob can now spend these 10000 satoshis
```

**Common Mistake**:

```typescript
// ❌ WRONG: Trying to internalize sender's change outputs
// This causes error: "paymentRemittance must be valid for protocol wallet payment"

// Change outputs are NOT BRC-29 payments
// They're regular P2PKH outputs already tracked by sender's wallet

await walletClient.internalizeAction({
  tx: paymentToken.transaction,
  outputs: [
    { outputIndex: 1, protocol: 'wallet payment' }  // ❌ Change output
  ]
})

// ✅ CORRECT: Only recipient internalizes their payment
await walletClient.internalizeAction({
  tx: paymentToken.transaction,
  outputs: [{
    outputIndex: 0,  // Payment output
    protocol: 'wallet payment',
    paymentRemittance: {  // BRC-29 derivation info
      senderIdentityKey,
      derivationPrefix,
      derivationSuffix
    }
  }]
})
```

---

## Session Management

### Frontend Session Manager

**File**: `frontend-next/lib/walletManager.ts`

```typescript
const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000'

interface SessionData {
  sessionId: string
  identityKey: string
}

/**
 * Helper to make authenticated API requests
 */
async function apiRequest(
  endpoint: string,
  options: RequestInit = {}
): Promise<Response> {
  const sessionId = localStorage.getItem('wallet_session_id')

  const headers = {
    'Content-Type': 'application/json',
    ...(sessionId && { 'X-Session-Id': sessionId }),
    ...options.headers
  }

  const response = await fetch(`${API_URL}${endpoint}`, {
    ...options,
    headers
  })

  // Handle session expiration
  if (response.status === 401) {
    // Clear expired session
    localStorage.removeItem('wallet_session_id')
    localStorage.removeItem('wallet_identity_key')
    throw new Error('Session expired. Please reconnect your wallet.')
  }

  return response
}

/**
 * Connect wallet and create server session
 */
export async function connect(): Promise<SessionData> {
  try {
    // Import and call connectWallet
    const { connectWallet } = await import('./walletClient')
    const { identityKey } = await connectWallet()

    // Create session on server
    const response = await apiRequest('/api/wallet/connect', {
      method: 'POST',
      body: JSON.stringify({ identityKey })
    })

    if (!response.ok) {
      const error = await response.json()
      throw new Error(error.message || 'Failed to connect wallet')
    }

    const data = await response.json()

    // Store session data
    localStorage.setItem('wallet_session_id', data.sessionId)
    localStorage.setItem('wallet_identity_key', data.identityKey)

    return {
      sessionId: data.sessionId,
      identityKey: data.identityKey
    }
  } catch (error: any) {
    console.error('Connection failed:', error)
    throw error
  }
}

/**
 * Disconnect wallet and destroy server session
 */
export async function disconnect(): Promise<void> {
  try {
    await apiRequest('/api/wallet/disconnect', {
      method: 'POST'
    })
  } catch (error) {
    console.error('Disconnect error:', error)
  } finally {
    // Always clear local storage
    localStorage.removeItem('wallet_session_id')
    localStorage.removeItem('wallet_identity_key')

    // Clear wallet client instance
    const { clearWallet } = await import('./walletClient')
    clearWallet()
  }
}

/**
 * Check if session is still valid
 */
export async function checkStatus(): Promise<{
  connected: boolean
  identityKey?: string
}> {
  try {
    const response = await apiRequest('/api/wallet/status')

    if (!response.ok) {
      return { connected: false }
    }

    const data = await response.json()
    return {
      connected: data.connected,
      identityKey: data.identityKey
    }
  } catch (error) {
    console.error('Status check failed:', error)
    return { connected: false }
  }
}

/**
 * Load session from localStorage on page load
 */
export function loadSession(): SessionData | null {
  const sessionId = localStorage.getItem('wallet_session_id')
  const identityKey = localStorage.getItem('wallet_identity_key')

  if (sessionId && identityKey) {
    return { sessionId, identityKey }
  }

  return null
}
```

### Server Session Manager

**File**: `server/src/wallet/WalletSessionManager.ts`

```typescript
import crypto from 'crypto'

interface WalletSession {
  sessionId: string
  walletClient: any | null
  identityKey: string
  createdAt: Date
  lastAccessedAt: Date
}

/**
 * Manages wallet sessions with 30-minute timeout
 * Singleton pattern ensures one instance across server
 */
export class WalletSessionManager {
  private static instance: WalletSessionManager
  private sessions: Map<string, WalletSession>
  private readonly SESSION_TIMEOUT = 30 * 60 * 1000 // 30 minutes
  private cleanupInterval: NodeJS.Timeout

  private constructor() {
    this.sessions = new Map()

    // Auto-cleanup every 5 minutes
    this.cleanupInterval = setInterval(() => {
      this.cleanupExpiredSessions()
    }, 5 * 60 * 1000)
  }

  public static getInstance(): WalletSessionManager {
    if (!WalletSessionManager.instance) {
      WalletSessionManager.instance = new WalletSessionManager()
    }
    return WalletSessionManager.instance
  }

  /**
   * Create new session with identity key
   */
  public createSessionWithIdentity(identityKey: string): string {
    const sessionId = crypto.randomBytes(32).toString('hex')

    this.sessions.set(sessionId, {
      sessionId,
      walletClient: null,
      identityKey,
      createdAt: new Date(),
      lastAccessedAt: new Date()
    })

    console.log(`Session created: ${sessionId} for identity: ${identityKey.substring(0, 20)}...`)
    return sessionId
  }

  /**
   * Get wallet client for session
   * Updates lastAccessedAt to keep session alive
   */
  public getWalletClient(sessionId: string): any | null {
    const session = this.sessions.get(sessionId)

    if (!session) {
      return null
    }

    // Check if expired
    const now = new Date()
    const timeSinceAccess = now.getTime() - session.lastAccessedAt.getTime()

    if (timeSinceAccess > this.SESSION_TIMEOUT) {
      console.log(`Session expired: ${sessionId}`)
      this.sessions.delete(sessionId)
      return null
    }

    // Update last accessed time
    session.lastAccessedAt = new Date()

    return session.walletClient
  }

  /**
   * Get session info (identity key, created time)
   */
  public getSessionInfo(sessionId: string): {
    identityKey: string
    createdAt: Date
  } | null {
    const session = this.sessions.get(sessionId)

    if (!session) {
      return null
    }

    // Check if expired
    const now = new Date()
    const timeSinceAccess = now.getTime() - session.lastAccessedAt.getTime()

    if (timeSinceAccess > this.SESSION_TIMEOUT) {
      this.sessions.delete(sessionId)
      return null
    }

    // Update last accessed time
    session.lastAccessedAt = new Date()

    return {
      identityKey: session.identityKey,
      createdAt: session.createdAt
    }
  }

  /**
   * Delete session (logout)
   */
  public deleteSession(sessionId: string): boolean {
    return this.sessions.delete(sessionId)
  }

  /**
   * Cleanup expired sessions
   */
  private cleanupExpiredSessions(): void {
    const now = new Date()
    let cleanedCount = 0

    for (const [sessionId, session] of this.sessions.entries()) {
      const timeSinceAccess = now.getTime() - session.lastAccessedAt.getTime()

      if (timeSinceAccess > this.SESSION_TIMEOUT) {
        this.sessions.delete(sessionId)
        cleanedCount++
      }
    }

    if (cleanedCount > 0) {
      console.log(`Cleaned up ${cleanedCount} expired session(s)`)
    }
  }

  /**
   * Get active session count
   */
  public getSessionCount(): number {
    return this.sessions.size
  }

  /**
   * Stop cleanup interval (for graceful shutdown)
   */
  public destroy(): void {
    clearInterval(this.cleanupInterval)
  }
}
```

---

## Server API Implementation

### Complete API Route Example

**File**: `server/src/routes/payment.ts`

```typescript
import { Router } from 'express'
import { PeerPayClient } from '@bsv/message-box-client'
import { WalletSessionManager } from '../wallet/WalletSessionManager'
import { CertificationStorage } from '../storage/CertificationStorage'

const router = Router()

/**
 * POST /api/initiate-payment
 * Initiate a payment from sender to recipient
 * Requires active session
 */
router.post('/initiate-payment', async (req, res) => {
  try {
    const { recipient, amount } = req.body
    const sessionId = req.headers['x-session-id'] as string

    // Validate input
    if (!recipient || !amount) {
      return res.status(400).json({
        success: false,
        error: 'Recipient and amount are required'
      })
    }

    if (!sessionId) {
      return res.status(401).json({
        success: false,
        error: 'Session ID required'
      })
    }

    // Get session
    const sessionManager = WalletSessionManager.getInstance()
    const sessionInfo = sessionManager.getSessionInfo(sessionId)

    if (!sessionInfo) {
      return res.status(401).json({
        success: false,
        error: 'Invalid or expired session'
      })
    }

    // Verify recipient is certified
    const storage = CertificationStorage.getInstance()
    const recipientUser = await storage.getCertifiedUser(recipient)

    if (!recipientUser) {
      return res.status(404).json({
        success: false,
        error: 'Recipient not found or not certified'
      })
    }

    // Get wallet client from session
    const walletClient = sessionManager.getWalletClient(sessionId)

    if (!walletClient) {
      return res.status(401).json({
        success: false,
        error: 'No wallet client in session'
      })
    }

    // Create PeerPay client
    const messageBoxHost = process.env.MESSAGEBOX_HOST ||
                          'https://messagebox.babbage.systems'

    const peerPayClient = new PeerPayClient({
      walletClient,
      messageBoxHost,
      enableLogging: true
    })

    // Send payment
    console.log(`Initiating payment: ${amount} satoshis to ${recipient}`)

    const result = await peerPayClient.sendPayment({
      recipient,
      amount
    })

    res.json({
      success: true,
      messageId: result.messageId,
      message: 'Payment initiated successfully'
    })
  } catch (error: any) {
    console.error('Payment error:', error)
    res.status(500).json({
      success: false,
      error: error.message
    })
  }
})

export default router
```

### MongoDB Storage Implementation

**File**: `server/src/storage/CertificationStorage.ts`

```typescript
import { MongoClient, Db, Collection } from 'mongodb'

export interface CertifiedUser {
  identityKey: string
  alias?: string
  certificationDate: Date
  certificationTxid: string
  host: string
}

/**
 * MongoDB storage for certified users
 * Singleton pattern
 */
export class CertificationStorage {
  private static instance: CertificationStorage
  private client: MongoClient | null = null
  private db: Db | null = null
  private collection: Collection<CertifiedUser> | null = null

  private constructor() {}

  public static getInstance(): CertificationStorage {
    if (!CertificationStorage.instance) {
      CertificationStorage.instance = new CertificationStorage()
    }
    return CertificationStorage.instance
  }

  /**
   * Connect to MongoDB
   */
  public async connect(uri: string, dbName: string): Promise<void> {
    try {
      this.client = new MongoClient(uri)
      await this.client.connect()

      this.db = this.client.db(dbName)
      this.collection = this.db.collection<CertifiedUser>('certified_users')

      // Create unique index on identityKey
      await this.collection.createIndex({ identityKey: 1 }, { unique: true })

      console.log('✓ MongoDB connected:', dbName)
    } catch (error) {
      console.error('MongoDB connection failed:', error)
      throw error
    }
  }

  /**
   * Store or update certification
   */
  public async storeCertification(user: {
    identityKey: string
    alias?: string
    txid: string
    host: string
  }): Promise<void> {
    if (!this.collection) {
      throw new Error('Database not connected')
    }

    await this.collection.updateOne(
      { identityKey: user.identityKey },
      {
        $set: {
          identityKey: user.identityKey,
          alias: user.alias,
          certificationTxid: user.txid,
          host: user.host,
          certificationDate: new Date()
        }
      },
      { upsert: true }
    )

    console.log(`Stored certification for: ${user.identityKey}`)
  }

  /**
   * Get certified user by identity key
   */
  public async getCertifiedUser(identityKey: string): Promise<CertifiedUser | null> {
    if (!this.collection) {
      throw new Error('Database not connected')
    }

    return await this.collection.findOne({ identityKey })
  }

  /**
   * List all certified users
   */
  public async listAllCertifiedUsers(): Promise<CertifiedUser[]> {
    if (!this.collection) {
      throw new Error('Database not connected')
    }

    return await this.collection
      .find({})
      .sort({ certificationDate: -1 })
      .toArray()
  }

  /**
   * Search users by alias (case-insensitive)
   */
  public async searchByAlias(query: string): Promise<CertifiedUser[]> {
    if (!this.collection) {
      throw new Error('Database not connected')
    }

    return await this.collection
      .find({
        alias: { $regex: query, $options: 'i' }
      })
      .sort({ certificationDate: -1 })
      .toArray()
  }

  /**
   * Close connection
   */
  public async disconnect(): Promise<void> {
    if (this.client) {
      await this.client.close()
      console.log('MongoDB disconnected')
    }
  }
}
```

---

## Complete Integration Example

### Full Payment Flow with All Components

```typescript
// ============================================
// SENDER SIDE
// ============================================

// 1. User connects wallet (one-time)
import * as walletManager from '@/lib/walletManager'

const session = await walletManager.connect()
// → Wallet prompts user for approval
// → Returns: { sessionId, identityKey }
// → Stored in localStorage

// 2. User selects recipient and amount
const recipientKey = "03def456..."
const amount = 10000  // satoshis

// 3. Send payment
import { sendPayment } from '@/lib/paymentClient'

const result = await sendPayment(recipientKey, amount)
// → Creates PeerPayClient
// → Builds transaction (sender's UTXO → recipient + change)
// → Signs with wallet (no new approval needed)
// → Broadcasts to BSV network
// → Sends payment token via MessageBox
// → Returns: { success: true, messageId }

// ============================================
// RECIPIENT SIDE
// ============================================

// 1. Auto-check inbox (every 45 seconds)
import { checkInbox } from '@/lib/messageClient'

const messages = await checkInbox('payment_inbox')
// → Creates PeerPayClient
// → Lists messages (auto-decrypts)
// → For each payment message:
//   • Parses payment token
//   • Calls walletClient.internalizeAction()
//   • Wallet derives spending key
//   • Adds UTXO to balance
// → Returns: [{ messageId, sender, body, timestamp }]

// 2. Display messages in UI
messages.forEach(msg => {
  console.log(`Payment from: ${msg.sender}`)
  console.log(`Amount: ${msg.body.amount} satoshis`)
  console.log(`Time: ${msg.timestamp}`)
})

// 3. Acknowledge message (remove from server)
import { acknowledgeMessage } from '@/lib/messageClient'

await acknowledgeMessage(messages[0].messageId)
// → Calls PeerPayClient.acknowledgeMessage()
// → Message deleted from server (ephemeral)

// ============================================
// BLOCKCHAIN STATE
// ============================================

// Transaction now confirmed on BSV network:
// TXID: abc123...
// Inputs:  Sender's UTXO (15000 sats)
// Output[0]: 10000 sats → Recipient (derived address)
// Output[1]: 4999 sats → Sender (change)
// Fee: 1 sat

// Recipient's wallet:
// • Has private key for output[0] (via derivation)
// • Can spend 10000 satoshis in future transactions
// • Tracked as "received_payment" label
```

---

## Next Steps

📖 Return to [BSV Patterns Guide](./BSV_PATTERNS_GUIDE.md)

📖 Continue to [Deployment Guide](./DEPLOYMENT.md)
