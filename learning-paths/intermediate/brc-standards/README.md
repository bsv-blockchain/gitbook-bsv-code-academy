# BRC Standards Implementation

## Overview

Bitcoin Request for Comments (BRC) standards define protocols and patterns for interoperability across the BSV ecosystem. This module teaches you how to implement key BRC standards in your applications.

## What are BRC Standards?

BRC standards are community-driven specifications that define:
- Common protocols and patterns
- Interoperability requirements
- Best practices
- Reference implementations

Think of them as the "RFC" or "EIP" equivalents for BSV blockchain.

## Why BRC Standards Matter

1. **Interoperability**: Applications can communicate seamlessly
2. **Consistency**: Shared patterns reduce development complexity
3. **Security**: Peer-reviewed security practices
4. **Compatibility**: Future-proof your applications
5. **Community**: Leverage collective knowledge

## Core BRC Standards

### BRC-42: Key Derivation Protocol

**Purpose**: Deterministic, protocol-specific key derivation

**Use Cases**:
- Application-specific keys
- Counterparty-specific encryption
- Invoice-specific payments
- Service authentication

**Implementation**:
```typescript
import { PrivateKey } from '@bsv/sdk'

const masterKey = PrivateKey.fromWif('your-master-key')

// Derive protocol-specific key
const protocolID = PrivateKey.fromString('my-app')
const keyID = PrivateKey.fromString('encryption')

const derivedKey = masterKey.deriveChild(protocolID, keyID)

// For counterparty-specific keys
const counterpartyPubKey = /* ... */
const sharedKey = masterKey.deriveChild(
  protocolID,
  keyID,
  counterpartyPubKey
)

// For invoice-specific keys
const invoiceNumber = '12345'
const invoiceKey = masterKey.deriveChild(
  protocolID,
  keyID,
  counterpartyPubKey,
  invoiceNumber
)
```

**Benefits**:
- No need to store multiple private keys
- Deterministic key recovery
- Privacy through key isolation
- Scalable to unlimited keys

**Learn More**: [BRC-42 SDK Component](../../../sdk-components/brc-42/README.md)

### BRC-43: Wallet Security Levels

**Purpose**: Define security contexts for key derivation

**Security Levels**:
- **0**: Encrypted storage (least secure, most convenient)
- **1**: Password required
- **2**: Two-factor authentication required

**Implementation**:
```typescript
interface SecurityContext {
  level: 0 | 1 | 2
  authenticate?: () => Promise<boolean>
}

async function deriveKeyWithSecurity(
  masterKey: PrivateKey,
  protocolID: PrivateKey,
  keyID: PrivateKey,
  securityLevel: number
): Promise<PrivateKey> {
  // Level 2: Require 2FA
  if (securityLevel === 2) {
    const twoFactorValid = await verify2FA()
    if (!twoFactorValid) throw new Error('2FA required')
  }

  // Level 1: Require password
  if (securityLevel >= 1) {
    const passwordValid = await verifyPassword()
    if (!passwordValid) throw new Error('Password required')
  }

  return masterKey.deriveChild(protocolID, keyID)
}
```

**Use Cases**:
- Low-value keys: Level 0 (cached)
- Medium-value keys: Level 1 (password)
- High-value keys: Level 2 (2FA)

### BRC-29: Payment Protocol

**Purpose**: Standardize payment requests and receipts

**Payment Flow**:
1. Merchant generates payment request
2. Customer reviews and authorizes payment
3. Wallet creates transaction
4. Merchant validates and accepts
5. Receipt provided to customer

**Implementation**:
```typescript
interface PaymentRequest {
  network: 'mainnet' | 'testnet'
  outputs: Array<{
    script: string  // Hex encoded locking script
    satoshis: number
  }>
  creationTimestamp: number
  expirationTimestamp?: number
  memo?: string
  merchantData?: any
}

interface Payment {
  merchantData?: any
  transaction: string  // Hex encoded transaction
  refundTo?: string    // Output script for refunds
  memo?: string
}

interface PaymentACK {
  payment: Payment
  memo?: string
  error?: string
}

// Create payment request
function createPaymentRequest(
  amount: number,
  address: string,
  memo: string
): PaymentRequest {
  return {
    network: 'mainnet',
    outputs: [{
      script: new P2PKH().lock(address).toHex(),
      satoshis: amount
    }],
    creationTimestamp: Date.now(),
    expirationTimestamp: Date.now() + (15 * 60 * 1000), // 15 minutes
    memo
  }
}

// Process payment
async function processPayment(
  request: PaymentRequest,
  privateKey: PrivateKey
): Promise<Payment> {
  const tx = new Transaction()

  // Add inputs from wallet
  // ... UTXO selection logic

  // Add requested outputs
  for (const output of request.outputs) {
    tx.addOutput({
      satoshis: output.satoshis,
      lockingScript: Script.fromHex(output.script)
    })
  }

  await tx.sign()

  return {
    transaction: tx.toHex(),
    merchantData: request.merchantData,
    memo: 'Payment completed'
  }
}
```

**Learn More**: [BRC-29 SDK Component](../../../sdk-components/brc-29/README.md)

### BRC-56: Certificate Management

**Purpose**: Identity certificates and attestations

**Components**:
- Certificate creation
- Certificate validation
- Revocation checking
- Trust chain verification

**Use Cases**:
- Identity verification
- Credential attestation
- Access control
- Age verification
- Professional licenses

**Basic Implementation**:
```typescript
interface Certificate {
  type: string
  subject: {
    publicKey: string
    name?: string
  }
  issuer: {
    publicKey: string
    name?: string
  }
  serialNumber: string
  validFrom: number
  validTo: number
  fields: Record<string, any>
  signature?: string
}

function createCertificate(
  issuerKey: PrivateKey,
  subjectPubKey: string,
  type: string,
  fields: Record<string, any>
): Certificate {
  const cert: Certificate = {
    type,
    subject: { publicKey: subjectPubKey },
    issuer: { publicKey: issuerKey.toPublicKey().toHex() },
    serialNumber: generateSerialNumber(),
    validFrom: Date.now(),
    validTo: Date.now() + (365 * 24 * 60 * 60 * 1000), // 1 year
    fields
  }

  // Sign certificate
  const certData = JSON.stringify(cert)
  cert.signature = issuerKey.sign(certData).toHex()

  return cert
}

function verifyCertificate(cert: Certificate): boolean {
  const { signature, ...certData } = cert
  const message = JSON.stringify(certData)

  const issuerPubKey = PublicKey.fromHex(cert.issuer.publicKey)
  const sig = Signature.fromHex(signature)

  return issuerPubKey.verify(message, sig)
}
```

### BRC-1: Merchant API Specification

**Purpose**: Standardize API for transaction submission and status

**Endpoints**:
- Submit transaction
- Query transaction status
- Fee quotes
- Policy information

**Implementation**:
```typescript
interface MerchantAPIClient {
  submitTransaction(tx: Transaction): Promise<SubmitResponse>
  getTransactionStatus(txid: string): Promise<StatusResponse>
  getFeeQuote(): Promise<FeeQuote>
}

interface SubmitResponse {
  txid: string
  returnResult: 'success' | 'failure'
  resultDescription: string
  conflictedWith?: string[]
}

class MerchantAPI implements MerchantAPIClient {
  constructor(private apiUrl: string) {}

  async submitTransaction(tx: Transaction): Promise<SubmitResponse> {
    const response = await fetch(`${this.apiUrl}/tx`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        rawtx: tx.toHex()
      })
    })

    return response.json()
  }

  async getTransactionStatus(txid: string): Promise<StatusResponse> {
    const response = await fetch(`${this.apiUrl}/tx/${txid}`)
    return response.json()
  }
}
```

## Implementing Multiple BRC Standards

### Example: Secure Payment Application

Combine BRC-42, BRC-43, and BRC-29:

```typescript
class SecurePaymentApp {
  constructor(private masterKey: PrivateKey) {}

  // BRC-42 + BRC-43: Derive payment key with security
  async getPaymentKey(securityLevel: number): Promise<PrivateKey> {
    const protocolID = PrivateKey.fromString('payments')
    const keyID = PrivateKey.fromString('receive')

    if (securityLevel >= 1) {
      await this.authenticate()
    }

    return this.masterKey.deriveChild(protocolID, keyID)
  }

  // BRC-29: Create payment request
  async createInvoice(amount: number, memo: string): Promise<PaymentRequest> {
    const paymentKey = await this.getPaymentKey(0)
    const address = paymentKey.toPublicKey().toAddress()

    return {
      network: 'mainnet',
      outputs: [{
        script: new P2PKH().lock(address).toHex(),
        satoshis: amount
      }],
      creationTimestamp: Date.now(),
      memo
    }
  }

  // BRC-29: Process received payment
  async processPayment(payment: Payment): Promise<PaymentACK> {
    const tx = Transaction.fromHex(payment.transaction)

    // Validate transaction
    const valid = await this.validatePayment(tx)

    if (!valid) {
      return {
        payment,
        error: 'Invalid payment'
      }
    }

    return {
      payment,
      memo: 'Payment accepted'
    }
  }

  private async authenticate(): Promise<void> {
    // Implement authentication
  }

  private async validatePayment(tx: Transaction): Promise<boolean> {
    // Validate payment transaction
    return true
  }
}
```

## Best Practices

1. **Follow the Spec**: Implement standards exactly as specified
2. **Version Awareness**: Track which BRC version you implement
3. **Graceful Degradation**: Handle missing features in older versions
4. **Security First**: Especially for key management (BRC-42/43)
5. **Test Interoperability**: Verify compatibility with other implementations
6. **Document Deviations**: Clearly note any non-standard extensions

## Testing BRC Implementations

```typescript
describe('BRC-42 Implementation', () => {
  it('should derive consistent keys', () => {
    const master = PrivateKey.fromWif('...')
    const protocol = PrivateKey.fromString('test')
    const keyID = PrivateKey.fromString('key1')

    const derived1 = master.deriveChild(protocol, keyID)
    const derived2 = master.deriveChild(protocol, keyID)

    expect(derived1.toWif()).toBe(derived2.toWif())
  })

  it('should derive different keys for different protocols', () => {
    const master = PrivateKey.fromWif('...')
    const key1 = master.deriveChild(
      PrivateKey.fromString('app1'),
      PrivateKey.fromString('key')
    )
    const key2 = master.deriveChild(
      PrivateKey.fromString('app2'),
      PrivateKey.fromString('key')
    )

    expect(key1.toWif()).not.toBe(key2.toWif())
  })
})
```

## Related Resources

### SDK Components
- [BRC-42](../../../sdk-components/brc-42/README.md)
- [BRC-29](../../../sdk-components/brc-29/README.md)

### Code Features
- [BRC-42 Key Derivation](../../../code-features/brc-42-derivation/README.md)

### External Links
- [BRC Specifications Repository](https://github.com/bitcoin-sv/BRCs)
- [BSV Skills Center - BRC Standards](https://hub.bsvblockchain.org)

## Hands-On Exercise

Build an application that:
1. Uses BRC-42 to derive app-specific keys
2. Implements BRC-43 security levels
3. Creates BRC-29 payment requests
4. Issues BRC-56 certificates

## Next Module

Continue to: [Advanced Learning Path](../../advanced/README.md) to learn about overlay networks and advanced protocols.

## Summary

BRC standards provide the foundation for interoperable BSV applications. By implementing these standards, you ensure your application can communicate with the broader ecosystem while following security and usability best practices.

Key takeaways:
- ✅ BRC-42 for key derivation
- ✅ BRC-43 for security levels
- ✅ BRC-29 for payment protocols
- ✅ BRC-56 for certificates
- ✅ Combine standards for robust applications
