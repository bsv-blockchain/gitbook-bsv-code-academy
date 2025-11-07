# BRC-42 Key Derivation

Complete examples for using BRC-42 hierarchical deterministic key derivation.

## Overview

BRC-42 defines a standard for deriving BSV keys hierarchically, enabling protocol isolation and secure key management. This code feature demonstrates practical BRC-42 key derivation patterns.

**Related SDK Components:**
- [HD Wallets](../../sdk-components/hd-wallets/README.md)
- [BRC-42](../../sdk-components/brc-42/README.md)
- [Private Keys](../../sdk-components/private-keys/README.md)

## Basic BRC-42 Derivation

```typescript
import { PrivateKey, KeyDeriver } from '@bsv/sdk'

/**
 * Basic BRC-42 Key Derivation
 *
 * Derive protocol-specific keys from a master key
 */
class BRC42BasicDerivation {
  /**
   * Derive a key for a specific protocol
   */
  deriveProtocolKey(
    masterPrivateKey: PrivateKey,
    protocolID: [number, string],
    keyID: string
  ): PrivateKey {
    const deriver = new KeyDeriver(masterPrivateKey)

    // Derive key for specific protocol and key ID
    const derivedKey = deriver.derivePrivateKey(
      protocolID,
      keyID
    )

    return derivedKey
  }

  /**
   * Derive a public key for a specific protocol
   */
  deriveProtocolPublicKey(
    masterPrivateKey: PrivateKey,
    protocolID: [number, string],
    keyID: string
  ): PublicKey {
    const deriver = new KeyDeriver(masterPrivateKey)

    const derivedPublicKey = deriver.derivePublicKey(
      protocolID,
      keyID
    )

    return derivedPublicKey
  }
}

/**
 * Usage Example
 */
async function example() {
  const derivation = new BRC42BasicDerivation()

  // Master key (from seed phrase or secure storage)
  const masterKey = PrivateKey.fromRandom()

  // Protocol ID: [security level, protocol name]
  // Security level 0 = no security, 1 = low, 2 = medium, etc.
  const protocolID: [number, string] = [2, 'my-app']

  // Derive key for specific purpose
  const invoiceKey = derivation.deriveProtocolKey(
    masterKey,
    protocolID,
    'invoices-2024-001'
  )

  console.log('Derived invoice key address:', invoiceKey.toAddress())
}
```

## BRC-43 Security Levels

```typescript
import { PrivateKey, KeyDeriver } from '@bsv/sdk'

/**
 * BRC-43 Security Levels
 *
 * Different security levels for different use cases
 */
class BRC43SecurityLevels {
  /**
   * Security Level 0 - No security
   * Used for public keys that don't need protection
   */
  derivePublicKey(
    masterKey: PrivateKey,
    protocolName: string,
    keyID: string
  ): PublicKey {
    const deriver = new KeyDeriver(masterKey)

    return deriver.derivePublicKey(
      [0, protocolName],
      keyID
    )
  }

  /**
   * Security Level 1 - Low security
   * Used for small amounts or testing
   */
  deriveLowSecurityKey(
    masterKey: PrivateKey,
    protocolName: string,
    keyID: string
  ): PrivateKey {
    const deriver = new KeyDeriver(masterKey)

    return deriver.derivePrivateKey(
      [1, protocolName],
      keyID
    )
  }

  /**
   * Security Level 2 - Standard security
   * Default for most applications
   */
  deriveStandardSecurityKey(
    masterKey: PrivateKey,
    protocolName: string,
    keyID: string
  ): PrivateKey {
    const deriver = new KeyDeriver(masterKey)

    return deriver.derivePrivateKey(
      [2, protocolName],
      keyID
    )
  }

  /**
   * Security Level 3 - High security
   * For sensitive operations or large amounts
   */
  deriveHighSecurityKey(
    masterKey: PrivateKey,
    protocolName: string,
    keyID: string,
    counterparty?: string,
    invoiceNumber?: string
  ): PrivateKey {
    const deriver = new KeyDeriver(masterKey)

    // High security requires additional context
    const secureKeyID = counterparty && invoiceNumber
      ? `${keyID}-${counterparty}-${invoiceNumber}`
      : keyID

    return deriver.derivePrivateKey(
      [3, protocolName],
      secureKeyID
    )
  }
}

/**
 * Usage Example
 */
async function securityLevelExample() {
  const security = new BRC43SecurityLevels()
  const masterKey = PrivateKey.fromRandom()

  // Public key for display (no security needed)
  const displayKey = security.derivePublicKey(
    masterKey,
    'my-app',
    'display-profile'
  )

  // Low security for test transactions
  const testKey = security.deriveLowSecurityKey(
    masterKey,
    'my-app',
    'test-payment'
  )

  // Standard security for normal payments
  const paymentKey = security.deriveStandardSecurityKey(
    masterKey,
    'my-app',
    'payment-001'
  )

  // High security for large transfers
  const transferKey = security.deriveHighSecurityKey(
    masterKey,
    'my-app',
    'large-transfer',
    'counterparty-pubkey',
    'invoice-12345'
  )

  console.log('Keys derived for different security levels')
}
```

## Protocol Isolation

```typescript
import { PrivateKey, KeyDeriver } from '@bsv/sdk'

/**
 * Protocol Isolation
 *
 * Keep different protocols completely isolated for security
 */
class ProtocolIsolation {
  private masterKey: PrivateKey
  private protocols: Map<string, KeyDeriver> = new Map()

  constructor(masterKey: PrivateKey) {
    this.masterKey = masterKey
  }

  /**
   * Get or create protocol-specific deriver
   */
  private getProtocolDeriver(
    protocolName: string,
    securityLevel: number = 2
  ): KeyDeriver {
    const key = `${securityLevel}:${protocolName}`

    if (!this.protocols.has(key)) {
      // Create new deriver for this protocol
      const deriver = new KeyDeriver(this.masterKey)
      this.protocols.set(key, deriver)
    }

    return this.protocols.get(key)!
  }

  /**
   * Derive key for payments protocol
   */
  derivePaymentKey(invoiceNumber: string): PrivateKey {
    const deriver = this.getProtocolDeriver('payments', 2)

    return deriver.derivePrivateKey(
      [2, 'payments'],
      `invoice-${invoiceNumber}`
    )
  }

  /**
   * Derive key for identity protocol
   */
  deriveIdentityKey(context: string): PrivateKey {
    const deriver = this.getProtocolDeriver('identity', 3)

    return deriver.derivePrivateKey(
      [3, 'identity'],
      context
    )
  }

  /**
   * Derive key for token protocol
   */
  deriveTokenKey(tokenID: string, operation: string): PrivateKey {
    const deriver = this.getProtocolDeriver('tokens', 2)

    return deriver.derivePrivateKey(
      [2, 'tokens'],
      `${tokenID}-${operation}`
    )
  }

  /**
   * Derive key for messaging protocol
   */
  deriveMessagingKey(conversationID: string): PrivateKey {
    const deriver = this.getProtocolDeriver('messaging', 1)

    return deriver.derivePrivateKey(
      [1, 'messaging'],
      `conversation-${conversationID}`
    )
  }
}

/**
 * Usage Example
 */
async function isolationExample() {
  const masterKey = PrivateKey.fromRandom()
  const isolation = new ProtocolIsolation(masterKey)

  // Each protocol gets isolated keys
  const paymentKey = isolation.derivePaymentKey('INV-001')
  const identityKey = isolation.deriveIdentityKey('login-session-abc')
  const tokenKey = isolation.deriveTokenKey('TOKEN-XYZ', 'transfer')
  const messagingKey = isolation.deriveMessagingKey('chat-123')

  console.log('Payment key:', paymentKey.toAddress())
  console.log('Identity key:', identityKey.toAddress())
  console.log('Token key:', tokenKey.toAddress())
  console.log('Messaging key:', messagingKey.toAddress())

  // Keys are completely isolated - compromise of one doesn't affect others
}
```

## Advanced Key Derivation Patterns

```typescript
import { PrivateKey, KeyDeriver, PublicKey } from '@bsv/sdk'

/**
 * Advanced Key Derivation Patterns
 */
class AdvancedDerivation {
  /**
   * Derive keys with counterparty for shared protocols
   */
  deriveSharedKey(
    masterKey: PrivateKey,
    protocolName: string,
    counterpartyPubKey: PublicKey,
    invoiceNumber: string
  ): PrivateKey {
    const deriver = new KeyDeriver(masterKey)

    // Include counterparty in key derivation for uniqueness
    const keyID = `${counterpartyPubKey.toString()}-${invoiceNumber}`

    return deriver.derivePrivateKey(
      [2, protocolName],
      keyID
    )
  }

  /**
   * Derive time-based keys (rotating keys)
   */
  deriveTimeBasedKey(
    masterKey: PrivateKey,
    protocolName: string,
    period: 'daily' | 'weekly' | 'monthly'
  ): PrivateKey {
    const deriver = new KeyDeriver(masterKey)
    const now = new Date()

    let keyID: string

    switch (period) {
      case 'daily':
        keyID = `${now.getFullYear()}-${now.getMonth() + 1}-${now.getDate()}`
        break
      case 'weekly':
        const weekNumber = this.getWeekNumber(now)
        keyID = `${now.getFullYear()}-W${weekNumber}`
        break
      case 'monthly':
        keyID = `${now.getFullYear()}-${now.getMonth() + 1}`
        break
    }

    return deriver.derivePrivateKey(
      [2, protocolName],
      `period-${keyID}`
    )
  }

  /**
   * Derive hierarchical keys (parent → child → grandchild)
   */
  deriveHierarchicalKey(
    masterKey: PrivateKey,
    protocolName: string,
    hierarchy: string[]
  ): PrivateKey {
    const deriver = new KeyDeriver(masterKey)

    // Combine hierarchy into key ID
    const keyID = hierarchy.join('/')

    return deriver.derivePrivateKey(
      [2, protocolName],
      keyID
    )
  }

  /**
   * Derive keys with metadata
   */
  deriveMetadataKey(
    masterKey: PrivateKey,
    protocolName: string,
    metadata: Record<string, string>
  ): PrivateKey {
    const deriver = new KeyDeriver(masterKey)

    // Sort and stringify metadata for consistent derivation
    const sortedKeys = Object.keys(metadata).sort()
    const metadataString = sortedKeys
      .map(key => `${key}=${metadata[key]}`)
      .join('&')

    return deriver.derivePrivateKey(
      [2, protocolName],
      metadataString
    )
  }

  /**
   * Get ISO week number
   */
  private getWeekNumber(date: Date): number {
    const d = new Date(Date.UTC(date.getFullYear(), date.getMonth(), date.getDate()))
    const dayNum = d.getUTCDay() || 7
    d.setUTCDate(d.getUTCDate() + 4 - dayNum)
    const yearStart = new Date(Date.UTC(d.getUTCFullYear(), 0, 1))
    return Math.ceil((((d.getTime() - yearStart.getTime()) / 86400000) + 1) / 7)
  }
}

/**
 * Usage Example
 */
async function advancedExample() {
  const advanced = new AdvancedDerivation()
  const masterKey = PrivateKey.fromRandom()
  const counterparty = PrivateKey.fromRandom().toPublicKey()

  // Shared key with counterparty
  const sharedKey = advanced.deriveSharedKey(
    masterKey,
    'payments',
    counterparty,
    'INV-001'
  )

  // Rotating daily key
  const dailyKey = advanced.deriveTimeBasedKey(
    masterKey,
    'analytics',
    'daily'
  )

  // Hierarchical key
  const hierarchicalKey = advanced.deriveHierarchicalKey(
    masterKey,
    'organization',
    ['company-abc', 'department-sales', 'employee-123']
  )

  // Metadata-based key
  const metadataKey = advanced.deriveMetadataKey(
    masterKey,
    'content',
    {
      type: 'article',
      category: 'technology',
      id: 'post-456'
    }
  )

  console.log('Advanced derivation patterns demonstrated')
}
```

## Related Examples

- [Create Wallet](../create-wallet/README.md)
- [Generate Private Key](../generate-private-key/README.md)
- [Message Signing](../message-signing/README.md)

## See Also

**SDK Components:**
- [HD Wallets](../../sdk-components/hd-wallets/README.md) - Hierarchical deterministic wallets
- [BRC-42](../../sdk-components/brc-42/README.md) - BRC-42 standard
- [Private Keys](../../sdk-components/private-keys/README.md) - Key management

**Learning Paths:**
- [First Wallet](../../learning-paths/beginner/first-wallet/README.md)
