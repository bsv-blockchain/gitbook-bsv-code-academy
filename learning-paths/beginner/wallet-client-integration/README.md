# Frontend Wallet Integration with WalletClient

**Module 3B: Connecting to User Wallets (Frontend Paradigm)**

This module teaches you how to integrate MetaNet Desktop Wallet into your dApp using the BSV SDK's WalletClient component. This is the **standard approach** for building non-custodial BSV applications where users control their own keys.

> **Note**: This module is for **Frontend Integration Development**. If you're building backend services where you control keys, see [Managing Wallets Server-Side](../first-wallet/) instead.

---

## What is WalletClient?

`WalletClient` is a pre-built SDK component that:
- ✅ Connects your dApp to MetaNet Desktop Wallet (BSV Desktop)
- ✅ Requests user signatures for transactions
- ✅ Handles all wallet communication via BRC protocol
- ✅ Manages UTXO selection (wallet handles)
- ✅ Calculates fees automatically (wallet handles)
- ✅ Broadcasts transactions (wallet handles)

**You don't need to**:
- ❌ Manage private keys
- ❌ Track UTXOs
- ❌ Calculate fees
- ❌ Handle change outputs
- ❌ Configure ARC broadcasting

**The wallet does all of this for you!**

---

## Prerequisites

Before starting:
- ✅ Completed [Development Environment Setup](../development-environment/)
- ✅ MetaNet Desktop Wallet installed and set up
- ✅ Basic understanding of React or your frontend framework
- ✅ Testnet BSV in your wallet (from faucet)

---

## Part 1: WalletClient Basics

### Installation

WalletClient is included in `@bsv/sdk`:

```bash
npm install @bsv/sdk
```

### Import

```typescript
import { WalletClient } from '@bsv/sdk'
```

### Core Methods

```typescript
import { WalletClient, P2PKH } from '@bsv/sdk'

const wallet = new WalletClient('auto')

// Connect to user's wallet substrate
await wallet.connectToSubstrate()

// Get user's identity public key
const { publicKey } = await wallet.getPublicKey({ identityKey: true })

// Create and send a transaction (auto-signed and broadcast)
const result = await wallet.createAction({
  description: 'Send payment',
  outputs: [{
    lockingScript: new P2PKH().lock('recipient-address').toHex(),
    satoshis: 1000,
    outputDescription: 'Payment output'
  }]
})

const txid = result.txid
```

**These are the main methods you'll use!** WalletClient handles the complexity of key management, UTXO selection, and broadcasting.

---

## Part 2: Connecting to MetaNet Desktop Wallet

### Basic Connection

```typescript
import { WalletClient } from '@bsv/sdk'

async function connectWallet() {
  try {
    // 1. Create WalletClient instance
    const wallet = new WalletClient('auto')

    // 2. Connect to the wallet substrate
    await wallet.connectToSubstrate()

    // 3. Get user's identity public key
    const { publicKey } = await wallet.getPublicKey({ identityKey: true })

    console.log('Connected to wallet, public key:', publicKey)
    return { wallet, publicKey }

  } catch (error: any) {
    if (error.code === 'USER_REJECTED') {
      console.log('User rejected connection')
    } else if (error.code === 'WALLET_NOT_FOUND') {
      console.log('MetaNet Desktop Wallet not installed')
    } else {
      console.error('Connection failed:', error.message)
    }
    throw error
  }
}
```

### Connection Flow

```
Your dApp                    MetaNet Desktop Wallet
    │                                │
    │  connectToSubstrate()          │
    ├───────────────────────────────>│
    │                                │
    │      [Popup appears]           │
    │   "Allow dApp to connect?"     │
    │                                │
    │<───────────────────────────────┤
    │     User clicks "Allow"        │
    │                                │
    │  Connection established        │
    │  getPublicKey({identityKey})   │
    ├───────────────────────────────>│
    │                                │
    │<───────────────────────────────┤
    │     Returns public key         │
    │                                │
```

### React Hook for Wallet Connection

```typescript
// hooks/useWallet.ts
import { useState, useEffect } from 'react'
import { WalletClient } from '@bsv/sdk'

export interface WalletState {
  wallet: WalletClient | null
  publicKey: string | null
  connected: boolean
  connecting: boolean
  error: string | null
}

export function useWallet() {
  const [state, setState] = useState<WalletState>({
    wallet: null,
    publicKey: null,
    connected: false,
    connecting: false,
    error: null
  })

  const connect = async () => {
    setState(prev => ({ ...prev, connecting: true, error: null }))

    try {
      const wallet = new WalletClient('auto')
      await wallet.connectToSubstrate()

      const { publicKey } = await wallet.getPublicKey({ identityKey: true })

      setState({
        wallet,
        publicKey,
        connected: true,
        connecting: false,
        error: null
      })

      console.log('✅ Wallet connected, public key:', publicKey)
    } catch (error: any) {
      setState({
        wallet: null,
        publicKey: null,
        connected: false,
        connecting: false,
        error: error.message
      })

      console.error('❌ Connection failed:', error)
    }
  }

  const disconnect = async () => {
    // Note: WalletClient doesn't have a disconnect method
    // Simply clear local state
    setState({
      wallet: null,
      publicKey: null,
      connected: false,
      connecting: false,
      error: null
    })
  }

  // Auto-reconnect if wallet was previously connected
  useEffect(() => {
    const autoConnect = async () => {
      const wasConnected = localStorage.getItem('wallet_connected')
      if (wasConnected === 'true') {
        await connect()
      }
    }

    autoConnect()
  }, [])

  // Save connection state
  useEffect(() => {
    if (state.connected) {
      localStorage.setItem('wallet_connected', 'true')
    } else {
      localStorage.removeItem('wallet_connected')
    }
  }, [state.connected])

  return {
    ...state,
    connect,
    disconnect
  }
}
```

### Wallet Connection Component

```typescript
// components/WalletConnect.tsx
import React from 'react'
import { useWallet } from '../hooks/useWallet'

export const WalletConnect: React.FC = () => {
  const { publicKey, connected, connecting, error, connect, disconnect } = useWallet()

  if (connected && publicKey) {
    return (
      <div className="wallet-connected">
        <p>Connected: {publicKey.substring(0, 12)}...{publicKey.substring(publicKey.length - 12)}</p>
        <button onClick={disconnect}>Disconnect</button>
      </div>
    )
  }

  return (
    <div className="wallet-not-connected">
      {error && <p className="error">{error}</p>}

      <button
        onClick={connect}
        disabled={connecting}
      >
        {connecting ? 'Connecting...' : 'Connect Wallet'}
      </button>

      {!connecting && (
        <div className="wallet-instructions">
          <p>To connect:</p>
          <ul>
            <li>Install MetaNet Desktop Wallet</li>
            <li>Create or import a wallet</li>
            <li>Click "Connect Wallet"</li>
          </ul>
        </div>
      )}
    </div>
  )
}
```

---

## Part 3: Sending Transactions

### Simple Payment

```typescript
import { WalletClient, P2PKH } from '@bsv/sdk'

async function sendPayment(
  wallet: WalletClient,
  recipientAddress: string,
  amountSatoshis: number
) {
  try {
    // Create action to send payment (auto-signed and broadcast)
    const result = await wallet.createAction({
      description: 'Send payment',
      outputs: [{
        lockingScript: new P2PKH().lock(recipientAddress).toHex(),
        satoshis: amountSatoshis,
        outputDescription: 'Payment to recipient'
      }]
    })

    console.log('Payment sent!')
    console.log('TXID:', result.txid)
    console.log('Amount:', amountSatoshis, 'satoshis')

    return result.txid

  } catch (error: any) {
    if (error.code === 'USER_REJECTED') {
      console.log('User cancelled payment')
    } else if (error.code === 'INSUFFICIENT_FUNDS') {
      console.log('User has insufficient balance')
    } else {
      console.error('Payment failed:', error.message)
    }
    throw error
  }
}

// Usage
const txid = await sendPayment(wallet, '1A1zP1...', 1000)
```

### What Happens Under the Hood

When you call `wallet.createAction()`:

1. **WalletClient** sends transaction request to MetaNet Desktop Wallet
2. **Wallet popup** appears showing transaction details
3. **User reviews**:
   - Recipient address
   - Amount to send
   - Fee (calculated by wallet)
   - Total deduction from balance
4. **User approves** or rejects
5. **Wallet**:
   - Selects UTXOs
   - Calculates optimal fee
   - Creates change output
   - Signs transaction
   - Broadcasts to BSV network
6. **WalletClient** returns TXID

**Your dApp never sees the private key!**

### Payment Component

```typescript
// components/SendPayment.tsx
import React, { useState } from 'react'
import { WalletClient, P2PKH } from '@bsv/sdk'

interface SendPaymentProps {
  wallet: WalletClient
}

export const SendPayment: React.FC<SendPaymentProps> = ({ wallet }) => {
  const [recipient, setRecipient] = useState('')
  const [amount, setAmount] = useState('')
  const [sending, setSending] = useState(false)
  const [txid, setTxid] = useState<string | null>(null)
  const [error, setError] = useState<string | null>(null)

  const handleSend = async () => {
    setSending(true)
    setError(null)
    setTxid(null)

    try {
      // Validate inputs
      if (!recipient) throw new Error('Recipient address required')
      if (!amount || isNaN(Number(amount))) throw new Error('Valid amount required')

      const amountSats = Math.floor(Number(amount) * 100_000_000) // BSV to satoshis

      // Create action to send payment
      const result = await wallet.createAction({
        description: 'Send payment',
        outputs: [{
          lockingScript: new P2PKH().lock(recipient).toHex(),
          satoshis: amountSats,
          outputDescription: 'Payment to recipient'
        }]
      })

      setTxid(result.txid!)
      setRecipient('')
      setAmount('')

      console.log('✅ Payment successful:', result.txid)

    } catch (err: any) {
      if (err.code === 'USER_REJECTED') {
        setError('Payment cancelled by user')
      } else if (err.code === 'INSUFFICIENT_FUNDS') {
        setError('Insufficient balance in wallet')
      } else {
        setError(err.message || 'Payment failed')
      }
      console.error('❌ Payment failed:', err)
    } finally {
      setSending(false)
    }
  }

  return (
    <div className="send-payment">
      <h3>Send Payment</h3>

      {txid && (
        <div className="success-message">
          <p>✅ Payment sent successfully!</p>
          <p>TXID: <a href={`https://test.whatsonchain.com/tx/${txid}`} target="_blank" rel="noreferrer">{txid.substring(0, 16)}...</a></p>
        </div>
      )}

      {error && <p className="error-message">❌ {error}</p>}

      <div className="form">
        <input
          type="text"
          placeholder="Recipient Address"
          value={recipient}
          onChange={(e) => setRecipient(e.target.value)}
          disabled={sending}
        />

        <input
          type="number"
          placeholder="Amount (BSV)"
          value={amount}
          onChange={(e) => setAmount(e.target.value)}
          disabled={sending}
          step="0.00000001"
          min="0"
        />

        <button
          onClick={handleSend}
          disabled={sending || !recipient || !amount}
        >
          {sending ? 'Sending...' : 'Send Payment'}
        </button>
      </div>

      <div className="info">
        <p>The wallet will calculate the fee and request your approval.</p>
      </div>
    </div>
  )
}
```

---

## Part 4: Multiple Outputs

### Batch Payments

```typescript
import { WalletClient, P2PKH } from '@bsv/sdk'

async function sendBatchPayment(wallet: WalletClient) {
  const result = await wallet.createAction({
    description: 'Batch payment',
    outputs: [
      {
        lockingScript: new P2PKH().lock('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa').toHex(),
        satoshis: 1000,
        outputDescription: 'Payment 1'
      },
      {
        lockingScript: new P2PKH().lock('1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2').toHex(),
        satoshis: 2000,
        outputDescription: 'Payment 2'
      },
      {
        lockingScript: new P2PKH().lock('1HLoD9E4SDFFPDiYfNYnkBLQ85Y51J3Zb1').toHex(),
        satoshis: 3000,
        outputDescription: 'Payment 3'
      }
    ]
  })

  console.log('Batch payment sent:', result.txid)
  console.log('Total sent: 6000 satoshis to 3 recipients')

  return result.txid
}
```

### Payment with Metadata (OP_RETURN)

```typescript
import { WalletClient, P2PKH, Script } from '@bsv/sdk'

async function sendPaymentWithMetadata(
  wallet: WalletClient,
  recipientAddress: string,
  amountSatoshis: number,
  metadata: any
) {
  // Create OP_RETURN script with metadata
  const metadataHex = Buffer.from(JSON.stringify(metadata)).toString('hex')
  const opReturnScript = new Script()
  opReturnScript.writeOpCode(Script.OP_FALSE)
  opReturnScript.writeOpCode(Script.OP_RETURN)
  opReturnScript.writeBin(Buffer.from(metadataHex, 'hex'))

  const result = await wallet.createAction({
    description: 'Payment with metadata',
    outputs: [
      {
        lockingScript: new P2PKH().lock(recipientAddress).toHex(),
        satoshis: amountSatoshis,
        outputDescription: 'Payment'
      },
      {
        lockingScript: opReturnScript.toHex(),
        satoshis: 0,
        outputDescription: 'Metadata'
      }
    ]
  })

  console.log('Payment with metadata sent:', result.txid)
  return result.txid
}

// Usage
await sendPaymentWithMetadata(
  wallet,
  '1A1zP1...',
  1000,
  {
    type: 'INVOICE_PAYMENT',
    invoiceId: 'INV-12345',
    description: 'Payment for services',
    timestamp: Date.now()
  }
)
```

---

## Part 5: Message Signing

### Sign and Verify Messages

```typescript
import { WalletClient } from '@bsv/sdk'

async function signMessage(wallet: WalletClient, message: string) {
  try {
    // Hash the message
    const messageBytes = Buffer.from(message, 'utf8')

    // Request wallet to sign the message data
    const { signature } = await wallet.createSignature({
      data: Array.from(messageBytes),
      protocolID: [0, 'message signing'],
      keyID: '1',
      description: 'Sign message'
    })

    console.log('Message signed!')
    console.log('Message:', message)
    console.log('Signature:', Buffer.from(signature).toString('hex'))

    return signature

  } catch (error: any) {
    if (error.code === 'USER_REJECTED') {
      console.log('User rejected signing')
    } else {
      console.error('Signing failed:', error.message)
    }
    throw error
  }
}

// Usage
const signature = await signMessage(wallet, 'I agree to terms of service')
```

### Authentication with Signatures

```typescript
// components/AuthWithWallet.tsx
import React, { useState } from 'react'
import { WalletClient } from '@bsv/sdk'

interface AuthProps {
  wallet: WalletClient
  onAuthenticated: (publicKey: string, signature: Uint8Array) => void
}

export const AuthWithWallet: React.FC<AuthProps> = ({ wallet, onAuthenticated }) => {
  const [authenticating, setAuthenticating] = useState(false)

  const handleAuth = async () => {
    setAuthenticating(true)

    try {
      // Get user's identity public key
      const { publicKey } = await wallet.getPublicKey({ identityKey: true })

      // Create challenge message
      const timestamp = Date.now()
      const challenge = `Login to MyApp at ${timestamp}`
      const messageBytes = Buffer.from(challenge, 'utf8')

      // Request signature
      const { signature } = await wallet.createSignature({
        data: Array.from(messageBytes),
        protocolID: [0, 'authentication'],
        keyID: '1',
        description: 'Authenticate to MyApp'
      })

      // Send to backend for verification
      const response = await fetch('/api/auth/wallet', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          publicKey,
          message: challenge,
          signature: Buffer.from(signature).toString('hex')
        })
      })

      if (response.ok) {
        const { token } = await response.json()
        localStorage.setItem('auth_token', token)
        onAuthenticated(publicKey, signature)
        console.log('✅ Authenticated with wallet')
      } else {
        throw new Error('Authentication failed')
      }

    } catch (error: any) {
      console.error('❌ Authentication failed:', error)
      alert(error.message)
    } finally {
      setAuthenticating(false)
    }
  }

  return (
    <button
      onClick={handleAuth}
      disabled={authenticating}
    >
      {authenticating ? 'Authenticating...' : 'Login with Wallet'}
    </button>
  )
}
```

---

## Part 6: Error Handling

### Common Error Codes

```typescript
try {
  await wallet.createAction({ description: '...', outputs: [...] })
} catch (error: any) {
  switch (error.code) {
    case 'USER_REJECTED':
      // User cancelled the transaction
      console.log('User rejected transaction')
      break

    case 'INSUFFICIENT_FUNDS':
      // User doesn't have enough balance
      console.log('Insufficient funds')
      alert('You don\'t have enough BSV for this transaction')
      break

    case 'WALLET_NOT_FOUND':
      // MetaNet Desktop Wallet not installed
      console.log('Wallet not installed')
      alert('Please install MetaNet Desktop Wallet')
      break

    case 'WALLET_LOCKED':
      // Wallet is locked
      console.log('Wallet is locked')
      alert('Please unlock your wallet')
      break

    case 'NETWORK_ERROR':
      // Broadcasting failed
      console.log('Network error')
      alert('Transaction failed to broadcast. Please try again.')
      break

    case 'INVALID_OUTPUT':
      // Output format is wrong
      console.log('Invalid output')
      alert('Invalid transaction output')
      break

    default:
      console.error('Unknown error:', error)
      alert(`Transaction failed: ${error.message}`)
  }
}
```

### Comprehensive Error Handler

```typescript
function handleWalletError(error: any, context: string) {
  const errorMessages: Record<string, string> = {
    'USER_REJECTED': 'You cancelled the request',
    'INSUFFICIENT_FUNDS': 'Insufficient balance in your wallet',
    'WALLET_NOT_FOUND': 'MetaNet Desktop Wallet not installed. Please install it from desktop.bsvb.tech',
    'WALLET_LOCKED': 'Please unlock your wallet and try again',
    'NETWORK_ERROR': 'Network error. Please check your connection and try again',
    'INVALID_OUTPUT': 'Invalid transaction format',
    'TIMEOUT': 'Request timed out. Please try again'
  }

  const userMessage = errorMessages[error.code] || `${context} failed: ${error.message}`

  console.error(`❌ ${context} error:`, {
    code: error.code,
    message: error.message,
    details: error
  })

  return userMessage
}

// Usage
try {
  await wallet.sendTransaction({ outputs: [...] })
} catch (error) {
  const message = handleWalletError(error, 'Payment')
  alert(message)
}
```

---

## Part 7: Best Practices

### 1. Always Check Connection State

```typescript
function ensureWalletConnected(wallet: WalletClient | null) {
  if (!wallet) {
    throw new Error('Wallet not connected. Please connect your wallet first.')
  }
}

// Usage
async function sendPayment(wallet: WalletClient | null, recipient: string, amount: number) {
  ensureWalletConnected(wallet)
  return await wallet!.sendTransaction({ outputs: [{ address: recipient, satoshis: amount }] })
}
```

### 2. Provide Clear Transaction Previews

```typescript
// Show user what they're approving BEFORE calling wallet
function TransactionPreview({ recipient, amount }: { recipient: string, amount: number }) {
  const fee = 500 // Estimated (wallet calculates actual)
  const total = amount + fee

  return (
    <div className="tx-preview">
      <h4>Transaction Preview</h4>
      <p>Recipient: {recipient}</p>
      <p>Amount: {amount} satoshis</p>
      <p>Est. Fee: ~{fee} satoshis</p>
      <p><strong>Total: ~{total} satoshis</strong></p>
      <p className="note">Final fee determined by wallet</p>
    </div>
  )
}
```

### 3. Handle User Cancellation Gracefully

```typescript
async function requestPayment(wallet: WalletClient, recipient: string, amount: number) {
  try {
    const result = await wallet.sendTransaction({
      outputs: [{ address: recipient, satoshis: amount }]
    })
    return { success: true, txid: result.txid }
  } catch (error: any) {
    if (error.code === 'USER_REJECTED') {
      // Don't show error for user cancellation - it's intentional
      return { success: false, cancelled: true }
    }
    // Show error for other failures
    return { success: false, error: error.message }
  }
}
```

### 4. Provide Fallback Instructions

```tsx
function WalletNotFound() {
  return (
    <div className="wallet-not-found">
      <h3>MetaNet Desktop Wallet Required</h3>
      <p>This dApp requires MetaNet Desktop Wallet to function.</p>

      <div className="instructions">
        <h4>Installation Steps:</h4>
        <ol>
          <li>Visit <a href="https://desktop.bsvb.tech" target="_blank" rel="noreferrer">desktop.bsvb.tech</a></li>
          <li>Download MetaNet Desktop Wallet</li>
          <li>Install and set up your wallet</li>
          <li>Return to this page and click "Connect Wallet"</li>
        </ol>
      </div>

      <div className="testnet-info">
        <h4>Get Testnet BSV:</h4>
        <p>Get testnet coins from MetaNet Desktop Wallet's faucet or <a href="https://discord.gg/bsv" target="_blank" rel="noreferrer">BSV Discord</a></p>
      </div>
    </div>
  )
}
```

### 5. Persist Connection State

```typescript
// Save connection preference
useEffect(() => {
  if (walletConnected) {
    localStorage.setItem('wallet_auto_connect', 'true')
  } else {
    localStorage.removeItem('wallet_auto_connect')
  }
}, [walletConnected])

// Auto-reconnect on page load
useEffect(() => {
  const shouldAutoConnect = localStorage.getItem('wallet_auto_connect') === 'true'
  if (shouldAutoConnect) {
    connectWallet()
  }
}, [])
```

---

## Part 8: Complete dApp Example

Here's a complete React dApp integrating all concepts:

```typescript
// App.tsx
import React from 'react'
import { useWallet } from './hooks/useWallet'
import { WalletConnect } from './components/WalletConnect'
import { SendPayment } from './components/SendPayment'
import { TransactionHistory } from './components/TransactionHistory'
import './App.css'

function App() {
  const { wallet, publicKey, connected, error } = useWallet()

  return (
    <div className="App">
      <header>
        <h1>My BSV dApp</h1>
        <WalletConnect />
      </header>

      <main>
        {connected && wallet ? (
          <>
            <div className="user-info">
              <h2>Your Wallet</h2>
              <p>Public Key: {publicKey?.substring(0, 20)}...</p>
            </div>

            <SendPayment wallet={wallet} />

            <TransactionHistory publicKey={publicKey!} />
          </>
        ) : (
          <div className="connect-prompt">
            <h2>Welcome to My BSV dApp</h2>
            <p>Connect your MetaNet Desktop Wallet to get started</p>
            {error && <p className="error">{error}</p>}
          </div>
        )}
      </main>

      <footer>
        <p>Built with BSV SDK and WalletClient</p>
        <a href="https://desktop.bsvb.tech" target="_blank" rel="noreferrer">
          Learn more about MetaNet Desktop Wallet
        </a>
      </footer>
    </div>
  )
}

export default App
```

---

## Summary

**What You Learned:**

- ✅ WalletClient connects your dApp to MetaNet Desktop Wallet
- ✅ Wallet handles all key management, UTXO selection, fees, and broadcasting
- ✅ Your dApp requests user approval for transactions
- ✅ Error handling for user rejections and failures
- ✅ Message signing for authentication
- ✅ Best practices for UX and state management

**What You DON'T Need to Do:**

- ❌ Manage private keys
- ❌ Track UTXOs
- ❌ Calculate fees
- ❌ Create change outputs
- ❌ Configure ARC
- ❌ Handle low-level transaction details

**The wallet does all of this automatically!**

---

## Next Steps

- **[Building Transactions](../first-transaction/)** - Learn transaction structure and SDK methods
- **[BSV Fundamentals](../bsv-fundamentals/)** - Understand UTXOs, scripts, and blockchain concepts
- **Intermediate Projects** - Build real-world dApps with WalletClient

---

## Resources

- **Wallet Toolbox API**: https://fast.brc.dev/
- **BRC Standards**: https://hub.bsvblockchain.org/brc
- **MetaNet Desktop Wallet**: https://desktop.bsvb.tech/
- **Get BSV - Orange Gateway**: https://hub.bsvblockchain.org/demos-and-onboardings/onboardings/onboarding-catalog/get-bsv/orange-gateway
- **SDK Documentation**: https://bsv-blockchain.github.io/ts-sdk/
- **Example dApps**: [code-features/](../../../code-features/)

---

**You're now ready to build non-custodial BSV dApps!**
