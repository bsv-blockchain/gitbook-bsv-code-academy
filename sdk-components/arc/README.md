# ARC (Arc API for Transaction Broadcasting)

## Overview

The `ARC` class in the BSV TypeScript SDK provides a production-ready interface for broadcasting Bitcoin transactions through the ARC (Arc) API. ARC is a modern transaction processor that offers advanced features like transaction status tracking, fee policy queries, double-spend notifications, and callback-based event handling. It implements the Broadcaster interface, making it a drop-in replacement for other broadcast methods.

## Purpose

- Broadcast transactions to the BSV network through the ARC API
- Query transaction status and confirmation details
- Calculate and verify transaction fees based on network policies
- Monitor double-spend attempts with callback notifications
- Handle transaction errors and rejection reasons
- Support deployment tracking and application identification
- Enable webhook callbacks for asynchronous transaction updates

## When to Use

The ARC class provides two main broadcasting approaches:

### 1. Simple Transaction Broadcasting: `tx.broadcast(arc)`

**Use when:**
- Broadcasting a single, independent transaction
- Transaction doesn't depend on unconfirmed parents
- You want simple, straightforward broadcasting

**Example:**
```typescript
const arc = new ARC('https://api.taal.com/arc', { apiKey: 'xxx' });
const response = await tx.broadcast(arc);
// Or use default broadcaster:
const response = await tx.broadcast();
```

### 2. BEEF Bundle Broadcasting: `arc.broadcastBEEF(beefHex)`

**Use when:**
- Broadcasting transaction chains with dependencies
- Child transactions spend from unconfirmed parent transactions
- You need atomic broadcasting of multiple related transactions
- You want to include merkle proofs for SPV validation

**Example:**
```typescript
const arc = new ARC('https://api.taal.com/arc', { apiKey: 'xxx' });

// Create BEEF bundle
const beef = new Beef();
beef.addTransaction(parentTx);
beef.addTransaction(childTx);

const beefHex = beef.toHex();
const response = await arc.broadcastBEEF(beefHex);
```

**Important:** `tx.broadcast()` does NOT automatically handle transaction chains. For chains, you MUST use `arc.broadcastBEEF()` with a properly constructed BEEF bundle.

## Basic Usage

### Broadcasting a Transaction

```typescript
import { ARC, Transaction, PrivateKey, P2PKH } from '@bsv/sdk';

// Initialize ARC broadcaster
const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'mainnet_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
  deploymentId: 'my-app-v1',
  callbackUrl: 'https://myapp.com/callbacks',
  callbackToken: 'secret_token'
});

// Create and broadcast transaction
const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');
const tx = new Transaction();

tx.addInput({
  sourceTransaction: Transaction.fromHex('...'),
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(privKey)
});

tx.addOutput({
  lockingScript: new P2PKH().lock('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa'),
  satoshis: 1000
});

await tx.fee();
await tx.sign();

// Broadcast with ARC
const response = await tx.broadcast(arc);
console.log('TXID:', response.txid);
console.log('Status:', response.status);
```

### Querying Transaction Status

```typescript
import { ARC } from '@bsv/sdk';

const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'mainnet_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
});

// Query transaction status
const txid = '1234567890abcdef...';
const status = await arc.getTransactionStatus(txid);

console.log('Status:', status.status); // 'SEEN', 'MINED', 'REJECTED', etc.
console.log('Block height:', status.blockHeight);
console.log('Block hash:', status.blockHash);
console.log('Confirmations:', status.confirmations);
```

## Key Features

### 1. Transaction Broadcasting with Enhanced Features

ARC provides production-grade transaction broadcasting with deployment tracking:

```typescript
import { ARC, Transaction, PrivateKey, P2PKH } from '@bsv/sdk';

// Initialize ARC with full configuration
const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'mainnet_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
  deploymentId: 'payment-processor-v2.1.0',
  callbackUrl: 'https://myapp.com/tx-callbacks',
  callbackToken: 'webhook_secret_abc123'
});

// Create transaction
const privKey = PrivateKey.fromRandom();
const tx = new Transaction();

// Add inputs and outputs
tx.addInput({
  sourceTransaction: Transaction.fromHex('...'),
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(privKey)
});

tx.addOutput({
  lockingScript: new P2PKH().lock('1RecipientAddress...'),
  satoshis: 5000
});

tx.addOutput({
  lockingScript: new P2PKH().lock(privKey.toPublicKey().toAddress()),
  change: true
});

await tx.fee();
await tx.sign();

// Broadcast with error handling
try {
  const response = await tx.broadcast(arc);

  console.log('Transaction broadcast successful');
  console.log('TXID:', response.txid);
  console.log('Status:', response.status);
  console.log('Timestamp:', response.timestamp);

  // Response includes:
  // - txid: Transaction ID
  // - status: 'SEEN', 'QUEUED', 'MINED'
  // - timestamp: Submission timestamp
  // - competingTxs: Any competing (double-spend) transactions

  if (response.competingTxs && response.competingTxs.length > 0) {
    console.warn('Warning: Competing transactions detected');
    response.competingTxs.forEach(competingTxid => {
      console.warn('  Competing TXID:', competingTxid);
    });
  }

} catch (error) {
  console.error('Broadcast failed:', error.message);

  // Handle specific error types
  if (error.code === 'INVALID_TRANSACTION') {
    console.error('Transaction validation failed:', error.details);
  } else if (error.code === 'INSUFFICIENT_FEE') {
    console.error('Fee too low. Minimum required:', error.minimumFee);
  } else if (error.code === 'DOUBLE_SPEND') {
    console.error('Double-spend detected. Original TXID:', error.originalTxid);
  }
}
```

### 2. Fee Policy Queries and Calculation

ARC provides fee policy information for accurate fee calculation:

```typescript
import { ARC, Transaction } from '@bsv/sdk';

const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'mainnet_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
});

// Query current fee policy
const feePolicy = await arc.getFeePolicy();

console.log('Fee policies:');
console.log('Standard fee:', feePolicy.standard, 'sats/KB');
console.log('Data fee:', feePolicy.data, 'sats/KB');
console.log('Mining fee:', feePolicy.mining, 'sats/KB');

// Calculate fee for a transaction
const tx = new Transaction();
// ... add inputs and outputs ...

// Get transaction size
const txSize = tx.toBinary().length;

// Calculate required fee
const requiredFee = Math.ceil((txSize / 1000) * feePolicy.standard);
console.log('Required fee:', requiredFee, 'satoshis');

// Verify transaction has sufficient fee
const actualFee = tx.getFee();
if (actualFee < requiredFee) {
  console.error('Insufficient fee!');
  console.error('Need additional:', requiredFee - actualFee, 'satoshis');
} else {
  console.log('Fee is sufficient');
}

// Use ARC's fee model directly
import { SatoshisPerKilobyte } from '@bsv/sdk/transaction/fee-models';

const arcFeeModel = new SatoshisPerKilobyte(feePolicy.standard);
await tx.fee(arcFeeModel);

console.log('Transaction fee set to:', tx.getFee(), 'satoshis');
```

### 3. Transaction Status Tracking

Monitor transaction lifecycle from broadcast to confirmation:

```typescript
import { ARC } from '@bsv/sdk';

const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'mainnet_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
});

class TransactionMonitor {
  /**
   * Poll transaction status until confirmed
   */
  static async waitForConfirmation(
    txid: string,
    requiredConfirmations: number = 6,
    pollIntervalMs: number = 5000,
    timeoutMs: number = 3600000 // 1 hour
  ): Promise<{ confirmed: boolean; blockHeight?: number; blockHash?: string }> {
    const startTime = Date.now();

    while (Date.now() - startTime < timeoutMs) {
      try {
        const status = await arc.getTransactionStatus(txid);

        console.log(`Status: ${status.status}, Confirmations: ${status.confirmations || 0}`);

        // Check status
        if (status.status === 'REJECTED') {
          throw new Error(`Transaction rejected: ${status.rejectReason}`);
        }

        if (status.status === 'MINED' &&
            status.confirmations &&
            status.confirmations >= requiredConfirmations) {
          return {
            confirmed: true,
            blockHeight: status.blockHeight,
            blockHash: status.blockHash
          };
        }

        // Wait before next poll
        await new Promise(resolve => setTimeout(resolve, pollIntervalMs));

      } catch (error) {
        console.error('Status check failed:', error.message);
        await new Promise(resolve => setTimeout(resolve, pollIntervalMs));
      }
    }

    return { confirmed: false };
  }

  /**
   * Get detailed transaction status
   */
  static async getDetailedStatus(txid: string): Promise<{
    status: string;
    stage: 'mempool' | 'mining' | 'confirmed' | 'unknown';
    details: any;
  }> {
    const status = await arc.getTransactionStatus(txid);

    let stage: 'mempool' | 'mining' | 'confirmed' | 'unknown' = 'unknown';

    if (status.status === 'SEEN' || status.status === 'QUEUED') {
      stage = 'mempool';
    } else if (status.status === 'MINED' && (!status.confirmations || status.confirmations < 6)) {
      stage = 'mining';
    } else if (status.status === 'MINED' && status.confirmations && status.confirmations >= 6) {
      stage = 'confirmed';
    }

    return {
      status: status.status,
      stage,
      details: {
        txid: status.txid,
        blockHeight: status.blockHeight,
        blockHash: status.blockHash,
        confirmations: status.confirmations,
        timestamp: status.timestamp,
        rejectReason: status.rejectReason
      }
    };
  }
}

// Usage
const txid = '1234567890abcdef...';

// Wait for confirmation
const result = await TransactionMonitor.waitForConfirmation(txid, 6);

if (result.confirmed) {
  console.log('Transaction confirmed!');
  console.log('Block height:', result.blockHeight);
  console.log('Block hash:', result.blockHash);
} else {
  console.log('Transaction not confirmed within timeout');
}

// Get detailed status
const detailedStatus = await TransactionMonitor.getDetailedStatus(txid);
console.log('Status:', detailedStatus.status);
console.log('Stage:', detailedStatus.stage);
console.log('Details:', detailedStatus.details);
```

### 4. Double-Spend Detection and Notification

ARC provides double-spend detection with callback notifications:

```typescript
import { ARC, Transaction } from '@bsv/sdk';
import express from 'express';

// Setup webhook server to receive callbacks
const app = express();
app.use(express.json());

const callbackSecret = 'webhook_secret_abc123';

app.post('/tx-callbacks', (req, res) => {
  // Verify callback token
  const token = req.headers['authorization'];
  if (token !== `Bearer ${callbackSecret}`) {
    return res.status(401).send('Unauthorized');
  }

  const event = req.body;

  console.log('Received callback:', event.type);

  switch (event.type) {
    case 'TRANSACTION_SEEN':
      console.log('Transaction seen in mempool:', event.txid);
      break;

    case 'TRANSACTION_MINED':
      console.log('Transaction mined:', event.txid);
      console.log('Block height:', event.blockHeight);
      console.log('Block hash:', event.blockHash);
      break;

    case 'TRANSACTION_REJECTED':
      console.error('Transaction rejected:', event.txid);
      console.error('Reason:', event.rejectReason);
      break;

    case 'DOUBLE_SPEND_ATTEMPT':
      console.warn('⚠️  Double-spend detected!');
      console.warn('Original TXID:', event.txid);
      console.warn('Competing TXID:', event.competingTxid);

      // Take action - e.g., alert merchants, freeze orders
      handleDoubleSpend(event.txid, event.competingTxid);
      break;

    case 'TRANSACTION_CONFIRMED':
      console.log('Transaction confirmed:', event.txid);
      console.log('Confirmations:', event.confirmations);
      break;
  }

  res.status(200).send('OK');
});

app.listen(3000, () => {
  console.log('Webhook server listening on port 3000');
});

// Initialize ARC with callback URL
const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'mainnet_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
  deploymentId: 'payment-processor',
  callbackUrl: 'https://myapp.com/tx-callbacks',
  callbackToken: callbackSecret
});

// Handle double-spend events
function handleDoubleSpend(originalTxid: string, competingTxid: string) {
  console.error('CRITICAL: Double-spend attempt detected');
  console.error('Original transaction:', originalTxid);
  console.error('Competing transaction:', competingTxid);

  // Business logic:
  // 1. Alert payment processor
  // 2. Freeze pending orders
  // 3. Notify merchant
  // 4. Log security incident
  // 5. Analyze competing transaction

  // Example: Check which transaction has higher fee
  arc.getTransactionStatus(originalTxid).then(originalStatus => {
    arc.getTransactionStatus(competingTxid).then(competingStatus => {
      console.log('Original fee:', originalStatus.fee);
      console.log('Competing fee:', competingStatus.fee);

      if (competingStatus.fee > originalStatus.fee) {
        console.warn('Competing transaction has higher fee - may confirm instead');
      }
    });
  });
}

// Broadcast transaction with double-spend monitoring
async function broadcastWithMonitoring(tx: Transaction) {
  try {
    const response = await tx.broadcast(arc);

    console.log('Transaction broadcast:', response.txid);

    // Check for immediate competing transactions
    if (response.competingTxs && response.competingTxs.length > 0) {
      console.warn('Competing transactions detected at broadcast:');
      response.competingTxs.forEach(competingTxid => {
        console.warn('  -', competingTxid);
        handleDoubleSpend(response.txid, competingTxid);
      });
    }

    return response;
  } catch (error) {
    if (error.code === 'DOUBLE_SPEND') {
      handleDoubleSpend(error.txid, error.competingTxid);
    }
    throw error;
  }
}
```

## API Reference

### Constructor

```typescript
constructor(url: string, options?: ARCOptions)
```

Creates a new ARC broadcaster instance.

**Parameters:**
- `url: string` - The ARC API endpoint URL
- `options?: ARCOptions` - Configuration options
  - `apiKey?: string` - API key for authentication
  - `deploymentId?: string` - Application deployment identifier
  - `callbackUrl?: string` - Webhook URL for transaction callbacks
  - `callbackToken?: string` - Bearer token for callback authentication

**Example:**
```typescript
const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'mainnet_xxx',
  deploymentId: 'my-app-v1',
  callbackUrl: 'https://myapp.com/callbacks',
  callbackToken: 'secret'
});
```

### Instance Methods

#### `broadcast(tx: Transaction): Promise<BroadcastResponse>`

Broadcasts a transaction to the network.

**Parameters:**
- `tx: Transaction` - The transaction to broadcast

**Returns:** `Promise<BroadcastResponse>`
- `txid: string` - Transaction ID
- `status: string` - Transaction status ('SEEN', 'QUEUED', 'MINED')
- `timestamp: number` - Submission timestamp
- `competingTxs?: string[]` - Array of competing transaction IDs

**Example:**
```typescript
const response = await arc.broadcast(transaction);
console.log('TXID:', response.txid);
```

#### `getTransactionStatus(txid: string): Promise<TransactionStatus>`

Queries the status of a transaction.

**Parameters:**
- `txid: string` - The transaction ID to query

**Returns:** `Promise<TransactionStatus>`
- `txid: string` - Transaction ID
- `status: string` - Current status
- `blockHeight?: number` - Block height if mined
- `blockHash?: string` - Block hash if mined
- `confirmations?: number` - Number of confirmations
- `timestamp?: number` - Status timestamp
- `rejectReason?: string` - Rejection reason if rejected
- `fee?: number` - Transaction fee in satoshis

**Example:**
```typescript
const status = await arc.getTransactionStatus(txid);
console.log('Status:', status.status);
console.log('Confirmations:', status.confirmations);
```

#### `getFeePolicy(): Promise<FeePolicy>`

Retrieves the current fee policy from ARC.

**Returns:** `Promise<FeePolicy>`
- `standard: number` - Standard transaction fee (sats/KB)
- `data: number` - Data transaction fee (sats/KB)
- `mining: number` - Mining fee (sats/KB)

**Example:**
```typescript
const policy = await arc.getFeePolicy();
console.log('Standard fee:', policy.standard, 'sats/KB');
```

#### `broadcastBEEF(beefHex: string): Promise<BroadcastResponse>`

Broadcasts a BEEF (Background Evaluation Extended Format) bundle containing transaction chains.

**Parameters:**
- `beefHex: string` - The BEEF bundle in hexadecimal format

**Returns:** `Promise<BroadcastResponse>`
- `txid: string` - Transaction ID of the final transaction
- `status: string` - Transaction status
- `timestamp: number` - Submission timestamp

**Use this method when:**
- Broadcasting transaction chains with dependencies
- Child transactions spend from unconfirmed parents
- You need atomic broadcasting of multiple transactions

**Example:**
```typescript
import { Beef } from '@bsv/sdk';

// Create BEEF bundle
const beef = new Beef();
beef.addTransaction(parentTx);
beef.addTransaction(childTx);

const beefHex = beef.toHex();
const response = await arc.broadcastBEEF(beefHex);
console.log('BEEF broadcast successful:', response.txid);
```

### Response Types

```typescript
interface BroadcastResponse {
  txid: string;
  status: 'SEEN' | 'QUEUED' | 'MINED' | 'REJECTED';
  timestamp: number;
  competingTxs?: string[];
}

interface TransactionStatus {
  txid: string;
  status: 'SEEN' | 'QUEUED' | 'MINED' | 'REJECTED' | 'UNKNOWN';
  blockHeight?: number;
  blockHash?: string;
  confirmations?: number;
  timestamp?: number;
  rejectReason?: string;
  fee?: number;
}

interface FeePolicy {
  standard: number;  // satoshis per KB
  data: number;      // satoshis per KB
  mining: number;    // satoshis per KB
}
```

## Common Patterns

### Pattern 1: Production Payment Processor

Complete payment processor with ARC integration:

```typescript
import { ARC, Transaction, PrivateKey, P2PKH } from '@bsv/sdk';
import { SatoshisPerKilobyte } from '@bsv/sdk/transaction/fee-models';

class PaymentProcessor {
  private arc: ARC;
  private privKey: PrivateKey;
  private pendingPayments: Map<string, PaymentRecord>;

  constructor(
    arcUrl: string,
    apiKey: string,
    walletPrivKey: PrivateKey,
    callbackUrl: string
  ) {
    this.arc = new ARC(arcUrl, {
      apiKey,
      deploymentId: 'payment-processor-v1.0.0',
      callbackUrl,
      callbackToken: process.env.CALLBACK_SECRET!
    });

    this.privKey = walletPrivKey;
    this.pendingPayments = new Map();
  }

  /**
   * Process payment transaction
   */
  async processPayment(
    recipientAddress: string,
    amount: number,
    utxos: Array<{ tx: Transaction; outputIndex: number }>,
    metadata?: any
  ): Promise<{ txid: string; status: string }> {
    try {
      // Get current fee policy
      const feePolicy = await this.arc.getFeePolicy();
      const feeModel = new SatoshisPerKilobyte(feePolicy.standard);

      // Create transaction
      const tx = new Transaction();

      // Add inputs
      for (const utxo of utxos) {
        tx.addInput({
          sourceTransaction: utxo.tx,
          sourceOutputIndex: utxo.outputIndex,
          unlockingScriptTemplate: new P2PKH().unlock(this.privKey)
        });
      }

      // Add payment output
      tx.addOutput({
        lockingScript: new P2PKH().lock(recipientAddress),
        satoshis: amount
      });

      // Add change output
      tx.addOutput({
        lockingScript: new P2PKH().lock(this.privKey.toPublicKey().toAddress()),
        change: true
      });

      // Calculate fee and sign
      await tx.fee(feeModel);
      await tx.sign();

      // Store payment record
      const txid = tx.id('hex');
      this.pendingPayments.set(txid, {
        txid,
        recipientAddress,
        amount,
        timestamp: Date.now(),
        status: 'pending',
        metadata
      });

      // Broadcast
      const response = await tx.broadcast(this.arc);

      // Update status
      this.pendingPayments.get(txid)!.status = response.status;

      console.log('Payment broadcast successful');
      console.log('TXID:', response.txid);
      console.log('Amount:', amount, 'satoshis');
      console.log('Recipient:', recipientAddress);

      return {
        txid: response.txid,
        status: response.status
      };

    } catch (error) {
      console.error('Payment processing failed:', error.message);

      // Handle specific errors
      if (error.code === 'INSUFFICIENT_FEE') {
        throw new Error(`Fee too low. Minimum: ${error.minimumFee} satoshis`);
      } else if (error.code === 'DOUBLE_SPEND') {
        throw new Error(`Double-spend detected: ${error.competingTxid}`);
      } else if (error.code === 'INVALID_TRANSACTION') {
        throw new Error(`Invalid transaction: ${error.details}`);
      }

      throw error;
    }
  }

  /**
   * Handle webhook callback
   */
  handleCallback(event: any) {
    const txid = event.txid;
    const payment = this.pendingPayments.get(txid);

    if (!payment) {
      console.warn('Received callback for unknown transaction:', txid);
      return;
    }

    switch (event.type) {
      case 'TRANSACTION_MINED':
        console.log('Payment mined:', txid);
        payment.status = 'mined';
        payment.blockHeight = event.blockHeight;
        payment.blockHash = event.blockHash;
        break;

      case 'TRANSACTION_CONFIRMED':
        console.log('Payment confirmed:', txid);
        payment.status = 'confirmed';
        payment.confirmations = event.confirmations;

        // Payment is now final - update business logic
        this.finalizePayment(payment);
        break;

      case 'TRANSACTION_REJECTED':
        console.error('Payment rejected:', txid);
        payment.status = 'rejected';
        payment.rejectReason = event.rejectReason;

        // Handle failed payment
        this.handleFailedPayment(payment);
        break;

      case 'DOUBLE_SPEND_ATTEMPT':
        console.error('Double-spend attempt:', txid);
        payment.status = 'double_spend';
        payment.competingTxid = event.competingTxid;

        // Alert and freeze
        this.handleDoubleSpend(payment);
        break;
    }
  }

  /**
   * Query payment status
   */
  async getPaymentStatus(txid: string): Promise<PaymentRecord> {
    const payment = this.pendingPayments.get(txid);
    if (!payment) {
      throw new Error('Payment not found');
    }

    // Refresh status from ARC
    try {
      const status = await this.arc.getTransactionStatus(txid);
      payment.status = status.status;
      payment.confirmations = status.confirmations;
      payment.blockHeight = status.blockHeight;
      payment.blockHash = status.blockHash;
    } catch (error) {
      console.error('Failed to refresh payment status:', error.message);
    }

    return payment;
  }

  private finalizePayment(payment: PaymentRecord) {
    console.log('Finalizing payment:', payment.txid);
    // Update database, send confirmation email, etc.
  }

  private handleFailedPayment(payment: PaymentRecord) {
    console.error('Handling failed payment:', payment.txid);
    // Refund, retry, notify user, etc.
  }

  private handleDoubleSpend(payment: PaymentRecord) {
    console.error('CRITICAL: Double-spend detected for payment:', payment.txid);
    // Freeze account, alert security, investigate, etc.
  }
}

interface PaymentRecord {
  txid: string;
  recipientAddress: string;
  amount: number;
  timestamp: number;
  status: string;
  metadata?: any;
  blockHeight?: number;
  blockHash?: string;
  confirmations?: number;
  rejectReason?: string;
  competingTxid?: string;
}

// Usage
const processor = new PaymentProcessor(
  'https://api.taal.com/arc',
  'mainnet_xxx',
  PrivateKey.fromWif('L5...'),
  'https://myapp.com/callbacks'
);

// Process payment
const result = await processor.processPayment(
  '1RecipientAddress...',
  10000, // 10000 satoshis
  [{ tx: utxoTx, outputIndex: 0 }],
  { orderId: '12345', customerId: 'user_abc' }
);

console.log('Payment TXID:', result.txid);
```

### Pattern 2: Batch Transaction Broadcasting

Efficiently broadcast multiple transactions with error handling:

```typescript
import { ARC, Transaction } from '@bsv/sdk';

class BatchBroadcaster {
  private arc: ARC;

  constructor(arcUrl: string, apiKey: string) {
    this.arc = new ARC(arcUrl, {
      apiKey,
      deploymentId: 'batch-broadcaster'
    });
  }

  /**
   * Broadcast multiple transactions with retry logic
   */
  async broadcastBatch(
    transactions: Transaction[],
    maxRetries: number = 3
  ): Promise<BatchResult> {
    const results: BatchResult = {
      successful: [],
      failed: [],
      total: transactions.length
    };

    // Get fee policy once for all transactions
    const feePolicy = await this.arc.getFeePolicy();
    console.log('Current fee policy:', feePolicy.standard, 'sats/KB');

    // Broadcast transactions sequentially (to maintain dependencies)
    for (const tx of transactions) {
      const txid = tx.id('hex');
      let attempts = 0;
      let success = false;

      while (attempts < maxRetries && !success) {
        attempts++;

        try {
          console.log(`Broadcasting ${txid} (attempt ${attempts}/${maxRetries})`);

          const response = await tx.broadcast(this.arc);

          results.successful.push({
            txid: response.txid,
            status: response.status,
            attempts
          });

          success = true;
          console.log(`✓ Success: ${txid}`);

        } catch (error) {
          console.error(`✗ Failed: ${txid} - ${error.message}`);

          if (attempts >= maxRetries) {
            results.failed.push({
              txid,
              error: error.message,
              code: error.code,
              attempts
            });
          } else {
            // Wait before retry
            await new Promise(resolve => setTimeout(resolve, 1000 * attempts));
          }
        }
      }
    }

    console.log('\nBatch broadcast complete:');
    console.log(`Total: ${results.total}`);
    console.log(`Successful: ${results.successful.length}`);
    console.log(`Failed: ${results.failed.length}`);

    return results;
  }

  /**
   * Broadcast with dependency ordering
   */
  async broadcastWithDependencies(
    transactions: Transaction[]
  ): Promise<BatchResult> {
    // Sort transactions by dependencies (parent before child)
    const sorted = this.topologicalSort(transactions);

    // Broadcast in order
    return this.broadcastBatch(sorted);
  }

  private topologicalSort(transactions: Transaction[]): Transaction[] {
    const txMap = new Map<string, Transaction>();
    const sorted: Transaction[] = [];
    const visited = new Set<string>();

    for (const tx of transactions) {
      txMap.set(tx.id('hex'), tx);
    }

    function visit(tx: Transaction) {
      const txid = tx.id('hex');
      if (visited.has(txid)) return;

      // Visit dependencies first
      for (const input of tx.inputs) {
        if (input.sourceTXID && txMap.has(input.sourceTXID)) {
          visit(txMap.get(input.sourceTXID)!);
        }
      }

      visited.add(txid);
      sorted.push(tx);
    }

    transactions.forEach(tx => visit(tx));
    return sorted;
  }
}

interface BatchResult {
  successful: Array<{ txid: string; status: string; attempts: number }>;
  failed: Array<{ txid: string; error: string; code?: string; attempts: number }>;
  total: number;
}

// Usage
const batcher = new BatchBroadcaster(
  'https://api.taal.com/arc',
  'mainnet_xxx'
);

const transactions = [tx1, tx2, tx3, tx4, tx5];

const results = await batcher.broadcastWithDependencies(transactions);

console.log('Broadcast results:');
console.log('Success rate:', (results.successful.length / results.total * 100).toFixed(2) + '%');

if (results.failed.length > 0) {
  console.error('Failed transactions:');
  results.failed.forEach(failure => {
    console.error(`  ${failure.txid}: ${failure.error}`);
  });
}
```

### Pattern 3: Transaction Monitoring Dashboard

Real-time transaction monitoring with ARC:

```typescript
import { ARC } from '@bsv/sdk';
import { EventEmitter } from 'events';

class TransactionMonitorDashboard extends EventEmitter {
  private arc: ARC;
  private transactions: Map<string, MonitoredTransaction>;
  private pollInterval: NodeJS.Timeout | null = null;

  constructor(arcUrl: string, apiKey: string) {
    super();
    this.arc = new ARC(arcUrl, { apiKey });
    this.transactions = new Map();
  }

  /**
   * Add transaction to monitor
   */
  addTransaction(txid: string, metadata?: any) {
    this.transactions.set(txid, {
      txid,
      status: 'UNKNOWN',
      addedAt: Date.now(),
      lastChecked: 0,
      checks: 0,
      metadata
    });

    console.log('Monitoring transaction:', txid);
    this.emit('added', { txid, metadata });
  }

  /**
   * Start monitoring
   */
  startMonitoring(intervalMs: number = 10000) {
    if (this.pollInterval) {
      console.warn('Monitoring already started');
      return;
    }

    console.log('Starting transaction monitoring');
    console.log('Poll interval:', intervalMs, 'ms');
    console.log('Transactions:', this.transactions.size);

    this.pollInterval = setInterval(async () => {
      await this.checkAllTransactions();
    }, intervalMs);

    // Initial check
    this.checkAllTransactions();
  }

  /**
   * Stop monitoring
   */
  stopMonitoring() {
    if (this.pollInterval) {
      clearInterval(this.pollInterval);
      this.pollInterval = null;
      console.log('Monitoring stopped');
    }
  }

  /**
   * Check all monitored transactions
   */
  private async checkAllTransactions() {
    const txids = Array.from(this.transactions.keys());

    console.log(`\nChecking ${txids.length} transactions...`);

    for (const txid of txids) {
      await this.checkTransaction(txid);
    }
  }

  /**
   * Check single transaction status
   */
  private async checkTransaction(txid: string) {
    const monitored = this.transactions.get(txid);
    if (!monitored) return;

    try {
      const status = await this.arc.getTransactionStatus(txid);

      monitored.lastChecked = Date.now();
      monitored.checks++;

      const previousStatus = monitored.status;
      monitored.status = status.status;
      monitored.blockHeight = status.blockHeight;
      monitored.blockHash = status.blockHash;
      monitored.confirmations = status.confirmations;

      // Emit event if status changed
      if (status.status !== previousStatus) {
        console.log(`Status change: ${txid} -> ${status.status}`);

        this.emit('statusChange', {
          txid,
          previousStatus,
          newStatus: status.status,
          transaction: monitored
        });
      }

      // Emit specific events
      if (status.status === 'MINED' && previousStatus !== 'MINED') {
        this.emit('mined', { txid, blockHeight: status.blockHeight });
      }

      if (status.status === 'REJECTED') {
        this.emit('rejected', { txid, reason: status.rejectReason });
        // Stop monitoring rejected transactions
        this.transactions.delete(txid);
      }

      if (status.confirmations && status.confirmations >= 6 && previousStatus !== 'CONFIRMED') {
        this.emit('confirmed', { txid, confirmations: status.confirmations });
        // Stop monitoring confirmed transactions
        this.transactions.delete(txid);
      }

    } catch (error) {
      console.error(`Failed to check ${txid}:`, error.message);
      monitored.lastError = error.message;
    }
  }

  /**
   * Get dashboard statistics
   */
  getStatistics(): DashboardStats {
    const stats: DashboardStats = {
      total: this.transactions.size,
      byStatus: {},
      averageAge: 0,
      oldestTransaction: null
    };

    let totalAge = 0;
    let oldestTime = Date.now();

    for (const [txid, tx] of this.transactions) {
      // Count by status
      stats.byStatus[tx.status] = (stats.byStatus[tx.status] || 0) + 1;

      // Calculate age
      const age = Date.now() - tx.addedAt;
      totalAge += age;

      if (tx.addedAt < oldestTime) {
        oldestTime = tx.addedAt;
        stats.oldestTransaction = {
          txid,
          age: age / 1000,
          status: tx.status
        };
      }
    }

    if (this.transactions.size > 0) {
      stats.averageAge = (totalAge / this.transactions.size) / 1000; // in seconds
    }

    return stats;
  }

  /**
   * Get all monitored transactions
   */
  getTransactions(): MonitoredTransaction[] {
    return Array.from(this.transactions.values());
  }
}

interface MonitoredTransaction {
  txid: string;
  status: string;
  addedAt: number;
  lastChecked: number;
  checks: number;
  blockHeight?: number;
  blockHash?: string;
  confirmations?: number;
  lastError?: string;
  metadata?: any;
}

interface DashboardStats {
  total: number;
  byStatus: Record<string, number>;
  averageAge: number;
  oldestTransaction: {
    txid: string;
    age: number;
    status: string;
  } | null;
}

// Usage
const dashboard = new TransactionMonitorDashboard(
  'https://api.taal.com/arc',
  'mainnet_xxx'
);

// Add event listeners
dashboard.on('statusChange', (event) => {
  console.log(`Status changed: ${event.txid}`);
  console.log(`  ${event.previousStatus} -> ${event.newStatus}`);
});

dashboard.on('mined', (event) => {
  console.log(`Transaction mined: ${event.txid}`);
  console.log(`  Block height: ${event.blockHeight}`);
});

dashboard.on('confirmed', (event) => {
  console.log(`Transaction confirmed: ${event.txid}`);
  console.log(`  Confirmations: ${event.confirmations}`);
});

dashboard.on('rejected', (event) => {
  console.error(`Transaction rejected: ${event.txid}`);
  console.error(`  Reason: ${event.reason}`);
});

// Add transactions to monitor
dashboard.addTransaction('txid1', { orderId: '12345' });
dashboard.addTransaction('txid2', { orderId: '12346' });
dashboard.addTransaction('txid3', { orderId: '12347' });

// Start monitoring (check every 10 seconds)
dashboard.startMonitoring(10000);

// Get statistics periodically
setInterval(() => {
  const stats = dashboard.getStatistics();
  console.log('\n--- Dashboard Statistics ---');
  console.log('Total monitored:', stats.total);
  console.log('By status:', stats.byStatus);
  console.log('Average age:', stats.averageAge.toFixed(2), 'seconds');
  if (stats.oldestTransaction) {
    console.log('Oldest:', stats.oldestTransaction.txid,
      `(${stats.oldestTransaction.age.toFixed(2)}s, ${stats.oldestTransaction.status})`);
  }
}, 30000);

// Stop monitoring after 1 hour
setTimeout(() => {
  dashboard.stopMonitoring();
  console.log('Monitoring stopped after 1 hour');
}, 3600000);
```

## Security Considerations

1. **API Key Protection**: Never expose API keys in client-side code. Store them securely in environment variables or secret management systems.

2. **Callback Authentication**: Always verify callback tokens to ensure webhooks are coming from ARC. Use HTTPS for callback URLs.

3. **Double-Spend Monitoring**: Implement immediate alerts for double-spend attempts. Consider transactions unsafe until they have sufficient confirmations.

4. **Rate Limiting**: Implement rate limiting for API calls to avoid service disruptions and additional costs.

5. **Error Handling**: Handle all possible error codes appropriately. Don't assume broadcasts always succeed.

6. **Fee Validation**: Always verify calculated fees meet current policy requirements before broadcasting.

## Performance Considerations

1. **Batch Status Queries**: When monitoring multiple transactions, batch status queries where possible to reduce API calls.

2. **Callback vs Polling**: Use callbacks instead of polling for production systems to reduce latency and API usage.

3. **Connection Pooling**: Reuse ARC instances across requests to benefit from HTTP connection pooling.

4. **Caching Fee Policies**: Cache fee policies for reasonable periods (e.g., 5 minutes) to reduce unnecessary queries.

5. **Async Operations**: Use async/await properly to avoid blocking operations. Consider Promise.all for parallel broadcasts when transactions are independent.

6. **Timeout Configuration**: Set appropriate timeouts for API calls based on your application's requirements.

## Related Components

- [Transaction](../transaction/README.md) - Create and manage transactions
- [BEEF](../beef/README.md) - Transaction envelope format
- <!-- [Broadcaster](../broadcaster/README.md) (Component pending) --> - Broadcasting interface
- <!-- [FeeModel](../fee-model/README.md) (Component pending) --> - Fee calculation strategies

## Code Examples

See complete working examples in:
- [Transaction Broadcasting](../../code-features/transaction-broadcasting/README.md)
- [Payment Processing](../../code-features/payment-processing/README.md)
- [Double-Spend Detection](../../code-features/double-spend-detection/README.md)
- [Batch Operations](../../code-features/batch-operations/README.md)

## Best Practices

1. **Always use API keys** for production environments to ensure authentication and rate limiting
2. **Implement webhook callbacks** instead of polling for better performance and lower latency
3. **Verify callback tokens** to ensure webhooks are authentic
4. **Monitor for double-spends** actively and implement alerts for suspicious activity
5. **Query fee policies** before broadcasting to ensure transactions meet requirements
6. **Handle all error codes** appropriately with user-friendly messages
7. **Use deployment IDs** to track which version of your application broadcast transactions
8. **Implement retry logic** for transient network failures
9. **Log all broadcast attempts** for audit trails and debugging
10. **Set reasonable timeouts** to prevent hanging operations

## Troubleshooting

### Issue: Broadcast fails with "INSUFFICIENT_FEE" error

**Solution**: Query current fee policy and recalculate fees.

```typescript
const feePolicy = await arc.getFeePolicy();
const feeModel = new SatoshisPerKilobyte(feePolicy.standard * 1.1); // 10% buffer
await tx.fee(feeModel);
await tx.sign();
await tx.broadcast(arc);
```

### Issue: Callback webhook not receiving events

**Solution**: Verify callback URL is publicly accessible and token is correct.

```typescript
// Test callback URL
const testCallback = async () => {
  const response = await fetch('https://myapp.com/callbacks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${callbackToken}`
    },
    body: JSON.stringify({
      type: 'TEST',
      txid: 'test123'
    })
  });
  console.log('Callback test:', response.status);
};
```

### Issue: Transaction status shows "UNKNOWN"

**Solution**: Transaction may not have reached ARC yet. Wait and retry.

```typescript
async function waitForStatus(txid: string, maxAttempts: number = 10) {
  for (let i = 0; i < maxAttempts; i++) {
    const status = await arc.getTransactionStatus(txid);
    if (status.status !== 'UNKNOWN') {
      return status;
    }
    await new Promise(resolve => setTimeout(resolve, 2000));
  }
  throw new Error('Transaction status remained UNKNOWN');
}
```

### Issue: API rate limit exceeded

**Solution**: Implement exponential backoff and request throttling.

```typescript
class RateLimitedARC {
  private lastRequest = 0;
  private minInterval = 100; // ms between requests

  async broadcast(tx: Transaction): Promise<any> {
    const now = Date.now();
    const elapsed = now - this.lastRequest;

    if (elapsed < this.minInterval) {
      await new Promise(resolve =>
        setTimeout(resolve, this.minInterval - elapsed)
      );
    }

    this.lastRequest = Date.now();
    return arc.broadcast(tx);
  }
}
```

## Further Reading

- [ARC API Documentation](https://docs.taal.com/arc) - Official ARC API documentation
- [Transaction Broadcasting](https://wiki.bitcoinsv.io/index.php/Transaction_broadcasting) - BSV Wiki on broadcasting
- [Double-Spend Prevention](https://wiki.bitcoinsv.io/index.php/Double-spending) - Understanding double-spends
- [Fee Calculation](https://wiki.bitcoinsv.io/index.php/Bitcoin_fees) - Bitcoin fee mechanisms
- [Webhook Security](https://webhooks.dev/security) - Webhook security best practices
- [BSV SDK Documentation](https://bsv-blockchain.github.io/ts-sdk) - Official SDK documentation

## Status

✅ Complete
