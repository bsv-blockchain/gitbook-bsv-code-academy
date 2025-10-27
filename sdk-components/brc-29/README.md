# BRC-29 (Simple Payments Protocol)

## Overview

BRC-29 is the Simple Payments Protocol standard that defines a structured way for applications to request, negotiate, and deliver payments on the BSV blockchain. It provides standardized PaymentRequest and PaymentResponse formats with integrated SPV envelope support, enabling direct payment negotiations between wallets and applications without requiring third-party payment processors. The BSV TypeScript SDK includes full BRC-29 implementation for building compliant payment systems.

## Purpose

- Create standardized payment requests with clear payment terms
- Generate payment responses with transaction and SPV proofs
- Enable direct wallet-to-application payment negotiations
- Provide proof of payment with merkle path verification
- Support multiple payment destinations in a single request
- Ensure payment authenticity through cryptographic verification
- Enable offline payment verification through SPV

## Basic Usage

### Creating a Payment Request

```typescript
import { PaymentRequest, P2PKH } from '@bsv/sdk';

// Create payment request
const paymentRequest: PaymentRequest = {
  network: 'mainnet',
  outputs: [
    {
      script: new P2PKH().lock('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa').toHex(),
      satoshis: 10000,
      description: 'Digital content purchase'
    }
  ],
  creationTimestamp: Date.now(),
  expirationTimestamp: Date.now() + (3600 * 1000), // 1 hour
  memo: 'Payment for article #12345',
  merchantData: JSON.stringify({
    orderId: '12345',
    customerId: 'user_abc',
    items: ['article-xyz']
  })
};

console.log('Payment request created');
console.log('Amount:', paymentRequest.outputs[0].satoshis, 'satoshis');
console.log('Expires:', new Date(paymentRequest.expirationTimestamp));
```

### Creating a Payment Response

```typescript
import { PaymentResponse, Transaction, PrivateKey, P2PKH, Beef } from '@bsv/sdk';

// Create transaction for payment
const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');
const tx = new Transaction();

// Add input
tx.addInput({
  sourceTransaction: Transaction.fromHex('...'),
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(privKey)
});

// Add payment output from request
tx.addOutput({
  lockingScript: new P2PKH().lock('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa'),
  satoshis: 10000
});

// Add change
tx.addOutput({
  lockingScript: new P2PKH().lock(privKey.toPublicKey().toAddress()),
  change: true
});

await tx.fee();
await tx.sign();

// Create BEEF envelope with transaction
const beef = new Beef();
beef.addTransaction(tx);

// Create payment response
const paymentResponse: PaymentResponse = {
  payment: beef.toHex(),
  memo: 'Payment for order #12345',
  paymentId: tx.id('hex')
};

console.log('Payment response created');
console.log('Transaction ID:', paymentResponse.paymentId);
```

## Key Features

### 1. PaymentRequest Structure and Creation

PaymentRequest defines the terms and requirements for a payment:

```typescript
import { PaymentRequest, P2PKH, Script, OP } from '@bsv/sdk';

interface PaymentRequest {
  network: 'mainnet' | 'testnet' | 'regtest';
  outputs: PaymentOutput[];
  creationTimestamp: number;
  expirationTimestamp?: number;
  memo?: string;
  merchantData?: string;
  paymentUrl?: string;
}

interface PaymentOutput {
  script: string;      // Hex-encoded locking script
  satoshis: number;    // Amount in satoshis
  description?: string; // Human-readable description
}

// Example: Simple payment request
function createSimplePaymentRequest(
  recipientAddress: string,
  amount: number,
  description: string,
  validForMinutes: number = 60
): PaymentRequest {
  return {
    network: 'mainnet',
    outputs: [
      {
        script: new P2PKH().lock(recipientAddress).toHex(),
        satoshis: amount,
        description
      }
    ],
    creationTimestamp: Date.now(),
    expirationTimestamp: Date.now() + (validForMinutes * 60 * 1000),
    memo: description
  };
}

// Example: Multi-output payment request
function createMultiOutputPaymentRequest(
  outputs: Array<{ address: string; amount: number; description: string }>,
  orderId: string
): PaymentRequest {
  return {
    network: 'mainnet',
    outputs: outputs.map(output => ({
      script: new P2PKH().lock(output.address).toHex(),
      satoshis: output.amount,
      description: output.description
    })),
    creationTimestamp: Date.now(),
    expirationTimestamp: Date.now() + (3600 * 1000), // 1 hour
    memo: `Payment for order ${orderId}`,
    merchantData: JSON.stringify({
      orderId,
      timestamp: Date.now(),
      outputs: outputs.length
    })
  };
}

// Example: Payment request with OP_RETURN data
function createPaymentWithData(
  recipientAddress: string,
  amount: number,
  data: string
): PaymentRequest {
  return {
    network: 'mainnet',
    outputs: [
      // Payment output
      {
        script: new P2PKH().lock(recipientAddress).toHex(),
        satoshis: amount,
        description: 'Payment'
      },
      // Data output
      {
        script: new Script()
          .writeOpCode(OP.OP_FALSE)
          .writeOpCode(OP.OP_RETURN)
          .writeBin(Buffer.from(data, 'utf8'))
          .toHex(),
        satoshis: 0,
        description: 'Data storage'
      }
    ],
    creationTimestamp: Date.now(),
    expirationTimestamp: Date.now() + (3600 * 1000),
    memo: 'Payment with attached data'
  };
}

// Usage
const simpleRequest = createSimplePaymentRequest(
  '1MerchantAddress...',
  5000,
  'Coffee purchase',
  30 // Valid for 30 minutes
);

const multiRequest = createMultiOutputPaymentRequest(
  [
    { address: '1Merchant1...', amount: 3000, description: 'Product A' },
    { address: '1Merchant2...', amount: 2000, description: 'Product B' },
    { address: '1Charity...', amount: 500, description: 'Donation' }
  ],
  'ORDER-12345'
);

const dataRequest = createPaymentWithData(
  '1MerchantAddress...',
  1000,
  JSON.stringify({ orderId: '12345', timestamp: Date.now() })
);
```

### 2. PaymentResponse with SPV Envelopes

PaymentResponse includes the transaction and SPV proof in BEEF format:

```typescript
import {
  PaymentResponse,
  Transaction,
  PrivateKey,
  P2PKH,
  Beef,
  MerklePath
} from '@bsv/sdk';

interface PaymentResponse {
  payment: string;      // BEEF-encoded transaction(s) in hex
  memo?: string;        // Optional note from payer
  paymentId?: string;   // Transaction ID for reference
}

// Create payment response with SPV proof
async function createPaymentResponse(
  paymentRequest: PaymentRequest,
  sourceUtxos: Array<{
    tx: Transaction;
    outputIndex: number;
    merklePath?: MerklePath;
  }>,
  payerPrivKey: PrivateKey
): Promise<PaymentResponse> {
  // Create transaction
  const tx = new Transaction();

  // Add inputs from UTXOs
  for (const utxo of sourceUtxos) {
    tx.addInput({
      sourceTransaction: utxo.tx,
      sourceOutputIndex: utxo.outputIndex,
      unlockingScriptTemplate: new P2PKH().unlock(payerPrivKey)
    });
  }

  // Add outputs from payment request
  for (const output of paymentRequest.outputs) {
    tx.addOutput({
      lockingScript: Script.fromHex(output.script),
      satoshis: output.satoshis
    });
  }

  // Add change output
  tx.addOutput({
    lockingScript: new P2PKH().lock(payerPrivKey.toPublicKey().toAddress()),
    change: true
  });

  // Calculate fee and sign
  await tx.fee();
  await tx.sign();

  // Create BEEF envelope
  const beef = new Beef();

  // Add source transactions with merkle proofs
  for (const utxo of sourceUtxos) {
    beef.addTransaction(utxo.tx);
    if (utxo.merklePath) {
      beef.addMerklePath(utxo.merklePath);
    }
  }

  // Add payment transaction
  beef.addTransaction(tx);

  // Create payment response
  const response: PaymentResponse = {
    payment: beef.toHex(),
    memo: 'Payment completed',
    paymentId: tx.id('hex')
  };

  return response;
}

// Verify payment response
async function verifyPaymentResponse(
  paymentRequest: PaymentRequest,
  paymentResponse: PaymentResponse,
  chainTracker: ChainTracker
): Promise<{ valid: boolean; errors: string[] }> {
  const errors: string[] = [];

  try {
    // Parse BEEF envelope
    const beef = Beef.fromHex(paymentResponse.payment);

    // Verify SPV proofs
    const spvValid = await beef.verify(chainTracker);
    if (!spvValid) {
      errors.push('SPV verification failed');
      return { valid: false, errors };
    }

    // Get payment transaction (last transaction in BEEF)
    const transactions = beef.getTransactions();
    const paymentTx = transactions[transactions.length - 1];

    // Verify payment ID matches
    if (paymentResponse.paymentId && paymentTx.id('hex') !== paymentResponse.paymentId) {
      errors.push('Payment ID mismatch');
    }

    // Verify each requested output is in the transaction
    for (let i = 0; i < paymentRequest.outputs.length; i++) {
      const requestedOutput = paymentRequest.outputs[i];
      const txOutput = paymentTx.outputs[i];

      if (!txOutput) {
        errors.push(`Missing output ${i}`);
        continue;
      }

      // Verify script matches
      if (txOutput.lockingScript?.toHex() !== requestedOutput.script) {
        errors.push(`Output ${i} script mismatch`);
      }

      // Verify amount matches
      if (txOutput.satoshis !== requestedOutput.satoshis) {
        errors.push(
          `Output ${i} amount mismatch: ` +
          `expected ${requestedOutput.satoshis}, got ${txOutput.satoshis}`
        );
      }
    }

    // Check expiration
    if (paymentRequest.expirationTimestamp &&
        Date.now() > paymentRequest.expirationTimestamp) {
      errors.push('Payment request expired');
    }

    return {
      valid: errors.length === 0,
      errors
    };

  } catch (e) {
    errors.push(`Verification error: ${e.message}`);
    return { valid: false, errors };
  }
}

// Usage
const request = createSimplePaymentRequest(
  '1MerchantAddress...',
  5000,
  'Coffee purchase'
);

const response = await createPaymentResponse(
  request,
  [{ tx: utxoTx, outputIndex: 0 }],
  payerPrivKey
);

const chainTracker = new WhatsOnChainTracker();
const verification = await verifyPaymentResponse(request, response, chainTracker);

if (verification.valid) {
  console.log('Payment verified successfully');
} else {
  console.error('Payment verification failed:', verification.errors);
}
```

### 3. Direct Payment Negotiation

BRC-29 enables direct payment negotiation between wallets and applications:

```typescript
import { PaymentRequest, PaymentResponse } from '@bsv/sdk';
import express from 'express';

// Merchant server implementing BRC-29
class BRC29PaymentServer {
  private app: express.Application;
  private pendingPayments: Map<string, PaymentRequest>;

  constructor(port: number) {
    this.app = express();
    this.app.use(express.json());
    this.pendingPayments = new Map();

    this.setupRoutes();
    this.app.listen(port, () => {
      console.log(`BRC-29 payment server listening on port ${port}`);
    });
  }

  private setupRoutes() {
    // Create payment request
    this.app.post('/payment-request', (req, res) => {
      const { amount, description, orderId } = req.body;

      const paymentRequest: PaymentRequest = {
        network: 'mainnet',
        outputs: [
          {
            script: new P2PKH().lock(this.getMerchantAddress()).toHex(),
            satoshis: amount,
            description
          }
        ],
        creationTimestamp: Date.now(),
        expirationTimestamp: Date.now() + (3600 * 1000), // 1 hour
        memo: description,
        merchantData: JSON.stringify({ orderId }),
        paymentUrl: `https://merchant.com/payment/${orderId}`
      };

      // Store payment request
      this.pendingPayments.set(orderId, paymentRequest);

      res.json(paymentRequest);
    });

    // Submit payment
    this.app.post('/payment/:orderId', async (req, res) => {
      const { orderId } = req.params;
      const paymentResponse: PaymentResponse = req.body;

      const paymentRequest = this.pendingPayments.get(orderId);
      if (!paymentRequest) {
        return res.status(404).json({ error: 'Payment request not found' });
      }

      // Verify payment
      const verification = await verifyPaymentResponse(
        paymentRequest,
        paymentResponse,
        chainTracker
      );

      if (!verification.valid) {
        return res.status(400).json({
          error: 'Payment verification failed',
          details: verification.errors
        });
      }

      // Payment is valid - fulfill order
      await this.fulfillOrder(orderId, paymentResponse.paymentId!);

      // Clean up
      this.pendingPayments.delete(orderId);

      res.json({
        success: true,
        message: 'Payment accepted',
        txid: paymentResponse.paymentId
      });
    });

    // Get payment status
    this.app.get('/payment-status/:orderId', (req, res) => {
      const { orderId } = req.params;
      const paymentRequest = this.pendingPayments.get(orderId);

      if (!paymentRequest) {
        return res.json({ status: 'completed' });
      }

      const isExpired = paymentRequest.expirationTimestamp &&
                       Date.now() > paymentRequest.expirationTimestamp;

      res.json({
        status: isExpired ? 'expired' : 'pending',
        expiresAt: paymentRequest.expirationTimestamp,
        amount: paymentRequest.outputs[0].satoshis
      });
    });
  }

  private getMerchantAddress(): string {
    return process.env.MERCHANT_ADDRESS!;
  }

  private async fulfillOrder(orderId: string, txid: string) {
    console.log(`Fulfilling order ${orderId} with payment ${txid}`);
    // Business logic to fulfill order
  }
}

// Wallet client implementing BRC-29
class BRC29WalletClient {
  private privKey: PrivateKey;
  private utxos: Array<{ tx: Transaction; outputIndex: number }>;

  constructor(privKey: PrivateKey) {
    this.privKey = privKey;
    this.utxos = [];
  }

  /**
   * Request payment from merchant
   */
  async requestPayment(
    merchantUrl: string,
    amount: number,
    description: string,
    orderId: string
  ): Promise<PaymentRequest> {
    const response = await fetch(`${merchantUrl}/payment-request`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ amount, description, orderId })
    });

    const paymentRequest: PaymentRequest = await response.json();
    return paymentRequest;
  }

  /**
   * Make payment
   */
  async makePayment(
    merchantUrl: string,
    paymentRequest: PaymentRequest,
    orderId: string
  ): Promise<{ success: boolean; txid?: string; error?: string }> {
    try {
      // Create payment response
      const paymentResponse = await createPaymentResponse(
        paymentRequest,
        this.utxos,
        this.privKey
      );

      // Submit payment
      const response = await fetch(`${merchantUrl}/payment/${orderId}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(paymentResponse)
      });

      const result = await response.json();

      if (response.ok) {
        return {
          success: true,
          txid: result.txid
        };
      } else {
        return {
          success: false,
          error: result.error
        };
      }
    } catch (e) {
      return {
        success: false,
        error: e.message
      };
    }
  }

  /**
   * Complete payment flow
   */
  async pay(
    merchantUrl: string,
    amount: number,
    description: string,
    orderId: string
  ): Promise<string> {
    // Step 1: Request payment terms
    console.log('Requesting payment terms...');
    const paymentRequest = await this.requestPayment(
      merchantUrl,
      amount,
      description,
      orderId
    );

    console.log('Payment request received:');
    console.log('Amount:', paymentRequest.outputs[0].satoshis, 'satoshis');
    console.log('Expires:', new Date(paymentRequest.expirationTimestamp!));

    // Step 2: User approves payment (in real app, show UI)
    const approved = true; // User approval

    if (!approved) {
      throw new Error('Payment cancelled by user');
    }

    // Step 3: Make payment
    console.log('Creating and submitting payment...');
    const result = await this.makePayment(merchantUrl, paymentRequest, orderId);

    if (result.success) {
      console.log('Payment successful!');
      console.log('Transaction ID:', result.txid);
      return result.txid!;
    } else {
      throw new Error(`Payment failed: ${result.error}`);
    }
  }
}

// Usage
const server = new BRC29PaymentServer(3000);

const wallet = new BRC29WalletClient(
  PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB')
);

const txid = await wallet.pay(
  'https://merchant.com',
  5000,
  'Coffee purchase',
  'ORDER-12345'
);

console.log('Payment completed:', txid);
```

### 4. Reference Implementation Patterns

Complete reference patterns for common BRC-29 use cases:

```typescript
import {
  PaymentRequest,
  PaymentResponse,
  Transaction,
  PrivateKey,
  P2PKH,
  Beef,
  MerklePath
} from '@bsv/sdk';

// Pattern: Subscription payment
function createSubscriptionPaymentRequest(
  recipientAddress: string,
  monthlyAmount: number,
  months: number,
  subscriptionId: string
): PaymentRequest {
  return {
    network: 'mainnet',
    outputs: [
      {
        script: new P2PKH().lock(recipientAddress).toHex(),
        satoshis: monthlyAmount * months,
        description: `${months}-month subscription`
      }
    ],
    creationTimestamp: Date.now(),
    expirationTimestamp: Date.now() + (24 * 3600 * 1000), // 24 hours
    memo: `Subscription payment for ${months} months`,
    merchantData: JSON.stringify({
      type: 'subscription',
      subscriptionId,
      months,
      monthlyAmount,
      startDate: Date.now()
    })
  };
}

// Pattern: Tipping / donation
function createTipPaymentRequest(
  creatorAddress: string,
  suggestedAmounts: number[],
  creatorName: string,
  contentId: string
): PaymentRequest {
  // Use first suggested amount as default
  const defaultAmount = suggestedAmounts[0];

  return {
    network: 'mainnet',
    outputs: [
      {
        script: new P2PKH().lock(creatorAddress).toHex(),
        satoshis: defaultAmount,
        description: `Tip for ${creatorName}`
      }
    ],
    creationTimestamp: Date.now(),
    memo: `Support ${creatorName}`,
    merchantData: JSON.stringify({
      type: 'tip',
      creatorName,
      contentId,
      suggestedAmounts
    })
  };
}

// Pattern: Split payment
function createSplitPaymentRequest(
  recipients: Array<{ address: string; amount: number; name: string }>,
  orderId: string,
  description: string
): PaymentRequest {
  return {
    network: 'mainnet',
    outputs: recipients.map(recipient => ({
      script: new P2PKH().lock(recipient.address).toHex(),
      satoshis: recipient.amount,
      description: `Payment to ${recipient.name}`
    })),
    creationTimestamp: Date.now(),
    expirationTimestamp: Date.now() + (3600 * 1000),
    memo: description,
    merchantData: JSON.stringify({
      type: 'split',
      orderId,
      recipients: recipients.map(r => ({ name: r.name, amount: r.amount }))
    })
  };
}

// Pattern: Conditional payment with refund address
function createConditionalPaymentRequest(
  merchantAddress: string,
  amount: number,
  refundAddress: string,
  condition: string,
  orderId: string
): PaymentRequest {
  return {
    network: 'mainnet',
    outputs: [
      {
        script: new P2PKH().lock(merchantAddress).toHex(),
        satoshis: amount,
        description: 'Conditional payment'
      }
    ],
    creationTimestamp: Date.now(),
    expirationTimestamp: Date.now() + (7 * 24 * 3600 * 1000), // 7 days
    memo: `Conditional payment: ${condition}`,
    merchantData: JSON.stringify({
      type: 'conditional',
      orderId,
      condition,
      refundAddress,
      terms: 'Refund if condition not met within 7 days'
    })
  };
}

// Pattern: Batch payment processing
class BatchPaymentProcessor {
  /**
   * Process multiple payments efficiently
   */
  static async processPaymentBatch(
    payments: Array<{
      request: PaymentRequest;
      orderId: string;
    }>
  ): Promise<Array<{ orderId: string; success: boolean; txid?: string; error?: string }>> {
    const results: Array<any> = [];

    for (const payment of payments) {
      try {
        // Verify payment request is valid
        if (payment.request.expirationTimestamp &&
            Date.now() > payment.request.expirationTimestamp) {
          results.push({
            orderId: payment.orderId,
            success: false,
            error: 'Payment request expired'
          });
          continue;
        }

        // Process payment
        // (In real implementation, would create transaction and broadcast)
        const txid = 'mock_txid_' + payment.orderId;

        results.push({
          orderId: payment.orderId,
          success: true,
          txid
        });

      } catch (e) {
        results.push({
          orderId: payment.orderId,
          success: false,
          error: e.message
        });
      }
    }

    return results;
  }
}

// Usage examples
const subscriptionRequest = createSubscriptionPaymentRequest(
  '1MerchantAddr...',
  1000,
  12,
  'SUB-123'
);

const tipRequest = createTipPaymentRequest(
  '1CreatorAddr...',
  [100, 500, 1000, 5000],
  'Alice Creator',
  'article-xyz'
);

const splitRequest = createSplitPaymentRequest(
  [
    { address: '1Merchant...', amount: 7000, name: 'Merchant' },
    { address: '1Platform...', amount: 2000, name: 'Platform' },
    { address: '1Charity...', amount: 1000, name: 'Charity' }
  ],
  'ORDER-999',
  'Split payment for order'
);

const conditionalRequest = createConditionalPaymentRequest(
  '1Merchant...',
  10000,
  '1Refund...',
  'Product delivered within 7 days',
  'ORDER-888'
);
```

## API Reference

### PaymentRequest Interface

```typescript
interface PaymentRequest {
  network: 'mainnet' | 'testnet' | 'regtest';
  outputs: PaymentOutput[];
  creationTimestamp: number;
  expirationTimestamp?: number;
  memo?: string;
  merchantData?: string;
  paymentUrl?: string;
}

interface PaymentOutput {
  script: string;       // Hex-encoded locking script
  satoshis: number;     // Amount in satoshis
  description?: string; // Human-readable output description
}
```

**Properties:**
- `network` - Target BSV network
- `outputs` - Array of payment outputs required
- `creationTimestamp` - When request was created (Unix timestamp ms)
- `expirationTimestamp` - Optional expiration time (Unix timestamp ms)
- `memo` - Optional human-readable note
- `merchantData` - Optional data for merchant use (recommend JSON string)
- `paymentUrl` - Optional URL where payment should be submitted

### PaymentResponse Interface

```typescript
interface PaymentResponse {
  payment: string;      // BEEF-encoded transaction(s) in hex
  memo?: string;        // Optional note from payer
  paymentId?: string;   // Transaction ID for reference
}
```

**Properties:**
- `payment` - BEEF envelope containing transaction and proofs (hex)
- `memo` - Optional note from payer
- `paymentId` - Transaction ID of payment transaction

### Helper Functions

#### `createPaymentRequest(params): PaymentRequest`

Creates a standardized payment request.

**Parameters:**
- `params.network` - Target network
- `params.outputs` - Payment outputs
- `params.validForMinutes` - Validity duration
- `params.memo` - Payment description
- `params.merchantData` - Additional data

#### `verifyPaymentResponse(request, response, chainTracker): Promise<VerificationResult>`

Verifies a payment response against its request.

**Parameters:**
- `request: PaymentRequest` - Original payment request
- `response: PaymentResponse` - Payment response to verify
- `chainTracker: ChainTracker` - For SPV verification

**Returns:** `Promise<VerificationResult>`
- `valid: boolean` - Whether payment is valid
- `errors: string[]` - Array of validation errors

## Common Patterns

### Pattern 1: E-commerce Checkout Integration

Complete e-commerce checkout with BRC-29:

```typescript
import {
  PaymentRequest,
  PaymentResponse,
  Transaction,
  PrivateKey,
  P2PKH,
  Beef
} from '@bsv/sdk';

class EcommerceCheckout {
  private merchantAddress: string;
  private orders: Map<string, Order>;

  constructor(merchantAddress: string) {
    this.merchantAddress = merchantAddress;
    this.orders = new Map();
  }

  /**
   * Create checkout session
   */
  createCheckout(
    items: Array<{ id: string; name: string; price: number }>,
    customerId: string
  ): { orderId: string; paymentRequest: PaymentRequest } {
    // Calculate total
    const total = items.reduce((sum, item) => sum + item.price, 0);

    // Generate order ID
    const orderId = `ORDER-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;

    // Create order record
    const order: Order = {
      orderId,
      customerId,
      items,
      total,
      status: 'pending',
      createdAt: Date.now()
    };

    this.orders.set(orderId, order);

    // Create payment request
    const paymentRequest: PaymentRequest = {
      network: 'mainnet',
      outputs: [
        {
          script: new P2PKH().lock(this.merchantAddress).toHex(),
          satoshis: total,
          description: `Order ${orderId}`
        }
      ],
      creationTimestamp: Date.now(),
      expirationTimestamp: Date.now() + (15 * 60 * 1000), // 15 minutes
      memo: `Payment for ${items.length} item(s)`,
      merchantData: JSON.stringify({
        orderId,
        customerId,
        items: items.map(item => ({ id: item.id, name: item.name, price: item.price }))
      }),
      paymentUrl: `https://shop.example.com/api/payment/${orderId}`
    };

    return { orderId, paymentRequest };
  }

  /**
   * Process payment
   */
  async processPayment(
    orderId: string,
    paymentResponse: PaymentResponse
  ): Promise<{ success: boolean; error?: string }> {
    const order = this.orders.get(orderId);
    if (!order) {
      return { success: false, error: 'Order not found' };
    }

    if (order.status !== 'pending') {
      return { success: false, error: 'Order already processed' };
    }

    try {
      // Parse BEEF
      const beef = Beef.fromHex(paymentResponse.payment);
      const transactions = beef.getTransactions();
      const paymentTx = transactions[transactions.length - 1];

      // Verify payment amount
      const paymentOutput = paymentTx.outputs[0];
      if (paymentOutput.satoshis !== order.total) {
        return {
          success: false,
          error: `Incorrect amount: expected ${order.total}, got ${paymentOutput.satoshis}`
        };
      }

      // Verify payment address
      const expectedScript = new P2PKH().lock(this.merchantAddress).toHex();
      if (paymentOutput.lockingScript?.toHex() !== expectedScript) {
        return { success: false, error: 'Incorrect payment address' };
      }

      // Update order
      order.status = 'paid';
      order.txid = paymentTx.id('hex');
      order.paidAt = Date.now();

      // Fulfill order
      await this.fulfillOrder(order);

      return { success: true };

    } catch (e) {
      return { success: false, error: e.message };
    }
  }

  /**
   * Get order status
   */
  getOrderStatus(orderId: string): OrderStatus | null {
    const order = this.orders.get(orderId);
    if (!order) {
      return null;
    }

    return {
      orderId: order.orderId,
      status: order.status,
      total: order.total,
      txid: order.txid,
      createdAt: order.createdAt,
      paidAt: order.paidAt
    };
  }

  private async fulfillOrder(order: Order) {
    console.log(`Fulfilling order ${order.orderId}`);
    // Ship products, send digital goods, etc.
  }
}

interface Order {
  orderId: string;
  customerId: string;
  items: Array<{ id: string; name: string; price: number }>;
  total: number;
  status: 'pending' | 'paid' | 'fulfilled' | 'cancelled';
  createdAt: number;
  txid?: string;
  paidAt?: number;
}

interface OrderStatus {
  orderId: string;
  status: string;
  total: number;
  txid?: string;
  createdAt: number;
  paidAt?: number;
}

// Usage
const checkout = new EcommerceCheckout('1MerchantAddr...');

// Customer creates checkout
const { orderId, paymentRequest } = checkout.createCheckout(
  [
    { id: 'prod1', name: 'Widget A', price: 2000 },
    { id: 'prod2', name: 'Widget B', price: 3000 }
  ],
  'customer@example.com'
);

console.log('Order ID:', orderId);
console.log('Total:', paymentRequest.outputs[0].satoshis, 'satoshis');
console.log('Expires:', new Date(paymentRequest.expirationTimestamp!));

// Customer makes payment
// (wallet creates payment response)
const paymentResponse: PaymentResponse = {
  payment: '...beef hex...',
  paymentId: 'txid...'
};

// Process payment
const result = await checkout.processPayment(orderId, paymentResponse);

if (result.success) {
  console.log('Payment successful!');
  const status = checkout.getOrderStatus(orderId);
  console.log('Order status:', status?.status);
} else {
  console.error('Payment failed:', result.error);
}
```

### Pattern 2: Content Paywall with BRC-29

Implement content paywall using BRC-29:

```typescript
import { PaymentRequest, PaymentResponse } from '@bsv/sdk';

class ContentPaywall {
  private content: Map<string, Content>;
  private accessGrants: Map<string, Set<string>>; // contentId -> Set of txids

  constructor() {
    this.content = new Map();
    this.accessGrants = new Map();
  }

  /**
   * Create content with paywall
   */
  publishContent(
    contentId: string,
    title: string,
    price: number,
    contentData: string
  ) {
    this.content.set(contentId, {
      id: contentId,
      title,
      price,
      contentData,
      publishedAt: Date.now()
    });

    this.accessGrants.set(contentId, new Set());
  }

  /**
   * Request access to content
   */
  requestAccess(contentId: string, recipientAddress: string): PaymentRequest | null {
    const content = this.content.get(contentId);
    if (!content) {
      return null;
    }

    return {
      network: 'mainnet',
      outputs: [
        {
          script: new P2PKH().lock(recipientAddress).toHex(),
          satoshis: content.price,
          description: `Access to: ${content.title}`
        }
      ],
      creationTimestamp: Date.now(),
      expirationTimestamp: Date.now() + (24 * 3600 * 1000), // 24 hours
      memo: `Purchase access to ${content.title}`,
      merchantData: JSON.stringify({
        type: 'content_access',
        contentId,
        title: content.title
      })
    };
  }

  /**
   * Grant access after payment
   */
  async grantAccess(
    contentId: string,
    paymentResponse: PaymentResponse
  ): Promise<{ success: boolean; accessToken?: string; error?: string }> {
    const content = this.content.get(contentId);
    if (!content) {
      return { success: false, error: 'Content not found' };
    }

    try {
      // Verify payment
      const beef = Beef.fromHex(paymentResponse.payment);
      const transactions = beef.getTransactions();
      const paymentTx = transactions[transactions.length - 1];
      const txid = paymentTx.id('hex');

      // Check if already granted
      if (this.accessGrants.get(contentId)?.has(txid)) {
        return { success: false, error: 'Access already granted for this payment' };
      }

      // Grant access
      this.accessGrants.get(contentId)!.add(txid);

      // Generate access token
      const accessToken = this.generateAccessToken(contentId, txid);

      return {
        success: true,
        accessToken
      };

    } catch (e) {
      return { success: false, error: e.message };
    }
  }

  /**
   * Retrieve content with access token
   */
  getContent(
    contentId: string,
    accessToken: string
  ): { success: boolean; content?: string; error?: string } {
    const content = this.content.get(contentId);
    if (!content) {
      return { success: false, error: 'Content not found' };
    }

    // Verify access token
    const { valid, txid } = this.verifyAccessToken(contentId, accessToken);
    if (!valid || !txid) {
      return { success: false, error: 'Invalid access token' };
    }

    // Check if access was granted
    if (!this.accessGrants.get(contentId)?.has(txid)) {
      return { success: false, error: 'Access not granted' };
    }

    return {
      success: true,
      content: content.contentData
    };
  }

  private generateAccessToken(contentId: string, txid: string): string {
    // In production, use proper JWT or similar
    return Buffer.from(`${contentId}:${txid}`).toString('base64');
  }

  private verifyAccessToken(
    contentId: string,
    accessToken: string
  ): { valid: boolean; txid?: string } {
    try {
      const decoded = Buffer.from(accessToken, 'base64').toString('utf8');
      const [tokenContentId, txid] = decoded.split(':');

      if (tokenContentId !== contentId) {
        return { valid: false };
      }

      return { valid: true, txid };
    } catch (e) {
      return { valid: false };
    }
  }
}

interface Content {
  id: string;
  title: string;
  price: number;
  contentData: string;
  publishedAt: number;
}

// Usage
const paywall = new ContentPaywall();

// Publish content
paywall.publishContent(
  'article-123',
  'Understanding Bitcoin',
  1000,
  'Full article content here...'
);

// User requests access
const paymentRequest = paywall.requestAccess('article-123', '1AuthorAddr...');
console.log('Price:', paymentRequest?.outputs[0].satoshis, 'satoshis');

// User makes payment
const paymentResponse: PaymentResponse = {
  payment: '...beef hex...',
  paymentId: 'txid...'
};

// Grant access
const accessResult = await paywall.grantAccess('article-123', paymentResponse);

if (accessResult.success) {
  console.log('Access granted!');
  console.log('Access token:', accessResult.accessToken);

  // Retrieve content
  const contentResult = paywall.getContent('article-123', accessResult.accessToken!);

  if (contentResult.success) {
    console.log('Content:', contentResult.content);
  }
}
```

### Pattern 3: Peer-to-Peer Marketplace

P2P marketplace using BRC-29:

```typescript
import { PaymentRequest, PaymentResponse, Transaction, PrivateKey, P2PKH } from '@bsv/sdk';

class P2PMarketplace {
  private listings: Map<string, Listing>;
  private escrow: Map<string, EscrowTransaction>;

  constructor() {
    this.listings = new Map();
    this.escrow = new Map();
  }

  /**
   * Create listing
   */
  createListing(
    sellerId: string,
    sellerAddress: string,
    item: {
      title: string;
      description: string;
      price: number;
      category: string;
    }
  ): string {
    const listingId = `LIST-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;

    this.listings.set(listingId, {
      id: listingId,
      sellerId,
      sellerAddress,
      item,
      status: 'active',
      createdAt: Date.now()
    });

    return listingId;
  }

  /**
   * Initiate purchase (creates payment request)
   */
  initiatePurchase(
    listingId: string,
    buyerId: string,
    escrowAddress: string
  ): { purchaseId: string; paymentRequest: PaymentRequest } | null {
    const listing = this.listings.get(listingId);
    if (!listing || listing.status !== 'active') {
      return null;
    }

    const purchaseId = `PURCH-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;

    // Create payment request to escrow
    const paymentRequest: PaymentRequest = {
      network: 'mainnet',
      outputs: [
        {
          script: new P2PKH().lock(escrowAddress).toHex(),
          satoshis: listing.item.price,
          description: `Escrow for ${listing.item.title}`
        }
      ],
      creationTimestamp: Date.now(),
      expirationTimestamp: Date.now() + (3600 * 1000),
      memo: `Purchase ${listing.item.title}`,
      merchantData: JSON.stringify({
        type: 'escrow',
        listingId,
        purchaseId,
        buyerId,
        sellerId: listing.sellerId
      })
    };

    // Store escrow transaction
    this.escrow.set(purchaseId, {
      purchaseId,
      listingId,
      buyerId,
      sellerId: listing.sellerId,
      amount: listing.item.price,
      escrowAddress,
      status: 'awaiting_payment',
      createdAt: Date.now()
    });

    return { purchaseId, paymentRequest };
  }

  /**
   * Complete purchase (buyer pays into escrow)
   */
  async completePurchase(
    purchaseId: string,
    paymentResponse: PaymentResponse
  ): Promise<{ success: boolean; error?: string }> {
    const escrowTx = this.escrow.get(purchaseId);
    if (!escrowTx) {
      return { success: false, error: 'Purchase not found' };
    }

    if (escrowTx.status !== 'awaiting_payment') {
      return { success: false, error: 'Purchase already paid' };
    }

    // Verify payment
    const beef = Beef.fromHex(paymentResponse.payment);
    const transactions = beef.getTransactions();
    const paymentTx = transactions[transactions.length - 1];

    escrowTx.escrowTxid = paymentTx.id('hex');
    escrowTx.status = 'in_escrow';
    escrowTx.paidAt = Date.now();

    // Update listing
    const listing = this.listings.get(escrowTx.listingId);
    if (listing) {
      listing.status = 'sold';
    }

    return { success: true };
  }

  /**
   * Release escrow to seller (after delivery confirmed)
   */
  async releaseEscrow(
    purchaseId: string,
    escrowPrivKey: PrivateKey
  ): Promise<{ success: boolean; txid?: string; error?: string }> {
    const escrowTx = this.escrow.get(purchaseId);
    if (!escrowTx || escrowTx.status !== 'in_escrow') {
      return { success: false, error: 'Invalid escrow state' };
    }

    const listing = this.listings.get(escrowTx.listingId);
    if (!listing) {
      return { success: false, error: 'Listing not found' };
    }

    try {
      // Create release transaction
      const releaseTx = new Transaction();

      // Spend from escrow
      // (In production, fetch actual escrow UTXO)
      releaseTx.addInput({
        sourceTXID: escrowTx.escrowTxid,
        sourceOutputIndex: 0,
        unlockingScriptTemplate: new P2PKH().unlock(escrowPrivKey)
      });

      // Pay seller
      releaseTx.addOutput({
        lockingScript: new P2PKH().lock(listing.sellerAddress),
        satoshis: escrowTx.amount
      });

      await releaseTx.fee();
      await releaseTx.sign();

      // Broadcast release transaction
      // await releaseTx.broadcast(broadcaster);

      escrowTx.status = 'released';
      escrowTx.releasedAt = Date.now();
      escrowTx.releaseTxid = releaseTx.id('hex');

      return {
        success: true,
        txid: releaseTx.id('hex')
      };

    } catch (e) {
      return { success: false, error: e.message };
    }
  }
}

interface Listing {
  id: string;
  sellerId: string;
  sellerAddress: string;
  item: {
    title: string;
    description: string;
    price: number;
    category: string;
  };
  status: 'active' | 'sold' | 'cancelled';
  createdAt: number;
}

interface EscrowTransaction {
  purchaseId: string;
  listingId: string;
  buyerId: string;
  sellerId: string;
  amount: number;
  escrowAddress: string;
  status: 'awaiting_payment' | 'in_escrow' | 'released' | 'refunded';
  createdAt: number;
  escrowTxid?: string;
  paidAt?: number;
  releaseTxid?: string;
  releasedAt?: number;
}

// Usage
const marketplace = new P2PMarketplace();

// Seller creates listing
const listingId = marketplace.createListing(
  'seller123',
  '1SellerAddr...',
  {
    title: 'Vintage Camera',
    description: 'Excellent condition',
    price: 50000,
    category: 'Electronics'
  }
);

// Buyer initiates purchase
const purchase = marketplace.initiatePurchase(
  listingId,
  'buyer456',
  '1EscrowAddr...'
);

if (purchase) {
  console.log('Purchase ID:', purchase.purchaseId);
  console.log('Payment required:', purchase.paymentRequest.outputs[0].satoshis);

  // Buyer makes payment
  const paymentResponse: PaymentResponse = {
    payment: '...beef hex...',
    paymentId: 'txid...'
  };

  await marketplace.completePurchase(purchase.purchaseId, paymentResponse);

  // After delivery confirmed, release escrow
  const escrowKey = PrivateKey.fromWif('...');
  const release = await marketplace.releaseEscrow(purchase.purchaseId, escrowKey);

  if (release.success) {
    console.log('Escrow released to seller:', release.txid);
  }
}
```

## Security Considerations

1. **Expiration Validation**: Always check expiration timestamps on payment requests. Reject expired requests to prevent replay attacks.

2. **Amount Verification**: Verify exact payment amounts match the request. Reject partial or excessive payments.

3. **SPV Proof Validation**: Always verify merkle proofs with a trusted chain tracker. Invalid proofs indicate fraudulent payments.

4. **Merchant Data Integrity**: Don't trust merchantData from payment responses. Always validate against original request data stored server-side.

5. **Double-Spend Monitoring**: Monitor for double-spend attempts on payment transactions. Wait for confirmations for high-value transactions.

6. **HTTPS Only**: Always use HTTPS for paymentUrl endpoints to prevent man-in-the-middle attacks.

## Performance Considerations

1. **Request Caching**: Cache payment requests server-side to avoid regenerating identical requests.

2. **BEEF Size**: For large transaction chains, BEEF envelopes can be substantial. Use compression for network transmission.

3. **Batch Processing**: Process multiple payments in batches when possible to improve throughput.

4. **Async Verification**: Perform SPV verification asynchronously to avoid blocking payment submission.

5. **Database Indexing**: Index orders by paymentId (txid) for fast payment lookup.

6. **Expiration Cleanup**: Periodically clean up expired payment requests to free resources.

## Related Components

- [Transaction](../transaction/README.md) - Create payment transactions
- [BEEF](../beef/README.md) - Transaction envelope format
- [P2PKH](../p2pkh/README.md) - Standard payment script
- [SPV](../spv/README.md) - Payment verification
- [ARC](../arc/README.md) - Transaction broadcasting

## Code Examples

See complete working examples in:
- [Payment Processing](../../code-features/payment-processing/README.md)
- [E-commerce Integration](../../code-features/ecommerce-integration/README.md)
- [Content Paywall](../../code-features/content-paywall/README.md)
- [Marketplace](../../code-features/marketplace/README.md)

## Best Practices

1. **Always set expiration timestamps** for payment requests to limit replay window
2. **Verify all payment details** against original request stored server-side
3. **Use BEEF format** for payment responses to include SPV proofs
4. **Implement proper error handling** for expired, invalid, or underpaid requests
5. **Store merchantData as JSON** for easy parsing and extensibility
6. **Generate unique order IDs** to prevent collisions and enable tracking
7. **Validate network field** matches your expected network (mainnet vs testnet)
8. **Monitor payment confirmations** for high-value transactions
9. **Provide clear descriptions** for each output to inform users
10. **Implement callback URLs** for asynchronous payment notifications

## Troubleshooting

### Issue: Payment verification fails with correct payment

**Solution**: Ensure output order matches request and verify script encoding.

```typescript
// Verify each output matches
for (let i = 0; i < paymentRequest.outputs.length; i++) {
  const expected = paymentRequest.outputs[i];
  const actual = paymentTx.outputs[i];

  console.log(`Output ${i}:`);
  console.log('  Expected script:', expected.script);
  console.log('  Actual script:', actual.lockingScript?.toHex());
  console.log('  Expected amount:', expected.satoshis);
  console.log('  Actual amount:', actual.satoshis);
}
```

### Issue: Payment request expired before user could pay

**Solution**: Set reasonable expiration times and provide expiration warnings.

```typescript
// Set expiration based on payment type
const getExpirationTime = (paymentType: string): number => {
  const now = Date.now();
  switch (paymentType) {
    case 'instant': return now + (5 * 60 * 1000);     // 5 minutes
    case 'standard': return now + (30 * 60 * 1000);   // 30 minutes
    case 'subscription': return now + (24 * 3600 * 1000); // 24 hours
    default: return now + (15 * 60 * 1000);           // 15 minutes
  }
};
```

### Issue: BEEF envelope too large for HTTP request

**Solution**: Use compression or split large transaction chains.

```typescript
import * as zlib from 'zlib';

// Compress BEEF before sending
const beefHex = beef.toHex();
const beefBuffer = Buffer.from(beefHex, 'hex');
const compressed = zlib.gzipSync(beefBuffer);

// Send compressed data
const response = await fetch(paymentUrl, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/octet-stream',
    'Content-Encoding': 'gzip'
  },
  body: compressed
});
```

### Issue: Cannot parse merchantData from old requests

**Solution**: Implement versioning in merchantData format.

```typescript
interface MerchantData {
  version: number;
  orderId: string;
  // ... other fields
}

// Create with version
const merchantData = JSON.stringify({
  version: 1,
  orderId: '12345',
  customerId: 'user_abc'
});

// Parse with version check
const parseMerchantData = (dataString: string): MerchantData | null => {
  try {
    const data = JSON.parse(dataString);
    if (!data.version) {
      // Migrate old format
      return { version: 1, ...data };
    }
    return data;
  } catch (e) {
    return null;
  }
};
```

## Further Reading

- [BRC-29 Specification](https://github.com/bitcoin-sv/BRCs/blob/master/payments/0029.md) - Official BRC-29 specification
- [BRC-8: Transaction Envelopes](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0008.md) - SPV envelope format
- [BRC-62: BEEF Format](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0062.md) - BEEF specification
- [Payment Protocols](https://wiki.bitcoinsv.io/index.php/Payment_Protocols) - BSV payment protocol overview
- [SPV](https://wiki.bitcoinsv.io/index.php/Simplified_Payment_Verification) - SPV verification
- [BSV SDK Documentation](https://bsv-blockchain.github.io/ts-sdk) - Official SDK documentation

## Status

✅ Complete
