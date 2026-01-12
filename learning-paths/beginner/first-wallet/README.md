# Your First Wallet

## Overview

In this module, you'll learn how to create BSV wallets using the TypeScript SDK. We'll cover two distinct approaches: **backend wallet management** (server-side key management) and **frontend wallet integration** (browser-based using WalletClient).

Understanding these two paradigms is crucial for building secure and user-friendly BSV applications.

## Learning Objectives

By the end of this module, you will be able to:
- Understand the difference between backend and frontend wallet paradigms
- Generate and manage private keys securely on the backend
- Implement HD wallets using BRC-42 key derivation
- Follow security best practices for key storage
- Understand when to use WalletClient for frontend applications
- Create production-ready wallet implementations

## What is a Wallet?

A BSV wallet is software that:
- **Manages private keys** securely
- **Generates addresses** for receiving payments
- **Signs transactions** to send payments
- **Tracks UTXOs** (Unspent Transaction Outputs)
- **Maintains transaction history**

**Important**: A wallet doesn't "store" BSV. It stores the private keys that control UTXOs on the blockchain.

## Backend vs Frontend Wallet Paradigm

### Backend Wallet (Custodial/Server-Side)

**Use Case**: When your application controls funds on behalf of users

**Characteristics**:
- Private keys stored server-side
- Application signs transactions
- Full control over key management
- Requires robust security infrastructure
- Examples: Exchanges, payment processors, custodial services

**Pros**:
- Simplified user experience
- No wallet setup required for users
- Can implement complex business logic

**Cons**:
- You're responsible for security
- Single point of failure
- Regulatory implications (you custody funds)

### Frontend Wallet (Non-Custodial)

**Use Case**: When users control their own funds

**Characteristics**:
- Private keys stored in user's browser/device
- User signs transactions via wallet extension
- No server-side key storage
- Uses WalletClient SDK integration
- Examples: DApps, web3 applications

**Pros**:
- Users control their own keys
- No custody liability
- Better privacy

**Cons**:
- Requires user to have wallet setup
- More complex UX
- Limited control over transaction signing

**This Module**: We'll focus on **backend wallet implementation**. For frontend wallet integration using WalletClient, see the [Wallet Client Integration](../wallet-client-integration/README.md) module.

## Backend Wallet Implementation

### Understanding Wallet Types

#### 1. Simple Wallet
Single private key controlling one or more addresses.

**Pros**: Simple to understand and implement
**Cons**: Must backup each key separately, not scalable

#### 2. HD Wallet (Hierarchical Deterministic)
One master seed generates unlimited keys deterministically using BRC-42.

**Pros**:
- One backup for all keys
- Deterministic key derivation
- Can generate keys without exposing master key
- Supports key hierarchy and organization

**Cons**: Slightly more complex implementation

**Recommendation**: Always use HD wallets for production applications.

## Creating a Simple Wallet

### Step 1: Generate a Private Key

```typescript
import { PrivateKey } from '@bsv/sdk'

// Generate a cryptographically secure random private key
const privateKey = PrivateKey.fromRandom()

// Export in different formats
console.log('=== Private Key Generation ===')
console.log('Private Key (WIF):', privateKey.toWif())
console.log('Private Key (Hex):', privateKey.toHex())
console.log('\n⚠️ Never share your private key!')
console.log('⚠️ This example is for learning purposes only')

// Save this code to a file and run it:
// npx ts-node generate-key.ts
```

**Output Example**:
```
=== Private Key Generation ===
Private Key (WIF): L4rK1yDtCWekvXuE6oXD9jCYfFNV2cWRpVuPLBcCU2z8TrisoyY1
Private Key (Hex): e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855

⚠️ Never share your private key!
⚠️ This example is for learning purposes only
```

**Security Note**: Never log private keys in production. This is for educational purposes only.

**To run this example:**
```bash
# Create the file
echo 'import { PrivateKey } from "@bsv/sdk"
const privateKey = PrivateKey.fromRandom()
console.log("Private Key (WIF):", privateKey.toWif())
console.log("Address:", privateKey.toPublicKey().toAddress())' > generate-key.ts

# Run it
npx ts-node generate-key.ts
```

### Step 2: Derive Public Key and Address

```typescript
import { PrivateKey } from '@bsv/sdk'

// Generate or load a private key
const privateKey = PrivateKey.fromRandom()

// Derive public key from private key (using elliptic curve cryptography)
const publicKey = privateKey.toPublicKey()

// Generate Bitcoin address from public key
const address = publicKey.toAddress()

console.log('=== Key Derivation ===')
console.log('Private Key (WIF):', privateKey.toWif())
console.log('Public Key (Hex):', publicKey.toString())
console.log('Address:', address)
console.log('\n✅ You can share the address to receive payments!')
console.log('💡 Fund this address with testnet BSV from BSV Desktop')
console.log('   Download: https://desktop.bsvb.tech/')

// Run this example: npx ts-node derive-address.ts
```

**Key Concepts**:
- **Private Key**: Secret number (256 bits). Must be kept secure.
- **Public Key**: Derived from private key via ECDSA. Can be shared publicly.
- **Address**: Hash of public key. Used for receiving payments.

**To run this example:**
```bash
# Create and run the complete example
cat > derive-address.ts << 'EOF'
import { PrivateKey } from '@bsv/sdk'

const privateKey = PrivateKey.fromRandom()
const publicKey = privateKey.toPublicKey()
const address = publicKey.toAddress()

console.log('Private Key (WIF):', privateKey.toWif())
console.log('Public Key:', publicKey.toString())
console.log('Address:', address)
console.log('\nFund at: https://desktop.bsvb.tech/')
EOF

npx ts-node derive-address.ts
```

### Step 3: Create a Basic Wallet Class

```typescript
import { PrivateKey } from '@bsv/sdk'

class SimpleWallet {
  private privateKey: PrivateKey
  public readonly address: string
  public readonly publicKey: string

  constructor(privateKey?: PrivateKey) {
    this.privateKey = privateKey || PrivateKey.fromRandom()

    const pubKey = this.privateKey.toPublicKey()
    this.publicKey = pubKey.toString()
    this.address = pubKey.toAddress()
  }

  // Get public wallet information
  getInfo() {
    return {
      address: this.address,
      publicKey: this.publicKey
    }
  }

  // Get private key for transaction signing
  // INTERNAL USE ONLY - never expose via API
  getPrivateKey(): PrivateKey {
    return this.privateKey
  }

  // Export private key for backup (encrypted storage only)
  exportPrivateKey(): string {
    return this.privateKey.toWif()
  }

  // Restore wallet from WIF-encoded private key
  static fromWIF(wif: string): SimpleWallet {
    const privateKey = PrivateKey.fromWif(wif)
    return new SimpleWallet(privateKey)
  }

  // Restore wallet from hex-encoded private key
  static fromHex(hex: string): SimpleWallet {
    const privateKey = PrivateKey.fromHex(hex)
    return new SimpleWallet(privateKey)
  }
}

// Usage Example
const wallet = new SimpleWallet()
console.log('=== Simple Wallet Created ===')
console.log('Wallet Address:', wallet.address)
console.log('Public Key:', wallet.publicKey)

// Backup (encrypt this in production!)
const backupWIF = wallet.exportPrivateKey()
console.log('\n=== Backup (Keep Secure!) ===')
console.log('Private Key (WIF):', backupWIF)

// Restore from backup
const restoredWallet = SimpleWallet.fromWIF(backupWIF)
console.log('\n=== Wallet Restored ===')
console.log('Restored Address:', restoredWallet.address)
console.log('✅ Addresses match:', wallet.address === restoredWallet.address)

console.log('\n💡 Fund this address with testnet BSV from BSV Desktop')
console.log('   Download: https://desktop.bsvb.tech/')

// Run this example: npx ts-node simple-wallet.ts
```

## Creating an HD Wallet with BRC-42

HD wallets provide a scalable, hierarchical key management system. The SDK uses **BRC-42** for deterministic key derivation.

### Understanding BRC-42 Key Derivation

BRC-42 defines how to derive child keys from a master key using:
- **Protocol ID**: Identifies the application/protocol
- **Key ID**: Identifies the specific key purpose
- **Counterparty**: Optional, for key exchange protocols

This creates a hierarchy: `Master Key → Protocol → Key ID → Derived Key`

### Step 1: Generate Mnemonic Seed Phrase

```typescript
import { Mnemonic } from '@bsv/sdk'

// Generate 12-word mnemonic (128 bits of entropy)
const mnemonic = Mnemonic.fromRandom()
const mnemonicPhrase = mnemonic.toString()

console.log('Mnemonic:', mnemonicPhrase)
// Example: abandon ability able about above absent absorb abstract absurd abuse access accident

// Convert to seed (used for key derivation)
const seed = mnemonic.toSeed()
console.log('Seed (Hex):', Buffer.from(seed).toString('hex'))
```

**Mnemonic Standards**:
- 12 words = 128 bits of entropy (standard, secure)
- 15 words = 160 bits of entropy
- 18 words = 192 bits of entropy
- 21 words = 224 bits of entropy
- 24 words = 256 bits of entropy (maximum security)

**Security**: The mnemonic phrase is the root of all keys. Protect it like the master password to all funds.

### Step 2: Derive Master Private Key

```typescript
import { Mnemonic, PrivateKey } from '@bsv/sdk'

// Generate or restore from mnemonic
const mnemonic = Mnemonic.fromRandom()

// Derive master private key from mnemonic seed
const seed = mnemonic.toSeed()
// Use first 32 bytes of seed as private key
const seedHex = Buffer.from(seed).toString('hex').substring(0, 64)
const masterKey = PrivateKey.fromHex(seedHex)

console.log('Master Private Key:', masterKey.toWif())
```

### Step 3: Implement HD Wallet with BRC-42

```typescript
import { Mnemonic, PrivateKey, PublicKey } from '@bsv/sdk'

class HDWallet {
  private masterKey: PrivateKey
  private mnemonic: Mnemonic
  private protocolID: string
  private counterparty: PublicKey  // Use 'anyone' for self-derivation
  private derivedKeys: Map<string, PrivateKey> = new Map()

  constructor(mnemonic?: Mnemonic, protocolID: string = 'wallet') {
    // Generate new mnemonic or use provided one
    this.mnemonic = mnemonic || Mnemonic.fromRandom()

    // Derive master key from mnemonic
    const seed = this.mnemonic.toSeed()
    // Use first 32 bytes of seed as private key
    const seedHex = Buffer.from(seed).toString('hex').substring(0, 64)
    this.masterKey = PrivateKey.fromHex(seedHex)

    // Set protocol ID for BRC-42 derivation
    this.protocolID = protocolID

    // Use 'anyone' (PrivateKey(1).toPublicKey()) as counterparty for self-derivation
    this.counterparty = new PrivateKey(1).toPublicKey()
  }

  /**
   * Derive a child private key using BRC-42
   *
   * @param keyID - Unique identifier for this key (e.g., 'address-0', 'signing-key')
   * @param counterparty - Optional counterparty for key exchange protocols
   * @returns Derived private key
   */
  deriveKey(keyID: string, counterparty?: string): PrivateKey {
    const cacheKey = counterparty ? `${keyID}-${counterparty}` : keyID

    // Return cached key if already derived
    if (this.derivedKeys.has(cacheKey)) {
      return this.derivedKeys.get(cacheKey)!
    }

    // Create invoice number in BRC-42 format: "<securityLevel>-<protocolID>-<keyID>"
    // Security level 0 = can be revealed, level 1 = with counterparty consent, level 2 = cannot be revealed
    const invoiceNumber = `0-${this.protocolID}-${keyID}`

    // Derive child key using BRC-42
    // For simple wallet: counterparty is 'anyone' (PrivateKey(1).toPublicKey())
    const derivedKey = this.masterKey.deriveChild(this.counterparty, invoiceNumber)

    // Cache derived key
    this.derivedKeys.set(cacheKey, derivedKey)

    return derivedKey
  }

  /**
   * Generate address at specific index
   * Uses pattern: protocol='wallet', keyID='address-{index}'
   */
  generateAddress(index: number): string {
    const key = this.deriveKey(`address-${index}`)
    return key.toPublicKey().toAddress()
  }

  /**
   * Generate multiple addresses
   */
  generateAddresses(count: number, startIndex: number = 0): string[] {
    const addresses: string[] = []
    for (let i = startIndex; i < startIndex + count; i++) {
      addresses.push(this.generateAddress(i))
    }
    return addresses
  }

  /**
   * Get private key for specific address index
   * INTERNAL USE ONLY - for transaction signing
   */
  getPrivateKeyForAddress(index: number): PrivateKey {
    return this.deriveKey(`address-${index}`)
  }

  /**
   * Derive specialized keys for different purposes
   */
  deriveSigningKey(purpose: string): PrivateKey {
    return this.deriveKey(`signing-${purpose}`)
  }

  deriveEncryptionKey(purpose: string): PrivateKey {
    return this.deriveKey(`encryption-${purpose}`)
  }

  /**
   * Export mnemonic for backup
   * WARNING: This should only be called in secure contexts
   * In production: encrypt before storing
   */
  exportMnemonic(): string {
    return this.mnemonic.toString()
  }

  /**
   * Restore HD wallet from mnemonic phrase
   */
  static fromMnemonic(mnemonicPhrase: string, protocolID: string = 'simple wallet'): HDWallet {
    const mnemonic = Mnemonic.fromString(mnemonicPhrase)
    return new HDWallet(mnemonic, protocolID)
  }

  /**
   * Get wallet metadata (safe to expose)
   */
  getInfo() {
    return {
      protocolID: 'wallet',
      firstAddress: this.generateAddress(0),
      // Never expose mnemonic or master key
    }
  }
}

// Example Usage
const wallet = new HDWallet()

console.log('=== HD Wallet Created ===')
console.log('Protocol ID:', wallet.getInfo().protocolID)
console.log('First Address:', wallet.getInfo().firstAddress)

// Generate receiving addresses
console.log('\n=== Generated Addresses ===')
const addresses = wallet.generateAddresses(5)
addresses.forEach((addr, idx) => {
  console.log(`Address ${idx}:`, addr)
})

// Derive specialized keys
console.log('\n=== Specialized Keys ===')
const signingKey = wallet.deriveSigningKey('document-signing')
console.log('Signing Key Address:', signingKey.toPublicKey().toAddress())

const encryptionKey = wallet.deriveEncryptionKey('message-encryption')
console.log('Encryption Key Address:', encryptionKey.toPublicKey().toAddress())

// Backup wallet (encrypt this!)
const backupPhrase = wallet.exportMnemonic()
console.log('\n=== Backup (KEEP SECURE!) ===')
console.log('Mnemonic:', backupPhrase)
console.log('⚠️ Write this down and store it safely!')

// Restore wallet from backup
console.log('\n=== Testing Restore ===')
const restored = HDWallet.fromMnemonic(backupPhrase)
console.log('Restored First Address:', restored.generateAddress(0))
console.log('✅ Addresses match:', wallet.generateAddress(0) === restored.generateAddress(0))

console.log('\n💡 Fund any of these addresses with testnet BSV from BSV Desktop')
console.log('   Download: https://desktop.bsvb.tech/')
console.log('\n📖 Complete onboarding guide:')
console.log('   https://hub.bsvblockchain.org/demos-and-onboardings/onboardings/onboarding-catalog/metanet-desktop-mainnet')

// Run this example: npx ts-node hd-wallet-example.ts
```

### Advanced: Using KeyDeriver for Complex Hierarchies

For more complex key derivation needs, the SDK provides the `KeyDeriver` class:

```typescript
import { PrivateKey, KeyDeriver } from '@bsv/sdk'

class AdvancedHDWallet {
  private masterKey: PrivateKey
  private deriver: KeyDeriver

  constructor(masterKey: PrivateKey) {
    this.masterKey = masterKey
    this.deriver = new KeyDeriver(masterKey)
  }

  /**
   * Derive keys with custom protocol and key IDs
   */
  deriveCustomKey(protocolID: string, keyID: string): PrivateKey {
    return this.deriver.deriveChild(
      PrivateKey.fromString(protocolID),
      PrivateKey.fromString(keyID)
    )
  }

  /**
   * Derive public key without exposing private key
   * Useful for watch-only wallets or key distribution
   */
  derivePublicKey(protocolID: string, keyID: string): string {
    const privateKey = this.deriveCustomKey(protocolID, keyID)
    return privateKey.toPublicKey().toHex()
  }

  /**
   * Create child deriver for delegated key management
   * This allows creating sub-wallets without exposing master key
   */
  createChildDeriver(protocolID: string): KeyDeriver {
    const childKey = this.masterKey.deriveChild(
      PrivateKey.fromString(protocolID),
      PrivateKey.fromString('master')
    )
    return new KeyDeriver(childKey)
  }
}
```

## UTXO Management and Balance Tracking

**Important**: In modern SDK-based applications, you should **NOT** manually track UTXOs by querying external APIs. The SDK provides built-in UTXO management.

### How UTXO Tracking Works

When you create transactions using the SDK:

1. **Backend Applications**: Use `Transaction` class which handles UTXO selection automatically
2. **Frontend Applications**: `WalletClient` manages UTXOs through wallet integration
3. **ARC Integration**: Submit transactions to ARC, which provides status and confirmation
4. **No Manual Fetching**: Don't query blockchain explorers for UTXOs

### Transaction Creation (Basic Pattern)

```typescript
import { PrivateKey, Transaction, P2PKH } from '@bsv/sdk'

class WalletWithTransactions {
  private wallet: HDWallet

  constructor(wallet: HDWallet) {
    this.wallet = wallet
  }

  /**
   * Create and sign a transaction
   * Note: UTXOs are provided by your application's UTXO tracking system,
   * NOT fetched from external APIs
   */
  async createTransaction(
    recipientAddress: string,
    amountSatoshis: number,
    changeIndex: number = 0
  ): Promise<Transaction> {
    // Create new transaction
    const tx = new Transaction()

    // Add outputs
    tx.addOutput({
      satoshis: amountSatoshis,
      lockingScript: new P2PKH().lock(recipientAddress)
    })

    // Note: In a real application, you would:
    // 1. Query your own database for available UTXOs
    // 2. Select appropriate UTXOs for the transaction
    // 3. Add them as inputs to the transaction
    // 4. Add change output to your own address
    // 5. Sign with appropriate private keys

    // Get signing key
    const privateKey = this.wallet.getPrivateKeyForAddress(0)

    // Sign transaction (SDK handles the signing logic)
    // In practice, you'd sign each input with the appropriate key
    await tx.sign()

    return tx
  }
}
```

**Key Point**: Your application should maintain its own UTXO state in a database. When transactions are created/confirmed, update your UTXO set accordingly. See the [Transaction Management](../first-transaction/README.md) module for details.

## Secure Key Storage Best Practices

Security is paramount when managing private keys on the backend. Follow these practices:

### 1. Never Hard-Code Keys

**NEVER DO THIS**:
```typescript
// ❌ WRONG - Never hard-code keys
const privateKey = PrivateKey.fromWif('L4rK1yDtCWekvXuE6oXD9jCYfFNV2cWRpVuPLBcCU2z8TrisoyY1')
```

### 2. Environment Variables (Development Only)

For development and testing, use environment variables:

```typescript
import { PrivateKey } from '@bsv/sdk'
import * as dotenv from 'dotenv'

dotenv.config()

// Load from environment
const masterKeyWIF = process.env.MASTER_PRIVATE_KEY
if (!masterKeyWIF) {
  throw new Error('MASTER_PRIVATE_KEY not set in environment')
}

const masterKey = PrivateKey.fromWif(masterKeyWIF)
```

**.env file** (NEVER commit to version control):
```env
MASTER_PRIVATE_KEY=L4rK1yDtCWekvXuE6oXD9jCYfFNV2cWRpVuPLBcCU2z8TrisoyY1
```

**.gitignore**:
```
.env
.env.*
*.key
*.pem
```

### 3. Production Key Storage

**For Production, Use**:

#### Option A: Hardware Security Modules (HSM)

```typescript
// Example: AWS CloudHSM, Azure Key Vault, Google Cloud KMS
import { KMSClient, SignCommand } from '@aws-sdk/client-kms'

class HSMWallet {
  private kmsClient: KMSClient
  private keyId: string

  constructor(keyId: string) {
    this.kmsClient = new KMSClient({ region: 'us-east-1' })
    this.keyId = keyId
  }

  /**
   * Sign transaction using HSM
   * Private key never leaves the HSM
   */
  async sign(message: Buffer): Promise<Buffer> {
    const command = new SignCommand({
      KeyId: this.keyId,
      Message: message,
      SigningAlgorithm: 'ECDSA_SHA_256'
    })

    const response = await this.kmsClient.send(command)
    return Buffer.from(response.Signature!)
  }
}
```

#### Option B: Encrypted Database Storage

```typescript
import * as crypto from 'crypto'
import { PrivateKey } from '@bsv/sdk'

class EncryptedKeyStore {
  private encryptionKey: Buffer

  constructor(encryptionKey: string) {
    // Derive encryption key from master password
    this.encryptionKey = crypto.scryptSync(encryptionKey, 'salt', 32)
  }

  /**
   * Encrypt private key for database storage
   */
  encryptPrivateKey(privateKey: PrivateKey): string {
    const algorithm = 'aes-256-gcm'
    const iv = crypto.randomBytes(16)
    const cipher = crypto.createCipheriv(algorithm, this.encryptionKey, iv)

    const keyWIF = privateKey.toWif()
    let encrypted = cipher.update(keyWIF, 'utf8', 'hex')
    encrypted += cipher.final('hex')

    const authTag = cipher.getAuthTag()

    // Store: iv + authTag + encrypted data
    return JSON.stringify({
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex'),
      data: encrypted
    })
  }

  /**
   * Decrypt private key from database
   */
  decryptPrivateKey(encryptedData: string): PrivateKey {
    const { iv, authTag, data } = JSON.parse(encryptedData)
    const algorithm = 'aes-256-gcm'

    const decipher = crypto.createDecipheriv(
      algorithm,
      this.encryptionKey,
      Buffer.from(iv, 'hex')
    )

    decipher.setAuthTag(Buffer.from(authTag, 'hex'))

    let decrypted = decipher.update(data, 'hex', 'utf8')
    decrypted += decipher.final('utf8')

    return PrivateKey.fromWif(decrypted)
  }

  /**
   * Encrypt mnemonic phrase
   */
  encryptMnemonic(mnemonic: string): string {
    const algorithm = 'aes-256-gcm'
    const iv = crypto.randomBytes(16)
    const cipher = crypto.createCipheriv(algorithm, this.encryptionKey, iv)

    let encrypted = cipher.update(mnemonic, 'utf8', 'hex')
    encrypted += cipher.final('hex')

    const authTag = cipher.getAuthTag()

    return JSON.stringify({
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex'),
      data: encrypted
    })
  }

  /**
   * Decrypt mnemonic phrase
   */
  decryptMnemonic(encryptedData: string): string {
    const { iv, authTag, data } = JSON.parse(encryptedData)
    const algorithm = 'aes-256-gcm'

    const decipher = crypto.createDecipheriv(
      algorithm,
      this.encryptionKey,
      Buffer.from(iv, 'hex')
    )

    decipher.setAuthTag(Buffer.from(authTag, 'hex'))

    let decrypted = decipher.update(data, 'hex', 'utf8')
    decrypted += decipher.final('utf8')

    return decrypted
  }
}

// Usage
const keyStore = new EncryptedKeyStore(process.env.MASTER_ENCRYPTION_KEY!)

// Encrypt for storage
const wallet = new HDWallet()
const mnemonic = wallet.exportMnemonic()
const encryptedMnemonic = keyStore.encryptMnemonic(mnemonic)

// Store in database
await db.wallets.insert({
  userId: 'user123',
  encryptedMnemonic,
  createdAt: new Date()
})

// Later: Retrieve and decrypt
const record = await db.wallets.findOne({ userId: 'user123' })
const decryptedMnemonic = keyStore.decryptMnemonic(record.encryptedMnemonic)
const restoredWallet = HDWallet.fromMnemonic(decryptedMnemonic)
```

#### Option C: Secret Management Services

```typescript
// Example: HashiCorp Vault, AWS Secrets Manager
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager'

class SecretManagerWallet {
  private client: SecretsManagerClient

  constructor() {
    this.client = new SecretsManagerClient({ region: 'us-east-1' })
  }

  /**
   * Load private key from AWS Secrets Manager
   */
  async loadPrivateKey(secretName: string): Promise<PrivateKey> {
    const command = new GetSecretValueCommand({
      SecretId: secretName
    })

    const response = await this.client.send(command)
    const secretString = response.SecretString!
    const secret = JSON.parse(secretString)

    return PrivateKey.fromWif(secret.privateKey)
  }

  /**
   * Load HD wallet from Secrets Manager
   */
  async loadHDWallet(secretName: string): Promise<HDWallet> {
    const command = new GetSecretValueCommand({
      SecretId: secretName
    })

    const response = await this.client.send(command)
    const secretString = response.SecretString!
    const secret = JSON.parse(secretString)

    return HDWallet.fromMnemonic(secret.mnemonic)
  }
}
```

### 4. Security Checklist

**Key Storage**:
- [ ] Use HSM/KMS for production environments
- [ ] Encrypt keys at rest in database
- [ ] Use secret management services (Vault, AWS Secrets Manager)
- [ ] Never log private keys or mnemonics
- [ ] Implement key rotation policies
- [ ] Use separate keys for different environments (dev/staging/prod)

**Access Control**:
- [ ] Limit access to key storage systems
- [ ] Implement role-based access control (RBAC)
- [ ] Audit all key access
- [ ] Use multi-signature for high-value operations
- [ ] Implement rate limiting on signing operations

**Operational Security**:
- [ ] Never commit keys to version control
- [ ] Use `.gitignore` for sensitive files
- [ ] Rotate keys regularly
- [ ] Monitor for unauthorized access
- [ ] Have key recovery procedures
- [ ] Test backup and restore procedures

**Development vs Production**:
- [ ] Use testnet for development
- [ ] Separate keys for dev/staging/prod
- [ ] Use environment-specific configuration
- [ ] Never use production keys in development

### 5. Key Rotation Example

```typescript
class RotatableWallet {
  private currentWallet: HDWallet
  private version: number

  constructor(mnemonic: string, version: number = 1) {
    this.currentWallet = HDWallet.fromMnemonic(mnemonic)
    this.version = version
  }

  /**
   * Rotate to new master key
   * Returns new mnemonic that should be securely stored
   */
  rotate(): string {
    this.version++
    const newWallet = new HDWallet(undefined, `wallet-v${this.version}`)
    const newMnemonic = newWallet.exportMnemonic()

    this.currentWallet = newWallet

    // In production:
    // 1. Generate new wallet
    // 2. Move funds from old wallet to new wallet
    // 3. Archive old wallet securely
    // 4. Update encrypted storage with new wallet

    return newMnemonic
  }

  getVersion(): number {
    return this.version
  }
}
```

## Complete Production Wallet Example

Here's a complete example demonstrating production-ready patterns:

```typescript
import { PrivateKey, Mnemonic } from '@bsv/sdk'
import * as crypto from 'crypto'

interface WalletConfig {
  encryptionKey: string
  network: 'mainnet' | 'testnet'
  protocolID: string
}

interface WalletBackup {
  encryptedMnemonic: string
  network: string
  version: number
  createdAt: string
}

class ProductionWallet {
  private wallet: HDWallet
  private keyStore: EncryptedKeyStore
  private network: 'mainnet' | 'testnet'
  private version: number

  private constructor(
    wallet: HDWallet,
    keyStore: EncryptedKeyStore,
    network: 'mainnet' | 'testnet',
    version: number = 1
  ) {
    this.wallet = wallet
    this.keyStore = keyStore
    this.network = network
    this.version = version
  }

  /**
   * Create new wallet
   */
  static create(config: WalletConfig): ProductionWallet {
    const wallet = new HDWallet(undefined, config.protocolID)
    const keyStore = new EncryptedKeyStore(config.encryptionKey)

    return new ProductionWallet(wallet, keyStore, config.network)
  }

  /**
   * Restore wallet from encrypted backup
   */
  static restore(backup: WalletBackup, config: WalletConfig): ProductionWallet {
    const keyStore = new EncryptedKeyStore(config.encryptionKey)
    const mnemonic = keyStore.decryptMnemonic(backup.encryptedMnemonic)
    const wallet = HDWallet.fromMnemonic(mnemonic, config.protocolID)

    return new ProductionWallet(
      wallet,
      keyStore,
      backup.network as 'mainnet' | 'testnet',
      backup.version
    )
  }

  /**
   * Generate new receiving address
   */
  generateReceiveAddress(index: number): string {
    return this.wallet.generateAddress(index)
  }

  /**
   * Get private key for signing (internal use only)
   */
  private getSigningKey(index: number): PrivateKey {
    return this.wallet.getPrivateKeyForAddress(index)
  }

  /**
   * Export encrypted backup
   */
  exportBackup(): WalletBackup {
    const mnemonic = this.wallet.exportMnemonic()
    const encryptedMnemonic = this.keyStore.encryptMnemonic(mnemonic)

    return {
      encryptedMnemonic,
      network: this.network,
      version: this.version,
      createdAt: new Date().toISOString()
    }
  }

  /**
   * Get public wallet information (safe to expose)
   */
  getPublicInfo() {
    return {
      network: this.network,
      version: this.version,
      firstAddress: this.wallet.generateAddress(0)
    }
  }

  /**
   * Verify backup can be restored
   */
  static verifyBackup(backup: WalletBackup, config: WalletConfig): boolean {
    try {
      const restored = ProductionWallet.restore(backup, config)
      return restored !== null
    } catch (error) {
      console.error('Backup verification failed:', error)
      return false
    }
  }
}

// Usage Example
const config: WalletConfig = {
  encryptionKey: process.env.WALLET_ENCRYPTION_KEY!,
  network: 'testnet',
  protocolID: 'my-app-wallet'
}

// Create new wallet
const wallet = ProductionWallet.create(config)
console.log('Wallet Info:', wallet.getPublicInfo())

// Generate addresses
const address1 = wallet.generateReceiveAddress(0)
const address2 = wallet.generateReceiveAddress(1)

// Create encrypted backup
const backup = wallet.exportBackup()

// Store backup in database (encrypted)
await db.wallets.insert({
  userId: 'user123',
  backup: backup,
  createdAt: new Date()
})

// Later: Restore from backup
const storedBackup = await db.wallets.findOne({ userId: 'user123' })
const restoredWallet = ProductionWallet.restore(storedBackup.backup, config)

// Verify backup integrity
const isValid = ProductionWallet.verifyBackup(backup, config)
console.log('Backup valid:', isValid)
```

## Frontend Wallet Alternative: WalletClient

For frontend applications where users control their own keys, use **WalletClient** instead of managing keys directly.

### When to Use WalletClient

- Building a DApp (decentralized application)
- Users need to control their own funds
- Non-custodial architecture
- Browser-based applications
- Need wallet extension integration

### WalletClient Quick Example

```typescript
import { WalletClient } from '@bsv/sdk'

// Create WalletClient and connect to user's wallet
const wallet = new WalletClient('auto')
await wallet.connectToSubstrate()

// Request user's identity public key
const { publicKey } = await wallet.getPublicKey({ identityKey: true })

// Create action (auto-signs and broadcasts)
const result = await wallet.createAction({
  description: 'Send payment',
  outputs: [{
    lockingScript: new P2PKH().lock(recipientAddress).toHex(),
    satoshis: 10000,
    outputDescription: 'Payment'
  }]
})

const txid = result.txid
```

**Key Difference**: With WalletClient, the user's browser wallet handles all key management and signing. Your application never touches private keys.

**Learn More**: See the [Wallet Client Integration](../wallet-client-integration/README.md) module for complete frontend wallet implementation.

## Testing Your Wallet

### Unit Tests Example

```typescript
import { describe, it, expect } from '@jest/globals'

describe('HDWallet', () => {
  it('should generate deterministic addresses from mnemonic', () => {
    const mnemonic = 'abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about'

    const wallet1 = HDWallet.fromMnemonic(mnemonic)
    const wallet2 = HDWallet.fromMnemonic(mnemonic)

    expect(wallet1.generateAddress(0)).toBe(wallet2.generateAddress(0))
    expect(wallet1.generateAddress(1)).toBe(wallet2.generateAddress(1))
  })

  it('should derive different keys for different indices', () => {
    const wallet = new HDWallet()

    const addr0 = wallet.generateAddress(0)
    const addr1 = wallet.generateAddress(1)

    expect(addr0).not.toBe(addr1)
  })

  it('should restore wallet from backup', () => {
    const original = new HDWallet()
    const backup = original.exportMnemonic()

    const restored = HDWallet.fromMnemonic(backup)

    expect(original.generateAddress(0)).toBe(restored.generateAddress(0))
  })

  it('should encrypt and decrypt mnemonic', () => {
    const keyStore = new EncryptedKeyStore('test-password')
    const original = 'abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about'

    const encrypted = keyStore.encryptMnemonic(original)
    const decrypted = keyStore.decryptMnemonic(encrypted)

    expect(decrypted).toBe(original)
  })
})
```

### Integration Testing on Testnet

```typescript
// Always use testnet for development and testing
const wallet = new HDWallet(undefined, 'testnet-wallet')

// Get testnet address
const testAddress = wallet.generateAddress(0)
console.log('Testnet Address:', testAddress)

// Fund this address from testnet faucet:
// Get testnet coins from MetaNet Desktop Wallet's built-in faucet or BSV Discord community

// Test transaction creation (covered in next module)
// ...
```

## Practice Exercises

1. **Simple Wallet**: Create a simple wallet and derive 10 addresses
2. **HD Wallet**: Create an HD wallet with BRC-42 and generate addresses
3. **Encryption**: Implement mnemonic encryption and decryption
4. **Backup & Restore**: Export wallet backup and restore it
5. **Key Derivation**: Derive specialized keys for signing, encryption
6. **Production Setup**: Configure encrypted key storage with environment variables

## Summary

**What You Learned**:
- Difference between backend and frontend wallet paradigms
- How to create simple and HD wallets using the SDK
- BRC-42 key derivation for hierarchical key management
- Security best practices for production key storage
- When to use WalletClient for frontend applications

**Key Takeaways**:
- **Backend wallets**: You control keys, you're responsible for security
- **Frontend wallets**: Users control keys via WalletClient
- **Always use HD wallets** for production (BRC-42)
- **Never expose private keys** in logs, APIs, or UI
- **Use HSM/encryption** for production key storage
- **Test on testnet** before deploying to mainnet

## Related Components

- [Private Keys](../../../sdk-components/private-keys/README.md)
- [HD Wallets](../../../sdk-components/hd-wallets/README.md)
- [BRC-42 Key Derivation](../../../sdk-components/brc-42/README.md)
- <!-- [Mnemonic Generation](../../../sdk-components/mnemonic/README.md) (Component pending) -->
- <!-- [WalletClient](../../../sdk-components/wallet-client/README.md) (Component pending) -->

## Related Code Features

- [Generate Private Key](../../../code-features/generate-private-key/README.md)
- [BRC-42 Key Derivation](../../../code-features/brc-42-derivation/README.md)

## Next Steps

Now that you understand wallet management, let's create and broadcast transactions!

Continue to: [Your First Transaction](../first-transaction/README.md)

## Security Reminders

**Critical Security Rules**:

1. **Never share your private key or mnemonic phrase** with anyone
2. **Always backup your mnemonic** in a secure location
3. **Use testnet for learning and development** until you're confident
4. **Encrypt all private keys** stored in databases or files
5. **Use HSM/KMS for production** environments
6. **Never commit keys to git** - use `.gitignore`
7. **Implement key rotation** policies for long-running services
8. **Test with small amounts** before moving to production
9. **Audit your security practices** regularly
10. **Have incident response plans** for key compromise

Your mnemonic phrase is the master key to all derived keys. Treat it like the password to your bank account - because it is!
