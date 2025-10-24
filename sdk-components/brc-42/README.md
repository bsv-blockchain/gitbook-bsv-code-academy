# BRC-42: Key Derivation Protocol

## Overview

BRC-42 defines a standardized protocol for deriving encryption and signing keys from a master private key. This enables secure, deterministic key generation for various applications and services.

## Purpose

BRC-42 solves the key management problem by:
- Providing deterministic key derivation
- Enabling per-protocol and per-counterparty keys
- Supporting both encryption and signing operations
- Maintaining privacy through key isolation

## Key Concepts

### Protocol ID
A unique identifier for each application or protocol (e.g., "hello world", "payments")

### Key ID
An identifier for specific use cases within a protocol (e.g., message encryption, data signing)

### Counterparty
The public key of the party you're interacting with (optional)

### Invoice Number
A unique number for per-interaction keys (optional)

## Basic Usage

```typescript
import { PrivateKey } from '@bsv/sdk'

const privateKey = PrivateKey.fromWif('your-wif-key')

// Derive a key for a specific protocol
const derivedKey = privateKey.deriveChild(
  PrivateKey.fromString('hello world'), // protocolID
  PrivateKey.fromString('encryption')    // keyID
)
```

## Use Cases

### Application-Specific Keys
Derive unique keys for different applications without exposing your master key.

### Encryption Keys
Generate encryption keys for secure communication with specific counterparties.

### Signing Keys
Create signing keys for authentication and message verification.

### Invoice-Specific Keys
Generate unique keys for each payment or interaction.

## Security Features

- **Key Isolation**: Each protocol/application uses separate keys
- **Deterministic**: Same inputs always produce same keys
- **Counterparty-Specific**: Keys can be unique per interaction partner
- **Master Key Protection**: Master key never needs to be exposed

## Related Components

- [Private Keys](../private-keys/README.md) - Key management fundamentals
- [HD Wallets](../hd-wallets/README.md) - Hierarchical key derivation
- [Encryption](../encryption/README.md) - Using derived keys for encryption
- [BRC-43](../brc-43/README.md) - Wallet security levels

## Code Examples

See [Code Features - BRC-42 Key Derivation](../../code-features/brc-42-derivation/README.md) for complete examples.

## Common Patterns

### Derive Application Key
```typescript
const appKey = masterKey.deriveChild(
  PrivateKey.fromString('myapp'),
  PrivateKey.fromString('primary')
)
```

### Derive Counterparty-Specific Key
```typescript
const sharedKey = masterKey.deriveChild(
  protocolID,
  keyID,
  counterpartyPublicKey
)
```

### Derive Invoice Key
```typescript
const invoiceKey = masterKey.deriveChild(
  protocolID,
  keyID,
  counterpartyPublicKey,
  invoiceNumber
)
```

## Best Practices

1. Use descriptive protocol IDs
2. Document your keyID conventions
3. Never reuse invoice numbers
4. Store protocol/key IDs securely
5. Use counterparty keys when possible for added security

## Learning Path References

- Intermediate: [BRC Standards](../../learning-paths/intermediate/brc-standards/README.md)
- Advanced: [Custom Protocols](../../learning-paths/advanced/custom-protocols/README.md)

## Specification

For the complete BRC-42 specification, visit the official BSV documentation.
