# Broadcast Arc

Complete examples for broadcasting transactions using ARC (Advanced Repository for Chains) - the modern transaction processing API for BSV.

## Overview

ARC is the next-generation transaction broadcast and processing system for BSV, replacing the legacy Merchant API. This guide demonstrates how to use ARC for broadcasting transactions, monitoring status, handling retries, and optimizing broadcast strategies with proper error handling.

**Related SDK Components:**
- [ARC](../../sdk-components/arc/README.md)
- [Transaction](../../sdk-components/transaction/README.md)
- [BEEF](../../sdk-components/beef/README.md)

## Basic ARC Broadcasting

```typescript
import { Transaction, PrivateKey, P2PKH, ARC, Script } from '@bsv/sdk'

/**
 * Basic ARC Transaction Broadcaster
 *
 * Demonstrates simple transaction broadcasting using ARC
 */
class BasicARCBroadcaster {
  private arc: ARC
  private arcUrl: string

  constructor(arcUrl: string = 'https://arc.taal.com') {
    this.arcUrl = arcUrl
    this.arc = new ARC(arcUrl)
  }

  /**
   * Broadcast a simple transaction to ARC
   */
  async broadcast(tx: Transaction): Promise<ARCBroadcastResult> {
    try {
      console.log('Broadcasting transaction to ARC...')
      console.log('ARC endpoint:', this.arcUrl)
      console.log('Transaction ID:', tx.id('hex'))
      console.log('Transaction size:', tx.toHex().length / 2, 'bytes')

      // Broadcast to ARC
      const response = await this.arc.broadcastTransaction(tx)

      const result: ARCBroadcastResult = {
        txid: response.txid,
        status: response.txStatus,
        timestamp: Date.now(),
        blockHash: response.blockHash,
        blockHeight: response.blockHeight,
        extraInfo: response.extraInfo
      }

      console.log('Broadcast successful!')
      console.log('TXID:', result.txid)
      console.log('Status:', result.status)

      return result
    } catch (error) {
      console.error('Broadcast failed:', error.message)
      throw new Error(`ARC broadcast failed: ${error.message}`)
    }
  }

  /**
   * Create and broadcast a simple payment transaction
   */
  async createAndBroadcastPayment(
    privateKey: PrivateKey,
    utxo: UTXO,
    recipientAddress: string,
    amount: number
  ): Promise<ARCBroadcastResult> {
    try {
      // Create transaction
      const tx = new Transaction()

      // Add input
      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        unlockingScriptTemplate: new P2PKH().unlock(privateKey),
        sequence: 0xffffffff
      })

      // Add payment output
      tx.addOutput({
        satoshis: amount,
        lockingScript: Script.fromAddress(recipientAddress)
      })

      // Add change output
      const fee = 200 // Simple fee estimation
      const change = utxo.satoshis - amount - fee

      if (change > 546) { // Dust threshold
        const changeAddress = privateKey.toPublicKey().toAddress()
        tx.addOutput({
          satoshis: change,
          lockingScript: Script.fromAddress(changeAddress)
        })
      }

      // Sign transaction
      await tx.sign()

      console.log('Transaction created and signed')
      console.log('Broadcasting to ARC...')

      // Broadcast
      return await this.broadcast(tx)
    } catch (error) {
      throw new Error(`Create and broadcast failed: ${error.message}`)
    }
  }

  /**
   * Get ARC endpoint information
   */
  async getARCInfo(): Promise<ARCInfo> {
    try {
      const response = await fetch(`${this.arcUrl}/v1/policy`)
      const policy = await response.json()

      return {
        endpoint: this.arcUrl,
        version: policy.apiVersion || '1.0',
        mining: policy.mining || {},
        fees: policy.fees || {}
      }
    } catch (error) {
      console.error('Failed to get ARC info:', error.message)
      return {
        endpoint: this.arcUrl,
        version: 'unknown',
        mining: {},
        fees: {}
      }
    }
  }
}

interface UTXO {
  txid: string
  vout: number
  satoshis: number
  script?: Script
}

interface ARCBroadcastResult {
  txid: string
  status: string
  timestamp: number
  blockHash?: string
  blockHeight?: number
  extraInfo?: string
}

interface ARCInfo {
  endpoint: string
  version: string
  mining: any
  fees: any
}

/**
 * Usage Example
 */
async function basicARCExample() {
  const broadcaster = new BasicARCBroadcaster('https://arc.taal.com')

  // Get ARC information
  const arcInfo = await broadcaster.getARCInfo()
  console.log('ARC Info:', arcInfo)

  // Create and broadcast payment
  const privateKey = PrivateKey.fromWif('your-wif-key')

  const utxo: UTXO = {
    txid: 'previous-tx-id...',
    vout: 0,
    satoshis: 100000
  }

  try {
    const result = await broadcaster.createAndBroadcastPayment(
      privateKey,
      utxo,
      '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
      50000
    )

    console.log('Payment broadcast successful:', result)
  } catch (error) {
    console.error('Payment failed:', error.message)
  }
}
```

## ARC Status Monitoring

```typescript
import { ARC, Transaction } from '@bsv/sdk'

/**
 * ARC Transaction Status Monitor
 *
 * Monitor transaction status and confirmations via ARC
 */
class ARCStatusMonitor {
  private arc: ARC
  private monitoredTxs: Map<string, MonitoredTransaction> = new Map()
  private pollInterval: number = 5000 // 5 seconds

  constructor(arcUrl: string = 'https://arc.taal.com') {
    this.arc = new ARC(arcUrl)
  }

  /**
   * Check transaction status
   */
  async checkStatus(txid: string): Promise<TransactionStatus> {
    try {
      const response = await this.arc.getTransactionStatus(txid)

      const status: TransactionStatus = {
        txid,
        status: response.txStatus,
        blockHash: response.blockHash,
        blockHeight: response.blockHeight,
        confirmations: response.confirmations || 0,
        timestamp: Date.now()
      }

      console.log(`Status for ${txid}:`, status.status)

      if (status.blockHeight) {
        console.log(`Block height: ${status.blockHeight}`)
        console.log(`Confirmations: ${status.confirmations}`)
      }

      return status
    } catch (error) {
      throw new Error(`Status check failed: ${error.message}`)
    }
  }

  /**
   * Start monitoring a transaction
   */
  async startMonitoring(
    txid: string,
    callback: (status: TransactionStatus) => void,
    options: MonitorOptions = {}
  ): Promise<void> {
    const {
      requiredConfirmations = 6,
      timeout = 600000, // 10 minutes
      onComplete,
      onError
    } = options

    const monitored: MonitoredTransaction = {
      txid,
      startTime: Date.now(),
      callback,
      requiredConfirmations,
      timeout,
      onComplete,
      onError
    }

    this.monitoredTxs.set(txid, monitored)
    console.log(`Started monitoring ${txid}`)
    console.log(`Required confirmations: ${requiredConfirmations}`)

    // Start monitoring loop
    this.monitorLoop(txid)
  }

  /**
   * Monitoring loop for a transaction
   */
  private async monitorLoop(txid: string): Promise<void> {
    const monitored = this.monitoredTxs.get(txid)
    if (!monitored) return

    try {
      // Check for timeout
      if (Date.now() - monitored.startTime > monitored.timeout) {
        console.log(`Monitoring timeout for ${txid}`)
        if (monitored.onError) {
          monitored.onError(new Error('Monitoring timeout'))
        }
        this.monitoredTxs.delete(txid)
        return
      }

      // Check status
      const status = await this.checkStatus(txid)

      // Call callback
      monitored.callback(status)

      // Check if confirmed
      if (status.confirmations >= monitored.requiredConfirmations) {
        console.log(`Transaction ${txid} confirmed with ${status.confirmations} confirmations`)
        if (monitored.onComplete) {
          monitored.onComplete(status)
        }
        this.monitoredTxs.delete(txid)
        return
      }

      // Continue monitoring
      setTimeout(() => this.monitorLoop(txid), this.pollInterval)
    } catch (error) {
      console.error(`Monitor error for ${txid}:`, error.message)

      if (monitored.onError) {
        monitored.onError(error)
      }

      // Retry after longer delay
      setTimeout(() => this.monitorLoop(txid), this.pollInterval * 2)
    }
  }

  /**
   * Stop monitoring a transaction
   */
  stopMonitoring(txid: string): void {
    this.monitoredTxs.delete(txid)
    console.log(`Stopped monitoring ${txid}`)
  }

  /**
   * Broadcast and monitor in one operation
   */
  async broadcastAndMonitor(
    tx: Transaction,
    requiredConfirmations: number = 6
  ): Promise<TransactionStatus> {
    try {
      // Broadcast
      console.log('Broadcasting transaction...')
      const broadcastResult = await this.arc.broadcastTransaction(tx)
      const txid = broadcastResult.txid

      console.log('Broadcast successful, monitoring...')

      // Wait for confirmations
      return await this.waitForConfirmations(txid, requiredConfirmations)
    } catch (error) {
      throw new Error(`Broadcast and monitor failed: ${error.message}`)
    }
  }

  /**
   * Wait for specific number of confirmations
   */
  async waitForConfirmations(
    txid: string,
    requiredConfirmations: number = 6,
    timeout: number = 600000
  ): Promise<TransactionStatus> {
    return new Promise((resolve, reject) => {
      const startTime = Date.now()

      const checkConfirmations = async () => {
        try {
          // Check timeout
          if (Date.now() - startTime > timeout) {
            reject(new Error('Confirmation timeout'))
            return
          }

          // Check status
          const status = await this.checkStatus(txid)

          console.log(`${txid}: ${status.confirmations}/${requiredConfirmations} confirmations`)

          // Check if confirmed
          if (status.confirmations >= requiredConfirmations) {
            console.log('Transaction confirmed!')
            resolve(status)
          } else {
            // Continue waiting
            setTimeout(checkConfirmations, this.pollInterval)
          }
        } catch (error) {
          reject(error)
        }
      }

      checkConfirmations()
    })
  }

  /**
   * Get all monitored transactions
   */
  getMonitoredTransactions(): string[] {
    return Array.from(this.monitoredTxs.keys())
  }
}

interface MonitoredTransaction {
  txid: string
  startTime: number
  callback: (status: TransactionStatus) => void
  requiredConfirmations: number
  timeout: number
  onComplete?: (status: TransactionStatus) => void
  onError?: (error: Error) => void
}

interface TransactionStatus {
  txid: string
  status: string
  blockHash?: string
  blockHeight?: number
  confirmations: number
  timestamp: number
}

interface MonitorOptions {
  requiredConfirmations?: number
  timeout?: number
  onComplete?: (status: TransactionStatus) => void
  onError?: (error: Error) => void
}

/**
 * Usage Example
 */
async function statusMonitoringExample() {
  const monitor = new ARCStatusMonitor('https://arc.taal.com')

  const txid = 'your-transaction-id...'

  // Simple status check
  const status = await monitor.checkStatus(txid)
  console.log('Current status:', status)

  // Monitor with callback
  await monitor.startMonitoring(
    txid,
    (status) => {
      console.log('Status update:', status)
    },
    {
      requiredConfirmations: 6,
      timeout: 600000,
      onComplete: (finalStatus) => {
        console.log('Monitoring complete:', finalStatus)
      },
      onError: (error) => {
        console.error('Monitoring error:', error)
      }
    }
  )

  // Or wait for confirmations
  try {
    const confirmed = await monitor.waitForConfirmations(txid, 6)
    console.log('Transaction confirmed:', confirmed)
  } catch (error) {
    console.error('Confirmation failed:', error.message)
  }
}
```

## Advanced ARC Features with Retry Logic

```typescript
import { Transaction, ARC } from '@bsv/sdk'

/**
 * Advanced ARC Broadcaster
 *
 * Implements retry logic, failover, and batch broadcasting
 */
class AdvancedARCBroadcaster {
  private arcEndpoints: string[]
  private currentEndpointIndex: number = 0
  private maxRetries: number = 3
  private retryDelay: number = 1000 // Start with 1 second

  constructor(arcEndpoints: string[] = [
    'https://arc.taal.com',
    'https://arc.gorillapool.io'
  ]) {
    this.arcEndpoints = arcEndpoints
  }

  /**
   * Broadcast with automatic retry and failover
   */
  async broadcastWithRetry(
    tx: Transaction,
    options: BroadcastOptions = {}
  ): Promise<ARCBroadcastResult> {
    const {
      maxRetries = this.maxRetries,
      retryDelay = this.retryDelay,
      validateBefore = true
    } = options

    // Validate transaction before broadcasting
    if (validateBefore) {
      const validation = this.validateTransaction(tx)
      if (!validation.valid) {
        throw new Error(`Transaction validation failed: ${validation.errors.join(', ')}`)
      }
      console.log('Transaction validation passed')
    }

    let lastError: Error | null = null

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        const arcUrl = this.arcEndpoints[this.currentEndpointIndex]
        console.log(`Broadcast attempt ${attempt}/${maxRetries}`)
        console.log(`Using ARC endpoint: ${arcUrl}`)

        const arc = new ARC(arcUrl)
        const response = await arc.broadcastTransaction(tx)

        console.log('Broadcast successful!')

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

        // Try next endpoint
        this.currentEndpointIndex = (this.currentEndpointIndex + 1) % this.arcEndpoints.length

        if (attempt < maxRetries) {
          // Exponential backoff
          const waitTime = Math.min(retryDelay * Math.pow(2, attempt - 1), 10000)
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
  async broadcastToMultipleArcs(tx: Transaction): Promise<MultiARCResult> {
    console.log(`Broadcasting to ${this.arcEndpoints.length} ARC endpoints`)

    const startTime = Date.now()

    const promises = this.arcEndpoints.map(async (arcUrl) => {
      try {
        const arc = new ARC(arcUrl)
        const response = await arc.broadcastTransaction(tx)

        return {
          arcUrl,
          success: true,
          txid: response.txid,
          status: response.txStatus,
          responseTime: Date.now() - startTime
        }
      } catch (error) {
        return {
          arcUrl,
          success: false,
          error: error.message,
          responseTime: Date.now() - startTime
        }
      }
    })

    const results = await Promise.all(promises)

    const successful = results.filter(r => r.success).length
    const failed = results.filter(r => !r.success).length

    console.log(`Broadcast results: ${successful} successful, ${failed} failed`)

    return {
      results,
      successful,
      failed,
      totalTime: Date.now() - startTime
    }
  }

  /**
   * Batch broadcast multiple transactions
   */
  async broadcastBatch(
    transactions: Transaction[],
    options: BatchOptions = {}
  ): Promise<BatchResult[]> {
    const {
      parallel = false,
      delayBetween = 100,
      continueOnError = true
    } = options

    console.log(`Broadcasting batch of ${transactions.length} transactions`)
    console.log(`Mode: ${parallel ? 'parallel' : 'sequential'}`)

    const results: BatchResult[] = []

    if (parallel) {
      // Broadcast all transactions in parallel
      const promises = transactions.map(async (tx, index) => {
        try {
          const result = await this.broadcastWithRetry(tx)
          return {
            index,
            success: true,
            txid: result.txid,
            status: result.status
          }
        } catch (error) {
          return {
            index,
            success: false,
            txid: tx.id('hex'),
            error: error.message
          }
        }
      })

      return await Promise.all(promises)
    } else {
      // Broadcast sequentially
      for (let i = 0; i < transactions.length; i++) {
        const tx = transactions[i]

        try {
          console.log(`Broadcasting transaction ${i + 1}/${transactions.length}`)
          const result = await this.broadcastWithRetry(tx)

          results.push({
            index: i,
            success: true,
            txid: result.txid,
            status: result.status
          })

          // Delay between broadcasts
          if (i < transactions.length - 1 && delayBetween > 0) {
            await new Promise(resolve => setTimeout(resolve, delayBetween))
          }
        } catch (error) {
          console.error(`Transaction ${i + 1} failed:`, error.message)

          results.push({
            index: i,
            success: false,
            txid: tx.id('hex'),
            error: error.message
          })

          if (!continueOnError) {
            break
          }
        }
      }

      const successful = results.filter(r => r.success).length
      console.log(`Batch complete: ${successful}/${transactions.length} successful`)

      return results
    }
  }

  /**
   * Validate transaction before broadcasting
   */
  private validateTransaction(tx: Transaction): ValidationResult {
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

    // Check for unlocking scripts
    for (let i = 0; i < tx.inputs.length; i++) {
      const input = tx.inputs[i]
      if (!input.unlockingScript || input.unlockingScript.chunks.length === 0) {
        errors.push(`Input ${i} has no unlocking script`)
      }
    }

    // Check transaction size (max 100MB but warn at 10MB)
    const txSize = tx.toHex().length / 2
    if (txSize > 10000000) {
      errors.push(`Transaction is very large (${txSize} bytes)`)
    }

    return {
      valid: errors.length === 0,
      errors
    }
  }

  /**
   * Get best ARC endpoint based on response time
   */
  async getBestEndpoint(): Promise<string> {
    console.log('Testing ARC endpoints...')

    const tests = this.arcEndpoints.map(async (arcUrl) => {
      const startTime = Date.now()

      try {
        await fetch(`${arcUrl}/v1/policy`)
        const responseTime = Date.now() - startTime

        return {
          arcUrl,
          responseTime,
          available: true
        }
      } catch (error) {
        return {
          arcUrl,
          responseTime: Infinity,
          available: false
        }
      }
    })

    const results = await Promise.all(tests)

    // Sort by response time
    results.sort((a, b) => a.responseTime - b.responseTime)

    const best = results[0]

    if (best.available) {
      console.log(`Best endpoint: ${best.arcUrl} (${best.responseTime}ms)`)
      return best.arcUrl
    }

    throw new Error('No ARC endpoints available')
  }
}

interface BroadcastOptions {
  maxRetries?: number
  retryDelay?: number
  validateBefore?: boolean
}

interface BatchOptions {
  parallel?: boolean
  delayBetween?: number
  continueOnError?: boolean
}

interface ARCBroadcastResult {
  txid: string
  status: string
  timestamp: number
  arcUrl?: string
  attempts?: number
}

interface MultiARCResult {
  results: Array<{
    arcUrl: string
    success: boolean
    txid?: string
    status?: string
    error?: string
    responseTime: number
  }>
  successful: number
  failed: number
  totalTime: number
}

interface BatchResult {
  index: number
  success: boolean
  txid: string
  status?: string
  error?: string
}

interface ValidationResult {
  valid: boolean
  errors: string[]
}

/**
 * Usage Example
 */
async function advancedARCExample() {
  const broadcaster = new AdvancedARCBroadcaster([
    'https://arc.taal.com',
    'https://arc.gorillapool.io'
  ])

  // Get best endpoint
  const bestEndpoint = await broadcaster.getBestEndpoint()
  console.log('Best endpoint:', bestEndpoint)

  // Create transaction
  const privateKey = PrivateKey.fromRandom()
  const tx = new Transaction()
  // ... add inputs and outputs ...
  await tx.sign()

  // Broadcast with retry
  try {
    const result = await broadcaster.broadcastWithRetry(tx, {
      maxRetries: 5,
      retryDelay: 1000,
      validateBefore: true
    })
    console.log('Broadcast successful:', result)
  } catch (error) {
    console.error('All retries failed:', error.message)
  }

  // Broadcast to multiple ARCs
  const multiResult = await broadcaster.broadcastToMultipleArcs(tx)
  console.log('Multi-ARC broadcast:', multiResult)

  // Batch broadcast
  const transactions: Transaction[] = [] // Array of transactions
  const batchResults = await broadcaster.broadcastBatch(transactions, {
    parallel: false,
    delayBetween: 100,
    continueOnError: true
  })
  console.log('Batch results:', batchResults)
}
```

## BEEF Format Broadcasting via ARC

```typescript
import { Transaction, ARC, BEEF } from '@bsv/sdk'

/**
 * BEEF Broadcaster for ARC
 *
 * Efficiently broadcast transaction chains using BEEF format
 */
class BEEFARCBroadcaster {
  private arc: ARC

  constructor(arcUrl: string = 'https://arc.taal.com') {
    this.arc = new ARC(arcUrl)
  }

  /**
   * Broadcast transaction chain using BEEF
   */
  async broadcastBEEF(transactions: Transaction[]): Promise<BEEFBroadcastResult> {
    try {
      console.log(`Creating BEEF from ${transactions.length} transactions`)

      // Create BEEF structure
      const beef = BEEF.fromTransactions(transactions)
      const beefHex = beef.toHex()

      const originalSize = transactions.reduce(
        (sum, tx) => sum + tx.toHex().length / 2,
        0
      )
      const beefSize = beefHex.length / 2

      console.log('Original size:', originalSize, 'bytes')
      console.log('BEEF size:', beefSize, 'bytes')
      console.log('Compression:', ((1 - beefSize / originalSize) * 100).toFixed(2), '%')

      // Broadcast BEEF to ARC
      console.log('Broadcasting BEEF to ARC...')
      const response = await this.arc.broadcastBEEF(beefHex)

      return {
        txids: transactions.map(tx => tx.id('hex')),
        beefSize,
        originalSize,
        compressionRatio: beefSize / originalSize,
        timestamp: Date.now(),
        response
      }
    } catch (error) {
      throw new Error(`BEEF broadcast failed: ${error.message}`)
    }
  }

  /**
   * Broadcast single transaction with BEEF format
   */
  async broadcastSingleWithBEEF(tx: Transaction): Promise<BEEFBroadcastResult> {
    return await this.broadcastBEEF([tx])
  }

  /**
   * Broadcast transaction with parent transactions (merkle proofs)
   */
  async broadcastWithParents(
    tx: Transaction,
    parentTransactions: Transaction[]
  ): Promise<BEEFBroadcastResult> {
    try {
      console.log('Broadcasting transaction with', parentTransactions.length, 'parents')

      // Combine transaction with its parents
      const allTransactions = [...parentTransactions, tx]

      // Create and broadcast BEEF
      return await this.broadcastBEEF(allTransactions)
    } catch (error) {
      throw new Error(`Broadcast with parents failed: ${error.message}`)
    }
  }

  /**
   * Create BEEF from transaction chain
   */
  createBEEFFromChain(
    transactions: Transaction[]
  ): {
    beef: BEEF
    hex: string
    size: number
  } {
    console.log('Creating BEEF from chain of', transactions.length, 'transactions')

    const beef = BEEF.fromTransactions(transactions)
    const hex = beef.toHex()
    const size = hex.length / 2

    console.log('BEEF created:', size, 'bytes')

    return {
      beef,
      hex,
      size
    }
  }

  /**
   * Validate BEEF structure before broadcasting
   */
  validateBEEF(beef: BEEF): ValidationResult {
    const errors: string[] = []

    try {
      // Check BEEF has transactions
      const transactions = beef.getTransactions()

      if (transactions.length === 0) {
        errors.push('BEEF contains no transactions')
      }

      // Validate each transaction in BEEF
      for (let i = 0; i < transactions.length; i++) {
        const tx = transactions[i]

        if (tx.inputs.length === 0) {
          errors.push(`Transaction ${i} has no inputs`)
        }

        if (tx.outputs.length === 0) {
          errors.push(`Transaction ${i} has no outputs`)
        }
      }

      return {
        valid: errors.length === 0,
        errors
      }
    } catch (error) {
      errors.push(`BEEF validation error: ${error.message}`)
      return {
        valid: false,
        errors
      }
    }
  }
}

interface BEEFBroadcastResult {
  txids: string[]
  beefSize: number
  originalSize: number
  compressionRatio: number
  timestamp: number
  response: any
}

interface ValidationResult {
  valid: boolean
  errors: string[]
}

/**
 * Usage Example
 */
async function beefARCExample() {
  const broadcaster = new BEEFARCBroadcaster('https://arc.taal.com')

  // Create transaction chain
  const tx1 = new Transaction()
  // ... configure tx1 ...

  const tx2 = new Transaction()
  // ... configure tx2 that spends from tx1 ...

  const tx3 = new Transaction()
  // ... configure tx3 that spends from tx2 ...

  const transactions = [tx1, tx2, tx3]

  // Broadcast as BEEF
  try {
    const result = await broadcaster.broadcastBEEF(transactions)
    console.log('BEEF broadcast successful')
    console.log('TXIDs:', result.txids)
    console.log('Compression ratio:', result.compressionRatio.toFixed(2))
  } catch (error) {
    console.error('BEEF broadcast failed:', error.message)
  }

  // Or broadcast single transaction with parents
  const parent1 = new Transaction() // Parent transaction
  const child = new Transaction()   // Child spending from parent

  const withParentsResult = await broadcaster.broadcastWithParents(
    child,
    [parent1]
  )
  console.log('Broadcast with parents:', withParentsResult)
}
```

## Related Examples

- [Transaction Broadcasting](../transaction-broadcasting/README.md)
- [Transaction Building](../transaction-building/README.md)
- [Transaction Chains](../transaction-chains/README.md)
- [Batch Operations](../batch-operations/README.md)

## See Also

**SDK Components:**
- [ARC](../../sdk-components/arc/README.md) - ARC API client
- [Transaction](../../sdk-components/transaction/README.md) - Transaction construction
- [BEEF](../../sdk-components/beef/README.md) - BEEF format handling

**Learning Paths:**
- [Broadcasting Transactions](../../learning-paths/intermediate/broadcasting/README.md)
- [ARC Integration](../../learning-paths/advanced/arc-integration/README.md)
- [Transaction Chains](../../learning-paths/advanced/transaction-chains/README.md)
