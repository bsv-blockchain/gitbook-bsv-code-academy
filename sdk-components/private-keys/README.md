# Private Keys

## Overview

The `PrivateKey` class is a fundamental component of the BSV TypeScript SDK that handles elliptic curve private key operations. Private keys are 256-bit numbers used to sign transactions and prove ownership of Bitcoin.

## Purpose

This component provides:
- Private key generation from random data or WIF (Wallet Import Format)
- DER signature creation and verification
- Public key derivation
- WIF encoding/decoding
- Message signing capabilities

## Basic Usage

### Generating a New Private Key

```typescript
import { PrivateKey } from '@bsv/sdk'

// Generate a random private key
const privateKey = PrivateKey.fromRandom()

// Get the private key as hex string
const hex = privateKey.toHex()

// Get WIF (Wallet Import Format) representation
const wif = privateKey.toWif()
```

### Creating from Existing Key Material

```typescript
// From WIF string
const privateKey = PrivateKey.fromWif('L1aW4aubDFB7yfras2S1mN3bqg9nwySY8nkoLmJebSLD5BWv3ENZ')

// From hex string
const privateKey = PrivateKey.fromHex('e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855')

// From random bytes
const privateKey = PrivateKey.fromRandom()
```

## Key Features

### 1. Signature Generation

Create DER-encoded signatures for transaction signing:

```typescript
const privateKey = PrivateKey.fromRandom()
const message = 'Hello BSV'
const messageHash = Hash.sha256(Buffer.from(message))

// Create signature
const signature = privateKey.sign(messageHash)

// Get DER encoding
const derSignature = signature.toDER()
```

### 2. Public Key Derivation

Derive the corresponding public key:

```typescript
const privateKey = PrivateKey.fromRandom()
const publicKey = privateKey.toPublicKey()

console.log('Public Key:', publicKey.toHex())
console.log('Address:', publicKey.toAddress())
```

### 3. WIF Encoding/Decoding

WIF (Wallet Import Format) provides a compact representation:

```typescript
const privateKey = PrivateKey.fromRandom()

// Export to WIF
const wif = privateKey.toWif()
console.log('WIF:', wif)

// Import from WIF
const imported = PrivateKey.fromWif(wif)
```

### 4. Message Signing

Sign arbitrary messages for authentication:

```typescript
const privateKey = PrivateKey.fromRandom()
const message = 'Prove ownership'

// Sign message
const signature = privateKey.signMessage(message)

// Verify with public key
const publicKey = privateKey.toPublicKey()
const isValid = publicKey.verifyMessage(message, signature)
```

## API Reference

### Constructor

```typescript
constructor(number?: BigNumber, compressed?: boolean)
```

- `number`: The private key as a BigNumber (optional)
- `compressed`: Whether to use compressed format (default: true)

### Static Methods

- `fromRandom(): PrivateKey` - Generate random private key
- `fromWif(wif: string): PrivateKey` - Import from WIF
- `fromHex(hex: string): PrivateKey` - Import from hex string
- `fromString(str: string): PrivateKey` - Import from hex or WIF string

### Instance Methods

- `toWif(): string` - Export to WIF format
- `toHex(): string` - Export to hex string
- `toPublicKey(): PublicKey` - Derive public key
- `sign(hash: number[]): Signature` - Sign a hash
- `signMessage(message: string): Signature` - Sign a message
- `verify(hash: number[], signature: Signature): boolean` - Verify signature

## Common Patterns

### Deterministic Key Generation

```typescript
import { PrivateKey, Hash } from '@bsv/sdk'

// Generate from seed phrase (simplified example)
function keyFromSeed(seed: string, index: number): PrivateKey {
  const hash = Hash.sha256(Buffer.from(`${seed}:${index}`))
  return PrivateKey.fromHex(hash.toString('hex'))
}

const key1 = keyFromSeed('my secret seed', 0)
const key2 = keyFromSeed('my secret seed', 1)
```

### Secure Key Storage

```typescript
import { PrivateKey } from '@bsv/sdk'
import { encrypt, decrypt } from './encryption-utils'

// Store encrypted private key
function storeKey(privateKey: PrivateKey, password: string): string {
  const wif = privateKey.toWif()
  return encrypt(wif, password)
}

// Retrieve and decrypt
function retrieveKey(encrypted: string, password: string): PrivateKey {
  const wif = decrypt(encrypted, password)
  return PrivateKey.fromWif(wif)
}
```

### Transaction Signing Pattern

```typescript
import { PrivateKey, Transaction } from '@bsv/sdk'

async function signTransaction(
  tx: Transaction,
  privateKey: PrivateKey,
  inputIndex: number
): Promise<void> {
  const publicKey = privateKey.toPublicKey()

  // Sign the input
  await tx.sign(inputIndex, privateKey, publicKey)
}
```

## Security Considerations

### 1. Key Generation

Always use cryptographically secure random number generation:

```typescript
// ✅ GOOD - Uses secure random
const privateKey = PrivateKey.fromRandom()

// ❌ BAD - Never use predictable sources
const badKey = PrivateKey.fromHex('0000000000000000000000000000000000000000000000000000000000000001')
```

### 2. Key Storage

Never store private keys in plain text:

```typescript
// ❌ BAD - Plain text storage
localStorage.setItem('privateKey', privateKey.toWif())

// ✅ GOOD - Encrypted storage
const encrypted = encryptKey(privateKey.toWif(), userPassword)
localStorage.setItem('encryptedKey', encrypted)
```

### 3. Memory Management

Clear sensitive data when no longer needed:

```typescript
function usePrivateKey(wif: string) {
  const privateKey = PrivateKey.fromWif(wif)

  try {
    // Use the key
    const signature = privateKey.sign(messageHash)
    return signature
  } finally {
    // Clear from memory (implementation dependent)
    wif = null
  }
}
```

## Related Components

- [Public Keys](../public-keys/README.md) - Derive public keys from private keys
- [Signatures](../signatures/README.md) - Work with ECDSA signatures
- [HD Wallets](../hd-wallets/README.md) - Hierarchical deterministic key derivation
- [Transaction](../transaction/README.md) - Sign transactions with private keys

## Code Examples

### Complete Working Examples

See these code features for full implementations:

- [Generate Private Key](../../code-features/generate-private-key/README.md)
- [Sign Transaction](../../code-features/sign-transaction/README.md)
- [Create Wallet](../../code-features/create-wallet/README.md)

### Example: Complete Key Management

```typescript
import { PrivateKey, PublicKey } from '@bsv/sdk'

class KeyManager {
  private privateKey: PrivateKey
  private publicKey: PublicKey

  constructor(wif?: string) {
    if (wif) {
      this.privateKey = PrivateKey.fromWif(wif)
    } else {
      this.privateKey = PrivateKey.fromRandom()
    }
    this.publicKey = this.privateKey.toPublicKey()
  }

  getAddress(): string {
    return this.publicKey.toAddress()
  }

  sign(messageHash: number[]): Signature {
    return this.privateKey.sign(messageHash)
  }

  export(): string {
    return this.privateKey.toWif()
  }

  verify(messageHash: number[], signature: Signature): boolean {
    return this.publicKey.verify(messageHash, signature)
  }
}

// Usage
const manager = new KeyManager()
console.log('Address:', manager.getAddress())

const exported = manager.export()
const restored = new KeyManager(exported)
```

## Best Practices

1. **Never hardcode private keys** in source code
2. **Use WIF format** for human-readable key export
3. **Always verify** signatures after creating them
4. **Use compressed keys** (default) to save space
5. **Implement proper key backup** procedures
6. **Test with testnet keys** before using mainnet
7. **Use HD wallets** (BRC-42) for multiple addresses

## Troubleshooting

### Common Issues

**Invalid WIF string:**
```typescript
try {
  const key = PrivateKey.fromWif(userInput)
} catch (error) {
  console.error('Invalid WIF format')
}
```

**Signature verification fails:**
```typescript
const signature = privateKey.sign(hash)
const valid = publicKey.verify(hash, signature)

if (!valid) {
  // Check that hash is correct
  // Ensure using matching public key
  // Verify signature wasn't corrupted
}
```

## Further Reading

- [ECDSA on Wikipedia](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm)
- [WIF Format](https://en.bitcoin.it/wiki/Wallet_import_format)
- [BRC-42: Key Derivation](../brc-42/README.md)

## Status

✅ **Complete** - Production ready
