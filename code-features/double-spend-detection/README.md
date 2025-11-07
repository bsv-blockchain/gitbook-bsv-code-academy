# Double-Spend Detection

Complete examples for detecting and handling double-spend attempts in BSV applications.

## Overview

Double-spending is when someone attempts to spend the same UTXO in multiple transactions. While the blockchain prevents double-spends from being confirmed, applications need to detect attempts early to prevent fraud. This guide covers detection strategies, monitoring, and mitigation techniques for zero-confirmation and confirmed transactions.

**Related SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md)
- [ARC](../../sdk-components/arc/README.md)
- [SPV](../../sdk-components/spv/README.md)
- [UTXO Management](../../sdk-components/utxo-management/README.md)

## UTXO Tracking and Monitoring

```typescript
import { Transaction, ARC } from '@bsv/sdk'

/**
 * UTXO Monitor
 *
 * Track UTXOs and detect double-spend attempts
 */
class UTXOMonitor {
  private trackedUTXOs: Map<string, TrackedUTXO> = new Map()
  private arc: ARC

  constructor(arcUrl: string = 'https://arc.taal.com') {
    this.arc = new ARC(arcUrl)
  }

  /**
   * Track a UTXO for double-spend detection
   */
  trackUTXO(utxo: {
    txid: string
    vout: number
    satoshis: number
    address: string
  }): void {
    const utxoKey = this.getUTXOKey(utxo.txid, utxo.vout)

    const tracked: TrackedUTXO = {
      txid: utxo.txid,
      vout: utxo.vout,
      satoshis: utxo.satoshis,
      address: utxo.address,
      status: 'unspent',
      trackedAt: Date.now(),
      spendingTxids: []
    }

    this.trackedUTXOs.set(utxoKey, tracked)

    console.log('Tracking UTXO:', utxoKey)
  }

  /**
   * Check if UTXO has been spent
   */
  async checkUTXOStatus(txid: string, vout: number): Promise<UTXOStatus> {
    try {
      const utxoKey = this.getUTXOKey(txid, vout)
      const tracked = this.trackedUTXOs.get(utxoKey)

      if (!tracked) {
        throw new Error('UTXO not tracked')
      }

      // Query transaction status
      const txStatus = await this.arc.getTransactionStatus(txid)

      // Check if any spending transactions exist
      const spendingTxs = await this.findSpendingTransactions(txid, vout)

      const status: UTXOStatus = {
        utxoKey,
        spent: spendingTxs.length > 0,
        spendingTxids: spendingTxs.map(tx => tx.txid),
        doubleSpendDetected: spendingTxs.length > 1,
        confirmedSpend: spendingTxs.some(tx => tx.confirmed),
        timestamp: Date.now()
      }

      // Update tracked UTXO
      if (status.spent) {
        tracked.status = status.doubleSpendDetected ? 'double-spend' : 'spent'
        tracked.spendingTxids = status.spendingTxids
      }

      if (status.doubleSpendDetected) {
        console.warn('DOUBLE-SPEND DETECTED:', utxoKey)
        console.warn('Spending transactions:', status.spendingTxids)
      }

      return status
    } catch (error) {
      throw new Error(`UTXO status check failed: ${error.message}`)
    }
  }

  /**
   * Find all transactions spending a UTXO
   */
  private async findSpendingTransactions(
    txid: string,
    vout: number
  ): Promise<Array<{ txid: string; confirmed: boolean }>> {
    try {
      // In production, query mempool and blockchain for spending txs
      // This is a simplified example
      const spendingTxs: Array<{ txid: string; confirmed: boolean }> = []

      // Query ARC for spending transactions
      // Note: Actual implementation depends on ARC API capabilities

      return spendingTxs
    } catch (error) {
      console.error('Error finding spending transactions:', error.message)
      return []
    }
  }

  /**
   * Monitor tracked UTXOs continuously
   */
  async startMonitoring(
    callback: (alert: DoubleSpendAlert) => void,
    interval: number = 5000
  ): Promise<void> {
    console.log('Starting UTXO monitoring...')

    const monitor = async () => {
      for (const [utxoKey, tracked] of this.trackedUTXOs.entries()) {
        if (tracked.status === 'unspent') {
          try {
            const status = await this.checkUTXOStatus(tracked.txid, tracked.vout)

            if (status.doubleSpendDetected) {
              const alert: DoubleSpendAlert = {
                utxoKey,
                txid: tracked.txid,
                vout: tracked.vout,
                spendingTxids: status.spendingTxids,
                detectedAt: Date.now(),
                severity: status.confirmedSpend ? 'high' : 'medium'
              }

              callback(alert)
            }
          } catch (error) {
            console.error('Monitoring error for', utxoKey, ':', error.message)
          }
        }
      }

      setTimeout(monitor, interval)
    }

    monitor()
  }

  /**
   * Remove UTXO from tracking
   */
  untrackUTXO(txid: string, vout: number): void {
    const utxoKey = this.getUTXOKey(txid, vout)
    this.trackedUTXOs.delete(utxoKey)
    console.log('Stopped tracking:', utxoKey)
  }

  /**
   * Get UTXO key
   */
  private getUTXOKey(txid: string, vout: number): string {
    return `${txid}:${vout}`
  }

  /**
   * Get all tracked UTXOs
   */
  getTrackedUTXOs(): TrackedUTXO[] {
    return Array.from(this.trackedUTXOs.values())
  }
}

interface TrackedUTXO {
  txid: string
  vout: number
  satoshis: number
  address: string
  status: 'unspent' | 'spent' | 'double-spend'
  trackedAt: number
  spendingTxids: string[]
}

interface UTXOStatus {
  utxoKey: string
  spent: boolean
  spendingTxids: string[]
  doubleSpendDetected: boolean
  confirmedSpend: boolean
  timestamp: number
}

interface DoubleSpendAlert {
  utxoKey: string
  txid: string
  vout: number
  spendingTxids: string[]
  detectedAt: number
  severity: 'low' | 'medium' | 'high'
}

/**
 * Usage Example
 */
async function utxoMonitorExample() {
  const monitor = new UTXOMonitor()

  // Track UTXO
  monitor.trackUTXO({
    txid: 'utxo-txid...',
    vout: 0,
    satoshis: 100000,
    address: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa'
  })

  // Check status
  const status = await monitor.checkUTXOStatus('utxo-txid...', 0)
  console.log('UTXO status:', status)

  // Start continuous monitoring
  await monitor.startMonitoring((alert) => {
    console.error('DOUBLE-SPEND ALERT:', alert)
    // Take action: notify admin, block payment, etc.
  })
}
```

## Transaction Conflict Detection

```typescript
import { Transaction } from '@bsv/sdk'

/**
 * Transaction Conflict Detector
 *
 * Detect conflicts between transactions
 */
class ConflictDetector {
  /**
   * Check if two transactions conflict (spend same inputs)
   */
  detectConflict(tx1: Transaction, tx2: Transaction): ConflictResult {
    const tx1Inputs = this.getInputKeys(tx1)
    const tx2Inputs = this.getInputKeys(tx2)

    const conflicts: string[] = []

    for (const input of tx1Inputs) {
      if (tx2Inputs.includes(input)) {
        conflicts.push(input)
      }
    }

    const result: ConflictResult = {
      hasConflict: conflicts.length > 0,
      conflictingInputs: conflicts,
      tx1Id: tx1.id('hex'),
      tx2Id: tx2.id('hex'),
      timestamp: Date.now()
    }

    if (result.hasConflict) {
      console.warn('CONFLICT DETECTED')
      console.warn('Transaction 1:', result.tx1Id)
      console.warn('Transaction 2:', result.tx2Id)
      console.warn('Conflicting inputs:', conflicts)
    }

    return result
  }

  /**
   * Check transaction against known transactions
   */
  checkAgainstKnown(
    tx: Transaction,
    knownTransactions: Transaction[]
  ): ConflictSummary {
    const conflicts: ConflictResult[] = []

    for (const knownTx of knownTransactions) {
      const conflict = this.detectConflict(tx, knownTx)
      if (conflict.hasConflict) {
        conflicts.push(conflict)
      }
    }

    return {
      txid: tx.id('hex'),
      conflictCount: conflicts.length,
      conflicts,
      riskLevel: this.assessRisk(conflicts.length)
    }
  }

  /**
   * Detect double-spend in transaction set
   */
  detectDoubleSpends(transactions: Transaction[]): DoubleSpendReport {
    const inputUsage = new Map<string, string[]>() // input -> [txids]

    // Track which transactions use which inputs
    for (const tx of transactions) {
      const txid = tx.id('hex')
      const inputs = this.getInputKeys(tx)

      for (const input of inputs) {
        if (!inputUsage.has(input)) {
          inputUsage.set(input, [])
        }
        inputUsage.get(input)!.push(txid)
      }
    }

    // Find inputs used by multiple transactions
    const doubleSpends: DoubleSpend[] = []

    for (const [input, txids] of inputUsage.entries()) {
      if (txids.length > 1) {
        doubleSpends.push({
          input,
          conflictingTxids: txids,
          count: txids.length
        })
      }
    }

    const report: DoubleSpendReport = {
      totalTransactions: transactions.length,
      doubleSpendCount: doubleSpends.length,
      doubleSpends,
      timestamp: Date.now()
    }

    if (doubleSpends.length > 0) {
      console.warn(`DETECTED ${doubleSpends.length} DOUBLE-SPEND ATTEMPTS`)
      for (const ds of doubleSpends) {
        console.warn(`  Input ${ds.input} spent by ${ds.count} transactions`)
      }
    }

    return report
  }

  /**
   * Get input keys from transaction
   */
  private getInputKeys(tx: Transaction): string[] {
    return tx.inputs.map(input => `${input.sourceTXID}:${input.sourceOutputIndex}`)
  }

  /**
   * Assess risk level based on conflict count
   */
  private assessRisk(conflictCount: number): 'low' | 'medium' | 'high' {
    if (conflictCount === 0) return 'low'
    if (conflictCount === 1) return 'medium'
    return 'high'
  }
}

interface ConflictResult {
  hasConflict: boolean
  conflictingInputs: string[]
  tx1Id: string
  tx2Id: string
  timestamp: number
}

interface ConflictSummary {
  txid: string
  conflictCount: number
  conflicts: ConflictResult[]
  riskLevel: 'low' | 'medium' | 'high'
}

interface DoubleSpend {
  input: string
  conflictingTxids: string[]
  count: number
}

interface DoubleSpendReport {
  totalTransactions: number
  doubleSpendCount: number
  doubleSpends: DoubleSpend[]
  timestamp: number
}

/**
 * Usage Example
 */
function conflictDetectionExample() {
  const detector = new ConflictDetector()

  const tx1 = new Transaction() // ... transaction 1 ...
  const tx2 = new Transaction() // ... transaction 2 ...

  // Check for conflicts
  const conflict = detector.detectConflict(tx1, tx2)

  if (conflict.hasConflict) {
    console.error('Transactions conflict!')
  }

  // Check against known transactions
  const knownTxs: Transaction[] = [/* ... */]
  const summary = detector.checkAgainstKnown(tx1, knownTxs)
  console.log('Conflict summary:', summary)

  // Detect double-spends in batch
  const transactions: Transaction[] = [/* ... */]
  const report = detector.detectDoubleSpends(transactions)
  console.log('Double-spend report:', report)
}
```

## Payment Security Manager

```typescript
import { Transaction, ARC } from '@bsv/sdk'

/**
 * Payment Security Manager
 *
 * Secure payment handling with double-spend protection
 */
class PaymentSecurityManager {
  private arc: ARC
  private pendingPayments: Map<string, PendingPayment> = new Map()

  constructor(arcUrl: string = 'https://arc.taal.com') {
    this.arc = new ARC(arcUrl)
  }

  /**
   * Accept payment with security checks
   */
  async acceptPayment(
    tx: Transaction,
    requiredAmount: number,
    recipientAddress: string,
    options: {
      requireConfirmations?: number
      allowZeroConf?: boolean
      timeout?: number
    } = {}
  ): Promise<PaymentResult> {
    try {
      const {
        requireConfirmations = 1,
        allowZeroConf = false,
        timeout = 60000
      } = options

      const txid = tx.id('hex')

      console.log('Accepting payment:', txid)
      console.log('Required amount:', requiredAmount)
      console.log('Recipient:', recipientAddress)

      // Validate payment amount
      const validation = this.validatePaymentAmount(tx, recipientAddress, requiredAmount)

      if (!validation.valid) {
        return {
          accepted: false,
          txid,
          reason: validation.reason,
          timestamp: Date.now()
        }
      }

      // Check for conflicts with pending payments
      const conflict = this.checkPendingConflicts(tx)

      if (conflict) {
        console.warn('Payment conflicts with pending transaction:', conflict)
        return {
          accepted: false,
          txid,
          reason: 'Conflicts with pending payment',
          conflictingTxid: conflict,
          timestamp: Date.now()
        }
      }

      // Broadcast transaction
      try {
        await this.arc.broadcastTransaction(tx)
      } catch (error) {
        return {
          accepted: false,
          txid,
          reason: `Broadcast failed: ${error.message}`,
          timestamp: Date.now()
        }
      }

      // Add to pending payments
      const pending: PendingPayment = {
        txid,
        amount: requiredAmount,
        recipientAddress,
        acceptedAt: Date.now(),
        confirmations: 0,
        status: 'pending'
      }

      this.pendingPayments.set(txid, pending)

      // If zero-conf allowed and no confirmations required
      if (allowZeroConf && requireConfirmations === 0) {
        pending.status = 'accepted'
        return {
          accepted: true,
          txid,
          confirmations: 0,
          timestamp: Date.now()
        }
      }

      // Wait for confirmations
      try {
        const status = await this.waitForConfirmations(
          txid,
          requireConfirmations,
          timeout
        )

        pending.confirmations = status.confirmations
        pending.status = 'confirmed'

        return {
          accepted: true,
          txid,
          confirmations: status.confirmations,
          blockHeight: status.blockHeight,
          timestamp: Date.now()
        }
      } catch (error) {
        pending.status = 'failed'
        return {
          accepted: false,
          txid,
          reason: error.message,
          timestamp: Date.now()
        }
      }
    } catch (error) {
      return {
        accepted: false,
        txid: tx.id('hex'),
        reason: `Payment acceptance failed: ${error.message}`,
        timestamp: Date.now()
      }
    }
  }

  /**
   * Validate payment amount
   */
  private validatePaymentAmount(
    tx: Transaction,
    recipientAddress: string,
    requiredAmount: number
  ): { valid: boolean; reason?: string } {
    let totalReceived = 0

    for (const output of tx.outputs) {
      try {
        const outputAddress = output.lockingScript.toAddress()
        if (outputAddress === recipientAddress) {
          totalReceived += output.satoshis
        }
      } catch {
        // Skip outputs that don't have addresses
        continue
      }
    }

    if (totalReceived < requiredAmount) {
      return {
        valid: false,
        reason: `Insufficient amount: received ${totalReceived}, required ${requiredAmount}`
      }
    }

    return { valid: true }
  }

  /**
   * Check if transaction conflicts with pending payments
   */
  private checkPendingConflicts(tx: Transaction): string | null {
    const inputKeys = new Set(
      tx.inputs.map(input => `${input.sourceTXID}:${input.sourceOutputIndex}`)
    )

    for (const [pendingTxid, pending] of this.pendingPayments.entries()) {
      if (pending.status === 'pending') {
        // In production, compare actual inputs
        // This is simplified
      }
    }

    return null
  }

  /**
   * Wait for transaction confirmations
   */
  private async waitForConfirmations(
    txid: string,
    required: number,
    timeout: number
  ): Promise<{ confirmations: number; blockHeight?: number }> {
    const startTime = Date.now()

    while (Date.now() - startTime < timeout) {
      try {
        const status = await this.arc.getTransactionStatus(txid)

        const confirmations = status.confirmations || 0

        if (confirmations >= required) {
          return {
            confirmations,
            blockHeight: status.blockHeight
          }
        }

        // Wait before checking again
        await new Promise(resolve => setTimeout(resolve, 5000))
      } catch (error) {
        console.error('Confirmation check error:', error.message)
      }
    }

    throw new Error('Confirmation timeout')
  }

  /**
   * Get pending payments
   */
  getPendingPayments(): PendingPayment[] {
    return Array.from(this.pendingPayments.values())
  }

  /**
   * Clear completed payments
   */
  clearCompletedPayments(): void {
    for (const [txid, payment] of this.pendingPayments.entries()) {
      if (payment.status === 'confirmed' || payment.status === 'failed') {
        this.pendingPayments.delete(txid)
      }
    }
  }
}

interface PendingPayment {
  txid: string
  amount: number
  recipientAddress: string
  acceptedAt: number
  confirmations: number
  status: 'pending' | 'accepted' | 'confirmed' | 'failed'
}

interface PaymentResult {
  accepted: boolean
  txid: string
  confirmations?: number
  blockHeight?: number
  reason?: string
  conflictingTxid?: string
  timestamp: number
}

/**
 * Usage Example
 */
async function paymentSecurityExample() {
  const security = new PaymentSecurityManager()

  const paymentTx = new Transaction()
  // ... payment transaction ...

  // Accept payment with 1 confirmation required
  const result = await security.acceptPayment(
    paymentTx,
    50000,
    '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
    {
      requireConfirmations: 1,
      allowZeroConf: false,
      timeout: 120000 // 2 minutes
    }
  )

  if (result.accepted) {
    console.log('Payment accepted!')
    console.log('Confirmations:', result.confirmations)
  } else {
    console.error('Payment rejected:', result.reason)
  }

  // Get pending payments
  const pending = security.getPendingPayments()
  console.log('Pending payments:', pending.length)
}
```

## Related Examples

- [Transaction Broadcasting](../transaction-broadcasting/README.md)
- [UTXO Management](../utxo-management/README.md)
- [SPV Verification](../spv-verification/README.md)
- [Marketplace](../marketplace/README.md)

## See Also

**SDK Components:**
- [Transaction](../../sdk-components/transaction/README.md) - Transaction handling
- [ARC](../../sdk-components/arc/README.md) - Transaction broadcasting and monitoring
- [SPV](../../sdk-components/spv/README.md) - SPV validation
- [UTXO Management](../../sdk-components/utxo-management/README.md) - UTXO tracking

**Learning Paths:**
- [Payment Security](../../learning-paths/intermediate/payment-security/README.md)
