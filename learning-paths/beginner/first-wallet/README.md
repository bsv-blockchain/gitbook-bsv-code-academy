# Your First Wallet

## Overview

In this module, you'll create your first BSV wallet using the TypeScript SDK. You'll learn how to generate keys, create addresses, and implement basic wallet functionality.

## Learning Objectives

By the end of this module, you will be able to:
- Generate private keys securely
- Create wallet addresses
- Understand HD wallet structure
- Back up and restore wallets
- Implement basic wallet operations
- Follow security best practices

## What is a Wallet?

A BSV wallet is software that:
- **Manages private keys**
- **Generates addresses** for receiving payments
- **Signs transactions** to send payments
- **Tracks balance** by monitoring UTXOs
- **Maintains history** of transactions

**Important**: A wallet doesn't "store" BSV. It stores keys that control UTXOs on the blockchain.

## Types of Wallets

### 1. Simple Wallet
Single private key controlling one or more addresses.

**Pros**: Simple to understand
**Cons**: Must backup each key separately

### 2. HD Wallet (Hierarchical Deterministic)
One master seed generates unlimited keys deterministically.

**Pros**: One backup for all keys
**Cons**: Slightly more complex

**Recommendation**: Use HD wallets for production applications.

## Creating Your First Simple Wallet

### Step 1: Generate a Private Key

```typescript
import { PrivateKey } from '@bsv/sdk'

// Generate a random private key
const privateKey = PrivateKey.fromRandom()

// IMPORTANT: Save this key securely!
console.log('Private Key (WIF):', privateKey.toWif())
console.log('Private Key (Hex):', privateKey.toHex())
```

**Output Example**:
```
Private Key (WIF): L4rK1yDtCWekvXuE6oXD9jCYfFNV2cWRpVuPLBcCU2z8TrisoyY1
Private Key (Hex): e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

### Step 2: Derive Public Key and Address

```typescript
// Derive public key from private key
const publicKey = privateKey.toPublicKey()

// Derive address from public key
const address = publicKey.toAddress()

console.log('Public Key:', publicKey.toHex())
console.log('Address:', address)
```

**Output Example**:
```
Public Key: 02a1b2c3d4e5f6...
Address: 1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa
```

### Step 3: Create a Basic Wallet Class

```typescript
import { PrivateKey, PublicKey } from '@bsv/sdk'

class SimpleWallet {
  private privateKey: PrivateKey
  public address: string

  constructor(privateKey?: PrivateKey) {
    // Use provided key or generate new one
    this.privateKey = privateKey || PrivateKey.fromRandom()
    this.address = this.privateKey.toPublicKey().toAddress()
  }

  // Get wallet info
  getInfo() {
    return {
      address: this.address,
      publicKey: this.privateKey.toPublicKey().toHex(),
      // Never expose private key in production!
    }
  }

  // Export private key (for backup)
  exportPrivateKey(): string {
    console.warn('⚠️  Keep this private key secure!')
    return this.privateKey.toWif()
  }

  // Get private key for signing
  getPrivateKey(): PrivateKey {
    return this.privateKey
  }

  // Static method to restore wallet
  static fromWIF(wif: string): SimpleWallet {
    const privateKey = PrivateKey.fromWif(wif)
    return new SimpleWallet(privateKey)
  }
}

// Usage
const wallet = new SimpleWallet()
console.log('Wallet Address:', wallet.getInfo().address)

// Backup
const backup = wallet.exportPrivateKey()

// Restore
const restoredWallet = SimpleWallet.fromWIF(backup)
```

## Creating an HD Wallet

HD wallets use a master seed to derive unlimited keys.

### Step 1: Generate Mnemonic

```typescript
import { HD } from '@bsv/sdk'

// Generate 12-word mnemonic (128 bits of entropy)
const mnemonic = HD.generateMnemonic(128)
console.log('Mnemonic:', mnemonic)

// Example output:
// abandon ability able about above absent absorb abstract absurd abuse access accident
```

**Mnemonic Lengths**:
- 12 words = 128 bits (standard)
- 24 words = 256 bits (extra security)

### Step 2: Derive Master Key

```typescript
// Create HD wallet from mnemonic
const hdSeed = HD.fromMnemonic(mnemonic)
const masterKey = PrivateKey.fromHex(hdSeed.privateKey.toHex())

console.log('Master Private Key:', masterKey.toWif())
```

### Step 3: Derive Child Keys

```typescript
class HDWallet {
  private masterKey: PrivateKey
  private mnemonic: string
  private addresses: Map<number, string> = new Map()

  constructor(mnemonic?: string) {
    if (mnemonic) {
      this.mnemonic = mnemonic
    } else {
      // Generate new mnemonic
      this.mnemonic = HD.generateMnemonic(128)
    }

    // Derive master key
    const hdSeed = HD.fromMnemonic(this.mnemonic)
    this.masterKey = PrivateKey.fromHex(hdSeed.privateKey.toHex())
  }

  // Derive key at index using BRC-42
  deriveKey(index: number): PrivateKey {
    const protocolID = PrivateKey.fromString('wallet')
    const keyID = PrivateKey.fromString(`address-${index}`)

    return this.masterKey.deriveChild(protocolID, keyID)
  }

  // Generate address at index
  generateAddress(index: number): string {
    if (this.addresses.has(index)) {
      return this.addresses.get(index)!
    }

    const key = this.deriveKey(index)
    const address = key.toPublicKey().toAddress()

    this.addresses.set(index, address)
    return address
  }

  // Get multiple addresses
  generateAddresses(count: number): string[] {
    const addresses: string[] = []
    for (let i = 0; i < count; i++) {
      addresses.push(this.generateAddress(i))
    }
    return addresses
  }

  // Get private key for specific address index
  getPrivateKeyForAddress(index: number): PrivateKey {
    return this.deriveKey(index)
  }

  // Export mnemonic for backup
  exportMnemonic(): string {
    console.warn('⚠️  Keep this mnemonic phrase secure!')
    console.warn('⚠️  Anyone with this phrase can access your funds!')
    return this.mnemonic
  }

  // Restore from mnemonic
  static fromMnemonic(mnemonic: string): HDWallet {
    return new HDWallet(mnemonic)
  }
}

// Usage
const hdWallet = new HDWallet()

// Generate first 5 addresses
const addresses = hdWallet.generateAddresses(5)
console.log('Addresses:', addresses)

// Backup mnemonic
const backup = hdWallet.exportMnemonic()
console.log('Backup phrase:', backup)

// Restore wallet
const restored = HDWallet.fromMnemonic(backup)
```

## Checking Wallet Balance

To check balance, you need to query UTXOs for your addresses.

```typescript
interface UTXO {
  txid: string
  vout: number
  satoshis: number
  script: string
}

class WalletWithBalance extends HDWallet {
  // Query UTXOs from block explorer API
  async getUTXOs(address: string): Promise<UTXO[]> {
    // Using WhatsOnChain API as example
    const response = await fetch(
      `https://api.whatsonchain.com/v1/bsv/main/address/${address}/unspent`
    )
    return await response.json()
  }

  // Get balance for specific address
  async getAddressBalance(index: number): Promise<number> {
    const address = this.generateAddress(index)
    const utxos = await this.getUTXOs(address)

    return utxos.reduce((total, utxo) => total + utxo.satoshis, 0)
  }

  // Get total wallet balance
  async getTotalBalance(addressCount: number = 20): Promise<number> {
    let total = 0

    for (let i = 0; i < addressCount; i++) {
      const balance = await this.getAddressBalance(i)
      total += balance

      // Stop if we find 5 consecutive empty addresses
      if (balance === 0 && i >= 5) {
        const lastFiveEmpty = await Promise.all([
          this.getAddressBalance(i - 1),
          this.getAddressBalance(i - 2),
          this.getAddressBalance(i - 3),
          this.getAddressBalance(i - 4)
        ])

        if (lastFiveEmpty.every(b => b === 0)) {
          break
        }
      }
    }

    return total
  }

  // Get all UTXOs for wallet
  async getAllUTXOs(addressCount: number = 20): Promise<UTXO[]> {
    const allUTXOs: UTXO[] = []

    for (let i = 0; i < addressCount; i++) {
      const address = this.generateAddress(i)
      const utxos = await this.getUTXOs(address)
      allUTXOs.push(...utxos)
    }

    return allUTXOs
  }
}

// Usage
const wallet = new WalletWithBalance()

// Check balance
const balance = await wallet.getTotalBalance()
console.log(`Balance: ${balance} satoshis (${balance / 100000000} BSV)`)
```

## Wallet Security Best Practices

### 1. Private Key Storage

**Never**:
- ❌ Hard-code private keys in source code
- ❌ Store private keys in plain text
- ❌ Commit private keys to version control
- ❌ Share private keys over insecure channels

**Do**:
- ✅ Use environment variables
- ✅ Encrypt private keys at rest
- ✅ Use secure key management services
- ✅ Implement proper access controls

### 2. Mnemonic Backup

```typescript
// Secure mnemonic storage example
import * as crypto from 'crypto'

class SecureWallet extends HDWallet {
  // Encrypt mnemonic with password
  static encryptMnemonic(mnemonic: string, password: string): string {
    const algorithm = 'aes-256-gcm'
    const key = crypto.scryptSync(password, 'salt', 32)
    const iv = crypto.randomBytes(16)
    const cipher = crypto.createCipheriv(algorithm, key, iv)

    let encrypted = cipher.update(mnemonic, 'utf8', 'hex')
    encrypted += cipher.final('hex')

    const authTag = cipher.getAuthTag()

    return JSON.stringify({
      iv: iv.toString('hex'),
      encrypted,
      authTag: authTag.toString('hex')
    })
  }

  // Decrypt mnemonic with password
  static decryptMnemonic(encryptedData: string, password: string): string {
    const { iv, encrypted, authTag } = JSON.parse(encryptedData)
    const algorithm = 'aes-256-gcm'
    const key = crypto.scryptSync(password, 'salt', 32)

    const decipher = crypto.createDecipheriv(
      algorithm,
      key,
      Buffer.from(iv, 'hex')
    )

    decipher.setAuthTag(Buffer.from(authTag, 'hex'))

    let decrypted = decipher.update(encrypted, 'hex', 'utf8')
    decrypted += decipher.final('utf8')

    return decrypted
  }
}

// Usage
const wallet = new SecureWallet()
const mnemonic = wallet.exportMnemonic()

// Encrypt for storage
const password = 'strong-user-password'
const encrypted = SecureWallet.encryptMnemonic(mnemonic, password)

// Save encrypted to file or database
// ...

// Later: decrypt and restore
const decrypted = SecureWallet.decryptMnemonic(encrypted, password)
const restored = SecureWallet.fromMnemonic(decrypted)
```

### 3. Testing on Testnet First

```typescript
// Use testnet for development
const isTestnet = true

if (isTestnet) {
  // Use testnet APIs
  const API_BASE = 'https://api.whatsonchain.com/v1/bsv/test'
} else {
  // Use mainnet APIs
  const API_BASE = 'https://api.whatsonchain.com/v1/bsv/main'
}
```

## Complete Wallet Example

```typescript
import { PrivateKey, HD } from '@bsv/sdk'

class BSVWallet {
  private masterKey: PrivateKey
  private mnemonic: string
  private network: 'mainnet' | 'testnet'
  private addressIndex: number = 0

  constructor(mnemonic?: string, network: 'mainnet' | 'testnet' = 'testnet') {
    this.network = network
    this.mnemonic = mnemonic || HD.generateMnemonic(128)

    const hdSeed = HD.fromMnemonic(this.mnemonic)
    this.masterKey = PrivateKey.fromHex(hdSeed.privateKey.toHex())
  }

  // Derive next unused address
  getNewAddress(): string {
    const address = this.generateAddress(this.addressIndex)
    this.addressIndex++
    return address
  }

  // Generate address at specific index
  generateAddress(index: number): string {
    const key = this.deriveKey(index)
    return key.toPublicKey().toAddress()
  }

  // Derive private key at index
  private deriveKey(index: number): PrivateKey {
    const protocolID = PrivateKey.fromString('bsv-wallet')
    const keyID = PrivateKey.fromString(`${this.network}-${index}`)
    return this.masterKey.deriveChild(protocolID, keyID)
  }

  // Get balance
  async getBalance(): Promise<number> {
    let total = 0
    let emptyCount = 0

    for (let i = 0; i <= this.addressIndex + 5; i++) {
      const address = this.generateAddress(i)
      const utxos = await this.fetchUTXOs(address)
      const balance = utxos.reduce((sum, u) => sum + u.satoshis, 0)

      total += balance

      if (balance === 0) {
        emptyCount++
        if (emptyCount >= 5) break
      } else {
        emptyCount = 0
      }
    }

    return total
  }

  // Fetch UTXOs from API
  private async fetchUTXOs(address: string): Promise<UTXO[]> {
    const networkPath = this.network === 'mainnet' ? 'main' : 'test'
    const url = `https://api.whatsonchain.com/v1/bsv/${networkPath}/address/${address}/unspent`

    try {
      const response = await fetch(url)
      if (!response.ok) return []
      return await response.json()
    } catch (error) {
      console.error('Error fetching UTXOs:', error)
      return []
    }
  }

  // Backup wallet
  exportBackup(): { mnemonic: string; network: string } {
    return {
      mnemonic: this.mnemonic,
      network: this.network
    }
  }

  // Restore wallet
  static restore(mnemonic: string, network: 'mainnet' | 'testnet' = 'testnet'): BSVWallet {
    return new BSVWallet(mnemonic, network)
  }
}

// Create new wallet
const wallet = new BSVWallet(undefined, 'testnet')

// Generate receiving address
const address = wallet.getNewAddress()
console.log('Send testnet BSV to:', address)

// Check balance
const balance = await wallet.getBalance()
console.log(`Balance: ${balance / 100000000} BSV`)

// Backup
const backup = wallet.exportBackup()
console.log('Save this backup:', backup)

// Restore from backup
const restored = BSVWallet.restore(backup.mnemonic, backup.network as any)
```

## Practice Exercises

1. **Create Simple Wallet**: Generate keys and addresses
2. **Create HD Wallet**: Generate mnemonic and derive keys
3. **Check Balance**: Query UTXOs for addresses
4. **Backup & Restore**: Export and import wallet
5. **Address Generation**: Derive 100 addresses from one seed

## Related Components

- [Private Keys](../../../sdk-components/private-keys/README.md)
- [HD Wallets](../../../sdk-components/hd-wallets/README.md)
- [BRC-42](../../../sdk-components/brc-42/README.md)

## Related Code Features

- [Generate Private Key](../../../code-features/generate-private-key/README.md)
- [Create Wallet](../../../code-features/create-wallet/README.md)

## Next Steps

Now that you have a wallet, let's create your first transaction!

Continue to: [Your First Transaction](../first-transaction/README.md)

## Security Reminders

⚠️ **Critical Security Points**:
1. Never share your private key or mnemonic
2. Always backup your mnemonic phrase
3. Use testnet for learning and development
4. Encrypt sensitive data at rest
5. Test with small amounts first

Your mnemonic phrase is like a master password. Anyone with it can access all your funds!
