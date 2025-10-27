# Transaction Broadcasting

Complete examples for broadcasting transactions to the BSV network using ARC (Advanced Repository for Chains).

## Overview

Broadcasting transactions is the final step in getting your transactions confirmed on the BSV blockchain. ARC is the modern transaction processing and broadcast API for BSV, replacing the legacy merchant API. This guide covers broadcast patterns, error handling, status monitoring, and broadcast optimization.

**Related SDK Components:**
- [ARC](../../sdk-components/arc/README.md)
- [Transaction](../../sdk-components/transaction/README.md)
- [BEEF](../../sdk-components/beef/README.md)

## Basic Transaction Broadcasting

```typescript
import { Transaction, PrivateKey, P2PKH, ARC } from '@bsv/sdk'

/**
 * Basic Transaction Broadcaster
 *
 * Broadcast transactions using ARC
 */
class TransactionBroadcaster {
  private arcClient: ARC

  constructor(arcUrl: string = 'https://arc.taal.com') {
    this.arcClient = new ARC(arcUrl)
  }

  /**
   * Broadcast a transaction
   */
  async broadcastTransaction(tx: Transaction): Promise<BroadcastResult> {
    try {
      console.log('Broadcasting transaction...')
      console.log('Transaction ID:', tx.id('hex'))
      console.log('Size:', tx.toHex().length / 2, 'bytes')

      // Broadcast to ARC
      const response = await this.arcClient.broadcastTransaction(tx)

      const result: BroadcastResult = {
        txid: response.txid,
        status: response.txStatus,
        timestamp: Date.now(),
        blockHash: response.blockHash,
        blockHeight: response.blockHeight
      }

      console.log('Transaction broadcast successful')
      console.log('Status:', result.status)

      return result
    } catch (error) {
      throw new Error(`Broadcast failed: ${error.message}`)
    }
  }

  /**
   * Broadcast and wait for confirmation
   */
  async broadcastAndWait(
    tx: Transaction,
    maxWaitTime: number = 60000 // 60 seconds default
  ): Promise<BroadcastResult> {
    try {
      const result = await this.broadcastTransaction(tx)

      console.log('Waiting for confirmation...')

      const startTime = Date.now()

      while (Date.now() - startTime < maxWaitTime) {
        const status = await this.checkTransactionStatus(result.txid)

        if (status.status === 'MINED') {
          console.log('Transaction confirmed')
          console.log('Block height:', status.blockHeight)
          return status
        }

        // Wait 2 seconds before next check
        await new Promise(resolve => setTimeout(resolve, 2000))
      }

      throw new Error('Transaction confirmation timeout')
    } catch (error) {
      throw new Error(`Broadcast and wait failed: ${error.message}`)
    }
  }

  /**
   * Check transaction status
   */
  async checkTransactionStatus(txid: string): Promise<BroadcastResult> {
    try {
      const status = await this.arcClient.getTransactionStatus(txid)

      return {
        txid,
        status: status.txStatus,
        timestamp: Date.now(),
        blockHash: status.blockHash,
        blockHeight: status.blockHeight
      }
    } catch (error) {
      throw new Error(`Status check failed: ${error.message}`)
    }
  }

  /**
   * Broadcast multiple transactions
   */
  async broadcastBatch(
    transactions: Transaction[]
  ): Promise<BroadcastResult[]> {
    try {
      console.log('Broadcasting batch of', transactions.length, 'transactions')

      const results: BroadcastResult[] = []

      for (const tx of transactions) {
        try {
          const result = await this.broadcastTransaction(tx)
          results.push(result)
        } catch (error) {
          console.error('Failed to broadcast tx:', tx.id('hex'), error.message)
          results.push({
            txid: tx.id('hex'),
            status: 'FAILED',
            timestamp: Date.now(),
            error: error.message
          })
        }
      }

      console.log('Batch broadcast completed')
      console.log('Successful:', results.filter(r => r.status !== 'FAILED').length)
      console.log('Failed:', results.filter(r => r.status === 'FAILED').length)

      return results
    } catch (error) {
      throw new Error(`Batch broadcast failed: ${error.message}`)
    }
  }
}

interface BroadcastResult {
  txid: string
  status: string
  timestamp: number
  blockHash?: string
  blockHeight?: number
  error?: string
}

/**
 * Usage Example
 */
async function basicBroadcastExample() {
  const broadcaster = new TransactionBroadcaster()
  const privateKey = PrivateKey.fromRandom()

  // Create transaction
  const tx = new Transaction()

  tx.addInput({
    sourceTXID: 'source-tx...',
    sourceOutputIndex: 0,
    unlockingScriptTemplate: new P2PKH().unlock(privateKey),
    sequence: 0xffffffff
  })

  tx.addOutput({
    satoshis: 50000,
    lockingScript: Script.fromAddress('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa')
  })

  await tx.sign()

  // Broadcast
  const result = await broadcaster.broadcastTransaction(tx)
  console.log('Result:', result)

  // Or broadcast and wait for confirmation
  const confirmedResult = await broadcaster.broadcastAndWait(tx, 30000)
  console.log('Confirmed:', confirmedResult)
}
```

## Advanced Broadcasting with Retry Logic

```typescript
import { Transaction, ARC } from '@bsv/sdk'

/**
 * Advanced Broadcaster with Retry Logic
 *
 * Handles broadcast failures with automatic retries
 */
class AdvancedBroadcaster {
  private arcUrls: string[]
  private currentArcIndex: number = 0

  constructor(arcUrls: string[] = [
    'https://arc.taal.com',
    'https://arc.gorillapool.io'
  ]) {
    this.arcUrls = arcUrls
  }

  /**
   * Broadcast with automatic retry and failover
   */
  async broadcastWithRetry(
    tx: Transaction,
    maxRetries: number = 3
  ): Promise<BroadcastResult> {
    let lastError: Error | null = null

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        console.log(`Broadcast attempt ${attempt}/${maxRetries}`)

        const arcUrl = this.arcUrls[this.currentArcIndex]
        const arc = new ARC(arcUrl)

        console.log('Using ARC:', arcUrl)

        const response = await arc.broadcastTransaction(tx)

        console.log('Broadcast successful')

        return {
          txid: response.txid,
          status: response.txStatus,
          timestamp: Date.now(),
          arcUrl,
          attempts: attempt
        }
      } catch (error) {
        lastError = error
        console.error(`Attempt ${attempt} failed:`, error.message)

        // Try next ARC endpoint
        this.currentArcIndex = (this.currentArcIndex + 1) % this.arcUrls.length

        if (attempt < maxRetries) {
          // Exponential backoff
          const waitTime = Math.min(1000 * Math.pow(2, attempt - 1), 10000)
          console.log(`Waiting ${waitTime}ms before retry...`)
          await new Promise(resolve => setTimeout(resolve, waitTime))
        }
      }
    }

    throw new Error(`Broadcast failed after ${maxRetries} attempts: ${lastError?.message}`)
  }

  /**
   * Broadcast to multiple ARC endpoints simultaneously
   */
  async broadcastMultipleArcs(tx: Transaction): Promise<BroadcastResult[]> {
    console.log('Broadcasting to', this.arcUrls.length, 'ARC endpoints')

    const promises = this.arcUrls.map(async (arcUrl) => {
      try {
        const arc = new ARC(arcUrl)
        const response = await arc.broadcastTransaction(tx)

        return {
          txid: response.txid,
          status: response.txStatus,
          timestamp: Date.now(),
          arcUrl,
          success: true
        }
      } catch (error) {
        return {
          txid: tx.id('hex'),
          status: 'FAILED',
          timestamp: Date.now(),
          arcUrl,
          success: false,
          error: error.message
        }
      }
    })

    const results = await Promise.all(promises)

    const successful = results.filter(r => r.success).length
    console.log(`Broadcast to ${successful}/${this.arcUrls.length} endpoints successful`)

    return results
  }

  /**
   * Broadcast with validation
   */
  async broadcastWithValidation(tx: Transaction): Promise<BroadcastResult> {
    try {
      // Validate transaction before broadcast
      const validation = this.validateTransaction(tx)

      if (!validation.valid) {
        throw new Error(`Transaction validation failed: ${validation.errors.join(', ')}`)
      }

      console.log('Transaction validation passed')
      console.log('Inputs:', tx.inputs.length)
      console.log('Outputs:', tx.outputs.length)
      console.log('Size:', tx.toHex().length / 2, 'bytes')

      // Broadcast with retry
      return await this.broadcastWithRetry(tx)
    } catch (error) {
      throw new Error(`Validated broadcast failed: ${error.message}`)
    }
  }

  /**
   * Validate transaction structure
   */
  private validateTransaction(tx: Transaction): {
    valid: boolean
    errors: string[]
  } {
    const errors: string[] = []

    // Check inputs
    if (tx.inputs.length === 0) {
      errors.push('Transaction has no inputs')
    }

    // Check outputs
    if (tx.outputs.length === 0) {
      errors.push('Transaction has no outputs')
    }

    // Check output amounts
    for (let i = 0; i < tx.outputs.length; i++) {
      const output = tx.outputs[i]
      if (output.satoshis < 0) {
        errors.push(`Output ${i} has negative amount`)
      }
      if (output.satoshis > 0 && output.satoshis < 546) {
        errors.push(`Output ${i} below dust threshold (546 sats)`)
      }
    }

    // Check signatures
    for (let i = 0; i < tx.inputs.length; i++) {
      const input = tx.inputs[i]
      if (!input.unlockingScript || input.unlockingScript.chunks.length === 0) {
        errors.push(`Input ${i} has no unlocking script`)
      }
    }

    return {
      valid: errors.length === 0,
      errors
    }
  }
}

interface BroadcastResult {
  txid: string
  status: string
  timestamp: number
  arcUrl?: string
  attempts?: number
  success?: boolean
  error?: string
  blockHash?: string
  blockHeight?: number
}

/**
 * Usage Example
 */
async function advancedBroadcastExample() {
  const broadcaster = new AdvancedBroadcaster([
    'https://arc.taal.com',
    'https://arc.gorillapool.io'
  ])

  const privateKey = PrivateKey.fromRandom()

  // Create transaction
  const tx = new Transaction()
  // ... add inputs and outputs ...
  await tx.sign()

  // Broadcast with retry
  try {
    const result = await broadcaster.broadcastWithRetry(tx, 5)
    console.log('Broadcast successful:', result)
  } catch (error) {
    console.error('All broadcast attempts failed:', error.message)
  }

  // Or broadcast to multiple ARCs
  const results = await broadcaster.broadcastMultipleArcs(tx)
  console.log('Multi-ARC results:', results)
}
```

## Transaction Monitoring

```typescript
import { Transaction, ARC } from '@bsv/sdk'

/**
 * Transaction Monitor
 *
 * Monitor transaction status and confirmations
 */
class TransactionMonitor {
  private arc: ARC
  private monitoredTxs: Map<string, MonitoredTransaction> = new Map()

  constructor(arcUrl: string = 'https://arc.taal.com') {
    this.arc = new ARC(arcUrl)
  }

  /**
   * Start monitoring a transaction
   */
  async monitorTransaction(
    txid: string,
    callback: (status: TransactionStatus) => void
  ): Promise<void> {
    const monitored: MonitoredTransaction = {
      txid,
      startTime: Date.now(),
      callback,
      status: 'PENDING',
      checkCount: 0
    }

    this.monitoredTxs.set(txid, monitored)

    console.log('Started monitoring transaction:', txid)

    // Start monitoring loop
    this.monitorLoop(txid)
  }

  /**
   * Monitor loop for a transaction
   */
  private async monitorLoop(txid: string): Promise<void> {
    const monitored = this.monitoredTxs.get(txid)
    if (!monitored) return

    try {
      monitored.checkCount++

      const status = await this.arc.getTransactionStatus(txid)

      const txStatus: TransactionStatus = {
        txid,
        status: status.txStatus,
        timestamp: Date.now(),
        blockHash: status.blockHash,
        blockHeight: status.blockHeight,
        confirmations: status.confirmations || 0
      }

      console.log(`Check ${monitored.checkCount}: ${txid} - ${txStatus.status}`)

      // Call callback
      monitored.callback(txStatus)

      // Update monitored transaction
      monitored.status = txStatus.status
      monitored.lastCheck = Date.now()

      // Stop monitoring if confirmed
      if (txStatus.status === 'MINED' && txStatus.confirmations >= 6) {
        console.log('Transaction confirmed with 6 blocks:', txid)
        this.monitoredTxs.delete(txid)
        return
      }

      // Continue monitoring
      setTimeout(() => this.monitorLoop(txid), 5000) // Check every 5 seconds
    } catch (error) {
      console.error('Monitor error:', error.message)

      // Retry after delay
      setTimeout(() => this.monitorLoop(txid), 10000)
    }
  }

  /**
   * Stop monitoring a transaction
   */
  stopMonitoring(txid: string): void {
    this.monitoredTxs.delete(txid)
    console.log('Stopped monitoring:', txid)
  }

  /**
   * Get monitored transactions
   */
  getMonitoredTransactions(): MonitoredTransaction[] {
    return Array.from(this.monitoredTxs.values())
  }

  /**
   * Wait for confirmations
   */
  async waitForConfirmations(
    txid: string,
    requiredConfirmations: number = 6,
    timeout: number = 300000 // 5 minutes
  ): Promise<TransactionStatus> {
    return new Promise((resolve, reject) => {
      const startTime = Date.now()

      const checkStatus = async () => {
        try {
          if (Date.now() - startTime > timeout) {
            reject(new Error('Timeout waiting for confirmations'))
            return
          }

          const status = await this.arc.getTransactionStatus(txid)

          const confirmations = status.confirmations || 0

          console.log(`${txid}: ${confirmations}/${requiredConfirmations} confirmations`)

          if (confirmations >= requiredConfirmations) {
            resolve({
              txid,
              status: status.txStatus,
              timestamp: Date.now(),
              blockHash: status.blockHash,
              blockHeight: status.blockHeight,
              confirmations
            })
          } else {
            setTimeout(checkStatus, 5000) // Check every 5 seconds
          }
        } catch (error) {
          reject(error)
        }
      }

      checkStatus()
    })
  }
}

interface MonitoredTransaction {
  txid: string
  startTime: number
  callback: (status: TransactionStatus) => void
  status: string
  checkCount: number
  lastCheck?: number
}

interface TransactionStatus {
  txid: string
  status: string
  timestamp: number
  blockHash?: string
  blockHeight?: number
  confirmations: number
}

/**
 * Usage Example
 */
async function monitoringExample() {
  const monitor = new TransactionMonitor()

  const txid = 'your-transaction-id...'

  // Start monitoring with callback
  await monitor.monitorTransaction(txid, (status) => {
    console.log('Status update:', status)

    if (status.status === 'MINED') {
      console.log('Transaction mined in block:', status.blockHeight)
    }
  })

  // Or wait for specific confirmations
  try {
    const status = await monitor.waitForConfirmations(txid, 6)
    console.log('Transaction confirmed:', status)
  } catch (error) {
    console.error('Confirmation timeout:', error.message)
  }

  // Get all monitored transactions
  const monitored = monitor.getMonitoredTransactions()
  console.log('Monitoring', monitored.length, 'transactions')
}
```

## BEEF Format Broadcasting

```typescript
import { Transaction, ARC, BEEF } from '@bsv/sdk'

/**
 * BEEF Broadcaster
 *
 * Broadcast transactions using BEEF format for efficiency
 */
class BEEFBroadcaster {
  private arc: ARC

  constructor(arcUrl: string = 'https://arc.taal.com') {
    this.arc = new ARC(arcUrl)
  }

  /**
   * Broadcast transaction chain using BEEF
   */
  async broadcastBEEF(
    transactions: Transaction[]
  ): Promise<BroadcastResult[]> {
    try {
      console.log('Creating BEEF from', transactions.length, 'transactions')

      // Create BEEF format
      const beef = BEEF.fromTransactions(transactions)
      const beefHex = beef.toHex()

      console.log('BEEF size:', beefHex.length / 2, 'bytes')
      console.log('Broadcasting BEEF...')

      // Broadcast BEEF
      const response = await this.arc.broadcastBEEF(beefHex)

      const results: BroadcastResult[] = transactions.map(tx => ({
        txid: tx.id('hex'),
        status: 'BROADCASTED',
        timestamp: Date.now(),
        format: 'BEEF'
      }))

      console.log('BEEF broadcast successful')

      return results
    } catch (error) {
      throw new Error(`BEEF broadcast failed: ${error.message}`)
    }
  }

  /**
   * Broadcast transaction with merkle proofs
   */
  async broadcastWithProofs(
    tx: Transaction,
    proofs: MerkleProof[]
  ): Promise<BroadcastResult> {
    try {
      // Create BEEF with proofs
      const beef = new BEEF()
      beef.addTransaction(tx)

      for (const proof of proofs) {
        beef.addMerkleProof(proof)
      }

      const beefHex = beef.toHex()

      console.log('Broadcasting transaction with proofs')
      console.log('BEEF size:', beefHex.length / 2, 'bytes')

      const response = await this.arc.broadcastBEEF(beefHex)

      return {
        txid: tx.id('hex'),
        status: 'BROADCASTED',
        timestamp: Date.now(),
        format: 'BEEF',
        proofs: proofs.length
      }
    } catch (error) {
      throw new Error(`Broadcast with proofs failed: ${error.message}`)
    }
  }
}

interface BroadcastResult {
  txid: string
  status: string
  timestamp: number
  format?: string
  proofs?: number
}

interface MerkleProof {
  txid: string
  blockHeight: number
  merkleRoot: string
  nodes: string[]
}

/**
 * Usage Example
 */
async function beefBroadcastExample() {
  const broadcaster = new BEEFBroadcaster()

  // Create transaction chain
  const transactions: Transaction[] = []
  // ... create linked transactions ...

  // Broadcast as BEEF
  const results = await broadcaster.broadcastBEEF(transactions)
  console.log('BEEF broadcast results:', results)
}
```

## Related Examples

- [Transaction Building](../transaction-building/README.md)
- [Transaction Chains](../transaction-chains/README.md)
- [Batch Operations](../batch-operations/README.md)
- [Double-Spend Detection](../double-spend-detection/README.md)

## See Also

**SDK Components:**
- [ARC](../../sdk-components/arc/README.md) - ARC API client
- [Transaction](../../sdk-components/transaction/README.md) - Transaction building
- [BEEF](../../sdk-components/beef/README.md) - BEEF format

**Learning Paths:**
- [Transaction Broadcasting](../../learning-paths/intermediate/transaction-broadcasting/README.md)
- [ARC Integration](../../learning-paths/advanced/arc-integration/README.md)
- [Transaction Monitoring](../../learning-paths/advanced/transaction-monitoring/README.md)
