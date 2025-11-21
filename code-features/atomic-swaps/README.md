# Atomic Swaps

Complete examples for implementing atomic swaps using Hash Time-Locked Contracts (HTLC) on the BSV blockchain, enabling trustless peer-to-peer exchanges.

## Overview

Atomic swaps allow two parties to exchange digital assets without requiring a trusted third party. Using Hash Time-Locked Contracts (HTLC), participants can either complete the swap or safely recover their funds if the counterparty fails to fulfill their obligations. This ensures atomicity: either both parties receive their assets or neither does.

**Related SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md)
- [Script](../../sdk-components/script/README.md)
- [Script Templates](../../sdk-components/script-templates/README.md)
- [Signatures](../../sdk-components/signatures/README.md)

## HTLC Script Builder

```typescript
import { Transaction, PrivateKey, PublicKey, Script, Hash, OP } from '@bsv/sdk'

/**
 * Hash Time-Locked Contract (HTLC) Builder
 *
 * Creates and manages HTLC scripts for atomic swaps.
 */
class HTLCBuilder {
  /**
   * Create an HTLC locking script
   *
   * The HTLC allows spending in two ways:
   * 1. With the secret preimage (before timeout) - for the recipient
   * 2. After timeout with refund key - for the sender
   *
   * @param recipientPubKey - Public key of the recipient
   * @param refundPubKey - Public key for refund (sender)
   * @param secretHash - Hash of the secret (SHA256)
   * @param lockTime - Unix timestamp for timeout
   * @returns HTLC locking script
   */
  createHTLCScript(
    recipientPubKey: PublicKey,
    refundPubKey: PublicKey,
    secretHash: string,
    lockTime: number
  ): Script {
    try {
      // HTLC Script:
      // IF
      //   OP_SHA256 <secretHash> OP_EQUALVERIFY <recipientPubKey> OP_CHECKSIG
      // ELSE
      //   <lockTime> OP_CHECKLOCKTIMEVERIFY OP_DROP <refundPubKey> OP_CHECKSIG
      // ENDIF

      const script = new Script()

      // IF branch (claim with secret)
      script.chunks.push({ op: OP.OP_IF })
      script.chunks.push({ op: OP.OP_SHA256 })
      script.chunks.push({ data: Buffer.from(secretHash, 'hex') })
      script.chunks.push({ op: OP.OP_EQUALVERIFY })
      script.chunks.push({ data: recipientPubKey.toBuffer() })
      script.chunks.push({ op: OP.OP_CHECKSIG })

      // ELSE branch (refund after timeout)
      script.chunks.push({ op: OP.OP_ELSE })
      script.chunks.push({ data: this.numberToBuffer(lockTime) })
      script.chunks.push({ op: OP.OP_CHECKLOCKTIMEVERIFY })
      script.chunks.push({ op: OP.OP_DROP })
      script.chunks.push({ data: refundPubKey.toBuffer() })
      script.chunks.push({ op: OP.OP_CHECKSIG })

      // End IF
      script.chunks.push({ op: OP.OP_ENDIF })

      return script
    } catch (error) {
      throw new Error(`HTLC script creation failed: ${error.message}`)
    }
  }

  /**
   * Create unlocking script to claim with secret
   */
  createClaimScript(
    secret: string,
    signature: Buffer,
    recipientPubKey: PublicKey
  ): Script {
    const script = new Script()

    // Push signature
    script.chunks.push({ data: signature })

    // Push secret preimage
    script.chunks.push({ data: Buffer.from(secret, 'hex') })

    // Push TRUE to select IF branch
    script.chunks.push({ op: OP.OP_TRUE })

    return script
  }

  /**
   * Create unlocking script for refund
   */
  createRefundScript(
    signature: Buffer,
    refundPubKey: PublicKey
  ): Script {
    const script = new Script()

    // Push signature
    script.chunks.push({ data: signature })

    // Push FALSE to select ELSE branch
    script.chunks.push({ op: OP.OP_FALSE })

    return script
  }

  /**
   * Generate a random secret
   */
  generateSecret(): { secret: string; hash: string } {
    const secret = Buffer.from(PrivateKey.fromRandom().toHex(), 'hex')
    const hash = Hash.sha256(secret)

    return {
      secret: secret.toString('hex'),
      hash: hash.toString('hex')
    }
  }

  /**
   * Convert number to buffer for script
   */
  private numberToBuffer(num: number): Buffer {
    if (num === 0) return Buffer.from([])
    if (num === -1) return Buffer.from([0x81])

    const isNegative = num < 0
    const absNum = Math.abs(num)
    const bytes: number[] = []

    let n = absNum
    while (n > 0) {
      bytes.push(n & 0xff)
      n >>= 8
    }

    if (bytes[bytes.length - 1] & 0x80) {
      bytes.push(isNegative ? 0x80 : 0x00)
    } else if (isNegative) {
      bytes[bytes.length - 1] |= 0x80
    }

    return Buffer.from(bytes)
  }
}

/**
 * Usage Example
 */
async function htlcExample() {
  const builder = new HTLCBuilder()

  // Generate keys
  const recipientKey = PrivateKey.fromRandom()
  const refundKey = PrivateKey.fromRandom()

  // Generate secret
  const { secret, hash } = builder.generateSecret()

  // Lock time: 24 hours from now
  const lockTime = Math.floor(Date.now() / 1000) + (24 * 60 * 60)

  // Create HTLC script
  const htlcScript = builder.createHTLCScript(
    recipientKey.toPublicKey(),
    refundKey.toPublicKey(),
    hash,
    lockTime
  )

  console.log('HTLC Script created')
  console.log('Secret hash:', hash)
  console.log('Lock time:', new Date(lockTime * 1000).toISOString())
}
```

## Atomic Swap Implementation

```typescript
import { Transaction, PrivateKey, PublicKey, Script, Hash } from '@bsv/sdk'

/**
 * Atomic Swap Manager
 *
 * Manages the full lifecycle of an atomic swap between two parties.
 */
class AtomicSwapManager {
  private htlcBuilder: HTLCBuilder

  constructor() {
    this.htlcBuilder = new HTLCBuilder()
  }

  /**
   * Initiate an atomic swap (Party A)
   *
   * @param partyA - Initiator's private key
   * @param partyBPubKey - Counterparty's public key
   * @param utxo - UTXO to lock in swap
   * @param lockTime - Timeout for swap (unix timestamp)
   * @returns Swap details including secret hash and transaction
   */
  async initiateSwap(
    partyA: PrivateKey,
    partyBPubKey: PublicKey,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    lockTime: number
  ): Promise<{
    secret: string
    secretHash: string
    transaction: Transaction
    htlcScript: Script
  }> {
    try {
      // Generate secret for this swap
      const { secret, hash } = this.htlcBuilder.generateSecret()

      // Create HTLC script
      // Party B can claim with secret, Party A can refund after timeout
      const htlcScript = this.htlcBuilder.createHTLCScript(
        partyBPubKey,
        partyA.toPublicKey(),
        hash,
        lockTime
      )

      // Create funding transaction
      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(partyA),
        sequence: 0xffffffff
      })

      // Lock funds in HTLC
      const fee = 500
      const htlcAmount = utxo.satoshis - fee

      tx.addOutput({
        satoshis: htlcAmount,
        lockingScript: htlcScript
      })

      await tx.sign()

      console.log('Swap initiated by Party A')
      console.log('Secret hash:', hash)
      console.log('HTLC funded with', htlcAmount, 'satoshis')

      return {
        secret,
        secretHash: hash,
        transaction: tx,
        htlcScript
      }
    } catch (error) {
      throw new Error(`Swap initiation failed: ${error.message}`)
    }
  }

  /**
   * Participate in atomic swap (Party B)
   *
   * Party B creates their HTLC using the same secret hash
   */
  async participateSwap(
    partyB: PrivateKey,
    partyAPubKey: PublicKey,
    secretHash: string,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    lockTime: number
  ): Promise<{
    transaction: Transaction
    htlcScript: Script
  }> {
    try {
      // Create HTLC with same secret hash
      // Party A can claim with secret, Party B can refund after timeout
      const htlcScript = this.htlcBuilder.createHTLCScript(
        partyAPubKey,
        partyB.toPublicKey(),
        secretHash,
        lockTime
      )

      // Create funding transaction
      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(partyB),
        sequence: 0xffffffff
      })

      const fee = 500
      const htlcAmount = utxo.satoshis - fee

      tx.addOutput({
        satoshis: htlcAmount,
        lockingScript: htlcScript
      })

      await tx.sign()

      console.log('Party B participated in swap')
      console.log('HTLC funded with', htlcAmount, 'satoshis')

      return {
        transaction: tx,
        htlcScript
      }
    } catch (error) {
      throw new Error(`Swap participation failed: ${error.message}`)
    }
  }

  /**
   * Claim funds from HTLC with secret
   *
   * @param claimant - Private key of the party claiming funds
   * @param secret - The secret preimage
   * @param htlcUTXO - UTXO locked in HTLC
   * @param htlcScript - The HTLC locking script
   * @returns Transaction claiming the funds
   */
  async claimHTLC(
    claimant: PrivateKey,
    secret: string,
    htlcUTXO: {
      txid: string
      vout: number
      satoshis: number
    },
    htlcScript: Script
  ): Promise<Transaction> {
    try {
      const tx = new Transaction()

      // Verify secret hash matches (in production, verify against HTLC script)
      const secretHash = Hash.sha256(Buffer.from(secret, 'hex'))

      tx.addInput({
        sourceTXID: htlcUTXO.txid,
        sourceOutputIndex: htlcUTXO.vout,
        unlockingScriptTemplate: {
          sign: async (tx: Transaction, inputIndex: number) => {
            // Create signature
            const signature = tx.sign(inputIndex, claimant, htlcScript)

            // Create claim script with secret
            return this.htlcBuilder.createClaimScript(
              secret,
              signature,
              claimant.toPublicKey()
            )
          },
          estimateLength: () => 150 // Approximate size
        },
        sequence: 0xffffffff
      })

      // Output to claimant's address
      const fee = 500
      const claimAmount = htlcUTXO.satoshis - fee

      tx.addOutput({
        satoshis: claimAmount,
        lockingScript: new P2PKH().lock(claimant.toPublicKey().toHash())
      })

      await tx.sign()

      console.log('HTLC claimed with secret')
      console.log('Claimed amount:', claimAmount, 'satoshis')

      return tx
    } catch (error) {
      throw new Error(`HTLC claim failed: ${error.message}`)
    }
  }

  /**
   * Refund HTLC after timeout
   *
   * @param refunder - Private key of the party requesting refund
   * @param htlcUTXO - UTXO locked in HTLC
   * @param htlcScript - The HTLC locking script
   * @param lockTime - The lock time from the HTLC
   * @returns Transaction refunding the funds
   */
  async refundHTLC(
    refunder: PrivateKey,
    htlcUTXO: {
      txid: string
      vout: number
      satoshis: number
    },
    htlcScript: Script,
    lockTime: number
  ): Promise<Transaction> {
    try {
      // Check if enough time has passed
      const currentTime = Math.floor(Date.now() / 1000)
      if (currentTime < lockTime) {
        throw new Error(`Cannot refund yet. Wait until ${new Date(lockTime * 1000).toISOString()}`)
      }

      const tx = new Transaction()
      tx.lockTime = lockTime

      tx.addInput({
        sourceTXID: htlcUTXO.txid,
        sourceOutputIndex: htlcUTXO.vout,
        unlockingScriptTemplate: {
          sign: async (tx: Transaction, inputIndex: number) => {
            // Create signature
            const signature = tx.sign(inputIndex, refunder, htlcScript)

            // Create refund script
            return this.htlcBuilder.createRefundScript(
              signature,
              refunder.toPublicKey()
            )
          },
          estimateLength: () => 120
        },
        sequence: 0xfffffffe // Required for lockTime
      })

      // Output back to refunder
      const fee = 500
      const refundAmount = htlcUTXO.satoshis - fee

      tx.addOutput({
        satoshis: refundAmount,
        lockingScript: new P2PKH().lock(refunder.toPublicKey().toHash())
      })

      await tx.sign()

      console.log('HTLC refunded after timeout')
      console.log('Refund amount:', refundAmount, 'satoshis')

      return tx
    } catch (error) {
      throw new Error(`HTLC refund failed: ${error.message}`)
    }
  }
}

/**
 * Complete Atomic Swap Example
 */
async function completeSwapExample() {
  const manager = new AtomicSwapManager()

  // Setup parties
  const aliceKey = PrivateKey.fromRandom()
  const bobKey = PrivateKey.fromRandom()

  console.log('=== Atomic Swap: Alice <-> Bob ===\n')

  // Alice's UTXO (100,000 satoshis)
  const aliceUTXO = {
    txid: 'alice-utxo-txid...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(aliceKey.toPublicKey().toHash())
  }

  // Bob's UTXO (150,000 satoshis)
  const bobUTXO = {
    txid: 'bob-utxo-txid...',
    vout: 0,
    satoshis: 150000,
    script: new P2PKH().lock(bobKey.toPublicKey().toHash())
  }

  // Lock time: 24 hours from now
  const lockTime = Math.floor(Date.now() / 1000) + (24 * 60 * 60)

  // Step 1: Alice initiates the swap
  console.log('Step 1: Alice initiates swap')
  const aliceSwap = await manager.initiateSwap(
    aliceKey,
    bobKey.toPublicKey(),
    aliceUTXO,
    lockTime
  )
  console.log('Alice locked 100,000 sats in HTLC\n')

  // Step 2: Bob participates using the same secret hash
  console.log('Step 2: Bob participates in swap')
  const bobSwap = await manager.participateSwap(
    bobKey,
    aliceKey.toPublicKey(),
    aliceSwap.secretHash,
    bobUTXO,
    lockTime
  )
  console.log('Bob locked 150,000 sats in HTLC\n')

  // Step 3: Bob claims Alice's funds (revealing the secret)
  console.log('Step 3: Bob claims Alice\'s HTLC')
  const bobClaim = await manager.claimHTLC(
    bobKey,
    aliceSwap.secret,
    {
      txid: aliceSwap.transaction.id('hex'),
      vout: 0,
      satoshis: 99500
    },
    aliceSwap.htlcScript
  )
  console.log('Bob claimed 99,500 sats from Alice\n')

  // Step 4: Alice learns the secret and claims Bob's funds
  console.log('Step 4: Alice claims Bob\'s HTLC')
  const aliceClaim = await manager.claimHTLC(
    aliceKey,
    aliceSwap.secret, // Alice uses the same secret Bob revealed
    {
      txid: bobSwap.transaction.id('hex'),
      vout: 0,
      satoshis: 149500
    },
    bobSwap.htlcScript
  )
  console.log('Alice claimed 149,500 sats from Bob\n')

  console.log('=== Atomic Swap Completed Successfully ===')
}
```

## Cross-Chain Atomic Swap

```typescript
import { Transaction, PrivateKey, PublicKey, Script } from '@bsv/sdk'

/**
 * Cross-Chain Atomic Swap Coordinator
 *
 * Coordinates atomic swaps between BSV and another blockchain.
 * This example shows the BSV side of the swap.
 */
class CrossChainSwapCoordinator {
  private swapManager: AtomicSwapManager

  constructor() {
    this.swapManager = new AtomicSwapManager()
  }

  /**
   * Create a cross-chain swap offer
   *
   * @param bsvParty - Private key on BSV side
   * @param otherChainAddress - Address on the other blockchain
   * @param bsvAmount - Amount to lock on BSV
   * @param otherChainAmount - Expected amount on other chain
   * @param utxo - BSV UTXO to lock
   * @returns Swap offer details
   */
  async createCrossChainOffer(
    bsvParty: PrivateKey,
    otherChainAddress: string,
    bsvAmount: number,
    otherChainAmount: number,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<{
    secretHash: string
    bsvHTLCAddress: string
    otherChainAddress: string
    amounts: {
      bsv: number
      otherChain: number
    }
    expiry: number
  }> {
    try {
      // Generate secret for cross-chain swap
      const builder = new HTLCBuilder()
      const { secret, hash } = builder.generateSecret()

      // Lock time: 48 hours (cross-chain swaps need more time)
      const lockTime = Math.floor(Date.now() / 1000) + (48 * 60 * 60)

      // Store secret securely for later use
      // In production, encrypt and store this safely
      this.storeSecret(hash, secret)

      console.log('Cross-chain swap offer created')
      console.log('BSV Amount:', bsvAmount)
      console.log('Other Chain Amount:', otherChainAmount)
      console.log('Secret Hash:', hash)
      console.log('Expires:', new Date(lockTime * 1000).toISOString())

      return {
        secretHash: hash,
        bsvHTLCAddress: 'bsv-htlc-address', // Derived from HTLC script
        otherChainAddress,
        amounts: {
          bsv: bsvAmount,
          otherChain: otherChainAmount
        },
        expiry: lockTime
      }
    } catch (error) {
      throw new Error(`Cross-chain offer creation failed: ${error.message}`)
    }
  }

  /**
   * Accept a cross-chain swap offer
   *
   * @param acceptor - Private key of accepting party
   * @param secretHash - Secret hash from the offer
   * @param offerDetails - Details of the swap offer
   * @param utxo - UTXO to lock on BSV side
   * @returns Transaction locking funds on BSV
   */
  async acceptCrossChainOffer(
    acceptor: PrivateKey,
    secretHash: string,
    offerDetails: {
      otherPartyPubKey: PublicKey
      bsvAmount: number
      expiry: number
    },
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<Transaction> {
    try {
      // Create HTLC on BSV side using the provided secret hash
      const swap = await this.swapManager.participateSwap(
        acceptor,
        offerDetails.otherPartyPubKey,
        secretHash,
        utxo,
        offerDetails.expiry
      )

      console.log('Accepted cross-chain offer')
      console.log('BSV funds locked in HTLC')

      return swap.transaction
    } catch (error) {
      throw new Error(`Cross-chain offer acceptance failed: ${error.message}`)
    }
  }

  /**
   * Monitor swap progress
   */
  async monitorSwapProgress(
    swapId: string,
    secretHash: string
  ): Promise<{
    status: 'pending' | 'claimed' | 'refunded' | 'expired'
    secret?: string
    claimTx?: string
  }> {
    // In production, this would check blockchain state
    // For now, return mock status
    console.log(`Monitoring swap ${swapId}`)
    console.log(`Secret hash: ${secretHash}`)

    return {
      status: 'pending'
    }
  }

  /**
   * Store secret securely (mock implementation)
   */
  private storeSecret(hash: string, secret: string): void {
    // In production, encrypt and store in secure storage
    console.log('Secret stored securely for hash:', hash)
  }

  /**
   * Retrieve stored secret (mock implementation)
   */
  private retrieveSecret(hash: string): string | null {
    // In production, retrieve from secure storage
    return null
  }
}

/**
 * Usage Example
 */
async function crossChainSwapExample() {
  const coordinator = new CrossChainSwapCoordinator()

  // Party A wants to swap BSV for Bitcoin
  const partyA = PrivateKey.fromRandom()

  const bsvUTXO = {
    txid: 'bsv-utxo...',
    vout: 0,
    satoshis: 1000000,
    script: new P2PKH().lock(partyA.toPublicKey().toHash())
  }

  // Create offer: 1,000,000 BSV sats for 0.001 BTC
  const offer = await coordinator.createCrossChainOffer(
    partyA,
    'bc1q...btc-address', // Bitcoin address
    1000000,
    100000, // 0.001 BTC in satoshis
    bsvUTXO
  )

  console.log('Cross-chain swap offer created')
  console.log('Share secret hash with counterparty:', offer.secretHash)
}
```

## Swap Safety and Recovery

```typescript
import { Transaction, PrivateKey } from '@bsv/sdk'

/**
 * Atomic Swap Safety Manager
 *
 * Handles error cases, timeouts, and recovery scenarios.
 */
class SwapSafetyManager {
  /**
   * Check if swap is safe to claim
   */
  canSafelyClaim(
    lockTime: number,
    bufferTime: number = 3600 // 1 hour buffer
  ): boolean {
    const currentTime = Math.floor(Date.now() / 1000)
    const timeRemaining = lockTime - currentTime

    if (timeRemaining < bufferTime) {
      console.warn(`Warning: Only ${timeRemaining} seconds until timeout`)
      return false
    }

    return true
  }

  /**
   * Check if refund is available
   */
  canRefund(lockTime: number): boolean {
    const currentTime = Math.floor(Date.now() / 1000)
    return currentTime >= lockTime
  }

  /**
   * Estimate time until refund is available
   */
  timeUntilRefund(lockTime: number): number {
    const currentTime = Math.floor(Date.now() / 1000)
    const remaining = lockTime - currentTime

    return Math.max(0, remaining)
  }

  /**
   * Verify secret matches hash before claiming
   */
  verifySecret(secret: string, expectedHash: string): boolean {
    const hash = Hash.sha256(Buffer.from(secret, 'hex'))
    return hash.toString('hex') === expectedHash
  }

  /**
   * Create emergency refund transaction
   */
  async createEmergencyRefund(
    refunder: PrivateKey,
    htlcUTXOs: Array<{
      txid: string
      vout: number
      satoshis: number
      script: Script
      lockTime: number
    }>
  ): Promise<Transaction[]> {
    const manager = new AtomicSwapManager()
    const refundTxs: Transaction[] = []

    for (const utxo of htlcUTXOs) {
      if (this.canRefund(utxo.lockTime)) {
        try {
          const refundTx = await manager.refundHTLC(
            refunder,
            utxo,
            utxo.script,
            utxo.lockTime
          )

          refundTxs.push(refundTx)
          console.log(`Created refund for HTLC ${utxo.txid}:${utxo.vout}`)
        } catch (error) {
          console.error(`Failed to create refund for ${utxo.txid}:`, error.message)
        }
      } else {
        const timeRemaining = this.timeUntilRefund(utxo.lockTime)
        console.log(`Cannot refund ${utxo.txid} yet. Wait ${timeRemaining} seconds`)
      }
    }

    return refundTxs
  }

  /**
   * Validate swap parameters before initiating
   */
  validateSwapParameters(params: {
    amount: number
    lockTime: number
    minLockTime: number
    maxLockTime: number
  }): { valid: boolean; errors: string[] } {
    const errors: string[] = []

    // Check amount
    if (params.amount < 546) {
      errors.push('Amount is below dust threshold')
    }

    // Check lock time is in the future
    const currentTime = Math.floor(Date.now() / 1000)
    if (params.lockTime <= currentTime) {
      errors.push('Lock time must be in the future')
    }

    // Check lock time is reasonable
    if (params.lockTime - currentTime < params.minLockTime) {
      errors.push(`Lock time must be at least ${params.minLockTime} seconds in the future`)
    }

    if (params.lockTime - currentTime > params.maxLockTime) {
      errors.push(`Lock time cannot be more than ${params.maxLockTime} seconds in the future`)
    }

    return {
      valid: errors.length === 0,
      errors
    }
  }
}

/**
 * Usage Example
 */
async function safetyExample() {
  const safety = new SwapSafetyManager()

  // Validate parameters before swap
  const lockTime = Math.floor(Date.now() / 1000) + (24 * 60 * 60)

  const validation = safety.validateSwapParameters({
    amount: 100000,
    lockTime,
    minLockTime: 3600, // At least 1 hour
    maxLockTime: 7 * 24 * 60 * 60 // At most 1 week
  })

  if (!validation.valid) {
    console.error('Invalid swap parameters:', validation.errors)
    return
  }

  // Check if safe to claim
  if (safety.canSafelyClaim(lockTime)) {
    console.log('Safe to claim funds')
  } else {
    console.log('Warning: Close to timeout, claim immediately or wait for refund')
  }

  // Check time until refund
  const timeUntilRefund = safety.timeUntilRefund(lockTime)
  console.log(`Time until refund available: ${timeUntilRefund} seconds`)
}
```

## Related Examples

- [Transaction Building](../transaction-building/README.md)
- [Transaction Chains](../transaction-chains/README.md)
- [Smart Contracts](../smart-contracts/README.md)

## See Also

**SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md) - Core transaction building
- [Script](../../sdk-components/script/README.md) - Bitcoin script operations
- [Script Templates](../../sdk-components/script-templates/README.md) - Custom script templates
- [Signatures](../../sdk-components/signatures/README.md) - Digital signatures

**Learning Paths:**
- [Script Programming](../../learning-paths/intermediate/script-programming/README.md)
