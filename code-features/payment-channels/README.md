# Payment Channels

Complete examples for implementing payment channels on BSV, enabling instant, low-cost off-chain transactions with on-chain settlement guarantees.

## Overview

Payment channels allow two parties to conduct multiple transactions off-chain while maintaining the security guarantees of on-chain settlement. By opening a channel with an initial funding transaction, parties can exchange signed state updates representing balance changes, and only broadcast the final state to the blockchain when closing the channel. This enables high-throughput, low-latency micropayments.

**Related SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md)
- [Script](../../sdk-components/script/README.md)
- [Script Templates](../../sdk-components/script-templates/README.md)
- [Signatures](../../sdk-components/signatures/README.md)

## Payment Channel Setup

```typescript
import { Transaction, PrivateKey, PublicKey, Script, P2PKH, OP } from '@bsv/sdk'

/**
 * Payment Channel Manager
 *
 * Manages the complete lifecycle of a unidirectional payment channel.
 */
class PaymentChannelManager {
  /**
   * Create a payment channel funding transaction
   *
   * @param sender - Channel sender (pays into channel)
   * @param receiver - Channel receiver
   * @param fundingAmount - Amount to lock in channel
   * @param duration - Channel duration in seconds
   * @param utxo - UTXO to fund the channel
   * @returns Channel setup details
   */
  async createChannel(
    sender: PrivateKey,
    receiver: PublicKey,
    fundingAmount: number,
    duration: number,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<{
    channelId: string
    fundingTx: Transaction
    expiryTime: number
    initialBalance: {
      sender: number
      receiver: number
    }
  }> {
    try {
      if (fundingAmount < 1000) {
        throw new Error('Funding amount too small for channel')
      }

      if (fundingAmount > utxo.satoshis) {
        throw new Error('Insufficient funds in UTXO')
      }

      // Calculate expiry time
      const currentTime = Math.floor(Date.now() / 1000)
      const expiryTime = currentTime + duration

      // Create funding transaction
      const fundingTx = new Transaction()

      fundingTx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(sender),
        sequence: 0xffffffff
      })

      // Create 2-of-2 multisig output for channel
      const channelScript = this.createChannelScript(
        sender.toPublicKey(),
        receiver,
        expiryTime
      )

      const fee = 500
      const channelAmount = fundingAmount - fee

      fundingTx.addOutput({
        satoshis: channelAmount,
        lockingScript: channelScript
      })

      // Add change output if needed
      const change = utxo.satoshis - fundingAmount - fee

      if (change > 546) {
        fundingTx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(sender.toPublicKey().toHash())
        })
      }

      await fundingTx.sign()

      const channelId = fundingTx.id('hex')

      console.log('Payment channel created')
      console.log('Channel ID:', channelId)
      console.log('Capacity:', channelAmount, 'satoshis')
      console.log('Expires:', new Date(expiryTime * 1000).toISOString())

      return {
        channelId,
        fundingTx,
        expiryTime,
        initialBalance: {
          sender: channelAmount,
          receiver: 0
        }
      }
    } catch (error) {
      throw new Error(`Channel creation failed: ${error.message}`)
    }
  }

  /**
   * Create channel locking script
   *
   * Allows spending with both signatures or sender signature after timeout
   */
  private createChannelScript(
    senderPubKey: PublicKey,
    receiverPubKey: PublicKey,
    expiryTime: number
  ): Script {
    // Channel script:
    // IF
    //   2 <senderPubKey> <receiverPubKey> 2 OP_CHECKMULTISIG
    // ELSE
    //   <expiryTime> OP_CHECKLOCKTIMEVERIFY OP_DROP <senderPubKey> OP_CHECKSIG
    // ENDIF

    const script = new Script()

    // IF branch (cooperative close with both signatures)
    script.chunks.push({ op: OP.OP_IF })
    script.chunks.push({ op: OP.OP_2 })
    script.chunks.push({ data: senderPubKey.toBuffer() })
    script.chunks.push({ data: receiverPubKey.toBuffer() })
    script.chunks.push({ op: OP.OP_2 })
    script.chunks.push({ op: OP.OP_CHECKMULTISIG })

    // ELSE branch (unilateral close after timeout)
    script.chunks.push({ op: OP.OP_ELSE })
    script.chunks.push({ data: this.numberToBuffer(expiryTime) })
    script.chunks.push({ op: OP.OP_CHECKLOCKTIMEVERIFY })
    script.chunks.push({ op: OP.OP_DROP })
    script.chunks.push({ data: senderPubKey.toBuffer() })
    script.chunks.push({ op: OP.OP_CHECKSIG })

    script.chunks.push({ op: OP.OP_ENDIF })

    return script
  }

  /**
   * Convert number to buffer for script
   */
  private numberToBuffer(num: number): Buffer {
    if (num === 0) return Buffer.from([])

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
async function setupChannelExample() {
  const manager = new PaymentChannelManager()

  const sender = PrivateKey.fromRandom()
  const receiver = PrivateKey.fromRandom()

  const utxo = {
    txid: 'funding-utxo...',
    vout: 0,
    satoshis: 1000000,
    script: new P2PKH().lock(sender.toPublicKey().toHash())
  }

  // Create channel with 500,000 sats for 24 hours
  const channel = await manager.createChannel(
    sender,
    receiver.toPublicKey(),
    500000,
    24 * 60 * 60,
    utxo
  )

  console.log('Channel ready for payments')
}
```

## Channel State Updates

```typescript
import { Transaction, PrivateKey, PublicKey, Script } from '@bsv/sdk'

/**
 * Channel State Manager
 *
 * Manages payment channel state updates and commitment transactions.
 */
class ChannelStateManager {
  private stateNumber: number = 0

  /**
   * Create a commitment transaction representing current channel state
   *
   * @param sender - Channel sender
   * @param receiver - Channel receiver
   * @param channelUTXO - The funding transaction output
   * @param senderBalance - Current balance for sender
   * @param receiverBalance - Current balance for receiver
   * @returns Commitment transaction
   */
  async createCommitmentTransaction(
    sender: PrivateKey,
    receiver: PrivateKey,
    channelUTXO: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    senderBalance: number,
    receiverBalance: number
  ): Promise<{
    tx: Transaction
    stateNumber: number
  }> {
    try {
      // Validate balances
      if (senderBalance + receiverBalance !== channelUTXO.satoshis) {
        throw new Error('Balances do not match channel capacity')
      }

      if (senderBalance < 0 || receiverBalance < 0) {
        throw new Error('Balances cannot be negative')
      }

      this.stateNumber++

      const tx = new Transaction()

      // Input from channel
      tx.addInput({
        sourceTXID: channelUTXO.txid,
        sourceOutputIndex: channelUTXO.vout,
        // In production, this would be a proper 2-of-2 multisig unlock
        unlockingScriptTemplate: new P2PKH().unlock(sender),
        sequence: this.stateNumber // State number as sequence
      })

      // Output to sender
      if (senderBalance > 546) {
        tx.addOutput({
          satoshis: senderBalance,
          lockingScript: new P2PKH().lock(sender.toPublicKey().toHash())
        })
      }

      // Output to receiver
      if (receiverBalance > 546) {
        tx.addOutput({
          satoshis: receiverBalance,
          lockingScript: new P2PKH().lock(receiver.toPublicKey().toHash())
        })
      }

      // Both parties sign
      await tx.sign()

      console.log(`Created commitment transaction #${this.stateNumber}`)
      console.log(`  Sender balance: ${senderBalance}`)
      console.log(`  Receiver balance: ${receiverBalance}`)

      return {
        tx,
        stateNumber: this.stateNumber
      }
    } catch (error) {
      throw new Error(`Commitment creation failed: ${error.message}`)
    }
  }

  /**
   * Process a payment through the channel
   *
   * @param currentState - Current channel state
   * @param paymentAmount - Amount to pay
   * @returns New state
   */
  processPayment(
    currentState: {
      senderBalance: number
      receiverBalance: number
      stateNumber: number
    },
    paymentAmount: number
  ): {
    senderBalance: number
    receiverBalance: number
    stateNumber: number
  } {
    if (paymentAmount <= 0) {
      throw new Error('Payment amount must be positive')
    }

    if (paymentAmount > currentState.senderBalance) {
      throw new Error('Insufficient balance in channel')
    }

    const newState = {
      senderBalance: currentState.senderBalance - paymentAmount,
      receiverBalance: currentState.receiverBalance + paymentAmount,
      stateNumber: currentState.stateNumber + 1
    }

    console.log(`Payment of ${paymentAmount} processed`)
    console.log(`New state: Sender=${newState.senderBalance}, Receiver=${newState.receiverBalance}`)

    return newState
  }

  /**
   * Create multiple payments in sequence
   */
  async createPaymentSequence(
    sender: PrivateKey,
    receiver: PrivateKey,
    channelUTXO: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    initialSenderBalance: number,
    payments: number[]
  ): Promise<Array<{ tx: Transaction; state: any }>> {
    const commitments: Array<{ tx: Transaction; state: any }> = []

    let state = {
      senderBalance: initialSenderBalance,
      receiverBalance: channelUTXO.satoshis - initialSenderBalance,
      stateNumber: 0
    }

    for (const payment of payments) {
      // Update state
      state = this.processPayment(state, payment)

      // Create commitment
      const commitment = await this.createCommitmentTransaction(
        sender,
        receiver,
        channelUTXO,
        state.senderBalance,
        state.receiverBalance
      )

      commitments.push({
        tx: commitment.tx,
        state: { ...state }
      })
    }

    return commitments
  }

  /**
   * Validate a commitment transaction
   */
  validateCommitment(
    commitment: Transaction,
    expectedState: {
      senderBalance: number
      receiverBalance: number
      stateNumber: number
    },
    channelCapacity: number
  ): { valid: boolean; errors: string[] } {
    const errors: string[] = []

    // Check total outputs equal channel capacity
    const totalOutput = commitment.outputs.reduce((sum, out) => sum + out.satoshis, 0)

    if (totalOutput !== channelCapacity) {
      errors.push(`Output total ${totalOutput} does not match capacity ${channelCapacity}`)
    }

    // Check sequence number
    const sequence = commitment.inputs[0]?.sequence
    if (sequence !== expectedState.stateNumber) {
      errors.push(`State number mismatch: expected ${expectedState.stateNumber}, got ${sequence}`)
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
async function channelPaymentsExample() {
  const stateManager = new ChannelStateManager()

  const sender = PrivateKey.fromRandom()
  const receiver = PrivateKey.fromRandom()

  const channelUTXO = {
    txid: 'channel-funding-tx...',
    vout: 0,
    satoshis: 100000,
    script: new Script() // Channel script
  }

  // Process a series of micropayments
  const payments = [1000, 500, 1500, 2000, 1000]

  const commitments = await stateManager.createPaymentSequence(
    sender,
    receiver,
    channelUTXO,
    100000, // Sender starts with full capacity
    payments
  )

  console.log(`\nCreated ${commitments.length} payment commitments`)
  console.log('Final state:', commitments[commitments.length - 1].state)
}
```

## Channel Closing

```typescript
import { Transaction, PrivateKey, Script } from '@bsv/sdk'

/**
 * Channel Close Manager
 *
 * Handles cooperative and unilateral channel closing.
 */
class ChannelCloseManager {
  /**
   * Close channel cooperatively with final settlement
   *
   * @param sender - Channel sender
   * @param receiver - Channel receiver
   * @param channelUTXO - Channel funding output
   * @param finalState - Final agreed state
   * @returns Settlement transaction
   */
  async closeChannelCooperative(
    sender: PrivateKey,
    receiver: PrivateKey,
    channelUTXO: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    finalState: {
      senderBalance: number
      receiverBalance: number
    }
  ): Promise<Transaction> {
    try {
      console.log('Closing channel cooperatively...')

      const tx = new Transaction()

      // Input from channel
      tx.addInput({
        sourceTXID: channelUTXO.txid,
        sourceOutputIndex: channelUTXO.vout,
        // Both parties sign (2-of-2 multisig)
        unlockingScriptTemplate: new P2PKH().unlock(sender),
        sequence: 0xffffffff
      })

      // Final settlement outputs
      if (finalState.senderBalance > 546) {
        tx.addOutput({
          satoshis: finalState.senderBalance,
          lockingScript: new P2PKH().lock(sender.toPublicKey().toHash())
        })
      }

      if (finalState.receiverBalance > 546) {
        tx.addOutput({
          satoshis: finalState.receiverBalance,
          lockingScript: new P2PKH().lock(receiver.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('Channel closed cooperatively')
      console.log(`  Sender receives: ${finalState.senderBalance}`)
      console.log(`  Receiver receives: ${finalState.receiverBalance}`)

      return tx
    } catch (error) {
      throw new Error(`Cooperative close failed: ${error.message}`)
    }
  }

  /**
   * Close channel unilaterally by broadcasting latest commitment
   *
   * @param party - Party closing the channel
   * @param latestCommitment - Latest commitment transaction
   * @param channelScript - The channel locking script
   * @returns Broadcasted transaction
   */
  async closeChannelUnilateral(
    party: PrivateKey,
    latestCommitment: Transaction,
    channelScript: Script
  ): Promise<Transaction> {
    try {
      console.log('Closing channel unilaterally...')
      console.log('Broadcasting latest commitment transaction')

      // In production, verify this is actually the latest commitment
      // and check for any challenge period

      // The commitment transaction is already signed and ready to broadcast
      console.log('Commitment transaction:', latestCommitment.id('hex'))

      return latestCommitment
    } catch (error) {
      throw new Error(`Unilateral close failed: ${error.message}`)
    }
  }

  /**
   * Close channel after timeout (refund to sender)
   *
   * @param sender - Channel sender
   * @param channelUTXO - Channel funding output
   * @param channelScript - Channel locking script
   * @param expiryTime - Channel expiry time
   * @returns Timeout refund transaction
   */
  async closeChannelTimeout(
    sender: PrivateKey,
    channelUTXO: {
      txid: string
      vout: number
      satoshis: number
    },
    channelScript: Script,
    expiryTime: number
  ): Promise<Transaction> {
    try {
      const currentTime = Math.floor(Date.now() / 1000)

      if (currentTime < expiryTime) {
        const remaining = expiryTime - currentTime
        throw new Error(`Channel not expired yet. Wait ${remaining} seconds`)
      }

      console.log('Closing channel after timeout...')

      const tx = new Transaction()
      tx.lockTime = expiryTime

      tx.addInput({
        sourceTXID: channelUTXO.txid,
        sourceOutputIndex: channelUTXO.vout,
        unlockingScriptTemplate: new P2PKH().unlock(sender),
        sequence: 0xfffffffe // Required for lockTime
      })

      // Refund all funds to sender
      const fee = 500
      const refundAmount = channelUTXO.satoshis - fee

      tx.addOutput({
        satoshis: refundAmount,
        lockingScript: new P2PKH().lock(sender.toPublicKey().toHash())
      })

      await tx.sign()

      console.log('Channel closed via timeout')
      console.log(`  Refunded to sender: ${refundAmount}`)

      return tx
    } catch (error) {
      throw new Error(`Timeout close failed: ${error.message}`)
    }
  }

  /**
   * Calculate final settlement from commitment history
   *
   * Finds the latest valid commitment transaction
   */
  calculateFinalSettlement(
    commitments: Array<{
      tx: Transaction
      stateNumber: number
      timestamp: number
    }>
  ): {
    latestCommitment: Transaction
    stateNumber: number
  } {
    if (commitments.length === 0) {
      throw new Error('No commitments found')
    }

    // Sort by state number (descending)
    const sorted = [...commitments].sort((a, b) => b.stateNumber - a.stateNumber)

    const latest = sorted[0]

    console.log('Final settlement calculated')
    console.log(`  Latest state number: ${latest.stateNumber}`)
    console.log(`  Total commitments: ${commitments.length}`)

    return {
      latestCommitment: latest.tx,
      stateNumber: latest.stateNumber
    }
  }
}

/**
 * Usage Example
 */
async function channelCloseExample() {
  const closeManager = new ChannelCloseManager()

  const sender = PrivateKey.fromRandom()
  const receiver = PrivateKey.fromRandom()

  const channelUTXO = {
    txid: 'channel-tx...',
    vout: 0,
    satoshis: 100000,
    script: new Script()
  }

  // Cooperative close with final balances
  const settlementTx = await closeManager.closeChannelCooperative(
    sender,
    receiver,
    channelUTXO,
    {
      senderBalance: 94000,
      receiverBalance: 6000
    }
  )

  console.log('Settlement transaction:', settlementTx.id('hex'))
}
```

## Bidirectional Payment Channel

```typescript
import { Transaction, PrivateKey, PublicKey, Script, P2PKH } from '@bsv/sdk'

/**
 * Bidirectional Payment Channel Manager
 *
 * Manages channels where both parties can send payments to each other.
 */
class BidirectionalChannelManager {
  /**
   * Create bidirectional channel with initial deposits from both parties
   *
   * @param party1 - First party
   * @param party2 - Second party
   * @param party1Deposit - Party 1's initial deposit
   * @param party2Deposit - Party 2's initial deposit
   * @param party1UTXO - Party 1's funding UTXO
   * @param party2UTXO - Party 2's funding UTXO
   * @param duration - Channel duration
   * @returns Channel setup
   */
  async createBidirectionalChannel(
    party1: PrivateKey,
    party2: PrivateKey,
    party1Deposit: number,
    party2Deposit: number,
    party1UTXO: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    party2UTXO: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    duration: number
  ): Promise<{
    fundingTx: Transaction
    channelCapacity: number
    initialBalances: {
      party1: number
      party2: number
    }
    expiryTime: number
  }> {
    try {
      console.log('Creating bidirectional payment channel...')

      const fundingTx = new Transaction()

      // Both parties contribute funds
      fundingTx.addInput({
        sourceTXID: party1UTXO.txid,
        sourceOutputIndex: party1UTXO.vout,
        unlockingScriptTemplate: new P2PKH().unlock(party1),
        sequence: 0xffffffff
      })

      fundingTx.addInput({
        sourceTXID: party2UTXO.txid,
        sourceOutputIndex: party2UTXO.vout,
        unlockingScriptTemplate: new P2PKH().unlock(party2),
        sequence: 0xffffffff
      })

      // Create 2-of-2 multisig channel output
      const expiryTime = Math.floor(Date.now() / 1000) + duration
      const channelScript = this.create2of2ChannelScript(
        party1.toPublicKey(),
        party2.toPublicKey(),
        expiryTime
      )

      const fee = 1000
      const channelCapacity = party1Deposit + party2Deposit - fee

      fundingTx.addOutput({
        satoshis: channelCapacity,
        lockingScript: channelScript
      })

      // Add change outputs if needed
      const party1Change = party1UTXO.satoshis - party1Deposit
      if (party1Change > 546) {
        fundingTx.addOutput({
          satoshis: party1Change,
          lockingScript: new P2PKH().lock(party1.toPublicKey().toHash())
        })
      }

      const party2Change = party2UTXO.satoshis - party2Deposit
      if (party2Change > 546) {
        fundingTx.addOutput({
          satoshis: party2Change,
          lockingScript: new P2PKH().lock(party2.toPublicKey().toHash())
        })
      }

      await fundingTx.sign()

      console.log('Bidirectional channel created')
      console.log(`  Total capacity: ${channelCapacity}`)
      console.log(`  Party 1 deposit: ${party1Deposit}`)
      console.log(`  Party 2 deposit: ${party2Deposit}`)

      return {
        fundingTx,
        channelCapacity,
        initialBalances: {
          party1: party1Deposit,
          party2: party2Deposit
        },
        expiryTime
      }
    } catch (error) {
      throw new Error(`Bidirectional channel creation failed: ${error.message}`)
    }
  }

  /**
   * Process bidirectional payment
   */
  processBidirectionalPayment(
    currentState: {
      party1Balance: number
      party2Balance: number
      stateNumber: number
    },
    from: 'party1' | 'party2',
    amount: number
  ): {
    party1Balance: number
    party2Balance: number
    stateNumber: number
  } {
    if (amount <= 0) {
      throw new Error('Payment amount must be positive')
    }

    const newState = {
      party1Balance: currentState.party1Balance,
      party2Balance: currentState.party2Balance,
      stateNumber: currentState.stateNumber + 1
    }

    if (from === 'party1') {
      if (amount > currentState.party1Balance) {
        throw new Error('Party 1 insufficient balance')
      }
      newState.party1Balance -= amount
      newState.party2Balance += amount
      console.log(`Party 1 → Party 2: ${amount} sats`)
    } else {
      if (amount > currentState.party2Balance) {
        throw new Error('Party 2 insufficient balance')
      }
      newState.party2Balance -= amount
      newState.party1Balance += amount
      console.log(`Party 2 → Party 1: ${amount} sats`)
    }

    console.log(`New balances: Party1=${newState.party1Balance}, Party2=${newState.party2Balance}`)

    return newState
  }

  /**
   * Create 2-of-2 multisig channel script
   */
  private create2of2ChannelScript(
    pubKey1: PublicKey,
    pubKey2: PublicKey,
    expiryTime: number
  ): Script {
    // Simplified 2-of-2 multisig
    // In production, would include timeout branches for both parties
    const script = new Script()

    script.chunks.push({ op: OP.OP_2 })
    script.chunks.push({ data: pubKey1.toBuffer() })
    script.chunks.push({ data: pubKey2.toBuffer() })
    script.chunks.push({ op: OP.OP_2 })
    script.chunks.push({ op: OP.OP_CHECKMULTISIG })

    return script
  }
}

/**
 * Usage Example
 */
async function bidirectionalChannelExample() {
  const manager = new BidirectionalChannelManager()

  const alice = PrivateKey.fromRandom()
  const bob = PrivateKey.fromRandom()

  const aliceUTXO = {
    txid: 'alice-utxo...',
    vout: 0,
    satoshis: 500000,
    script: new P2PKH().lock(alice.toPublicKey().toHash())
  }

  const bobUTXO = {
    txid: 'bob-utxo...',
    vout: 0,
    satoshis: 500000,
    script: new P2PKH().lock(bob.toPublicKey().toHash())
  }

  // Create channel with equal deposits
  const channel = await manager.createBidirectionalChannel(
    alice,
    bob,
    100000,
    100000,
    aliceUTXO,
    bobUTXO,
    24 * 60 * 60
  )

  console.log('Bidirectional channel ready')

  // Simulate bidirectional payments
  let state = {
    party1Balance: channel.initialBalances.party1,
    party2Balance: channel.initialBalances.party2,
    stateNumber: 0
  }

  // Alice pays Bob
  state = manager.processBidirectionalPayment(state, 'party1', 5000)

  // Bob pays Alice
  state = manager.processBidirectionalPayment(state, 'party2', 3000)

  // Alice pays Bob again
  state = manager.processBidirectionalPayment(state, 'party1', 2000)

  console.log('Final balances:', state)
}
```

## Related Examples

- [Transaction Building](../transaction-building/README.md)
- [Transaction Chains](../transaction-chains/README.md)
- [Atomic Swaps](../atomic-swaps/README.md)
- [Smart Contracts](../smart-contracts/README.md)

## See Also

**SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md) - Core transaction building
- [Script](../../sdk-components/script/README.md) - Bitcoin script operations
- [Script Templates](../../sdk-components/script-templates/README.md) - Custom script templates
- [Signatures](../../sdk-components/signatures/README.md) - Digital signatures

**Learning Paths:**
- [Script Programming](../../learning-paths/intermediate/script-programming/README.md)
- [Payment Channels](../../learning-paths/advanced/payment-channels/README.md)
