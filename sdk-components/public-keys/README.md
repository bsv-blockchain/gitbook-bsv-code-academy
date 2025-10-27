# Public Keys

## Overview

The `PublicKey` class represents elliptic curve public keys in the BSV TypeScript SDK. Public keys are derived from private keys and are used to verify signatures and generate Bitcoin addresses.

## Purpose

This component provides:
- Public key derivation from private keys
- Signature verification
- Address generation (P2PKH)
- DER and compressed/uncompressed encoding
- Point operations on the secp256k1 curve

## Basic Usage

### Deriving from Private Key

```typescript
import { PrivateKey, PublicKey } from '@bsv/sdk'

// Create private key
const privateKey = PrivateKey.fromRandom()

// Derive public key
const publicKey = privateKey.toPublicKey()

console.log('Public Key:', publicKey.toHex())
```

### Creating from Existing Data

```typescript
// From hex string (compressed)
const publicKey = PublicKey.fromHex('02f6c29...')

// From DER encoding
const publicKey = PublicKey.fromDER([0x02, 0xf6, 0xc2, ...])

// From private key
const privateKey = PrivateKey.fromWif('L1aW4aubDFB7...')
const publicKey = privateKey.toPublicKey()
```

## Key Features

### 1. Address Generation

Generate Bitcoin addresses from public keys:

```typescript
const privateKey = PrivateKey.fromRandom()
const publicKey = privateKey.toPublicKey()

// Generate P2PKH address
const address = publicKey.toAddress()
console.log('Address:', address) // e.g., "1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa"

// Testnet address
const testnetAddress = publicKey.toAddress('testnet')
```

### 2. Signature Verification

Verify ECDSA signatures:

```typescript
import { PrivateKey, Hash } from '@bsv/sdk'

const privateKey = PrivateKey.fromRandom()
const publicKey = privateKey.toPublicKey()

const message = 'Hello BSV'
const messageHash = Hash.sha256(Buffer.from(message))

// Sign with private key
const signature = privateKey.sign(messageHash)

// Verify with public key
const isValid = publicKey.verify(messageHash, signature)
console.log('Valid:', isValid) // true
```

### 3. Compressed vs Uncompressed

Public keys can be represented in compressed or uncompressed format:

```typescript
const privateKey = PrivateKey.fromRandom()

// Compressed (33 bytes) - default and recommended
const compressedPubKey = privateKey.toPublicKey()
console.log('Compressed:', compressedPubKey.toHex())
// e.g., "02f6c29..."

// Uncompressed (65 bytes)
const uncompressedPubKey = PublicKey.fromPrivateKey(privateKey, false)
console.log('Uncompressed:', uncompressedPubKey.toHex())
// e.g., "04f6c29...x...y..."
```

### 4. DER Encoding

Get DER-encoded representation:

```typescript
const publicKey = privateKey.toPublicKey()

// Get DER encoding
const der = publicKey.toDER()
console.log('DER:', Buffer.from(der).toString('hex'))

// Create from DER
const restored = PublicKey.fromDER(der)
```

## API Reference

### Constructor

```typescript
constructor(x: BigNumber, y: BigNumber, compressed?: boolean)
```

- `x`: X coordinate of the point
- `y`: Y coordinate of the point
- `compressed`: Whether to use compressed format (default: true)

### Static Methods

- `fromPrivateKey(privateKey: PrivateKey, compressed?: boolean): PublicKey` - Derive from private key
- `fromHex(hex: string): PublicKey` - Import from hex string
- `fromDER(der: number[]): PublicKey` - Import from DER encoding
- `fromString(str: string): PublicKey` - Import from hex or DER string
- `fromPoint(point: Point): PublicKey` - Create from secp256k1 point

### Instance Methods

- `toHex(): string` - Export to hex string
- `toDER(): number[]` - Export to DER encoding
- `toAddress(network?: 'mainnet' | 'testnet'): string` - Generate address
- `toHash(): number[]` - Get Hash160 of public key
- `verify(hash: number[], signature: Signature): boolean` - Verify signature
- `add(other: PublicKey): PublicKey` - Add another public key (point addition)
- `mul(scalar: BigNumber): PublicKey` - Scalar multiplication

## Common Patterns

### Verify Message Authentication

```typescript
import { PrivateKey, PublicKey, Hash } from '@bsv/sdk'

function authenticateMessage(
  message: string,
  signature: Signature,
  publicKeyHex: string
): boolean {
  const publicKey = PublicKey.fromHex(publicKeyHex)
  const messageHash = Hash.sha256(Buffer.from(message))

  return publicKey.verify(messageHash, signature)
}

// Usage
const privateKey = PrivateKey.fromRandom()
const publicKey = privateKey.toPublicKey()
const message = 'Authenticate me'
const messageHash = Hash.sha256(Buffer.from(message))
const signature = privateKey.sign(messageHash)

const isValid = authenticateMessage(
  message,
  signature,
  publicKey.toHex()
)
```

### Multi-Signature Key Aggregation

```typescript
function aggregatePublicKeys(publicKeys: PublicKey[]): PublicKey {
  if (publicKeys.length === 0) {
    throw new Error('Need at least one public key')
  }

  let aggregated = publicKeys[0]
  for (let i = 1; i < publicKeys.length; i++) {
    aggregated = aggregated.add(publicKeys[i])
  }

  return aggregated
}

// Usage
const key1 = PrivateKey.fromRandom().toPublicKey()
const key2 = PrivateKey.fromRandom().toPublicKey()
const key3 = PrivateKey.fromRandom().toPublicKey()

const aggregated = aggregatePublicKeys([key1, key2, key3])
console.log('Aggregated address:', aggregated.toAddress())
```

### Address Validation

```typescript
function validateAddress(address: string, publicKeyHex: string): boolean {
  try {
    const publicKey = PublicKey.fromHex(publicKeyHex)
    const generatedAddress = publicKey.toAddress()

    return address === generatedAddress
  } catch (error) {
    return false
  }
}

// Usage
const privateKey = PrivateKey.fromRandom()
const publicKey = privateKey.toPublicKey()
const address = publicKey.toAddress()

console.log(validateAddress(address, publicKey.toHex())) // true
console.log(validateAddress('1InvalidAddr', publicKey.toHex())) // false
```

## Security Considerations

### 1. Public Key Reuse

Be cautious about address reuse:

```typescript
// ✅ GOOD - Generate new address for each transaction
function generateNewReceivingAddress(hdWallet: HD): string {
  const privateKey = hdWallet.deriveChild(nextIndex)
  return privateKey.toPublicKey().toAddress()
}

// ⚠️ CAUTION - Reusing addresses reduces privacy
const singleAddress = publicKey.toAddress()
// Using this for all transactions
```

### 2. Signature Verification

Always verify signatures properly:

```typescript
// ✅ GOOD - Verify hash matches
function verifySignature(
  message: string,
  signature: Signature,
  publicKey: PublicKey
): boolean {
  const hash = Hash.sha256(Buffer.from(message))
  return publicKey.verify(hash, signature)
}

// ❌ BAD - Don't skip verification
function trustSignature(signature: Signature): boolean {
  return true // Never do this!
}
```

### 3. Point Operations

Be careful with point arithmetic:

```typescript
// ✅ GOOD - Validate points are on curve
function safeAdd(pubKey1: PublicKey, pubKey2: PublicKey): PublicKey {
  try {
    return pubKey1.add(pubKey2)
  } catch (error) {
    throw new Error('Invalid point operation')
  }
}
```

## Performance Considerations

### Compressed vs Uncompressed

Compressed keys save space and are faster to process:

```typescript
// ✅ Compressed (33 bytes) - Recommended
const compressed = privateKey.toPublicKey() // default

// ❌ Uncompressed (65 bytes) - Larger, slower
const uncompressed = PublicKey.fromPrivateKey(privateKey, false)

// Compressed keys:
// - Smaller transaction size
// - Lower fees
// - Faster verification
```

### Caching Public Keys

Cache derived public keys when used multiple times:

```typescript
class WalletCache {
  private publicKeyCache: Map<string, PublicKey> = new Map()

  getPublicKey(privateKey: PrivateKey): PublicKey {
    const wif = privateKey.toWif()

    if (!this.publicKeyCache.has(wif)) {
      this.publicKeyCache.set(wif, privateKey.toPublicKey())
    }

    return this.publicKeyCache.get(wif)!
  }
}
```

## Related Components

- [Private Keys](../private-keys/README.md) - Generate and manage private keys
- [Signatures](../signatures/README.md) - Create and verify signatures
- [P2PKH](../p2pkh/README.md) - Pay-to-Public-Key-Hash transactions
- [HD Wallets](../hd-wallets/README.md) - Hierarchical key derivation

## Code Examples

### Complete Working Examples

See these code features for full implementations:

- [Generate Private Key](../../code-features/generate-private-key/README.md)
- [Create Wallet](../../code-features/create-wallet/README.md)
- [P2PKH Template](../../code-features/p2pkh-template/README.md)

### Example: Complete Address Manager

```typescript
import { PrivateKey, PublicKey } from '@bsv/sdk'

class AddressManager {
  private privateKey: PrivateKey
  private publicKey: PublicKey
  private mainnetAddress: string
  private testnetAddress: string

  constructor(privateKey?: PrivateKey) {
    this.privateKey = privateKey || PrivateKey.fromRandom()
    this.publicKey = this.privateKey.toPublicKey()
    this.mainnetAddress = this.publicKey.toAddress('mainnet')
    this.testnetAddress = this.publicKey.toAddress('testnet')
  }

  getMainnetAddress(): string {
    return this.mainnetAddress
  }

  getTestnetAddress(): string {
    return this.testnetAddress
  }

  getPublicKeyHex(): string {
    return this.publicKey.toHex()
  }

  verifyOwnership(message: string, signature: Signature): boolean {
    const hash = Hash.sha256(Buffer.from(message))
    return this.publicKey.verify(hash, signature)
  }

  export(): {
    privateKey: string
    publicKey: string
    mainnetAddress: string
    testnetAddress: string
  } {
    return {
      privateKey: this.privateKey.toWif(),
      publicKey: this.publicKey.toHex(),
      mainnetAddress: this.mainnetAddress,
      testnetAddress: this.testnetAddress
    }
  }
}

// Usage
const manager = new AddressManager()
console.log(manager.export())
```

## Best Practices

1. **Use compressed keys** (default) for efficiency
2. **Verify all signatures** before trusting them
3. **Generate new addresses** for each transaction (privacy)
4. **Cache public keys** when using repeatedly
5. **Validate address format** before using
6. **Store public keys** safely but don't treat as secrets
7. **Use testnet** for development and testing

## Troubleshooting

### Common Issues

**Invalid public key format:**
```typescript
try {
  const pubKey = PublicKey.fromHex(userInput)
} catch (error) {
  console.error('Invalid public key hex:', error.message)
}
```

**Point not on curve:**
```typescript
try {
  const pubKey = new PublicKey(x, y)
  // Verify point is valid
  const address = pubKey.toAddress()
} catch (error) {
  console.error('Invalid point:', error.message)
}
```

**Verification fails:**
```typescript
const isValid = publicKey.verify(hash, signature)

if (!isValid) {
  // Check: Is the hash correct?
  // Check: Is this the right public key?
  // Check: Is the signature corrupted?
  // Check: Was the message modified?
}
```

## Further Reading

- [Elliptic Curve Cryptography](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography)
- [secp256k1 Curve](https://en.bitcoin.it/wiki/Secp256k1)
- [Bitcoin Address](https://en.bitcoin.it/wiki/Address)
- [Public Key Compression](https://bitcoin.stackexchange.com/questions/3059/what-is-a-compressed-bitcoin-key)

## Status

✅ **Complete** - Production ready
