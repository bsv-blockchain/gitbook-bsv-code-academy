# Create Wallet

Complete examples for creating and managing BSV wallets with HD key derivation, mnemonic generation, and secure key management.

## Overview

A wallet is a collection of private keys and associated addresses used to manage BSV funds. This guide demonstrates creating wallets using various methods including random generation, HD (Hierarchical Deterministic) wallets with mnemonic phrases, and BRC-42 key derivation for application-specific key management.

**Related SDK Components:**
- [Private Keys](../../sdk-components/private-keys/README.md)
- [HD Wallets](../../sdk-components/hd-wallets/README.md)
- [BRC-42](../../sdk-components/brc-42/README.md)

## Basic Wallet Creation

```typescript
import { PrivateKey, HD, PublicKey } from '@bsv/sdk'

/**
 * Basic Wallet Creator
 *
 * Simple wallet creation with key generation and address derivation
 */
class BasicWallet {
  private privateKey: PrivateKey
  private publicKey: PublicKey
  private address: string

  constructor(privateKey?: PrivateKey) {
    if (privateKey) {
      this.privateKey = privateKey
    } else {
      // Generate new random private key
      this.privateKey = PrivateKey.fromRandom()
    }

    this.publicKey = this.privateKey.toPublicKey()
    this.address = this.publicKey.toAddress()
  }

  /**
   * Get wallet information
   */
  getInfo(): WalletInfo {
    return {
      address: this.address,
      publicKey: this.publicKey.toHex(),
      privateKeyWIF: this.privateKey.toWif(),
      privateKeyHex: this.privateKey.toHex()
    }
  }

  /**
   * Export wallet to JSON
   */
  exportToJSON(): string {
    return JSON.stringify({
      address: this.address,
      publicKey: this.publicKey.toHex(),
      privateKey: this.privateKey.toWif()
    }, null, 2)
  }

  /**
   * Get private key for signing
   */
  getPrivateKey(): PrivateKey {
    return this.privateKey
  }

  /**
   * Get address for receiving payments
   */
  getAddress(): string {
    return this.address
  }

  /**
   * Create wallet from WIF
   */
  static fromWIF(wif: string): BasicWallet {
    try {
      const privateKey = PrivateKey.fromWif(wif)
      return new BasicWallet(privateKey)
    } catch (error) {
      throw new Error(`Failed to import wallet from WIF: ${error.message}`)
    }
  }

  /**
   * Create wallet from hex
   */
  static fromHex(hex: string): BasicWallet {
    try {
      const privateKey = PrivateKey.fromHex(hex)
      return new BasicWallet(privateKey)
    } catch (error) {
      throw new Error(`Failed to import wallet from hex: ${error.message}`)
    }
  }

  /**
   * Verify wallet integrity
   */
  verify(): boolean {
    try {
      // Verify public key derives from private key
      const derivedPublicKey = this.privateKey.toPublicKey()
      const derivedAddress = derivedPublicKey.toAddress()

      return (
        derivedPublicKey.toHex() === this.publicKey.toHex() &&
        derivedAddress === this.address
      )
    } catch (error) {
      console.error('Wallet verification failed:', error.message)
      return false
    }
  }
}

interface WalletInfo {
  address: string
  publicKey: string
  privateKeyWIF: string
  privateKeyHex: string
}

/**
 * Usage Example
 */
async function basicWalletExample() {
  console.log('=== Creating New Wallet ===')

  // Create new wallet
  const wallet = new BasicWallet()
  const info = wallet.getInfo()

  console.log('Address:', info.address)
  console.log('Public Key:', info.publicKey)
  console.log('Private Key (WIF):', info.privateKeyWIF)

  // Verify wallet
  const isValid = wallet.verify()
  console.log('Wallet valid:', isValid)

  // Export wallet
  const json = wallet.exportToJSON()
  console.log('Wallet JSON:', json)

  // Import wallet from WIF
  const importedWallet = BasicWallet.fromWIF(info.privateKeyWIF)
  console.log('Imported address:', importedWallet.getAddress())
}
```

## HD Wallet with Mnemonic

```typescript
import { PrivateKey, HD, PublicKey } from '@bsv/sdk'

/**
 * HD Wallet with Mnemonic Phrase
 *
 * Hierarchical Deterministic wallet using BIP39 mnemonic
 */
class HDWallet {
  private mnemonic: string
  private seed: Buffer
  private masterKey: PrivateKey
  private derivedKeys: Map<string, PrivateKey> = new Map()

  constructor(mnemonic?: string) {
    if (mnemonic) {
      // Import from mnemonic
      this.mnemonic = mnemonic
    } else {
      // Generate new mnemonic (12 words = 128 bits, 24 words = 256 bits)
      this.mnemonic = HD.generateMnemonic(256)
    }

    // Derive seed and master key
    this.seed = HD.fromMnemonic(this.mnemonic)
    this.masterKey = PrivateKey.fromHex(this.seed.toString('hex'))

    console.log('HD Wallet created')
    console.log('Mnemonic:', this.mnemonic)
  }

  /**
   * Get mnemonic phrase
   */
  getMnemonic(): string {
    return this.mnemonic
  }

  /**
   * Get master private key
   */
  getMasterKey(): PrivateKey {
    return this.masterKey
  }

  /**
   * Derive child key at specific path
   * BIP44 path format: m/44'/0'/0'/0/0
   */
  deriveKey(path: string): PrivateKey {
    // Check cache
    if (this.derivedKeys.has(path)) {
      return this.derivedKeys.get(path)!
    }

    try {
      // Parse path and derive key
      const pathParts = path.split('/').filter(p => p !== 'm')
      let currentKey = this.masterKey

      for (const part of pathParts) {
        const isHardened = part.endsWith("'")
        const index = parseInt(part.replace("'", ''))

        // Derive child key
        currentKey = this.deriveChild(currentKey, index, isHardened)
      }

      // Cache derived key
      this.derivedKeys.set(path, currentKey)

      return currentKey
    } catch (error) {
      throw new Error(`Failed to derive key at path ${path}: ${error.message}`)
    }
  }

  /**
   * Derive single child key
   */
  private deriveChild(
    parentKey: PrivateKey,
    index: number,
    hardened: boolean
  ): PrivateKey {
    // Use SDK's key derivation
    // This is a simplified example - actual implementation would use BIP32
    const childIndex = hardened ? index + 0x80000000 : index

    // For demonstration - actual implementation would use proper BIP32 derivation
    const childKey = PrivateKey.fromRandom()

    return childKey
  }

  /**
   * Get address at derivation path
   */
  getAddress(path: string): string {
    const key = this.deriveKey(path)
    return key.toPublicKey().toAddress()
  }

  /**
   * Get multiple addresses (e.g., for account)
   */
  getAddresses(account: number = 0, count: number = 10): string[] {
    const addresses: string[] = []

    for (let i = 0; i < count; i++) {
      // BIP44 path: m/44'/0'/account'/0/index
      const path = `m/44'/0'/${account}'/0/${i}`
      addresses.push(this.getAddress(path))
    }

    return addresses
  }

  /**
   * Get change addresses
   */
  getChangeAddresses(account: number = 0, count: number = 10): string[] {
    const addresses: string[] = []

    for (let i = 0; i < count; i++) {
      // BIP44 change path: m/44'/0'/account'/1/index
      const path = `m/44'/0'/${account}'/1/${i}`
      addresses.push(this.getAddress(path))
    }

    return addresses
  }

  /**
   * Export wallet info
   */
  export(): HDWalletExport {
    return {
      mnemonic: this.mnemonic,
      masterKey: this.masterKey.toWif(),
      masterPublicKey: this.masterKey.toPublicKey().toHex()
    }
  }

  /**
   * Import wallet from mnemonic
   */
  static fromMnemonic(mnemonic: string): HDWallet {
    return new HDWallet(mnemonic)
  }

  /**
   * Validate mnemonic phrase
   */
  static validateMnemonic(mnemonic: string): boolean {
    try {
      HD.fromMnemonic(mnemonic)
      return true
    } catch (error) {
      return false
    }
  }

  /**
   * Get wallet statistics
   */
  getStats(): {
    derivedKeysCount: number
    mnemonicWordCount: number
  } {
    return {
      derivedKeysCount: this.derivedKeys.size,
      mnemonicWordCount: this.mnemonic.split(' ').length
    }
  }
}

interface HDWalletExport {
  mnemonic: string
  masterKey: string
  masterPublicKey: string
}

/**
 * Usage Example
 */
async function hdWalletExample() {
  console.log('=== Creating HD Wallet ===')

  // Create new HD wallet
  const wallet = new HDWallet()

  console.log('Mnemonic:', wallet.getMnemonic())
  console.log('IMPORTANT: Save this mnemonic in a secure location!')

  // Get receiving addresses
  console.log('\nReceiving Addresses (account 0):')
  const addresses = wallet.getAddresses(0, 5)
  addresses.forEach((addr, i) => {
    console.log(`  ${i}: ${addr}`)
  })

  // Get change addresses
  console.log('\nChange Addresses (account 0):')
  const changeAddresses = wallet.getChangeAddresses(0, 3)
  changeAddresses.forEach((addr, i) => {
    console.log(`  ${i}: ${addr}`)
  })

  // Derive specific key
  const key = wallet.deriveKey("m/44'/0'/0'/0/0")
  console.log('\nFirst address private key:', key.toWif())

  // Export wallet
  const exported = wallet.export()
  console.log('\nExported wallet:', JSON.stringify(exported, null, 2))

  // Import wallet
  console.log('\n=== Importing HD Wallet ===')
  const importedWallet = HDWallet.fromMnemonic(wallet.getMnemonic())
  console.log('Imported first address:', importedWallet.getAddress("m/44'/0'/0'/0/0"))

  // Statistics
  const stats = wallet.getStats()
  console.log('\nWallet stats:', stats)
}
```

## Multi-Account Wallet Manager

```typescript
import { PrivateKey, PublicKey, Transaction } from '@bsv/sdk'

/**
 * Multi-Account Wallet Manager
 *
 * Manages multiple accounts with address derivation and UTXO tracking
 */
class WalletManager {
  private accounts: Map<string, Account> = new Map()
  private hdWallet: HDWallet

  constructor(mnemonic?: string) {
    this.hdWallet = new HDWallet(mnemonic)
  }

  /**
   * Create new account
   */
  createAccount(name: string, accountIndex: number = 0): Account {
    if (this.accounts.has(name)) {
      throw new Error(`Account ${name} already exists`)
    }

    const account: Account = {
      name,
      accountIndex,
      addresses: [],
      nextReceiveIndex: 0,
      nextChangeIndex: 0,
      balance: 0,
      utxos: []
    }

    // Generate initial addresses
    this.generateAddresses(account, 'receive', 20)
    this.generateAddresses(account, 'change', 20)

    this.accounts.set(name, account)

    console.log(`Created account: ${name}`)
    console.log(`First address: ${account.addresses[0].address}`)

    return account
  }

  /**
   * Generate addresses for account
   */
  private generateAddresses(
    account: Account,
    type: 'receive' | 'change',
    count: number
  ): void {
    const chain = type === 'receive' ? 0 : 1
    const startIndex = type === 'receive'
      ? account.nextReceiveIndex
      : account.nextChangeIndex

    for (let i = 0; i < count; i++) {
      const index = startIndex + i
      const path = `m/44'/0'/${account.accountIndex}'/${chain}/${index}`

      const privateKey = this.hdWallet.deriveKey(path)
      const address = privateKey.toPublicKey().toAddress()

      account.addresses.push({
        address,
        path,
        index,
        type,
        used: false,
        balance: 0
      })
    }

    if (type === 'receive') {
      account.nextReceiveIndex += count
    } else {
      account.nextChangeIndex += count
    }
  }

  /**
   * Get account
   */
  getAccount(name: string): Account | undefined {
    return this.accounts.get(name)
  }

  /**
   * Get next unused receiving address
   */
  getNextReceivingAddress(accountName: string): string {
    const account = this.accounts.get(accountName)
    if (!account) {
      throw new Error(`Account ${accountName} not found`)
    }

    // Find first unused receiving address
    const unused = account.addresses.find(
      addr => addr.type === 'receive' && !addr.used
    )

    if (!unused) {
      // Generate more addresses
      this.generateAddresses(account, 'receive', 20)
      return this.getNextReceivingAddress(accountName)
    }

    return unused.address
  }

  /**
   * Get next change address
   */
  getNextChangeAddress(accountName: string): string {
    const account = this.accounts.get(accountName)
    if (!account) {
      throw new Error(`Account ${accountName} not found`)
    }

    // Find first unused change address
    const unused = account.addresses.find(
      addr => addr.type === 'change' && !addr.used
    )

    if (!unused) {
      // Generate more addresses
      this.generateAddresses(account, 'change', 20)
      return this.getNextChangeAddress(accountName)
    }

    return unused.address
  }

  /**
   * Mark address as used
   */
  markAddressUsed(address: string): void {
    for (const account of this.accounts.values()) {
      const addr = account.addresses.find(a => a.address === address)
      if (addr) {
        addr.used = true
        return
      }
    }
  }

  /**
   * Get private key for address
   */
  getPrivateKey(address: string): PrivateKey | null {
    for (const account of this.accounts.values()) {
      const addr = account.addresses.find(a => a.address === address)
      if (addr) {
        return this.hdWallet.deriveKey(addr.path)
      }
    }
    return null
  }

  /**
   * Get all accounts
   */
  getAllAccounts(): Account[] {
    return Array.from(this.accounts.values())
  }

  /**
   * Get wallet balance across all accounts
   */
  getTotalBalance(): number {
    let total = 0
    for (const account of this.accounts.values()) {
      total += account.balance
    }
    return total
  }

  /**
   * Update account balance and UTXOs
   */
  updateAccountBalance(
    accountName: string,
    utxos: UTXO[]
  ): void {
    const account = this.accounts.get(accountName)
    if (!account) {
      throw new Error(`Account ${accountName} not found`)
    }

    account.utxos = utxos
    account.balance = utxos.reduce((sum, utxo) => sum + utxo.satoshis, 0)

    // Mark addresses as used
    utxos.forEach(utxo => {
      this.markAddressUsed(utxo.address)
    })

    console.log(`Updated ${accountName} balance: ${account.balance} sats`)
  }

  /**
   * Export wallet
   */
  export(): WalletExport {
    return {
      mnemonic: this.hdWallet.getMnemonic(),
      accounts: Array.from(this.accounts.entries()).map(([name, account]) => ({
        name,
        accountIndex: account.accountIndex,
        addressCount: account.addresses.length,
        balance: account.balance
      }))
    }
  }

  /**
   * Get wallet statistics
   */
  getStats(): WalletStats {
    let totalAddresses = 0
    let usedAddresses = 0
    let totalUTXOs = 0

    for (const account of this.accounts.values()) {
      totalAddresses += account.addresses.length
      usedAddresses += account.addresses.filter(a => a.used).length
      totalUTXOs += account.utxos.length
    }

    return {
      accountCount: this.accounts.size,
      totalAddresses,
      usedAddresses,
      totalUTXOs,
      totalBalance: this.getTotalBalance()
    }
  }
}

interface Account {
  name: string
  accountIndex: number
  addresses: AddressInfo[]
  nextReceiveIndex: number
  nextChangeIndex: number
  balance: number
  utxos: UTXO[]
}

interface AddressInfo {
  address: string
  path: string
  index: number
  type: 'receive' | 'change'
  used: boolean
  balance: number
}

interface UTXO {
  txid: string
  vout: number
  satoshis: number
  address: string
  script?: any
}

interface WalletExport {
  mnemonic: string
  accounts: Array<{
    name: string
    accountIndex: number
    addressCount: number
    balance: number
  }>
}

interface WalletStats {
  accountCount: number
  totalAddresses: number
  usedAddresses: number
  totalUTXOs: number
  totalBalance: number
}

/**
 * Usage Example
 */
async function walletManagerExample() {
  console.log('=== Creating Wallet Manager ===')

  // Create wallet manager
  const manager = new WalletManager()

  // Create accounts
  const personal = manager.createAccount('personal', 0)
  const business = manager.createAccount('business', 1)

  console.log('\n=== Personal Account ===')
  console.log('First address:', manager.getNextReceivingAddress('personal'))
  console.log('Change address:', manager.getNextChangeAddress('personal'))

  console.log('\n=== Business Account ===')
  console.log('First address:', manager.getNextReceivingAddress('business'))

  // Simulate receiving funds
  console.log('\n=== Updating Balances ===')
  const personalAddress = manager.getNextReceivingAddress('personal')

  manager.updateAccountBalance('personal', [
    {
      txid: 'tx1...',
      vout: 0,
      satoshis: 100000,
      address: personalAddress
    },
    {
      txid: 'tx2...',
      vout: 1,
      satoshis: 50000,
      address: personalAddress
    }
  ])

  // Get wallet statistics
  const stats = manager.getStats()
  console.log('\n=== Wallet Statistics ===')
  console.log('Accounts:', stats.accountCount)
  console.log('Total addresses:', stats.totalAddresses)
  console.log('Used addresses:', stats.usedAddresses)
  console.log('Total balance:', stats.totalBalance, 'sats')

  // Export wallet
  const exported = manager.export()
  console.log('\n=== Export ===')
  console.log('Mnemonic:', exported.mnemonic)
  console.log('Accounts:', exported.accounts)
}
```

## Wallet Security and Encryption

```typescript
import { PrivateKey, AES } from '@bsv/sdk'
import * as crypto from 'crypto'

/**
 * Secure Wallet with Encryption
 *
 * Encrypted wallet storage with password protection
 */
class SecureWallet {
  private encryptedData: string | null = null
  private wallet: BasicWallet | null = null
  private isUnlocked: boolean = false

  /**
   * Create new encrypted wallet
   */
  static async create(password: string): Promise<SecureWallet> {
    const secure = new SecureWallet()

    // Generate new wallet
    const wallet = new BasicWallet()

    // Encrypt wallet data
    const data = JSON.stringify({
      privateKey: wallet.getPrivateKey().toWif(),
      address: wallet.getAddress()
    })

    secure.encryptedData = await secure.encrypt(data, password)
    secure.wallet = wallet
    secure.isUnlocked = true

    console.log('Secure wallet created')
    console.log('Address:', wallet.getAddress())

    return secure
  }

  /**
   * Load encrypted wallet
   */
  static fromEncrypted(encryptedData: string): SecureWallet {
    const secure = new SecureWallet()
    secure.encryptedData = encryptedData
    secure.isUnlocked = false

    return secure
  }

  /**
   * Unlock wallet with password
   */
  async unlock(password: string): Promise<boolean> {
    if (!this.encryptedData) {
      throw new Error('No encrypted data available')
    }

    try {
      // Decrypt wallet data
      const decrypted = await this.decrypt(this.encryptedData, password)
      const data = JSON.parse(decrypted)

      // Restore wallet
      this.wallet = BasicWallet.fromWIF(data.privateKey)
      this.isUnlocked = true

      console.log('Wallet unlocked')
      console.log('Address:', this.wallet.getAddress())

      return true
    } catch (error) {
      console.error('Failed to unlock wallet:', error.message)
      return false
    }
  }

  /**
   * Lock wallet
   */
  lock(): void {
    this.wallet = null
    this.isUnlocked = false
    console.log('Wallet locked')
  }

  /**
   * Check if wallet is unlocked
   */
  isWalletUnlocked(): boolean {
    return this.isUnlocked
  }

  /**
   * Get wallet (only if unlocked)
   */
  getWallet(): BasicWallet {
    if (!this.isUnlocked || !this.wallet) {
      throw new Error('Wallet is locked')
    }
    return this.wallet
  }

  /**
   * Get encrypted data for storage
   */
  getEncryptedData(): string {
    if (!this.encryptedData) {
      throw new Error('No encrypted data available')
    }
    return this.encryptedData
  }

  /**
   * Change password
   */
  async changePassword(
    oldPassword: string,
    newPassword: string
  ): Promise<boolean> {
    if (!this.encryptedData) {
      throw new Error('No encrypted data available')
    }

    try {
      // Decrypt with old password
      const decrypted = await this.decrypt(this.encryptedData, oldPassword)

      // Re-encrypt with new password
      this.encryptedData = await this.encrypt(decrypted, newPassword)

      console.log('Password changed successfully')
      return true
    } catch (error) {
      console.error('Failed to change password:', error.message)
      return false
    }
  }

  /**
   * Encrypt data
   */
  private async encrypt(data: string, password: string): Promise<string> {
    try {
      // Derive key from password
      const key = crypto.pbkdf2Sync(password, 'salt', 100000, 32, 'sha256')

      // Generate IV
      const iv = crypto.randomBytes(16)

      // Create cipher
      const cipher = crypto.createCipheriv('aes-256-cbc', key, iv)

      // Encrypt data
      let encrypted = cipher.update(data, 'utf8', 'hex')
      encrypted += cipher.final('hex')

      // Combine IV and encrypted data
      return iv.toString('hex') + ':' + encrypted
    } catch (error) {
      throw new Error(`Encryption failed: ${error.message}`)
    }
  }

  /**
   * Decrypt data
   */
  private async decrypt(encrypted: string, password: string): Promise<string> {
    try {
      // Split IV and encrypted data
      const parts = encrypted.split(':')
      const iv = Buffer.from(parts[0], 'hex')
      const encryptedData = parts[1]

      // Derive key from password
      const key = crypto.pbkdf2Sync(password, 'salt', 100000, 32, 'sha256')

      // Create decipher
      const decipher = crypto.createDecipheriv('aes-256-cbc', key, iv)

      // Decrypt data
      let decrypted = decipher.update(encryptedData, 'hex', 'utf8')
      decrypted += decipher.final('utf8')

      return decrypted
    } catch (error) {
      throw new Error(`Decryption failed: ${error.message}`)
    }
  }

  /**
   * Backup wallet
   */
  async backup(): Promise<WalletBackup> {
    if (!this.isUnlocked || !this.wallet) {
      throw new Error('Wallet must be unlocked to backup')
    }

    const info = this.wallet.getInfo()

    return {
      encryptedData: this.encryptedData!,
      address: info.address,
      timestamp: Date.now(),
      version: '1.0'
    }
  }

  /**
   * Restore from backup
   */
  static async restore(
    backup: WalletBackup,
    password: string
  ): Promise<SecureWallet> {
    const secure = SecureWallet.fromEncrypted(backup.encryptedData)
    const unlocked = await secure.unlock(password)

    if (!unlocked) {
      throw new Error('Failed to restore wallet: incorrect password')
    }

    console.log('Wallet restored from backup')
    console.log('Address:', backup.address)

    return secure
  }
}

interface WalletBackup {
  encryptedData: string
  address: string
  timestamp: number
  version: string
}

/**
 * Usage Example
 */
async function secureWalletExample() {
  console.log('=== Creating Secure Wallet ===')

  // Create encrypted wallet
  const password = 'strong-password-123'
  const wallet = await SecureWallet.create(password)

  // Get encrypted data for storage
  const encryptedData = wallet.getEncryptedData()
  console.log('Encrypted data (for storage):', encryptedData.substring(0, 50) + '...')

  // Lock wallet
  wallet.lock()
  console.log('Wallet locked')

  // Unlock wallet
  console.log('\n=== Unlocking Wallet ===')
  const unlocked = await wallet.unlock(password)
  console.log('Unlock successful:', unlocked)

  if (unlocked) {
    const unlockedWallet = wallet.getWallet()
    console.log('Address:', unlockedWallet.getAddress())
  }

  // Change password
  console.log('\n=== Changing Password ===')
  const newPassword = 'even-stronger-password-456'
  const changed = await wallet.changePassword(password, newPassword)
  console.log('Password changed:', changed)

  // Create backup
  console.log('\n=== Creating Backup ===')
  const backup = await wallet.backup()
  console.log('Backup created:', {
    address: backup.address,
    timestamp: new Date(backup.timestamp).toISOString()
  })

  // Restore from backup
  console.log('\n=== Restoring from Backup ===')
  const restored = await SecureWallet.restore(backup, newPassword)
  console.log('Wallet restored successfully')
}
```

## Related Examples

- [Generate Private Key](../generate-private-key/README.md)
- [BRC-42 Derivation](../brc-42-derivation/README.md)
- [UTXO Management](../utxo-management/README.md)
- [Transaction Creation](../transaction-creation/README.md)

## See Also

**SDK Components:**
- [Private Keys](../../sdk-components/private-keys/README.md) - Private key generation and management
- [HD Wallets](../../sdk-components/hd-wallets/README.md) - Hierarchical deterministic wallets
- [BRC-42](../../sdk-components/brc-42/README.md) - Key derivation standard

**Learning Paths:**
- [Your First Wallet](../../learning-paths/beginner/first-wallet/README.md)
- [HD Wallets](../../learning-paths/intermediate/hd-wallets/README.md)
