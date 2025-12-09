# Project: Desktop Wallet Management

**Comprehensive Wallet Data Integration with WalletClient**

Build a Next.js application that connects to the BSV Desktop Wallet to retrieve and display comprehensive wallet information. This project demonstrates how to use the BSV SDK's WalletClient to access identity keys, derive keys using different protocols, manage baskets, check balances, and explore the full capabilities of wallet integration.

**Repository**: [github.com/bsv-blockchain-demos/desktop-wallet-data](https://github.com/bsv-blockchain-demos/desktop-wallet-data)

---

## What You'll Build

A production-ready wallet information dashboard featuring:

- Wallet connection and disconnection flow
- Identity key retrieval and display
- Public key derivation (Identity, Payment, Data)
- Address generation from public keys
- Custom key derivation with different protocols
- Network status detection (mainnet/testnet)
- Wallet balance checking
- Basket management and output listing
- Complete wallet capabilities discovery

---

## Learning Objectives

By completing this project, you will learn:

- **WalletClient Integration** - Connecting to and communicating with BSV Desktop Wallet
- **Identity Management** - Retrieving and working with identity keys
- **Key Derivation** - Using different protocol IDs to derive specialized keys
- **Public Key Types** - Understanding Identity, Payment, and Data key types
- **Address Generation** - Converting public keys to BSV addresses
- **Basket Management** - Organizing wallet outputs using baskets
- **Balance Checking** - Querying wallet balance for different addresses
- **Wallet Capabilities** - Discovering available wallet methods (encryption, signing, certificates, HMAC, etc.)

---

## Architecture Overview

```
┌─────────────────────────────────────────┐
│         Frontend (Next.js/React)        │
│  - WalletClient connection              │
│  - Identity key retrieval               │
│  - Public key derivation                │
│  - Address generation                   │
│  - Balance queries                      │
│  - Basket management                    │
│  - Capabilities discovery               │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│      BSV Desktop Wallet (Local)         │
│  - User authentication                  │
│  - Key management                       │
│  - Signature generation                 │
│  - Transaction creation                 │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│           BSV Blockchain                │
│  - Network queries                      │
│  - Balance lookups                      │
│  - UTXO management                      │
└─────────────────────────────────────────┘
```

---

## Key Patterns

### 1. Wallet Connection

Initialize and connect to the BSV Desktop Wallet using a React hook:

```typescript
import { useState, useEffect } from 'react'
import { WalletClient } from '@bsv/sdk'

export const useWallet = () => {
  const [wallet, setWallet] = useState<WalletClient | null>(null)
  const [identityKey, setIdentityKey] = useState<string | null>(null)
  const [isConnected, setIsConnected] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const connect = async () => {
    try {
      const walletClient = new WalletClient()
      const { publicKey } = await walletClient.getPublicKey({
        identityKey: true
      })

      setWallet(walletClient)
      setIdentityKey(publicKey)
      setIsConnected(true)
      setError(null)
    } catch (err) {
      setError('Failed to connect to wallet')
      console.error(err)
    }
  }

  const disconnect = () => {
    setWallet(null)
    setIdentityKey(null)
    setIsConnected(false)
  }

  return { wallet, identityKey, isConnected, error, connect, disconnect }
}
```

> **Reference**: [WalletClient Integration](../../beginner/wallet-client-integration/README.md)

### 2. Identity Key Retrieval

The identity key is the master key that identifies the wallet:

```typescript
// Retrieve identity key
const { publicKey: identityKey } = await wallet.getPublicKey({
  identityKey: true
})

console.log('Identity Key:', identityKey)
```

> **Reference**: [Public Keys](../../../sdk-components/public-keys/README.md)

### 3. Public Key Derivation

Derive different types of public keys using protocol IDs:

```typescript
// Get Identity public key
const { publicKey: identityPubKey } = await wallet.getPublicKey({
  identityKey: true
})

// Get Payment public key
const { publicKey: paymentPubKey } = await wallet.getPublicKey({
  protocolID: 'payments',
  keyID: '1'
})

// Get Data public key
const { publicKey: dataPubKey } = await wallet.getPublicKey({
  protocolID: 'data',
  keyID: '1'
})
```

**Key Types Explained**:
- **Identity Key**: Master key representing the wallet's identity
- **Payment Keys**: Keys used for receiving payments
- **Data Keys**: Keys used for data encryption and signing

> **Reference**: [HD Wallets](../../../sdk-components/hd-wallets/README.md)

### 4. Address Generation

Convert public keys to BSV addresses:

```typescript
import { PublicKey } from '@bsv/sdk'

// Convert public key to address
const pubKey = PublicKey.fromString(publicKeyString)
const address = pubKey.toAddress()

console.log('BSV Address:', address)
```

**Address Types**:
```typescript
// Identity Address
const identityAddress = PublicKey.fromString(identityKey).toAddress()

// Payment Address
const paymentAddress = PublicKey.fromString(paymentKey).toAddress()

// Data Address
const dataAddress = PublicKey.fromString(dataKey).toAddress()
```

> **Reference**: [Public Keys](../../../sdk-components/public-keys/README.md) | [P2PKH](../../../sdk-components/p2pkh/README.md)

### 5. Custom Key Derivation

Derive keys using custom protocols and key IDs:

```typescript
// Derive key with custom protocol
const { publicKey: derivedKey } = await wallet.getPublicKey({
  protocolID: 'custom-protocol',
  keyID: 'my-key-id',
  counterparty: 'self',
  forSelf: true
})

console.log('Derived Key:', derivedKey)
```

**Protocol Examples**:
```typescript
// Test protocol
const testKey = await wallet.getPublicKey({
  protocolID: 'test',
  keyID: '1'
})

// Custom application protocol
const appKey = await wallet.getPublicKey({
  protocolID: [0, 'my-app'],
  keyID: 'user-123'
})
```

> **Reference**: [BRC-42](../../../sdk-components/brc-42/README.md) | [HD Wallets](../../../sdk-components/hd-wallets/README.md)

### 6. Network Detection

Detect which network the wallet is connected to:

```typescript
// Get network information
const network = await wallet.getNetwork()

console.log('Network:', network) // 'mainnet' or 'testnet'
```

This is useful for:
- Displaying network status to users
- Adjusting UI based on network
- Preventing mainnet operations in testnet environment
- Showing appropriate block explorers

### 7. Balance Checking

Query wallet balance for specific addresses:

```typescript
// Check balance for an address
const balance = await wallet.getBalance({
  address: bsvAddress
})

console.log('Balance:', balance, 'satoshis')
```

**Multiple Address Balances**:
```typescript
// Check identity address balance
const identityBalance = await wallet.getBalance({
  address: identityAddress
})

// Check payment address balance
const paymentBalance = await wallet.getBalance({
  address: paymentAddress
})

// Total balance
const totalBalance = identityBalance + paymentBalance
```

> **Reference**: [UTXO Management](../../../sdk-components/utxo-management/README.md)

### 8. Basket Management

Baskets organize wallet outputs into categories:

```typescript
// List available baskets
const baskets = await wallet.listBaskets()

console.log('Available Baskets:', baskets)

// Get outputs from specific basket
const outputs = await wallet.listOutputs({
  basket: 'tokens',
  includeEnvelope: true
})

console.log('Token Outputs:', outputs)
```

**Basket Use Cases**:
- **tokens**: Store token outputs separately from payment UTXOs
- **receipts**: Keep payment receipts organized
- **certificates**: Manage certificate outputs
- **custom**: Application-specific organization

**Creating Outputs in Baskets**:
```typescript
await wallet.createAction({
  outputs: [{
    lockingScript: script.toHex(),
    satoshis: 1000,
    basket: 'tokens',
    outputDescription: 'My Token'
  }]
})
```

> **Reference**: [UTXO Management](../../../sdk-components/utxo-management/README.md)

### 9. Wallet Capabilities Discovery

Discover all available wallet methods:

```typescript
// Check available wallet capabilities
const capabilities = {
  // Key management
  hasGetPublicKey: typeof wallet.getPublicKey === 'function',
  hasRevealCounterpartyKeyLinkage: typeof wallet.revealCounterpartyKeyLinkage === 'function',
  hasRevealSpecificKeyLinkage: typeof wallet.revealSpecificKeyLinkage === 'function',

  // Encryption
  hasEncrypt: typeof wallet.encrypt === 'function',
  hasDecrypt: typeof wallet.decrypt === 'function',

  // Signing
  hasCreateSignature: typeof wallet.createSignature === 'function',
  hasVerifySignature: typeof wallet.verifySignature === 'function',

  // HMAC
  hasCreateHmac: typeof wallet.createHmac === 'function',
  hasVerifyHmac: typeof wallet.verifyHmac === 'function',

  // Certificates
  hasAcquireCertificate: typeof wallet.acquireCertificate === 'function',
  hasListCertificates: typeof wallet.listCertificates === 'function',
  hasProveCertificate: typeof wallet.proveCertificate === 'function',

  // Transactions
  hasCreateAction: typeof wallet.createAction === 'function',
  hasInternalizeAction: typeof wallet.internalizeAction === 'function',
  hasListActions: typeof wallet.listActions === 'function',

  // Outputs
  hasListOutputs: typeof wallet.listOutputs === 'function',
  hasRelinquishOutput: typeof wallet.relinquishOutput === 'function',

  // Baskets
  hasGetBasket: typeof wallet.getBasket === 'function',
  hasListBaskets: typeof wallet.listBaskets === 'function',

  // Network
  hasGetNetwork: typeof wallet.getNetwork === 'function',
  hasGetBalance: typeof wallet.getBalance === 'function'
}

console.log('Wallet Capabilities:', capabilities)
```

This allows you to:
- Check which features are available
- Build adaptive UIs based on capabilities
- Provide fallback functionality
- Discover new wallet features

---

## Component Structure

### WalletConnector Component

The main component that displays all wallet information:

```typescript
import { useWallet } from '../hooks/useWallet'
import { PublicKey } from '@bsv/sdk'

export default function WalletConnector() {
  const { wallet, identityKey, isConnected, error, connect, disconnect } = useWallet()
  const [walletData, setWalletData] = useState(null)

  const fetchWalletData = async () => {
    if (!wallet) return

    try {
      // Get public keys
      const { publicKey: identityPubKey } = await wallet.getPublicKey({
        identityKey: true
      })
      const { publicKey: paymentPubKey } = await wallet.getPublicKey({
        protocolID: 'payments',
        keyID: '1'
      })
      const { publicKey: dataPubKey } = await wallet.getPublicKey({
        protocolID: 'data',
        keyID: '1'
      })

      // Generate addresses
      const identityAddress = PublicKey.fromString(identityPubKey).toAddress()
      const paymentAddress = PublicKey.fromString(paymentPubKey).toAddress()
      const dataAddress = PublicKey.fromString(dataPubKey).toAddress()

      // Get network
      const network = await wallet.getNetwork()

      // Get balances
      const identityBalance = await wallet.getBalance({ address: identityAddress })
      const paymentBalance = await wallet.getBalance({ address: paymentAddress })

      // Get baskets
      const baskets = await wallet.listBaskets()

      setWalletData({
        identityKey: identityPubKey,
        publicKeys: {
          identity: identityPubKey,
          payment: paymentPubKey,
          data: dataPubKey
        },
        addresses: {
          identity: identityAddress,
          payment: paymentAddress,
          data: dataAddress
        },
        network,
        balances: {
          identity: identityBalance,
          payment: paymentBalance,
          total: identityBalance + paymentBalance
        },
        baskets
      })
    } catch (err) {
      console.error('Error fetching wallet data:', err)
    }
  }

  return (
    <div>
      {!isConnected ? (
        <button onClick={connect}>Connect Wallet</button>
      ) : (
        <>
          <button onClick={disconnect}>Disconnect</button>
          <button onClick={fetchWalletData}>Fetch All Data</button>

          {walletData && (
            <div>
              <section>
                <h2>Identity Key</h2>
                <code>{walletData.identityKey}</code>
              </section>

              <section>
                <h2>Addresses</h2>
                <div>Identity: {walletData.addresses.identity}</div>
                <div>Payment: {walletData.addresses.payment}</div>
                <div>Data: {walletData.addresses.data}</div>
              </section>

              <section>
                <h2>Public Keys</h2>
                <div>Identity: {walletData.publicKeys.identity}</div>
                <div>Payment: {walletData.publicKeys.payment}</div>
                <div>Data: {walletData.publicKeys.data}</div>
              </section>

              <section>
                <h2>Network</h2>
                <div>{walletData.network}</div>
              </section>

              <section>
                <h2>Balances</h2>
                <div>Identity: {walletData.balances.identity} sats</div>
                <div>Payment: {walletData.balances.payment} sats</div>
                <div>Total: {walletData.balances.total} sats</div>
              </section>

              <section>
                <h2>Baskets</h2>
                <div>{JSON.stringify(walletData.baskets, null, 2)}</div>
              </section>
            </div>
          )}
        </>
      )}
    </div>
  )
}
```

### Additional Components

**DerivedKeyGenerator**: Interactive component for deriving keys with custom protocols

**WalletBalance**: Real-time balance display for multiple addresses

**BasketManager**: Interface for creating, viewing, and managing baskets

---

## Project Structure

```
desktop-wallet-data/
├── app/
│   ├── layout.tsx         # Root layout
│   ├── page.tsx           # Main page
│   └── globals.css        # Global styles
├── components/
│   ├── WalletConnector.tsx         # Main wallet component
│   ├── DerivedKeyGenerator.tsx     # Custom key derivation
│   ├── WalletBalance.tsx           # Balance display
│   └── BasketManager.tsx           # Basket management
├── hooks/
│   └── useWallet.ts       # Wallet connection hook
├── package.json           # Dependencies
└── tsconfig.json          # TypeScript config
```

---

## Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/bsv-blockchain-demos/desktop-wallet-data
   cd desktop-wallet-data
   npm install
   ```

2. **Install BSV Desktop Wallet**

   Download and install the BSV Desktop Wallet from [desktop.bsvb.tech](https://desktop.bsvb.tech/)

   Make sure the wallet is running before connecting from the application.

3. **Start the application**
   ```bash
   npm run dev
   ```

   Open [http://localhost:3000](http://localhost:3000) in your browser.

4. **Connect to Wallet**

   Click "Connect Wallet" and approve the connection in your BSV Desktop Wallet.

5. **Explore Wallet Data**

   Click "Fetch All Data" to retrieve and display comprehensive wallet information.

---

## Important Concepts

### WalletClient vs Wallet Toolbox

**WalletClient** (used in this project):
- Frontend integration for user-controlled wallets
- Connects to BSV Desktop Wallet
- User must approve operations
- Keys remain secure in user's wallet
- Best for dApps and consumer applications

**Wallet Toolbox** (not used in this project):
- Backend wallet management
- Application controls keys
- Automatic transaction signing
- Best for services and custodial solutions

> **Reference**: [WalletClient Integration](../../beginner/wallet-client-integration/README.md)

### Protocol IDs

Protocol IDs are used to derive different types of keys:

```typescript
// String protocol
protocolID: 'payments'

// Array protocol [securityLevel, protocolName]
protocolID: [0, 'my-app']
protocolID: [2, 'high-security']
```

**Security Levels**:
- `0`: No security requirements
- `1`: Medium security
- `2`: High security (requires user authentication)

> **Reference**: [BRC-42](../../../sdk-components/brc-42/README.md) | [BRC-43](https://brc.dev/43)

### Public Key vs Address

**Public Key**: Raw cryptographic key (hex string, 66 characters)
```
03f3d5d6f4a4b8c9e2f1a3b5c7d9e1f3a5b7c9d1e3f5a7b9c1d3e5f7a9b1c3d5e7
```

**Address**: Human-readable identifier derived from public key
```
1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa
```

**Conversion**:
```typescript
const pubKey = PublicKey.fromString(publicKeyString)
const address = pubKey.toAddress()
```

### Baskets

Baskets are named containers for organizing wallet outputs:

- Keep different types of outputs separated
- Easier to query specific output types
- Better organization for complex applications
- Prevents accidental spending of important outputs

**Common Basket Names**:
- `default`: Standard payment outputs
- `tokens`: Token outputs
- `receipts`: Payment receipts
- `certificates`: Certificate outputs
- Custom names for your application

---

## Use Cases

This project's patterns enable:

1. **Wallet Information Dashboards**
   - Display user wallet details
   - Show balances across addresses
   - List available baskets and outputs

2. **Multi-Key Applications**
   - Derive different keys for different purposes
   - Separate payment and data keys
   - Protocol-specific key generation

3. **Developer Tools**
   - Wallet capability discovery
   - Key derivation testing
   - Network status monitoring

4. **Portfolio Management**
   - Track balances across multiple addresses
   - Organize outputs using baskets
   - Monitor token holdings

5. **Identity Management**
   - Identity key display and verification
   - Address generation for different contexts
   - Counterparty key derivation

---

## Summary

This project demonstrates:

- **WalletClient Integration** - Connecting to BSV Desktop Wallet
- **Identity Management** - Retrieving and displaying identity keys
- **Key Derivation** - Deriving keys with different protocols and key IDs
- **Address Generation** - Converting public keys to addresses
- **Balance Queries** - Checking balances for different addresses
- **Basket Management** - Organizing wallet outputs
- **Capabilities Discovery** - Exploring available wallet features
- **Network Detection** - Identifying mainnet vs testnet

These patterns form the foundation for building user-facing BSV applications that integrate with the BSV Desktop Wallet, giving users full control over their keys and transactions.

---

## Related Resources

- [WalletClient Integration](../../beginner/wallet-client-integration/README.md)
- [Public Keys](../../../sdk-components/public-keys/README.md)
- [HD Wallets](../../../sdk-components/hd-wallets/README.md)
- [BRC-42 (Key Derivation)](../../../sdk-components/brc-42/README.md)
- [UTXO Management](../../../sdk-components/utxo-management/README.md)
- [P2PKH](../../../sdk-components/p2pkh/README.md)
- [BSV Desktop Wallet](https://desktop.bsvb.tech/)
- [Wallet Toolbox Documentation](https://fast.brc.dev/)
