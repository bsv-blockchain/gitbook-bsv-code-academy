# Content Paywall with Microtransactions

Complete examples for implementing paywalled content with BSV microtransactions, including pay-per-view, subscriptions, and metered access.

## Overview

BSV's low transaction fees make microtransactions economically viable, enabling new monetization models for digital content. This guide covers pay-per-view content, token-based access, time-limited access, metered consumption, and subscription tiers. These patterns work for articles, videos, APIs, and any digital resource.

**Related SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md)
- [P2PKH](../../sdk-components/p2pkh/README.md)
- [Script](../../sdk-components/script/README.md)
- [OP_RETURN](../../sdk-components/script/README.md)

## Pay-Per-View Content

```typescript
import { Transaction, PrivateKey, P2PKH, Script, OP, Hash } from '@bsv/sdk'

/**
 * Pay-Per-View Content Manager
 *
 * Manage single payment access to content
 */
class PayPerViewManager {
  private contentPublisher: PrivateKey
  private accessTokens: Map<string, AccessToken> = new Map()

  constructor(publisherKey: PrivateKey) {
    this.contentPublisher = publisherKey
  }

  /**
   * Purchase content access
   */
  async purchaseAccess(params: {
    userKey: PrivateKey
    contentId: string
    price: number
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  }): Promise<{
    tx: Transaction
    accessToken: AccessToken
  }> {
    try {
      console.log('Processing content purchase')
      console.log('Content ID:', params.contentId)
      console.log('Price:', params.price, 'satoshis')

      const tx = new Transaction()

      // Add user's payment input
      tx.addInput({
        sourceTXID: params.utxo.txid,
        sourceOutputIndex: params.utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.userKey),
        sequence: 0xffffffff
      })

      // Payment output to publisher
      tx.addOutput({
        satoshis: params.price,
        lockingScript: new P2PKH().lock(this.contentPublisher.toPublicKey().toHash())
      })

      // OP_RETURN with access proof
      const accessProof = this.createAccessProof(
        params.contentId,
        params.userKey.toPublicKey().toAddress()
      )

      const opReturnScript = new Script()
      opReturnScript.writeOpCode(OP.OP_FALSE)
      opReturnScript.writeOpCode(OP.OP_RETURN)
      opReturnScript.writeBin(Buffer.from('CONTENT_ACCESS', 'utf8'))
      opReturnScript.writeBin(Buffer.from(params.contentId, 'utf8'))
      opReturnScript.writeBin(Buffer.from(accessProof, 'hex'))

      tx.addOutput({
        satoshis: 0,
        lockingScript: opReturnScript
      })

      // Change output
      const fee = 500
      const change = params.utxo.satoshis - params.price - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(params.userKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      // Generate access token
      const accessToken: AccessToken = {
        tokenId: this.generateTokenId(),
        contentId: params.contentId,
        userAddress: params.userKey.toPublicKey().toAddress(),
        paymentTxid: tx.id('hex'),
        accessProof,
        createdAt: Date.now(),
        expiresAt: Date.now() + 30 * 24 * 60 * 60 * 1000, // 30 days
        status: 'active'
      }

      this.accessTokens.set(accessToken.tokenId, accessToken)

      console.log('Access purchased successfully')
      console.log('Access token:', accessToken.tokenId)

      return { tx, accessToken }
    } catch (error) {
      throw new Error(`Purchase failed: ${error.message}`)
    }
  }

  /**
   * Verify access token
   */
  verifyAccess(tokenId: string, contentId: string): AccessVerification {
    const token = this.accessTokens.get(tokenId)

    if (!token) {
      return {
        valid: false,
        reason: 'Token not found'
      }
    }

    if (token.contentId !== contentId) {
      return {
        valid: false,
        reason: 'Token not valid for this content'
      }
    }

    if (token.status !== 'active') {
      return {
        valid: false,
        reason: `Token status: ${token.status}`
      }
    }

    if (Date.now() > token.expiresAt) {
      token.status = 'expired'
      return {
        valid: false,
        reason: 'Token expired'
      }
    }

    return {
      valid: true,
      token
    }
  }

  /**
   * Revoke access
   */
  revokeAccess(tokenId: string): void {
    const token = this.accessTokens.get(tokenId)
    if (token) {
      token.status = 'revoked'
      console.log('Access revoked:', tokenId)
    }
  }

  /**
   * Create access proof
   */
  private createAccessProof(contentId: string, userAddress: string): string {
    const data = `${contentId}:${userAddress}:${Date.now()}`
    return Hash.sha256(Buffer.from(data, 'utf8')).toString('hex')
  }

  /**
   * Generate token ID
   */
  private generateTokenId(): string {
    return `TOKEN-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
  }
}

interface AccessToken {
  tokenId: string
  contentId: string
  userAddress: string
  paymentTxid: string
  accessProof: string
  createdAt: number
  expiresAt: number
  status: 'active' | 'expired' | 'revoked'
}

interface AccessVerification {
  valid: boolean
  token?: AccessToken
  reason?: string
}

/**
 * Usage Example
 */
async function payPerViewExample() {
  const publisher = PrivateKey.fromRandom()
  const manager = new PayPerViewManager(publisher)

  const user = PrivateKey.fromRandom()

  // User purchases access
  const { accessToken } = await manager.purchaseAccess({
    userKey: user,
    contentId: 'ARTICLE-001',
    price: 1000, // 1000 satoshis
    utxo: {
      txid: 'user-utxo...',
      vout: 0,
      satoshis: 10000,
      script: new P2PKH().lock(user.toPublicKey().toHash())
    }
  })

  console.log('Access token:', accessToken)

  // Verify access
  const verification = manager.verifyAccess(accessToken.tokenId, 'ARTICLE-001')

  if (verification.valid) {
    console.log('Access granted!')
    // Deliver content...
  } else {
    console.log('Access denied:', verification.reason)
  }
}
```

## Metered Content Access

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Metered Access Manager
 *
 * Pay for content consumption by usage (views, API calls, etc.)
 */
class MeteredAccessManager {
  private accounts: Map<string, MeteredAccount> = new Map()
  private publisherKey: PrivateKey

  constructor(publisherKey: PrivateKey) {
    this.publisherKey = publisherKey
  }

  /**
   * Create metered account
   */
  async createAccount(params: {
    userKey: PrivateKey
    initialCredits: number
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  }): Promise<{
    account: MeteredAccount
    tx: Transaction
  }> {
    try {
      console.log('Creating metered account')
      console.log('Initial credits:', params.initialCredits)

      const tx = new Transaction()

      tx.addInput({
        sourceTXID: params.utxo.txid,
        sourceOutputIndex: params.utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.userKey),
        sequence: 0xffffffff
      })

      // Payment for credits
      const creditPrice = 10 // 10 satoshis per credit
      const totalPrice = params.initialCredits * creditPrice

      tx.addOutput({
        satoshis: totalPrice,
        lockingScript: new P2PKH().lock(this.publisherKey.toPublicKey().toHash())
      })

      // Change
      const fee = 500
      const change = params.utxo.satoshis - totalPrice - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(params.userKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      const account: MeteredAccount = {
        accountId: this.generateAccountId(),
        userAddress: params.userKey.toPublicKey().toAddress(),
        credits: params.initialCredits,
        totalSpent: 0,
        createdAt: Date.now(),
        lastTopUpTxid: tx.id('hex'),
        usageHistory: []
      }

      this.accounts.set(account.accountId, account)

      console.log('Account created:', account.accountId)

      return { account, tx }
    } catch (error) {
      throw new Error(`Account creation failed: ${error.message}`)
    }
  }

  /**
   * Top up account credits
   */
  async topUpCredits(params: {
    accountId: string
    userKey: PrivateKey
    additionalCredits: number
    utxo: any
  }): Promise<Transaction> {
    try {
      const account = this.accounts.get(params.accountId)
      if (!account) {
        throw new Error('Account not found')
      }

      console.log('Topping up account:', params.accountId)
      console.log('Additional credits:', params.additionalCredits)

      const tx = new Transaction()

      tx.addInput({
        sourceTXID: params.utxo.txid,
        sourceOutputIndex: params.utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.userKey),
        sequence: 0xffffffff
      })

      const creditPrice = 10
      const totalPrice = params.additionalCredits * creditPrice

      tx.addOutput({
        satoshis: totalPrice,
        lockingScript: new P2PKH().lock(this.publisherKey.toPublicKey().toHash())
      })

      const fee = 500
      const change = params.utxo.satoshis - totalPrice - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(params.userKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      // Update account
      account.credits += params.additionalCredits
      account.lastTopUpTxid = tx.id('hex')

      console.log('Credits topped up')
      console.log('New balance:', account.credits)

      return tx
    } catch (error) {
      throw new Error(`Top-up failed: ${error.message}`)
    }
  }

  /**
   * Consume credits for content access
   */
  consumeCredits(params: {
    accountId: string
    contentId: string
    creditCost: number
  }): ConsumptionResult {
    const account = this.accounts.get(params.accountId)

    if (!account) {
      return {
        success: false,
        reason: 'Account not found'
      }
    }

    if (account.credits < params.creditCost) {
      return {
        success: false,
        reason: 'Insufficient credits',
        currentBalance: account.credits,
        required: params.creditCost
      }
    }

    // Deduct credits
    account.credits -= params.creditCost
    account.totalSpent += params.creditCost

    // Record usage
    account.usageHistory.push({
      contentId: params.contentId,
      creditsUsed: params.creditCost,
      timestamp: Date.now()
    })

    console.log('Credits consumed')
    console.log('Content:', params.contentId)
    console.log('Cost:', params.creditCost)
    console.log('Remaining:', account.credits)

    return {
      success: true,
      remainingCredits: account.credits
    }
  }

  /**
   * Get account balance
   */
  getBalance(accountId: string): number {
    const account = this.accounts.get(accountId)
    return account ? account.credits : 0
  }

  /**
   * Get usage statistics
   */
  getUsageStats(accountId: string): UsageStats {
    const account = this.accounts.get(accountId)

    if (!account) {
      throw new Error('Account not found')
    }

    return {
      totalSpent: account.totalSpent,
      currentBalance: account.credits,
      totalAccessed: account.usageHistory.length,
      recentUsage: account.usageHistory.slice(-10)
    }
  }

  /**
   * Generate account ID
   */
  private generateAccountId(): string {
    return `ACCT-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
  }
}

interface MeteredAccount {
  accountId: string
  userAddress: string
  credits: number
  totalSpent: number
  createdAt: number
  lastTopUpTxid: string
  usageHistory: UsageRecord[]
}

interface UsageRecord {
  contentId: string
  creditsUsed: number
  timestamp: number
}

interface ConsumptionResult {
  success: boolean
  remainingCredits?: number
  reason?: string
  currentBalance?: number
  required?: number
}

interface UsageStats {
  totalSpent: number
  currentBalance: number
  totalAccessed: number
  recentUsage: UsageRecord[]
}

/**
 * Usage Example
 */
async function meteredAccessExample() {
  const publisher = PrivateKey.fromRandom()
  const manager = new MeteredAccessManager(publisher)

  const user = PrivateKey.fromRandom()

  // Create account with initial credits
  const { account } = await manager.createAccount({
    userKey: user,
    initialCredits: 100,
    utxo: {
      txid: 'user-utxo...',
      vout: 0,
      satoshis: 10000,
      script: new P2PKH().lock(user.toPublicKey().toHash())
    }
  })

  console.log('Account created:', account)

  // Consume credits for content
  const result = manager.consumeCredits({
    accountId: account.accountId,
    contentId: 'VIDEO-001',
    creditCost: 5
  })

  if (result.success) {
    console.log('Access granted!')
    console.log('Remaining credits:', result.remainingCredits)
  }

  // Get usage stats
  const stats = manager.getUsageStats(account.accountId)
  console.log('Usage stats:', stats)
}
```

## Time-Limited Access

```typescript
import { Transaction, PrivateKey, P2PKH, Script } from '@bsv/sdk'

/**
 * Time-Limited Access Manager
 *
 * Grant access for specific time periods
 */
class TimeLimitedAccessManager {
  private passes: Map<string, TimePass> = new Map()
  private publisherKey: PrivateKey

  constructor(publisherKey: PrivateKey) {
    this.publisherKey = publisherKey
  }

  /**
   * Purchase time-limited pass
   */
  async purchasePass(params: {
    userKey: PrivateKey
    duration: number // milliseconds
    price: number
    utxo: any
  }): Promise<{
    pass: TimePass
    tx: Transaction
  }> {
    try {
      console.log('Purchasing time pass')
      console.log('Duration:', params.duration / 1000 / 60, 'minutes')
      console.log('Price:', params.price)

      const tx = new Transaction()

      tx.addInput({
        sourceTXID: params.utxo.txid,
        sourceOutputIndex: params.utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.userKey),
        sequence: 0xffffffff
      })

      tx.addOutput({
        satoshis: params.price,
        lockingScript: new P2PKH().lock(this.publisherKey.toPublicKey().toHash())
      })

      const fee = 500
      const change = params.utxo.satoshis - params.price - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(params.userKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      const pass: TimePass = {
        passId: this.generatePassId(),
        userAddress: params.userKey.toPublicKey().toAddress(),
        paymentTxid: tx.id('hex'),
        startTime: Date.now(),
        endTime: Date.now() + params.duration,
        duration: params.duration,
        status: 'active'
      }

      this.passes.set(pass.passId, pass)

      console.log('Pass purchased')
      console.log('Pass ID:', pass.passId)
      console.log('Valid until:', new Date(pass.endTime).toISOString())

      return { pass, tx }
    } catch (error) {
      throw new Error(`Pass purchase failed: ${error.message}`)
    }
  }

  /**
   * Verify pass is valid
   */
  verifyPass(passId: string): PassVerification {
    const pass = this.passes.get(passId)

    if (!pass) {
      return {
        valid: false,
        reason: 'Pass not found'
      }
    }

    if (pass.status !== 'active') {
      return {
        valid: false,
        reason: `Pass status: ${pass.status}`
      }
    }

    const now = Date.now()

    if (now < pass.startTime) {
      return {
        valid: false,
        reason: 'Pass not yet active'
      }
    }

    if (now > pass.endTime) {
      pass.status = 'expired'
      return {
        valid: false,
        reason: 'Pass expired'
      }
    }

    const remainingTime = pass.endTime - now

    return {
      valid: true,
      pass,
      remainingTime
    }
  }

  /**
   * Extend pass duration
   */
  async extendPass(params: {
    passId: string
    userKey: PrivateKey
    additionalDuration: number
    price: number
    utxo: any
  }): Promise<Transaction> {
    try {
      const pass = this.passes.get(params.passId)
      if (!pass) {
        throw new Error('Pass not found')
      }

      console.log('Extending pass:', params.passId)
      console.log('Additional duration:', params.additionalDuration / 1000 / 60, 'minutes')

      const tx = new Transaction()

      tx.addInput({
        sourceTXID: params.utxo.txid,
        sourceOutputIndex: params.utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(params.userKey),
        sequence: 0xffffffff
      })

      tx.addOutput({
        satoshis: params.price,
        lockingScript: new P2PKH().lock(this.publisherKey.toPublicKey().toHash())
      })

      const fee = 500
      const change = params.utxo.satoshis - params.price - fee

      if (change >= 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(params.userKey.toPublicKey().toHash())
        })
      }

      await tx.sign()

      // Extend pass
      pass.endTime += params.additionalDuration
      pass.duration += params.additionalDuration

      console.log('Pass extended')
      console.log('New expiry:', new Date(pass.endTime).toISOString())

      return tx
    } catch (error) {
      throw new Error(`Pass extension failed: ${error.message}`)
    }
  }

  /**
   * Get all active passes for user
   */
  getUserPasses(userAddress: string): TimePass[] {
    return Array.from(this.passes.values())
      .filter(pass => pass.userAddress === userAddress && pass.status === 'active')
  }

  /**
   * Generate pass ID
   */
  private generatePassId(): string {
    return `PASS-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
  }
}

interface TimePass {
  passId: string
  userAddress: string
  paymentTxid: string
  startTime: number
  endTime: number
  duration: number
  status: 'active' | 'expired' | 'cancelled'
}

interface PassVerification {
  valid: boolean
  pass?: TimePass
  remainingTime?: number
  reason?: string
}

/**
 * Usage Example
 */
async function timeLimitedAccessExample() {
  const publisher = PrivateKey.fromRandom()
  const manager = new TimeLimitedAccessManager(publisher)

  const user = PrivateKey.fromRandom()

  // Purchase 1-hour pass
  const { pass } = await manager.purchasePass({
    userKey: user,
    duration: 60 * 60 * 1000, // 1 hour
    price: 10000,
    utxo: {/* ... */}
  })

  console.log('Pass:', pass)

  // Verify pass
  const verification = manager.verifyPass(pass.passId)

  if (verification.valid) {
    console.log('Access granted!')
    console.log('Time remaining:', verification.remainingTime! / 1000 / 60, 'minutes')
  }

  // Extend pass
  await manager.extendPass({
    passId: pass.passId,
    userKey: user,
    additionalDuration: 30 * 60 * 1000, // 30 more minutes
    price: 5000,
    utxo: {/* ... */}
  })
}
```

## Related Examples

- [Payment Processing](../payment-processing/README.md)
- [OP_RETURN](../op-return/README.md)
- [E-commerce Integration](../ecommerce-integration/README.md)
- [Subscription Management](../payment-processing/README.md)

## See Also

**SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md) - Transaction building
- [P2PKH](../../sdk-components/p2pkh/README.md) - Standard payments
- [Script](../../sdk-components/script/README.md) - Script operations

**Learning Paths:**
- [Micropayments](../../learning-paths/intermediate/micropayments/README.md)
