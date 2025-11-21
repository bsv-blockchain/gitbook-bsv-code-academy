# Development Paradigms in BSV

**Module 0: Understanding Your Development Approach**

Before diving into BSV development, it's crucial to understand the two fundamental architectural paradigms for building BSV applications. Your choice determines how you structure your application, manage keys, handle transactions, and integrate with users.

---

## Overview

BSV applications can be built using two distinct paradigms:

1. **Backend/Service Development** - Custodial wallet management where your application controls private keys
2. **Frontend Integration Development** - Non-custodial where users control their own wallets

Each paradigm serves different use cases, has different security considerations, and requires different SDK components.

---

## Paradigm 1: Backend/Service Development

### What It Is

Backend development is when **your application manages wallets and private keys on behalf of users or for internal business operations**. The application acts as a custodian, controlling the signing of transactions server-side.

### Architecture

```
┌──────────────────────────────────────┐
│      Your Backend Service            │
│                                      │
│  ┌────────────────────────────────┐ │
│  │  Private Key Storage           │ │
│  │  (Secure server-side)          │ │
│  └────────────────────────────────┘ │
│              ↓                       │
│  ┌────────────────────────────────┐ │
│  │  BSV SDK                       │ │
│  │  - Transaction building        │ │
│  │  - Signing with stored keys    │ │
│  │  - Broadcasting                │ │
│  └────────────────────────────────┘ │
│              ↓                       │
│  ┌────────────────────────────────┐ │
│  │  BSV Blockchain                │ │
│  └────────────────────────────────┘ │
└──────────────────────────────────────┘
           ↕ (API)
┌──────────────────────────────────────┐
│      User Interface                  │
│  (Web/Mobile - no keys)              │
└──────────────────────────────────────┘
```

### Use Cases

**Perfect for:**
- ✅ **Automated payment systems** - Server processes payments without user interaction
- ✅ **Internal business operations** - Supply chain tracking, invoicing, B2B transactions
- ✅ **Enterprise wallet services** - Companies managing wallets for employees
- ✅ **Custody solutions** - Exchanges, payment processors, merchant services
- ✅ **Tokenization platforms** - Server mints and transfers tokens programmatically
- ✅ **Recurring billing** - Subscription services with automatic charges
- ✅ **Batch processing** - High-volume transaction processing
- ✅ **Business wallets** - Corporate treasury management

**Not ideal for:**
- ❌ Consumer-facing dApps where users expect wallet sovereignty
- ❌ Applications where regulatory compliance requires non-custodial solutions
- ❌ Use cases requiring user signature authentication

### Technical Stack

**Core SDK Components:**
```typescript
import {
  PrivateKey,     // Generate and manage keys server-side
  Transaction,    // Build transactions programmatically
  P2PKH,          // Standard payment template
  ARC            // Broadcast to BSV network
} from '@bsv/sdk'
```

**Key Operations:**
- **Key Management**: Generate and securely store private keys in your database/vault
- **UTXO Tracking**: Maintain UTXO database for your managed wallets
- **Transaction Building**: Create transactions programmatically using SDK
- **Automated Signing**: Sign with stored private keys
- **Broadcasting**: Submit transactions to blockchain via ARC

### Simple Backend Example

```typescript
import { PrivateKey, Transaction, P2PKH, ARC } from '@bsv/sdk'

class BackendWalletService {
  private privateKey: PrivateKey

  constructor() {
    // In production: load from secure storage/vault
    this.privateKey = PrivateKey.fromWIF('your-stored-wif')
  }

  async sendPayment(recipientAddress: string, amountSatoshis: number) {
    // 1. Build transaction (SDK handles UTXO selection, fees, change)
    const tx = new Transaction()

    // 2. Add payment output
    tx.addOutput({
      satoshis: amountSatoshis,
      lockingScript: new P2PKH().lock(recipientAddress)
    })

    // 3. Sign with your stored key
    await tx.sign(this.privateKey)

    // 4. Broadcast (SDK handles ARC connection)
    const txid = await tx.broadcast()

    return txid
  }

  // Your API endpoint
  async handlePaymentRequest(req, res) {
    const { recipient, amount } = req.body
    const txid = await this.sendPayment(recipient, amount)
    res.json({ success: true, txid })
  }
}
```

### Security Considerations

**Critical Requirements:**
- 🔒 **Secure key storage** - Use HSMs, key vaults, or encrypted databases
- 🔒 **Access control** - Strict authentication for key access
- 🔒 **Audit logging** - Track all transaction creation and signing
- 🔒 **Key rotation** - Regular key updates for high-value wallets
- 🔒 **Cold storage** - Keep most funds in offline wallets
- 🔒 **Rate limiting** - Prevent automated attacks on your service
- 🔒 **Multi-signature** - Require multiple approvals for large transactions

**Compliance:**
- You are a **custodian** - subject to financial regulations
- May require KYC/AML procedures
- Responsible for user funds security
- Need insurance and security audits

---

## Paradigm 2: Frontend Integration Development

### What It Is

Frontend development is when **users control their own wallets and private keys**. Your application requests the user's wallet to sign transactions, but never has access to their private keys.

### Architecture

```
┌──────────────────────────────────────┐
│      User's Browser/Device           │
│                                      │
│  ┌────────────────────────────────┐ │
│  │  Your dApp Frontend            │ │
│  │                                │ │
│  │  - UI/UX                       │ │
│  │  - Transaction requests        │ │
│  │  - WalletClient integration    │ │
│  └────────────────────────────────┘ │
│              ↕                       │
│        (BRC Protocol)                │
│              ↕                       │
│  ┌────────────────────────────────┐ │
│  │  MetaNet Desktop Wallet        │ │
│  │  (BSV Desktop)                 │ │
│  │                                │ │
│  │  - Private keys (user-owned)   │ │
│  │  - Transaction signing         │ │
│  │  - UTXO management             │ │
│  │  - Broadcasting                │ │
│  └────────────────────────────────┘ │
│              ↓                       │
└──────────────────────────────────────┘
               ↓
┌──────────────────────────────────────┐
│      BSV Blockchain                  │
└──────────────────────────────────────┘
```

### Use Cases

**Perfect for:**
- ✅ **Consumer dApps** - Social platforms, marketplaces, content platforms
- ✅ **Web3 applications** - Where user sovereignty is expected
- ✅ **Peer-to-peer systems** - Direct user-to-user transactions
- ✅ **NFT marketplaces** - User-owned digital assets
- ✅ **DeFi applications** - Decentralized finance requiring non-custodial approach
- ✅ **Identity applications** - Self-sovereign identity systems
- ✅ **Gaming dApps** - User owns in-game assets
- ✅ **Voting systems** - Cryptographic signature authentication

**Not ideal for:**
- ❌ Automated backend processes without user interaction
- ❌ High-frequency trading (too much user confirmation friction)
- ❌ Internal business processes where company controls keys
- ❌ Services requiring custody (exchanges, payment processors)

### Technical Stack

**Core SDK Components:**
```typescript
import { WalletClient } from '@bsv/sdk'
```

**That's it!** The WalletClient handles everything:
- Connection to MetaNet Desktop Wallet
- Transaction signing requests
- UTXO management (wallet handles)
- Fee calculation (wallet handles)
- Broadcasting (wallet handles)

### Simple Frontend Example

```typescript
import { WalletClient, P2PKH } from '@bsv/sdk'

class PaymentDApp {
  private wallet: WalletClient

  async connectWallet() {
    // 1. Connect to user's MetaNet Desktop Wallet
    this.wallet = new WalletClient('auto')
    await this.wallet.connectToSubstrate()

    // 2. Get user's identity public key
    const { publicKey } = await this.wallet.getPublicKey({
      identityKey: true
    })

    console.log('Connected to wallet, public key:', publicKey)
    return publicKey
  }

  async requestPayment(recipientAddress: string, amountSatoshis: number) {
    // 1. Create action (wallet automatically signs and broadcasts)
    const result = await this.wallet.createAction({
      description: 'Send payment',
      outputs: [{
        lockingScript: new P2PKH().lock(recipientAddress).toHex(),
        satoshis: amountSatoshis,
        outputDescription: 'Payment to recipient'
      }]
    })

    // Wallet handles:
    // - UTXO selection
    // - Fee calculation
    // - Change outputs
    // - Transaction signing
    // - Broadcasting

    // 2. Get transaction ID
    return result.txid
  }

  // React component example
  async handlePayButtonClick() {
    try {
      const txid = await this.requestPayment(
        'recipient-address',
        1000 // satoshis
      )

      alert(`Payment sent! TXID: ${txid}`)
    } catch (error) {
      if (error.code === 'USER_REJECTED') {
        alert('Payment cancelled by user')
      } else {
        alert(`Payment failed: ${error.message}`)
      }
    }
  }
}
```

### MetaNet Desktop Wallet (BSV Desktop)

**The Standard Wallet:**

This course **exclusively teaches MetaNet Desktop Wallet integration**. It is the reference implementation of the BRC wallet specification and provides:

- ✅ Full BRC standard compliance
- ✅ Seamless WalletClient integration
- ✅ Best-in-class user experience
- ✅ Complete UTXO and key management
- ✅ Automatic fee calculation
- ✅ Built-in transaction broadcasting

**Official Resources:**
- **Wallet Toolbox Documentation**: https://fast.brc.dev/
- **BRC Standards**: https://hub.bsvblockchain.org/brc
- **MetaNet Desktop Wallet**: https://desktop.bsvb.tech/

**Other wallets** (Panda, Yours, etc.) are **not covered** in this course. Stick with MetaNet Desktop Wallet for standardized, reliable integration.

### Security Considerations

**Advantages:**
- ✅ **Non-custodial** - Users control their keys
- ✅ **No key storage risk** - Your application never sees private keys
- ✅ **Regulatory clarity** - Not a custodian, fewer compliance burdens
- ✅ **User sovereignty** - Users can export and use keys elsewhere
- ✅ **Lower liability** - User responsible for key security

**User Responsibilities:**
- 🔑 User must secure their seed phrase
- 🔑 User must manage wallet backups
- 🔑 User must approve each transaction
- 🔑 Lost keys = lost funds (no recovery)

**Your Responsibilities:**
- ✅ Clear UX for wallet connection
- ✅ Transaction request transparency (show amounts, fees)
- ✅ Error handling for user rejections
- ✅ Guidance on wallet security best practices

---

## Comparison Table

| Aspect | Backend/Service | Frontend Integration |
|--------|-----------------|----------------------|
| **Key Control** | Application controls keys | User controls keys |
| **Best For** | Business operations, automated systems | Consumer dApps, peer-to-peer apps |
| **SDK Usage** | PrivateKey, Transaction, ARC | WalletClient only |
| **Complexity** | Higher - manage keys, UTXOs, security | Lower - wallet handles everything |
| **User Experience** | Seamless (no wallet prompts) | Requires user approval per transaction |
| **Regulatory** | Custodial (strict regulations) | Non-custodial (fewer regulations) |
| **Security Risk** | High (you secure keys) | Lower (user secures keys) |
| **Scalability** | High (automated processing) | Medium (user confirmation friction) |
| **Transaction Speed** | Instant (programmatic) | Depends on user approval |

---

## Choosing Your Paradigm

### Ask These Questions:

1. **Who owns the funds?**
   - Company/service → Backend
   - End users → Frontend

2. **Is user interaction required?**
   - No (automated) → Backend
   - Yes (user approves) → Frontend

3. **What's the transaction volume?**
   - High-frequency → Backend
   - User-initiated → Frontend

4. **What's the regulatory environment?**
   - Can handle custody regulations → Backend
   - Need non-custodial → Frontend

5. **What's the use case?**
   - Internal business → Backend
   - Consumer dApp → Frontend

---

## Hybrid Approaches

Many applications use **both paradigms**:

### Example: E-commerce Platform

**Backend (Merchant wallet):**
- Merchant's business wallet managed server-side
- Receives customer payments
- Processes refunds programmatically
- Handles batch payouts

**Frontend (Customer wallet):**
- Customers use MetaNet Desktop Wallet
- Connect wallet to make purchases
- Sign payment transactions
- Receive refunds to their wallet

```typescript
// Backend: Merchant service
class MerchantWallet {
  async receivePayment(txid: string) {
    // Monitor blockchain for incoming payment
    // Credit customer account
    // Process order
  }

  async processRefund(customerAddress: string, amount: number) {
    // Sign refund transaction with merchant key
    // Broadcast automatically
  }
}

// Frontend: Customer checkout
class CheckoutPage {
  async completePayment() {
    // Request user's wallet to send payment
    const wallet = new WalletClient('auto')
    await wallet.connectToSubstrate()

    const result = await wallet.createAction({
      description: 'Payment for order',
      outputs: [{
        lockingScript: new P2PKH().lock(merchantAddress).toHex(),
        satoshis: orderTotal,
        outputDescription: 'Order payment'
      }]
    })

    // Send txid to backend for confirmation
    await api.confirmPayment(result.txid)
  }
}
```

---

## Next Steps

Now that you understand the two paradigms, the course content is organized to teach both:

### For Backend Development:
- **Module 1**: Environment Setup (Backend)
- **Module 2**: BSV Fundamentals
- **Module 3**: Managing Wallets Server-Side
- **Module 4**: Building Transactions Programmatically

### For Frontend Development:
- **Module 1**: Environment Setup (Frontend)
- **Module 2**: BSV Fundamentals
- **Module 3**: WalletClient Integration
- **Module 4**: Building dApps with WalletClient

### Universal:
- **Module 2: BSV Fundamentals** applies to both paradigms (UTXOs, transactions, scripts)

**Most developers will use both paradigms in different projects**, so it's valuable to understand both approaches.

---

## Key Takeaways

1. **Backend/Service** = You control keys, programmatic signing, custodial
2. **Frontend/Integration** = User controls keys, wallet signing, non-custodial
3. **Backend** uses `PrivateKey`, `Transaction`, `ARC` from SDK
4. **Frontend** uses `WalletClient` exclusively
5. **MetaNet Desktop Wallet** is the standard wallet for frontend integration
6. Most production applications use **both paradigms** for different parts

**Choose the paradigm** that matches your use case, security requirements, and user experience goals.

---

## Resources

- **Wallet Toolbox Documentation**: https://fast.brc.dev/
- **BRC Standards**: https://hub.bsvblockchain.org/brc
- **MetaNet Desktop Wallet**: https://desktop.bsvb.tech/
- **Get BSV - Orange Gateway**: https://hub.bsvblockchain.org/demos-and-onboardings/onboardings/onboarding-catalog/get-bsv/orange-gateway
- **SDK Documentation**: https://bsv-blockchain.github.io/ts-sdk/
- **Security Best Practices**: [sdk-components/private-keys/](../../../sdk-components/private-keys/)

---

**Next Module**: [Environment Setup](../development-environment/) - Setting up your development environment for your chosen paradigm
