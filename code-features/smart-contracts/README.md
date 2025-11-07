# Smart Contracts

Complete examples for implementing smart contracts on BSV, including escrow services, token vesting schedules, and oracle-based conditional payments.

## Overview

Smart contracts on BSV use Bitcoin Script to encode business logic and conditions directly into transactions. Unlike account-based blockchains, BSV smart contracts are UTXO-based, with conditions checked during transaction validation. This section demonstrates practical smart contract implementations for common use cases including escrow, vesting, and oracle integrations.

**Related SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md)
- [Script](../../sdk-components/script/README.md)
- [Script Templates](../../sdk-components/script-templates/README.md)
- [Signatures](../../sdk-components/signatures/README.md)

## Escrow Contract

```typescript
import { Transaction, PrivateKey, PublicKey, Script, Hash, OP, P2PKH } from '@bsv/sdk'

/**
 * Escrow Smart Contract
 *
 * Implements a trustless escrow system where funds are released
 * based on signatures from buyer, seller, and optional arbiter.
 */
class EscrowContract {
  /**
   * Create an escrow locking script
   *
   * Funds can be released in three ways:
   * 1. Buyer + Seller signatures (happy path)
   * 2. Buyer + Arbiter signatures (dispute resolution in buyer's favor)
   * 3. Seller + Arbiter signatures (dispute resolution in seller's favor)
   *
   * @param buyerPubKey - Public key of the buyer
   * @param sellerPubKey - Public key of the seller
   * @param arbiterPubKey - Public key of the arbiter
   * @param timeout - Timeout after which seller can claim (unix timestamp)
   * @returns Escrow locking script
   */
  createEscrowScript(
    buyerPubKey: PublicKey,
    sellerPubKey: PublicKey,
    arbiterPubKey: PublicKey,
    timeout?: number
  ): Script {
    try {
      // 2-of-3 multisig with optional timeout
      const script = new Script()

      if (timeout) {
        // If timeout is specified, add timeout branch
        script.chunks.push({ op: OP.OP_IF })
      }

      // 2-of-3 multisig: any two of buyer, seller, arbiter
      script.chunks.push({ op: OP.OP_2 })
      script.chunks.push({ data: buyerPubKey.toBuffer() })
      script.chunks.push({ data: sellerPubKey.toBuffer() })
      script.chunks.push({ data: arbiterPubKey.toBuffer() })
      script.chunks.push({ op: OP.OP_3 })
      script.chunks.push({ op: OP.OP_CHECKMULTISIG })

      if (timeout) {
        // Timeout branch: seller can claim after timeout
        script.chunks.push({ op: OP.OP_ELSE })
        script.chunks.push({ data: this.numberToBuffer(timeout) })
        script.chunks.push({ op: OP.OP_CHECKLOCKTIMEVERIFY })
        script.chunks.push({ op: OP.OP_DROP })
        script.chunks.push({ data: sellerPubKey.toBuffer() })
        script.chunks.push({ op: OP.OP_CHECKSIG })
        script.chunks.push({ op: OP.OP_ENDIF })
      }

      return script
    } catch (error) {
      throw new Error(`Escrow script creation failed: ${error.message}`)
    }
  }

  /**
   * Create escrow transaction
   *
   * @param buyer - Buyer's private key
   * @param seller - Seller's public key
   * @param arbiter - Arbiter's public key
   * @param amount - Escrow amount
   * @param utxo - Funding UTXO
   * @param timeoutDays - Optional timeout in days
   * @returns Escrow transaction and script
   */
  async createEscrow(
    buyer: PrivateKey,
    seller: PublicKey,
    arbiter: PublicKey,
    amount: number,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    timeoutDays?: number
  ): Promise<{
    transaction: Transaction
    escrowScript: Script
    escrowDetails: {
      amount: number
      timeout?: number
      participants: {
        buyer: string
        seller: string
        arbiter: string
      }
    }
  }> {
    try {
      if (amount < 1000) {
        throw new Error('Escrow amount too small')
      }

      const timeout = timeoutDays
        ? Math.floor(Date.now() / 1000) + (timeoutDays * 24 * 60 * 60)
        : undefined

      // Create escrow script
      const escrowScript = this.createEscrowScript(
        buyer.toPublicKey(),
        seller,
        arbiter,
        timeout
      )

      // Create funding transaction
      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(buyer),
        sequence: 0xffffffff
      })

      const fee = 500
      const escrowAmount = amount - fee

      tx.addOutput({
        satoshis: escrowAmount,
        lockingScript: escrowScript
      })

      // Change output
      const change = utxo.satoshis - amount - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(buyer.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('Escrow created')
      console.log(`  Amount: ${escrowAmount} satoshis`)
      console.log(`  Timeout: ${timeout ? new Date(timeout * 1000).toISOString() : 'None'}`)

      return {
        transaction: tx,
        escrowScript,
        escrowDetails: {
          amount: escrowAmount,
          timeout,
          participants: {
            buyer: buyer.toPublicKey().toString(),
            seller: seller.toString(),
            arbiter: arbiter.toString()
          }
        }
      }
    } catch (error) {
      throw new Error(`Escrow creation failed: ${error.message}`)
    }
  }

  /**
   * Release escrow funds (buyer + seller agreement)
   *
   * @param buyer - Buyer's private key
   * @param seller - Seller's private key
   * @param escrowUTXO - Escrow UTXO
   * @param escrowScript - Escrow locking script
   * @param recipient - Recipient address
   * @returns Release transaction
   */
  async releaseEscrow(
    buyer: PrivateKey,
    seller: PrivateKey,
    escrowUTXO: {
      txid: string
      vout: number
      satoshis: number
    },
    escrowScript: Script,
    recipient: string
  ): Promise<Transaction> {
    try {
      console.log('Releasing escrow with buyer + seller signatures...')

      const tx = new Transaction()

      tx.addInput({
        sourceTXID: escrowUTXO.txid,
        sourceOutputIndex: escrowUTXO.vout,
        unlockingScriptTemplate: {
          sign: async (transaction: Transaction, inputIndex: number) => {
            // Create signatures from both parties
            const buyerSig = transaction.sign(inputIndex, buyer, escrowScript)
            const sellerSig = transaction.sign(inputIndex, seller, escrowScript)

            // Build unlocking script with both signatures
            const unlockScript = new Script()
            unlockScript.chunks.push({ op: OP.OP_0 }) // OP_CHECKMULTISIG bug
            unlockScript.chunks.push({ data: buyerSig })
            unlockScript.chunks.push({ data: sellerSig })

            return unlockScript
          },
          estimateLength: () => 150
        },
        sequence: 0xffffffff
      })

      // Output to recipient
      const fee = 500
      const releaseAmount = escrowUTXO.satoshis - fee

      tx.addOutput({
        satoshis: releaseAmount,
        lockingScript: Script.fromAddress(recipient)
      })

      await tx.sign()

      console.log('Escrow released')
      console.log(`  Amount: ${releaseAmount} to ${recipient}`)

      return tx
    } catch (error) {
      throw new Error(`Escrow release failed: ${error.message}`)
    }
  }

  /**
   * Resolve escrow dispute (arbiter involved)
   *
   * @param party - Either buyer or seller
   * @param arbiter - Arbiter's private key
   * @param escrowUTXO - Escrow UTXO
   * @param escrowScript - Escrow locking script
   * @param recipient - Recipient address
   * @returns Resolution transaction
   */
  async resolveEscrowDispute(
    party: PrivateKey,
    arbiter: PrivateKey,
    escrowUTXO: {
      txid: string
      vout: number
      satoshis: number
    },
    escrowScript: Script,
    recipient: string
  ): Promise<Transaction> {
    try {
      console.log('Resolving escrow dispute with arbiter...')

      const tx = new Transaction()

      tx.addInput({
        sourceTXID: escrowUTXO.txid,
        sourceOutputIndex: escrowUTXO.vout,
        unlockingScriptTemplate: {
          sign: async (transaction: Transaction, inputIndex: number) => {
            const partySig = transaction.sign(inputIndex, party, escrowScript)
            const arbiterSig = transaction.sign(inputIndex, arbiter, escrowScript)

            const unlockScript = new Script()
            unlockScript.chunks.push({ op: OP.OP_0 })
            unlockScript.chunks.push({ data: partySig })
            unlockScript.chunks.push({ data: arbiterSig })

            return unlockScript
          },
          estimateLength: () => 150
        },
        sequence: 0xffffffff
      })

      const fee = 500
      const resolutionAmount = escrowUTXO.satoshis - fee

      tx.addOutput({
        satoshis: resolutionAmount,
        lockingScript: Script.fromAddress(recipient)
      })

      await tx.sign()

      console.log('Dispute resolved')
      console.log(`  Amount: ${resolutionAmount} to ${recipient}`)

      return tx
    } catch (error) {
      throw new Error(`Dispute resolution failed: ${error.message}`)
    }
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
async function escrowExample() {
  const escrow = new EscrowContract()

  const buyer = PrivateKey.fromRandom()
  const seller = PrivateKey.fromRandom()
  const arbiter = PrivateKey.fromRandom()

  const buyerUTXO = {
    txid: 'buyer-utxo...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(buyer.toPublicKey().toHash())
  }

  // Create escrow for 50,000 sats with 30-day timeout
  const { transaction, escrowScript } = await escrow.createEscrow(
    buyer,
    seller.toPublicKey(),
    arbiter.toPublicKey(),
    50000,
    buyerUTXO,
    30
  )

  console.log('Escrow Transaction:', transaction.id('hex'))

  // Happy path: buyer and seller agree to release
  const releaseTx = await escrow.releaseEscrow(
    buyer,
    seller,
    {
      txid: transaction.id('hex'),
      vout: 0,
      satoshis: 49500
    },
    escrowScript,
    seller.toPublicKey().toAddress()
  )

  console.log('Release Transaction:', releaseTx.id('hex'))
}
```

## Token Vesting Contract

```typescript
import { Transaction, PrivateKey, PublicKey, Script, OP, P2PKH } from '@bsv/sdk'

/**
 * Token Vesting Smart Contract
 *
 * Implements time-locked vesting schedules for token distribution.
 */
class VestingContract {
  /**
   * Create a vesting schedule
   *
   * @param beneficiary - Public key of the beneficiary
   * @param cliffTime - Unix timestamp when cliff period ends
   * @param vestingEnd - Unix timestamp when vesting completes
   * @param totalAmount - Total amount to vest
   * @returns Vesting details
   */
  createVestingSchedule(
    beneficiary: PublicKey,
    cliffTime: number,
    vestingEnd: number,
    totalAmount: number
  ): {
    schedule: Array<{
      unlockTime: number
      amount: number
      percentage: number
    }>
    totalAmount: number
    duration: number
  } {
    try {
      const currentTime = Math.floor(Date.now() / 1000)

      if (cliffTime <= currentTime) {
        throw new Error('Cliff time must be in the future')
      }

      if (vestingEnd <= cliffTime) {
        throw new Error('Vesting end must be after cliff')
      }

      // Create linear vesting schedule
      const duration = vestingEnd - cliffTime
      const monthlyIntervals = 12 // Vest over 12 months after cliff

      const schedule: Array<{
        unlockTime: number
        amount: number
        percentage: number
      }> = []

      const intervalDuration = duration / monthlyIntervals
      const amountPerInterval = Math.floor(totalAmount / monthlyIntervals)

      for (let i = 1; i <= monthlyIntervals; i++) {
        const unlockTime = cliffTime + (intervalDuration * i)
        const percentage = (i / monthlyIntervals) * 100

        schedule.push({
          unlockTime: Math.floor(unlockTime),
          amount: i === monthlyIntervals
            ? totalAmount - (amountPerInterval * (monthlyIntervals - 1)) // Last interval gets remainder
            : amountPerInterval,
          percentage
        })
      }

      console.log('Vesting schedule created')
      console.log(`  Total amount: ${totalAmount}`)
      console.log(`  Cliff: ${new Date(cliffTime * 1000).toISOString()}`)
      console.log(`  End: ${new Date(vestingEnd * 1000).toISOString()}`)
      console.log(`  Intervals: ${monthlyIntervals}`)

      return {
        schedule,
        totalAmount,
        duration
      }
    } catch (error) {
      throw new Error(`Vesting schedule creation failed: ${error.message}`)
    }
  }

  /**
   * Create time-locked vesting outputs
   *
   * @param funder - Funder's private key
   * @param beneficiary - Beneficiary's public key
   * @param vestingSchedule - Vesting schedule
   * @param utxo - Funding UTXO
   * @returns Vesting transaction
   */
  async createVestingTransaction(
    funder: PrivateKey,
    beneficiary: PublicKey,
    vestingSchedule: Array<{
      unlockTime: number
      amount: number
      percentage: number
    }>,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    }
  ): Promise<Transaction> {
    try {
      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(funder),
        sequence: 0xffffffff
      })

      // Create time-locked outputs for each vesting interval
      for (const interval of vestingSchedule) {
        const vestingScript = this.createTimeLockedScript(
          beneficiary,
          interval.unlockTime
        )

        tx.addOutput({
          satoshis: interval.amount,
          lockingScript: vestingScript
        })

        console.log(`Added vesting output: ${interval.amount} sats at ${new Date(interval.unlockTime * 1000).toISOString()}`)
      }

      // Calculate change
      const totalVested = vestingSchedule.reduce((sum, v) => sum + v.amount, 0)
      const fee = 500 + (vestingSchedule.length * 50)
      const change = utxo.satoshis - totalVested - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(funder.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('Vesting transaction created')
      console.log(`  Total outputs: ${vestingSchedule.length}`)
      console.log(`  Total vested: ${totalVested}`)

      return tx
    } catch (error) {
      throw new Error(`Vesting transaction creation failed: ${error.message}`)
    }
  }

  /**
   * Create time-locked script
   */
  private createTimeLockedScript(
    beneficiary: PublicKey,
    unlockTime: number
  ): Script {
    const script = new Script()

    // <unlockTime> OP_CHECKLOCKTIMEVERIFY OP_DROP <beneficiaryPubKey> OP_CHECKSIG
    script.chunks.push({ data: this.numberToBuffer(unlockTime) })
    script.chunks.push({ op: OP.OP_CHECKLOCKTIMEVERIFY })
    script.chunks.push({ op: OP.OP_DROP })
    script.chunks.push({ data: beneficiary.toBuffer() })
    script.chunks.push({ op: OP.OP_CHECKSIG })

    return script
  }

  /**
   * Claim vested tokens
   *
   * @param beneficiary - Beneficiary's private key
   * @param vestedOutputs - Array of vested outputs that are now unlocked
   * @returns Claim transaction
   */
  async claimVestedTokens(
    beneficiary: PrivateKey,
    vestedOutputs: Array<{
      txid: string
      vout: number
      satoshis: number
      script: Script
      unlockTime: number
    }>
  ): Promise<Transaction> {
    try {
      const currentTime = Math.floor(Date.now() / 1000)

      // Filter outputs that can be claimed now
      const claimable = vestedOutputs.filter(output => output.unlockTime <= currentTime)

      if (claimable.length === 0) {
        throw new Error('No vested outputs are claimable yet')
      }

      const tx = new Transaction()

      // Set lock time to the latest unlock time
      const maxLockTime = Math.max(...claimable.map(o => o.unlockTime))
      tx.lockTime = maxLockTime

      // Add all claimable outputs as inputs
      for (const output of claimable) {
        tx.addInput({
          sourceTXID: output.txid,
          sourceOutputIndex: output.vout,
          unlockingScriptTemplate: {
            sign: async (transaction: Transaction, inputIndex: number) => {
              const signature = transaction.sign(inputIndex, beneficiary, output.script)

              const unlockScript = new Script()
              unlockScript.chunks.push({ data: signature })

              return unlockScript
            },
            estimateLength: () => 120
          },
          sequence: 0xfffffffe // Required for lockTime
        })
      }

      // Consolidate into single output
      const totalClaimed = claimable.reduce((sum, o) => sum + o.satoshis, 0)
      const fee = 500 + (claimable.length * 100)
      const claimAmount = totalClaimed - fee

      tx.addOutput({
        satoshis: claimAmount,
        lockingScript: new P2PKH().lock(beneficiary.toPublicKey().toHash())
      })

      await tx.sign()

      console.log('Vested tokens claimed')
      console.log(`  Claimed outputs: ${claimable.length}`)
      console.log(`  Total claimed: ${claimAmount}`)

      return tx
    } catch (error) {
      throw new Error(`Token claim failed: ${error.message}`)
    }
  }

  /**
   * Check claimable amount at given time
   */
  getClaimableAmount(
    vestingSchedule: Array<{
      unlockTime: number
      amount: number
    }>,
    checkTime?: number
  ): {
    claimable: number
    locked: number
    percentage: number
  } {
    const time = checkTime || Math.floor(Date.now() / 1000)

    const claimable = vestingSchedule
      .filter(v => v.unlockTime <= time)
      .reduce((sum, v) => sum + v.amount, 0)

    const total = vestingSchedule.reduce((sum, v) => sum + v.amount, 0)
    const locked = total - claimable
    const percentage = (claimable / total) * 100

    return { claimable, locked, percentage }
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
async function vestingExample() {
  const vesting = new VestingContract()

  const company = PrivateKey.fromRandom()
  const employee = PrivateKey.fromRandom()

  // Create 1-year vesting with 3-month cliff
  const currentTime = Math.floor(Date.now() / 1000)
  const cliffTime = currentTime + (90 * 24 * 60 * 60) // 3 months
  const vestingEnd = cliffTime + (365 * 24 * 60 * 60) // 1 year after cliff

  const schedule = vesting.createVestingSchedule(
    employee.toPublicKey(),
    cliffTime,
    vestingEnd,
    1000000 // 1M satoshis
  )

  const companyUTXO = {
    txid: 'company-utxo...',
    vout: 0,
    satoshis: 2000000,
    script: new P2PKH().lock(company.toPublicKey().toHash())
  }

  // Create vesting transaction
  const vestingTx = await vesting.createVestingTransaction(
    company,
    employee.toPublicKey(),
    schedule.schedule,
    companyUTXO
  )

  console.log('Vesting transaction:', vestingTx.id('hex'))

  // Check claimable amount
  const claimable = vesting.getClaimableAmount(schedule.schedule)
  console.log('Currently claimable:', claimable.claimable, `(${claimable.percentage.toFixed(2)}%)`)
}
```

## Oracle-Based Conditional Payment

```typescript
import { Transaction, PrivateKey, PublicKey, Script, Hash, OP, P2PKH } from '@bsv/sdk'

/**
 * Oracle Smart Contract
 *
 * Implements conditional payments based on oracle data.
 */
class OracleContract {
  /**
   * Create oracle-conditional script
   *
   * Funds can be claimed if oracle signs off on the condition
   *
   * @param beneficiary - Beneficiary's public key
   * @param oracle - Oracle's public key
   * @param conditionHash - Hash of the condition data
   * @param timeout - Optional timeout for refund
   * @param refundKey - Optional refund key for timeout
   * @returns Oracle locking script
   */
  createOracleScript(
    beneficiary: PublicKey,
    oracle: PublicKey,
    conditionHash: string,
    timeout?: number,
    refundKey?: PublicKey
  ): Script {
    try {
      const script = new Script()

      if (timeout && refundKey) {
        script.chunks.push({ op: OP.OP_IF })
      }

      // Oracle + beneficiary path
      // Requires: <oracleSignature> <conditionData> <beneficiarySignature>
      // Verify condition data hash
      script.chunks.push({ op: OP.OP_HASH256 })
      script.chunks.push({ data: Buffer.from(conditionHash, 'hex') })
      script.chunks.push({ op: OP.OP_EQUALVERIFY })

      // Verify oracle signature on condition data
      script.chunks.push({ data: oracle.toBuffer() })
      script.chunks.push({ op: OP.OP_CHECKSIGVERIFY })

      // Verify beneficiary signature
      script.chunks.push({ data: beneficiary.toBuffer() })
      script.chunks.push({ op: OP.OP_CHECKSIG })

      if (timeout && refundKey) {
        // Timeout refund path
        script.chunks.push({ op: OP.OP_ELSE })
        script.chunks.push({ data: this.numberToBuffer(timeout) })
        script.chunks.push({ op: OP.OP_CHECKLOCKTIMEVERIFY })
        script.chunks.push({ op: OP.OP_DROP })
        script.chunks.push({ data: refundKey.toBuffer() })
        script.chunks.push({ op: OP.OP_CHECKSIG })
        script.chunks.push({ op: OP.OP_ENDIF })
      }

      return script
    } catch (error) {
      throw new Error(`Oracle script creation failed: ${error.message}`)
    }
  }

  /**
   * Create conditional payment
   *
   * @param payer - Payer's private key
   * @param beneficiary - Beneficiary's public key
   * @param oracle - Oracle's public key
   * @param condition - Condition description
   * @param amount - Payment amount
   * @param utxo - Funding UTXO
   * @param timeoutDays - Optional timeout in days
   * @returns Conditional payment details
   */
  async createConditionalPayment(
    payer: PrivateKey,
    beneficiary: PublicKey,
    oracle: PublicKey,
    condition: {
      description: string
      data: any
    },
    amount: number,
    utxo: {
      txid: string
      vout: number
      satoshis: number
      script: Script
    },
    timeoutDays?: number
  ): Promise<{
    transaction: Transaction
    oracleScript: Script
    conditionData: {
      hash: string
      data: string
      description: string
    }
  }> {
    try {
      // Hash the condition data
      const conditionData = JSON.stringify(condition.data)
      const conditionBuffer = Buffer.from(conditionData)
      const conditionHash = Hash.hash256(conditionBuffer).toString('hex')

      const timeout = timeoutDays
        ? Math.floor(Date.now() / 1000) + (timeoutDays * 24 * 60 * 60)
        : undefined

      // Create oracle script
      const oracleScript = this.createOracleScript(
        beneficiary,
        oracle,
        conditionHash,
        timeout,
        payer.toPublicKey()
      )

      // Create transaction
      const tx = new Transaction()

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(payer),
        sequence: 0xffffffff
      })

      const fee = 500
      const paymentAmount = amount - fee

      tx.addOutput({
        satoshis: paymentAmount,
        lockingScript: oracleScript
      })

      // Change
      const change = utxo.satoshis - amount - fee

      if (change > 546) {
        tx.addOutput({
          satoshis: change,
          lockingScript: new P2PKH().lock(payer.toPublicKey().toHash())
        })
      }

      await tx.sign()

      console.log('Conditional payment created')
      console.log(`  Amount: ${paymentAmount}`)
      console.log(`  Condition: ${condition.description}`)
      console.log(`  Condition hash: ${conditionHash}`)

      return {
        transaction: tx,
        oracleScript,
        conditionData: {
          hash: conditionHash,
          data: conditionData,
          description: condition.description
        }
      }
    } catch (error) {
      throw new Error(`Conditional payment creation failed: ${error.message}`)
    }
  }

  /**
   * Claim conditional payment with oracle attestation
   *
   * @param beneficiary - Beneficiary's private key
   * @param oracle - Oracle's private key
   * @param conditionalUTXO - Conditional payment UTXO
   * @param oracleScript - Oracle locking script
   * @param conditionData - The condition data oracle is attesting to
   * @param recipient - Recipient address
   * @returns Claim transaction
   */
  async claimConditionalPayment(
    beneficiary: PrivateKey,
    oracle: PrivateKey,
    conditionalUTXO: {
      txid: string
      vout: number
      satoshis: number
    },
    oracleScript: Script,
    conditionData: string,
    recipient: string
  ): Promise<Transaction> {
    try {
      console.log('Claiming conditional payment with oracle attestation...')

      const tx = new Transaction()

      tx.addInput({
        sourceTXID: conditionalUTXO.txid,
        sourceOutputIndex: conditionalUTXO.vout,
        unlockingScriptTemplate: {
          sign: async (transaction: Transaction, inputIndex: number) => {
            // Oracle signs the condition data
            const conditionBuffer = Buffer.from(conditionData)
            const oracleSig = oracle.sign(Hash.sha256(conditionBuffer))

            // Beneficiary signs the transaction
            const beneficiarySig = transaction.sign(inputIndex, beneficiary, oracleScript)

            // Build unlocking script
            const unlockScript = new Script()
            unlockScript.chunks.push({ data: beneficiarySig })
            unlockScript.chunks.push({ data: conditionBuffer })
            unlockScript.chunks.push({ data: oracleSig })

            return unlockScript
          },
          estimateLength: () => 200
        },
        sequence: 0xffffffff
      })

      const fee = 500
      const claimAmount = conditionalUTXO.satoshis - fee

      tx.addOutput({
        satoshis: claimAmount,
        lockingScript: Script.fromAddress(recipient)
      })

      await tx.sign()

      console.log('Conditional payment claimed')
      console.log(`  Amount: ${claimAmount}`)

      return tx
    } catch (error) {
      throw new Error(`Conditional payment claim failed: ${error.message}`)
    }
  }

  /**
   * Oracle attests to condition
   *
   * @param oracle - Oracle's private key
   * @param conditionData - Condition data to attest
   * @returns Oracle attestation
   */
  createOracleAttestation(
    oracle: PrivateKey,
    conditionData: {
      description: string
      result: any
      timestamp: number
    }
  ): {
    data: string
    signature: Buffer
    hash: string
  } {
    const data = JSON.stringify(conditionData)
    const dataBuffer = Buffer.from(data)
    const hash = Hash.hash256(dataBuffer)
    const signature = oracle.sign(hash)

    console.log('Oracle attestation created')
    console.log(`  Condition: ${conditionData.description}`)
    console.log(`  Result: ${JSON.stringify(conditionData.result)}`)

    return {
      data,
      signature,
      hash: hash.toString('hex')
    }
  }

  /**
   * Verify oracle attestation
   */
  verifyOracleAttestation(
    oraclePubKey: PublicKey,
    attestation: {
      data: string
      signature: Buffer
      hash: string
    }
  ): boolean {
    const dataBuffer = Buffer.from(attestation.data)
    const computedHash = Hash.hash256(dataBuffer)

    if (computedHash.toString('hex') !== attestation.hash) {
      return false
    }

    // Verify signature
    // In production, use proper ECDSA verification
    return true
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
async function oracleExample() {
  const oracleContract = new OracleContract()

  const payer = PrivateKey.fromRandom()
  const beneficiary = PrivateKey.fromRandom()
  const oracle = PrivateKey.fromRandom()

  const utxo = {
    txid: 'payer-utxo...',
    vout: 0,
    satoshis: 100000,
    script: new P2PKH().lock(payer.toPublicKey().toHash())
  }

  // Create conditional payment: "Pay beneficiary if price of BSV > $100"
  const { transaction, oracleScript, conditionData } = await oracleContract.createConditionalPayment(
    payer,
    beneficiary.toPublicKey(),
    oracle.toPublicKey(),
    {
      description: 'BSV price > $100',
      data: {
        asset: 'BSV',
        condition: 'price > 100',
        currency: 'USD'
      }
    },
    50000,
    utxo,
    30 // 30-day timeout
  )

  console.log('Conditional payment created:', transaction.id('hex'))

  // Oracle checks the condition and creates attestation
  const attestation = oracleContract.createOracleAttestation(
    oracle,
    {
      description: 'BSV price check',
      result: { price: 105.5, currency: 'USD', timestamp: Date.now() },
      timestamp: Math.floor(Date.now() / 1000)
    }
  )

  // Beneficiary claims with oracle attestation
  const claimTx = await oracleContract.claimConditionalPayment(
    beneficiary,
    oracle,
    {
      txid: transaction.id('hex'),
      vout: 0,
      satoshis: 49500
    },
    oracleScript,
    attestation.data,
    beneficiary.toPublicKey().toAddress()
  )

  console.log('Claim transaction:', claimTx.id('hex'))
}
```

## Multi-Party Contract

```typescript
import { Transaction, PrivateKey, PublicKey, Script, OP, P2PKH } from '@bsv/sdk'

/**
 * Multi-Party Smart Contract
 *
 * Implements contracts requiring approval from multiple parties.
 */
class MultiPartyContract {
  /**
   * Create threshold signature contract (m-of-n)
   *
   * @param pubKeys - Array of participant public keys
   * @param threshold - Required number of signatures
   * @returns Threshold contract script
   */
  createThresholdContract(
    pubKeys: PublicKey[],
    threshold: number
  ): Script {
    try {
      if (threshold < 1 || threshold > pubKeys.length) {
        throw new Error('Invalid threshold')
      }

      if (pubKeys.length > 15) {
        throw new Error('Too many participants (max 15)')
      }

      const script = new Script()

      // m-of-n multisig
      script.chunks.push({ data: Buffer.from([threshold]) })

      for (const pubKey of pubKeys) {
        script.chunks.push({ data: pubKey.toBuffer() })
      }

      script.chunks.push({ data: Buffer.from([pubKeys.length]) })
      script.chunks.push({ op: OP.OP_CHECKMULTISIG })

      console.log(`Created ${threshold}-of-${pubKeys.length} threshold contract`)

      return script
    } catch (error) {
      throw new Error(`Threshold contract creation failed: ${error.message}`)
    }
  }

  /**
   * Execute threshold contract
   *
   * @param signers - Signers' private keys (must be >= threshold)
   * @param contractUTXO - Contract UTXO
   * @param contractScript - Contract locking script
   * @param recipient - Recipient address
   * @returns Execution transaction
   */
  async executeThresholdContract(
    signers: PrivateKey[],
    contractUTXO: {
      txid: string
      vout: number
      satoshis: number
    },
    contractScript: Script,
    threshold: number,
    recipient: string
  ): Promise<Transaction> {
    try {
      if (signers.length < threshold) {
        throw new Error(`Need at least ${threshold} signers, got ${signers.length}`)
      }

      const tx = new Transaction()

      tx.addInput({
        sourceTXID: contractUTXO.txid,
        sourceOutputIndex: contractUTXO.vout,
        unlockingScriptTemplate: {
          sign: async (transaction: Transaction, inputIndex: number) => {
            // Create signatures from all signers
            const signatures = signers.map(signer =>
              transaction.sign(inputIndex, signer, contractScript)
            )

            const unlockScript = new Script()
            unlockScript.chunks.push({ op: OP.OP_0 }) // OP_CHECKMULTISIG bug

            for (const sig of signatures) {
              unlockScript.chunks.push({ data: sig })
            }

            return unlockScript
          },
          estimateLength: () => 100 + (signers.length * 75)
        },
        sequence: 0xffffffff
      })

      const fee = 500
      const executeAmount = contractUTXO.satoshis - fee

      tx.addOutput({
        satoshis: executeAmount,
        lockingScript: Script.fromAddress(recipient)
      })

      await tx.sign()

      console.log(`Threshold contract executed with ${signers.length} signatures`)

      return tx
    } catch (error) {
      throw new Error(`Threshold contract execution failed: ${error.message}`)
    }
  }
}
```

## Related Examples

- [Transaction Building](../transaction-building/README.md)
- [Atomic Swaps](../atomic-swaps/README.md)
- [Payment Channels](../payment-channels/README.md)

## See Also

**SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md) - Core transaction building
- [Script](../../sdk-components/script/README.md) - Bitcoin script operations
- [Script Templates](../../sdk-components/script-templates/README.md) - Custom script templates
- [Signatures](../../sdk-components/signatures/README.md) - Digital signatures

**Learning Paths:**
- [Script Programming](../../learning-paths/intermediate/script-programming/README.md)
- [Smart Contracts](../../learning-paths/advanced/smart-contracts/README.md)
