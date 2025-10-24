# Generate Private Key

## Overview

Learn how to securely generate, import, export, and manage private keys for BSV blockchain development.

## Prerequisites

- BSV TypeScript SDK installed
- Secure environment for key generation

## Code Example

```typescript
import { PrivateKey, HD } from '@bsv/sdk'

// Method 1: Generate a random private key
function generateRandomKey() {
  const privateKey = PrivateKey.fromRandom()

  console.log('Private Key (WIF):', privateKey.toWif())
  console.log('Private Key (Hex):', privateKey.toHex())
  console.log('Public Key:', privateKey.toPublicKey().toHex())
  console.log('Address:', privateKey.toPublicKey().toAddress())

  return privateKey
}

// Method 2: Generate from seed phrase (HD wallet)
function generateFromSeed() {
  // Generate a random mnemonic (12 or 24 words)
  const mnemonic = HD.generateMnemonic(128) // 128 bits = 12 words, 256 bits = 24 words
  console.log('Mnemonic:', mnemonic)

  // Derive master key from mnemonic
  const seed = HD.fromMnemonic(mnemonic)
  const masterKey = PrivateKey.fromHex(seed.privateKey.toHex())

  console.log('Master Private Key:', masterKey.toWif())

  return { mnemonic, masterKey }
}

// Method 3: Import existing key
function importKey(wifOrHex: string) {
  let privateKey: PrivateKey

  try {
    // Try WIF format first
    privateKey = PrivateKey.fromWif(wifOrHex)
  } catch {
    // Try hex format
    privateKey = PrivateKey.fromHex(wifOrHex)
  }

  console.log('Imported successfully')
  console.log('Address:', privateKey.toPublicKey().toAddress())

  return privateKey
}

// Method 4: Derive child keys (HD wallet)
function deriveChildKey(masterKey: PrivateKey, path: string) {
  // Derive using BIP32 path (e.g., "m/44'/0'/0'/0/0")
  const childKey = masterKey.deriveChild(/* derivation params */)

  console.log('Child Key:', childKey.toWif())
  console.log('Child Address:', childKey.toPublicKey().toAddress())

  return childKey
}

// Export keys securely
function exportKey(privateKey: PrivateKey) {
  return {
    wif: privateKey.toWif(),
    hex: privateKey.toHex(),
    publicKey: privateKey.toPublicKey().toHex(),
    address: privateKey.toPublicKey().toAddress()
  }
}

// Example usage
console.log('=== Generate Random Key ===')
const randomKey = generateRandomKey()

console.log('\n=== Generate from Seed Phrase ===')
const { mnemonic, masterKey } = generateFromSeed()

console.log('\n=== Import Existing Key ===')
const importedKey = importKey('L1234...') // Replace with actual WIF

console.log('\n=== Export Key Info ===')
const keyInfo = exportKey(randomKey)
console.log(keyInfo)
```

## Explanation

### Key Generation Methods

#### 1. Random Key Generation
- Uses cryptographically secure random number generator
- Suitable for single-use or simple wallets
- No backup beyond the key itself

#### 2. HD Wallet (Mnemonic)
- Generates 12 or 24 word mnemonic phrase
- Deterministic: same phrase always produces same keys
- Can derive unlimited child keys
- Industry standard (BIP39)

#### 3. Import Existing
- Supports WIF (Wallet Import Format) and hex
- Useful for restoring or migrating wallets

#### 4. Child Key Derivation
- Derive keys from master key
- Hierarchical structure (BIP32)
- Enables organized key management

## Key Formats

### WIF (Wallet Import Format)
```
L4rK1yDtCWekvXuE6oXD9jCYfFNV2cWRpVuPLBcCU2z8TrisoyY1
```
- Base58 encoded
- Includes checksum
- Standard for import/export

### Hexadecimal
```
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```
- 64 characters (32 bytes)
- Raw private key data

### Public Key
```
03a1b2c3d4e5f6...
```
- Derived from private key
- Safe to share publicly

### Address
```
1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa
```
- Derived from public key
- Used for receiving payments

## Security Best Practices

1. **Never Share Private Keys**: Keep them secret
2. **Secure Storage**: Use encryption for stored keys
3. **Backup Mnemonics**: Write down and store safely offline
4. **Test with Small Amounts**: Verify before large transactions
5. **Use HD Wallets**: Better organization and backup

## Common Use Cases

### Development Testing
```typescript
// Generate temporary key for testing
const testKey = PrivateKey.fromRandom()
```

### Production Wallet
```typescript
// Use HD wallet for production
const mnemonic = HD.generateMnemonic(256) // 24 words
// Store mnemonic securely!
```

### Key Recovery
```typescript
// Restore from mnemonic
const seed = HD.fromMnemonic(storedMnemonic)
const restoredKey = PrivateKey.fromHex(seed.privateKey.toHex())
```

## Related Components

- [Private Keys SDK Component](../../sdk-components/private-keys/README.md)
- [HD Wallets SDK Component](../../sdk-components/hd-wallets/README.md)
- [Public Keys SDK Component](../../sdk-components/public-keys/README.md)

## Related Code Features

- [Create Wallet](../create-wallet/README.md)
- [BRC-42 Key Derivation](../brc-42-derivation/README.md)
- [Import/Export Keys](../import-export-keys/README.md)

## Learning Path References

- Beginner: [Your First Wallet](../../learning-paths/beginner/first-wallet/README.md)
- Intermediate: [BRC Standards](../../learning-paths/intermediate/brc-standards/README.md)

## Warning

Never commit private keys to version control or share them publicly. Always use environment variables or secure key management systems in production.
