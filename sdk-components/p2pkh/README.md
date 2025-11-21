# P2PKH (Pay to Public Key Hash)

## Overview

The P2PKH component implements the most common Bitcoin transaction script template - Pay to Public Key Hash. The `P2PKH` class provides a simple interface for creating locking scripts that secure coins to a Bitcoin address and unlocking script templates that spend those coins using the corresponding private key.

P2PKH is the standard script template for basic Bitcoin payments, implementing the classic `OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG` script pattern that has been the foundation of Bitcoin transactions since its inception.

## Purpose

P2PKH in the BSV SDK solves several critical problems:

- **Standard Payments**: Implement the most widely-used Bitcoin payment script
- **Address-Based Transactions**: Send and receive coins using Bitcoin addresses
- **Signature Verification**: Cryptographically prove ownership of coins
- **Template Abstraction**: Simplify script creation without manual script construction
- **SIGHASH Support**: Flexible signature types (ALL, NONE, SINGLE, ANYONECANPAY)
- **Fee Estimation**: Accurate script length estimation for fee calculation

This component is fundamental for all standard BSV transactions, providing a secure and well-tested template for everyday Bitcoin payments.

## Basic Usage

### Create Locking Script (Receive Payment)

```typescript
import { P2PKH, PrivateKey } from '@bsv/sdk';

// Generate a key pair
const privateKey = PrivateKey.fromRandom();
const publicKey = privateKey.toPublicKey();
const address = publicKey.toAddress();

console.log('Your address:', address);

// Create P2PKH template
const p2pkh = new P2PKH();

// Create locking script for this address
const lockingScript = p2pkh.lock(address);

console.log('Locking script:', lockingScript.toHex());
// Output: OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
```

### Create Unlocking Script (Spend Payment)

```typescript
import { P2PKH, PrivateKey, Transaction } from '@bsv/sdk';

// Your private key
const privateKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');

// Create P2PKH template
const p2pkh = new P2PKH();

// Create unlocking script template
const unlockingScriptTemplate = p2pkh.unlock(privateKey);

// Use in transaction input
const tx = new Transaction();
tx.addInput({
  sourceTransaction: Transaction.fromHex('...'),
  sourceOutputIndex: 0,
  unlockingScriptTemplate  // Will be signed when tx.sign() is called
});
```

### Complete Payment Transaction

```typescript
import { Transaction, P2PKH, PrivateKey, ARC } from '@bsv/sdk';

async function sendPayment(
  sourceTransactionHex: string,
  sourceOutputIndex: number,
  recipientAddress: string,
  amount: number,
  privateKey: PrivateKey
): Promise<Transaction> {
  // Create transaction
  const tx = new Transaction();
  const p2pkh = new P2PKH();

  // Add input (spending from your address)
  tx.addInput({
    sourceTransaction: Transaction.fromHex(sourceTransactionHex),
    sourceOutputIndex,
    unlockingScriptTemplate: p2pkh.unlock(privateKey)
  });

  // Add payment output
  tx.addOutput({
    lockingScript: p2pkh.lock(recipientAddress),
    satoshis: amount
  });

  // Add change output (never reuse addresses!)
  tx.addOutput({
    lockingScript: p2pkh.lock(privateKey.toPublicKey().toAddress()),
    change: true
  });

  // Calculate fee and sign
  await tx.fee();
  await tx.sign();

  console.log('Transaction ID:', tx.id('hex'));
  return tx;
}

// Usage
const privKey = PrivateKey.fromWif('L5EY...');
const paymentTx = await sendPayment(
  '0100000001...',                        // Source transaction
  0,                                      // Output index
  '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa', // Recipient
  5000,                                   // Amount in satoshis
  privKey
);

// Broadcast
const arc = new ARC('https://api.taal.com/arc', { apiKey: 'mainnet_xxx' });
await paymentTx.broadcast(arc);
```

### SIGHASH Flags

```typescript
import { P2PKH, PrivateKey } from '@bsv/sdk';

const privateKey = PrivateKey.fromRandom();
const p2pkh = new P2PKH();

// Sign all outputs (default - most secure)
const unlockAll = p2pkh.unlock(
  privateKey,
  'all',    // signOutputs: 'all'
  false     // anyoneCanPay: false
);

// Sign no outputs (allows output modification)
const unlockNone = p2pkh.unlock(
  privateKey,
  'none',   // signOutputs: 'none'
  false
);

// Sign only corresponding output
const unlockSingle = p2pkh.unlock(
  privateKey,
  'single', // signOutputs: 'single'
  false
);

// Anyone can add inputs
const unlockAnyoneCanPay = p2pkh.unlock(
  privateKey,
  'all',
  true      // anyoneCanPay: true
);
```

## Key Features

### 1. Standard Script Template

P2PKH implements the classic Bitcoin script pattern:

```typescript
import { P2PKH, PublicKey, Script, OP } from '@bsv/sdk';

const p2pkh = new P2PKH();
const address = '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa';

// Create locking script
const lockingScript = p2pkh.lock(address);

// The locking script structure is:
// OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG

// Inspect the script
const scriptChunks = lockingScript.chunks;
console.log('Script operations:');
console.log('1. OP_DUP:', scriptChunks[0].op === OP.OP_DUP);
console.log('2. OP_HASH160:', scriptChunks[1].op === OP.OP_HASH160);
console.log('3. Public Key Hash:', scriptChunks[2].data);
console.log('4. OP_EQUALVERIFY:', scriptChunks[3].op === OP.OP_EQUALVERIFY);
console.log('5. OP_CHECKSIG:', scriptChunks[4].op === OP.OP_CHECKSIG);

// Get script as hex
console.log('Locking script hex:', lockingScript.toHex());

// Get script as assembly (human-readable)
console.log('Script assembly:', lockingScript.toASM());
// Output: OP_DUP OP_HASH160 62e907b15cbf27d5425399ebf6f0fb50ebb88f18 OP_EQUALVERIFY OP_CHECKSIG
```

### 2. Flexible SIGHASH Types

P2PKH supports all standard SIGHASH flags for different signing scenarios:

```typescript
import { Transaction, P2PKH, PrivateKey } from '@bsv/sdk';

/**
 * SIGHASH_ALL (default): Signs all inputs and outputs
 * Most common and secure option
 */
function createStandardPayment(privateKey: PrivateKey): Transaction {
  const tx = new Transaction();
  const p2pkh = new P2PKH();

  tx.addInput({
    sourceTransaction: Transaction.fromHex('...'),
    sourceOutputIndex: 0,
    unlockingScriptTemplate: p2pkh.unlock(
      privateKey,
      'all',    // Signs all outputs
      false     // Not anyoneCanPay
    )
  });

  // Outputs are locked in - cannot be modified after signing
  tx.addOutput({
    lockingScript: p2pkh.lock('1A1zP1...'),
    satoshis: 5000
  });

  return tx;
}

/**
 * SIGHASH_NONE: Signs inputs but no outputs
 * Allows outputs to be modified after signing
 */
function createBlankCheck(privateKey: PrivateKey): Transaction {
  const tx = new Transaction();
  const p2pkh = new P2PKH();

  tx.addInput({
    sourceTransaction: Transaction.fromHex('...'),
    sourceOutputIndex: 0,
    unlockingScriptTemplate: p2pkh.unlock(
      privateKey,
      'none',   // Don't sign outputs
      false
    )
  });

  // Outputs can be modified by anyone
  // Useful for donation or payment forwarding scenarios

  return tx;
}

/**
 * SIGHASH_SINGLE: Signs inputs and corresponding output
 * Only locks in the output at same index as input
 */
function createLinkedPayment(privateKey: PrivateKey): Transaction {
  const tx = new Transaction();
  const p2pkh = new P2PKH();

  tx.addInput({
    sourceTransaction: Transaction.fromHex('...'),
    sourceOutputIndex: 0,
    unlockingScriptTemplate: p2pkh.unlock(
      privateKey,
      'single', // Only sign corresponding output
      false
    )
  });

  // Output at index 0 is locked in
  tx.addOutput({
    lockingScript: p2pkh.lock('1A1zP1...'),
    satoshis: 5000
  });

  // Other outputs can be added/modified
  // Useful for partial transaction templates

  return tx;
}

/**
 * SIGHASH_ANYONECANPAY: Allows additional inputs
 * Can be combined with ALL, NONE, or SINGLE
 */
function createCrowdfundingPayment(privateKey: PrivateKey): Transaction {
  const tx = new Transaction();
  const p2pkh = new P2PKH();

  tx.addInput({
    sourceTransaction: Transaction.fromHex('...'),
    sourceOutputIndex: 0,
    unlockingScriptTemplate: p2pkh.unlock(
      privateKey,
      'all',
      true      // Anyone can add inputs
    )
  });

  // Fixed output (crowdfunding target)
  tx.addOutput({
    lockingScript: p2pkh.lock('1ProjectAddr...'),
    satoshis: 100000
  });

  // Others can add their inputs to contribute
  // Useful for crowdfunding or multi-party payments

  return tx;
}
```

### 3. Address and Public Key Support

P2PKH works with both Bitcoin addresses and public keys:

```typescript
import { P2PKH, PrivateKey, PublicKey } from '@bsv/sdk';

const privateKey = PrivateKey.fromRandom();
const publicKey = privateKey.toPublicKey();
const address = publicKey.toAddress();

const p2pkh = new P2PKH();

// Lock using address (most common)
const lockByAddress = p2pkh.lock(address);
console.log('Lock by address:', lockByAddress.toHex());

// Lock using public key directly
const lockByPubKey = p2pkh.lock(publicKey.toString());
console.log('Lock by public key:', lockByPubKey.toHex());

// Both produce the same locking script
console.log('Scripts match:', lockByAddress.toHex() === lockByPubKey.toHex());

// Addresses are base58check encoded public key hashes
console.log('Address:', address);                    // 1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa
console.log('Public key hash:', publicKey.toHash()); // Binary hash

// Convert between formats
const addressFromPubKey = publicKey.toAddress();
console.log('Derived address:', addressFromPubKey);
```

### 4. Fee Estimation with Unlocking Script Length

P2PKH provides accurate fee estimation by calculating unlocking script size:

```typescript
import { Transaction, P2PKH, PrivateKey, SatoshisPerKilobyte } from '@bsv/sdk';

async function createTransactionWithAccurateFee(
  privateKey: PrivateKey,
  recipientAddress: string,
  amount: number
): Promise<Transaction> {
  const tx = new Transaction();
  const p2pkh = new P2PKH();

  // Add input with unlocking template
  const unlockingTemplate = p2pkh.unlock(privateKey);

  tx.addInput({
    sourceTransaction: Transaction.fromHex('...'),
    sourceOutputIndex: 0,
    unlockingScriptTemplate: unlockingTemplate
  });

  // Add outputs
  tx.addOutput({
    lockingScript: p2pkh.lock(recipientAddress),
    satoshis: amount
  });

  tx.addOutput({
    lockingScript: p2pkh.lock(privateKey.toPublicKey().toAddress()),
    change: true
  });

  // The fee calculation uses estimateLength() from the unlocking template
  // P2PKH unlocking script: <signature> <publicKey>
  // Estimated length: ~107 bytes (71-73 byte signature + 33-65 byte pubkey)
  const estimatedLength = await unlockingTemplate.estimateLength(tx, 0);
  console.log('Estimated unlocking script length:', estimatedLength, 'bytes');

  // Calculate fee (uses unlocking script length estimate)
  const feeModel = new SatoshisPerKilobyte(50); // 50 sats/KB
  await tx.fee(feeModel);

  console.log('Transaction size:', tx.toBinary().length, 'bytes');
  console.log('Fee:', tx.getFee(), 'satoshis');

  // Sign (creates actual unlocking scripts)
  await tx.sign();

  // Verify actual size matches estimate
  const actualSize = tx.toBinary().length;
  console.log('Actual transaction size:', actualSize, 'bytes');

  return tx;
}
```

## API Reference

### P2PKH Class

```typescript
class P2PKH implements ScriptTemplate {
  // Create locking script for an address or public key
  lock(address: string): LockingScript;

  // Create unlocking script template for spending
  unlock(
    privateKey: PrivateKey,
    signOutputs?: 'all' | 'none' | 'single',
    anyoneCanPay?: boolean,
    sourceSatoshis?: number,
    lockingScript?: LockingScript
  ): {
    sign: (tx: Transaction, inputIndex: number) => Promise<UnlockingScript>;
    estimateLength: (tx: Transaction, inputIndex: number) => Promise<number>;
  };
}
```

### lock() Method

Creates a locking script that secures coins to a Bitcoin address.

**Parameters:**
- `address: string` - Bitcoin address (base58check encoded) or public key string

**Returns:**
- `LockingScript` - Script that locks coins to the address

**Example:**
```typescript
const p2pkh = new P2PKH();
const lockingScript = p2pkh.lock('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa');
```

### unlock() Method

Creates an unlocking script template that spends coins locked to an address.

**Parameters:**
- `privateKey: PrivateKey` - Private key corresponding to the locking address
- `signOutputs?: 'all' | 'none' | 'single'` - SIGHASH type for outputs (default: 'all')
- `anyoneCanPay?: boolean` - Allow additional inputs (default: false)
- `sourceSatoshis?: number` - Satoshis in the output being spent (optional)
- `lockingScript?: LockingScript` - Locking script being unlocked (optional)

**Returns:**
- Object with `sign()` and `estimateLength()` methods

**Example:**
```typescript
const p2pkh = new P2PKH();
const privateKey = PrivateKey.fromWif('L5EY...');

const unlockingTemplate = p2pkh.unlock(
  privateKey,
  'all',    // Sign all outputs
  false     // Not anyoneCanPay
);
```

### ScriptTemplate Interface

P2PKH implements the `ScriptTemplate` interface:

```typescript
interface ScriptTemplate {
  lock(...args: any[]): LockingScript;
  unlock(...args: any[]): UnlockingScriptTemplate;
}

interface UnlockingScriptTemplate {
  sign: (tx: Transaction, inputIndex: number) => Promise<UnlockingScript>;
  estimateLength: (tx: Transaction, inputIndex: number) => Promise<number>;
}
```

## Common Patterns

### 1. Simple Payment Workflow

```typescript
import { Transaction, P2PKH, PrivateKey, ARC } from '@bsv/sdk';

/**
 * Complete payment workflow with P2PKH
 */
class PaymentProcessor {
  private privateKey: PrivateKey;
  private p2pkh: P2PKH;
  private arc: ARC;

  constructor(privateKey: PrivateKey, arcApiKey: string) {
    this.privateKey = privateKey;
    this.p2pkh = new P2PKH();
    this.arc = new ARC('https://api.taal.com/arc', { apiKey: arcApiKey });
  }

  /**
   * Get your receiving address
   */
  getAddress(): string {
    return this.privateKey.toPublicKey().toAddress();
  }

  /**
   * Create a payment transaction
   */
  async createPayment(
    utxos: Array<{ txHex: string; outputIndex: number; satoshis: number }>,
    recipient: string,
    amount: number
  ): Promise<Transaction> {
    const tx = new Transaction();

    // Add all UTXOs as inputs
    let totalInput = 0;
    for (const utxo of utxos) {
      tx.addInput({
        sourceTransaction: Transaction.fromHex(utxo.txHex),
        sourceOutputIndex: utxo.outputIndex,
        unlockingScriptTemplate: this.p2pkh.unlock(this.privateKey)
      });
      totalInput += utxo.satoshis;
    }

    console.log(`Total input: ${totalInput} satoshis`);

    // Add payment output
    tx.addOutput({
      lockingScript: this.p2pkh.lock(recipient),
      satoshis: amount
    });

    // Add change output
    tx.addOutput({
      lockingScript: this.p2pkh.lock(this.getAddress()),
      change: true
    });

    // Calculate fee and sign
    await tx.fee();
    await tx.sign();

    console.log(`Payment created: ${tx.id('hex')}`);
    console.log(`Fee: ${tx.getFee()} satoshis`);

    return tx;
  }

  /**
   * Broadcast payment transaction
   */
  async sendPayment(
    utxos: Array<{ txHex: string; outputIndex: number; satoshis: number }>,
    recipient: string,
    amount: number
  ): Promise<string> {
    const tx = await this.createPayment(utxos, recipient, amount);

    // Broadcast to network
    await tx.broadcast(this.arc);

    console.log(`Payment sent to ${recipient}: ${amount} satoshis`);
    console.log(`Transaction ID: ${tx.id('hex')}`);

    return tx.id('hex');
  }

  /**
   * Create batch payment (multiple recipients)
   */
  async createBatchPayment(
    utxos: Array<{ txHex: string; outputIndex: number; satoshis: number }>,
    payments: Array<{ address: string; amount: number }>
  ): Promise<Transaction> {
    const tx = new Transaction();

    // Add inputs
    for (const utxo of utxos) {
      tx.addInput({
        sourceTransaction: Transaction.fromHex(utxo.txHex),
        sourceOutputIndex: utxo.outputIndex,
        unlockingScriptTemplate: this.p2pkh.unlock(this.privateKey)
      });
    }

    // Add payment outputs
    for (const payment of payments) {
      tx.addOutput({
        lockingScript: this.p2pkh.lock(payment.address),
        satoshis: payment.amount
      });
    }

    // Add change output
    tx.addOutput({
      lockingScript: this.p2pkh.lock(this.getAddress()),
      change: true
    });

    await tx.fee();
    await tx.sign();

    console.log(`Batch payment created: ${payments.length} recipients`);
    return tx;
  }
}

// Usage
const privateKey = PrivateKey.fromWif('L5EY...');
const processor = new PaymentProcessor(privateKey, 'mainnet_xxx');

console.log('Your address:', processor.getAddress());

// Send payment
const utxos = [
  { txHex: '0100000001...', outputIndex: 0, satoshis: 10000 }
];

await processor.sendPayment(
  utxos,
  '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
  5000
);

// Batch payment
await processor.createBatchPayment(utxos, [
  { address: '1Addr1...', amount: 1000 },
  { address: '1Addr2...', amount: 2000 },
  { address: '1Addr3...', amount: 3000 }
]);
```

### 2. Multi-Signature Coordination with SIGHASH

```typescript
import { Transaction, P2PKH, PrivateKey } from '@bsv/sdk';

/**
 * Coordinated multi-party payment using SIGHASH_ANYONECANPAY
 */
class CrowdfundingTransaction {
  private targetAddress: string;
  private targetAmount: number;
  private contributions: Map<string, number>;
  private tx: Transaction;
  private p2pkh: P2PKH;

  constructor(targetAddress: string, targetAmount: number) {
    this.targetAddress = targetAddress;
    this.targetAmount = targetAmount;
    this.contributions = new Map();
    this.tx = new Transaction();
    this.p2pkh = new P2PKH();

    // Create target output
    this.tx.addOutput({
      lockingScript: this.p2pkh.lock(targetAddress),
      satoshis: targetAmount
    });
  }

  /**
   * Add contribution from a participant
   */
  async addContribution(
    privateKey: PrivateKey,
    utxoTxHex: string,
    utxoIndex: number,
    amount: number
  ): Promise<void> {
    const contributorAddress = privateKey.toPublicKey().toAddress();

    // Add input with ANYONECANPAY flag
    this.tx.addInput({
      sourceTransaction: Transaction.fromHex(utxoTxHex),
      sourceOutputIndex: utxoIndex,
      unlockingScriptTemplate: this.p2pkh.unlock(
        privateKey,
        'all',
        true    // ANYONECANPAY - allows others to add inputs
      )
    });

    this.contributions.set(contributorAddress, amount);
    console.log(`Contribution added from ${contributorAddress}: ${amount} sats`);
  }

  /**
   * Get total contributions
   */
  getTotalContributions(): number {
    return Array.from(this.contributions.values()).reduce((a, b) => a + b, 0);
  }

  /**
   * Check if funding goal is met
   */
  isFunded(): boolean {
    return this.getTotalContributions() >= this.targetAmount;
  }

  /**
   * Finalize and broadcast crowdfunding transaction
   */
  async finalize(arc: ARC): Promise<string> {
    if (!this.isFunded()) {
      throw new Error(
        `Insufficient funds: ${this.getTotalContributions()}/${this.targetAmount}`
      );
    }

    await this.tx.fee();
    await this.tx.sign();

    await this.tx.broadcast(arc);

    console.log(`Crowdfunding complete: ${this.getTotalContributions()} sats raised`);
    return this.tx.id('hex');
  }
}

// Usage: Crowdfunding campaign
const crowdfund = new CrowdfundingTransaction(
  '1ProjectAddr...',
  100000  // Target: 100,000 satoshis
);

// Contributor 1
const contributor1 = PrivateKey.fromWif('L5EY...');
await crowdfund.addContribution(
  contributor1,
  '0100000001...',
  0,
  30000
);

// Contributor 2
const contributor2 = PrivateKey.fromWif('KxYZ...');
await crowdfund.addContribution(
  contributor2,
  '0100000002...',
  0,
  40000
);

// Contributor 3
const contributor3 = PrivateKey.fromWif('L1AB...');
await crowdfund.addContribution(
  contributor3,
  '0100000003...',
  0,
  35000
);

// Check if funded
console.log('Funded:', crowdfund.isFunded());
console.log('Total:', crowdfund.getTotalContributions());

// Finalize
const arc = new ARC('https://api.taal.com/arc', { apiKey: 'mainnet_xxx' });
const txid = await crowdfund.finalize(arc);
console.log('Transaction:', txid);
```

### 3. Payment Channel with P2PKH

```typescript
import { Transaction, P2PKH, PrivateKey } from '@bsv/sdk';

/**
 * Simple payment channel using P2PKH
 */
class PaymentChannel {
  private aliceKey: PrivateKey;
  private bobKey: PrivateKey;
  private fundingTx: Transaction;
  private fundingOutputIndex: number;
  private capacity: number;
  private aliceBalance: number;
  private bobBalance: number;
  private p2pkh: P2PKH;

  constructor(
    aliceKey: PrivateKey,
    bobKey: PrivateKey,
    fundingTxHex: string,
    fundingOutputIndex: number,
    capacity: number
  ) {
    this.aliceKey = aliceKey;
    this.bobKey = bobKey;
    this.fundingTx = Transaction.fromHex(fundingTxHex);
    this.fundingOutputIndex = fundingOutputIndex;
    this.capacity = capacity;
    this.aliceBalance = capacity; // Alice starts with full capacity
    this.bobBalance = 0;
    this.p2pkh = new P2PKH();
  }

  /**
   * Create payment from Alice to Bob
   */
  async createPayment(amount: number): Promise<Transaction> {
    if (amount > this.aliceBalance) {
      throw new Error('Insufficient balance');
    }

    // Update balances
    this.aliceBalance -= amount;
    this.bobBalance += amount;

    // Create settlement transaction
    const tx = new Transaction();

    // Input from funding transaction
    tx.addInput({
      sourceTransaction: this.fundingTx,
      sourceOutputIndex: this.fundingOutputIndex,
      unlockingScriptTemplate: this.p2pkh.unlock(this.aliceKey)
    });

    // Output to Bob
    if (this.bobBalance > 0) {
      tx.addOutput({
        lockingScript: this.p2pkh.lock(this.bobKey.toPublicKey().toAddress()),
        satoshis: this.bobBalance
      });
    }

    // Change to Alice
    if (this.aliceBalance > 0) {
      tx.addOutput({
        lockingScript: this.p2pkh.lock(this.aliceKey.toPublicKey().toAddress()),
        satoshis: this.aliceBalance
      });
    }

    await tx.fee();
    await tx.sign();

    console.log(`Payment: ${amount} sats to Bob`);
    console.log(`Alice balance: ${this.aliceBalance}`);
    console.log(`Bob balance: ${this.bobBalance}`);

    return tx;
  }

  /**
   * Get current balances
   */
  getBalances(): { alice: number; bob: number } {
    return {
      alice: this.aliceBalance,
      bob: this.bobBalance
    };
  }

  /**
   * Close channel and settle on-chain
   */
  async closeChannel(arc: ARC): Promise<string> {
    const settlementTx = await this.createPayment(0); // No new payment
    await settlementTx.broadcast(arc);

    console.log('Channel closed');
    console.log(`Final - Alice: ${this.aliceBalance}, Bob: ${this.bobBalance}`);

    return settlementTx.id('hex');
  }
}

// Usage: Payment channel
const aliceKey = PrivateKey.fromWif('L5EY...');
const bobKey = PrivateKey.fromWif('KxYZ...');

// Open channel with 100,000 satoshis
const channel = new PaymentChannel(
  aliceKey,
  bobKey,
  '0100000001...',  // Funding transaction
  0,                 // Output index
  100000             // Capacity
);

// Alice pays Bob 10,000 sats
await channel.createPayment(10000);

// Alice pays Bob another 5,000 sats
await channel.createPayment(5000);

// Check balances
console.log('Balances:', channel.getBalances());
// Output: { alice: 85000, bob: 15000 }

// Close channel
const arc = new ARC('https://api.taal.com/arc', { apiKey: 'mainnet_xxx' });
const closeTxid = await channel.closeChannel(arc);
```

## Security Considerations

### 1. Never Reuse Addresses

**BAD - Address reuse destroys privacy:**
```typescript
// DANGEROUS: Reusing the same address
const myAddress = privateKey.toPublicKey().toAddress();

// Payment 1
tx1.addOutput({
  lockingScript: p2pkh.lock(myAddress),
  satoshis: 5000
});

// Payment 2 - same address (BAD!)
tx2.addOutput({
  lockingScript: p2pkh.lock(myAddress),
  satoshis: 3000
});

// Change output - same address (BAD!)
tx3.addOutput({
  lockingScript: p2pkh.lock(myAddress),
  change: true
});
```

**GOOD - Generate new addresses:**
```typescript
// SECURE: Use BRC-42 key derivation for unique addresses
import { KeyDeriver, PrivateKey } from '@bsv/sdk';

const rootKey = PrivateKey.fromRandom();
const keyDeriver = new KeyDeriver(rootKey);

// Generate unique key for each payment
function getNewAddress(purpose: string, counter: number): string {
  const derivedKey = keyDeriver.derivePrivateKey(
    [2, 'payment-addresses'],
    `${purpose}-${counter}`,
    'self'
  );
  return derivedKey.toPublicKey().toAddress();
}

// Each payment gets a unique address
tx1.addOutput({
  lockingScript: p2pkh.lock(getNewAddress('receive', 1)),
  satoshis: 5000
});

tx2.addOutput({
  lockingScript: p2pkh.lock(getNewAddress('receive', 2)),
  satoshis: 3000
});

// Change also gets unique address
tx3.addOutput({
  lockingScript: p2pkh.lock(getNewAddress('change', 1)),
  change: true
});
```

### 2. Validate Addresses Before Use

**BAD - Using unvalidated addresses:**
```typescript
// DANGEROUS: No validation
const recipientAddress = userInput; // Could be malformed
const lockingScript = p2pkh.lock(recipientAddress);
// May throw error or create invalid script
```

**GOOD - Validate and handle errors:**
```typescript
// SECURE: Validate addresses
import { PublicKey } from '@bsv/sdk';

function validateAndLockAddress(address: string): LockingScript {
  try {
    // Attempt to parse as address
    const pubKey = PublicKey.fromString(address);
    const derivedAddress = pubKey.toAddress();

    // Verify it's a valid base58check address
    if (!derivedAddress || derivedAddress.length < 26) {
      throw new Error('Invalid address format');
    }

    return p2pkh.lock(address);
  } catch (error) {
    throw new Error(`Invalid Bitcoin address: ${error.message}`);
  }
}

// Usage with validation
try {
  const lockingScript = validateAndLockAddress(recipientAddress);
  tx.addOutput({ lockingScript, satoshis: 5000 });
} catch (error) {
  console.error('Address validation failed:', error);
}
```

### 3. Protect Private Keys

**BAD - Exposing private keys:**
```typescript
// DANGEROUS: Storing private keys in plain text
localStorage.setItem('privateKey', privateKey.toWif());

// DANGEROUS: Logging private keys
console.log('Private key:', privateKey.toWif());

// DANGEROUS: Sending private keys over network
await fetch('/api/save-key', {
  method: 'POST',
  body: JSON.stringify({ key: privateKey.toWif() })
});
```

**GOOD - Encrypt and secure private keys:**
```typescript
// SECURE: Encrypt private keys before storage
import { PrivateKey, encrypt } from '@bsv/sdk';

async function storeEncryptedKey(
  privateKey: PrivateKey,
  password: string
): Promise<void> {
  // Derive encryption key from password (use proper KDF in production)
  const passwordHash = sha256(Buffer.from(password));

  // Encrypt private key
  const plaintext = Array.from(Buffer.from(privateKey.toWif()));
  const ciphertext = encrypt(plaintext, privateKey, privateKey.toPublicKey());

  // Store encrypted key
  localStorage.setItem('encryptedKey', JSON.stringify(ciphertext));
}

// SECURE: Never log private keys
console.log('Address:', privateKey.toPublicKey().toAddress()); // Public info only

// SECURE: Use server-side key management
// Never send private keys to servers
// Use signing requests instead
async function requestSignature(tx: Transaction): Promise<Transaction> {
  // Send unsigned transaction to secure signing service
  const response = await fetch('/api/sign', {
    method: 'POST',
    body: JSON.stringify({ txHex: tx.toHex() })
  });
  const { signedTxHex } = await response.json();
  return Transaction.fromHex(signedTxHex);
}
```

### 4. Use Appropriate SIGHASH Flags

**BAD - Using SIGHASH_NONE without understanding risks:**
```typescript
// DANGEROUS: SIGHASH_NONE allows output modification
const unlocking = p2pkh.unlock(
  privateKey,
  'none',   // Anyone can change outputs!
  false
);

// After signing, outputs can be redirected
// Only use SIGHASH_NONE when you specifically need this behavior
```

**GOOD - Use SIGHASH_ALL for standard payments:**
```typescript
// SECURE: SIGHASH_ALL locks all outputs
const unlocking = p2pkh.unlock(
  privateKey,
  'all',    // Outputs are locked in
  false
);

// Outputs cannot be modified after signing
// This is the safest option for most use cases
```

## Performance Considerations

### 1. Batch Transaction Creation

```typescript
// Create multiple transactions efficiently
async function batchCreateTransactions(
  privateKey: PrivateKey,
  utxos: Array<{ txHex: string; outputIndex: number; satoshis: number }>,
  payments: Array<{ address: string; amount: number }>
): Promise<Transaction[]> {
  const p2pkh = new P2PKH();
  const transactions: Transaction[] = [];

  // Reuse unlocking template for all inputs with same key
  const unlockingTemplate = p2pkh.unlock(privateKey);

  for (const payment of payments) {
    const tx = new Transaction();

    // Add input (reuses template)
    tx.addInput({
      sourceTransaction: Transaction.fromHex(utxos[0].txHex),
      sourceOutputIndex: utxos[0].outputIndex,
      unlockingScriptTemplate: unlockingTemplate
    });

    // Add output
    tx.addOutput({
      lockingScript: p2pkh.lock(payment.address),
      satoshis: payment.amount
    });

    // Add change
    tx.addOutput({
      lockingScript: p2pkh.lock(privateKey.toPublicKey().toAddress()),
      change: true
    });

    await tx.fee();
    await tx.sign();

    transactions.push(tx);
  }

  return transactions;
}
```

### 2. Optimize Script Creation

```typescript
// Cache locking scripts for frequently used addresses
class OptimizedPaymentProcessor {
  private lockingScriptCache: Map<string, LockingScript>;
  private p2pkh: P2PKH;

  constructor() {
    this.lockingScriptCache = new Map();
    this.p2pkh = new P2PKH();
  }

  getLockingScript(address: string): LockingScript {
    // Check cache first
    if (this.lockingScriptCache.has(address)) {
      return this.lockingScriptCache.get(address)!;
    }

    // Create and cache
    const lockingScript = this.p2pkh.lock(address);
    this.lockingScriptCache.set(address, lockingScript);

    return lockingScript;
  }

  clearCache(): void {
    this.lockingScriptCache.clear();
  }
}
```

### 3. Parallel Transaction Signing

```typescript
// Sign multiple transactions in parallel
async function signTransactionsInParallel(
  transactions: Transaction[]
): Promise<Transaction[]> {
  return await Promise.all(
    transactions.map(async (tx) => {
      await tx.fee();
      await tx.sign();
      return tx;
    })
  );
}
```

## Related Components

- **[Transaction](../transaction/README.md)** - Use P2PKH templates in transactions
- **[Script](../script/README.md)** - Underlying script implementation
- **[Private Keys](../private-keys/README.md)** - Keys used for P2PKH unlocking
- **[Public Keys](../public-keys/README.md)** - Keys used for P2PKH addresses
- **[Signatures](../signatures/README.md)** - ECDSA signatures in unlocking scripts
- **[Script Templates](../script-templates/README.md)** - Other script template types
- **[HD Wallets](../hd-wallets/README.md)** - Generate addresses with key derivation

## Best Practices

### 1. Always Use Change Outputs

```typescript
// GOOD: Always include change output
tx.addOutput({
  lockingScript: p2pkh.lock(recipientAddress),
  satoshis: amount
});

tx.addOutput({
  lockingScript: p2pkh.lock(getNewChangeAddress()),
  change: true
});
```

### 2. Use BRC-42 for Address Generation

```typescript
// GOOD: Derive unique addresses using BRC-42
const keyDeriver = new KeyDeriver(rootKey);
const paymentKey = keyDeriver.derivePrivateKey(
  [2, 'payments'],
  `invoice-${invoiceNumber}`,
  'self'
);
const address = paymentKey.toPublicKey().toAddress();
```

### 3. Validate Transaction Before Broadcasting

```typescript
// GOOD: Validate transaction structure
async function validateTransaction(tx: Transaction): Promise<boolean> {
  // Check inputs
  if (tx.inputs.length === 0) {
    throw new Error('No inputs');
  }

  // Check outputs
  if (tx.outputs.length === 0) {
    throw new Error('No outputs');
  }

  // Verify signatures
  const isValid = await tx.verify();
  if (!isValid) {
    throw new Error('Invalid signatures');
  }

  return true;
}

await validateTransaction(tx);
await tx.broadcast(arc);
```

### 4. Handle Fee Calculation Errors

```typescript
// GOOD: Handle fee calculation failures
try {
  await tx.fee(feeModel);
  await tx.sign();
} catch (error) {
  if (error.message.includes('Insufficient funds')) {
    console.error('Not enough satoshis to cover fee');
    // Handle insufficient funds
  } else {
    console.error('Fee calculation failed:', error);
  }
}
```

### 5. Use Appropriate SIGHASH for Use Case

```typescript
// Standard payment: SIGHASH_ALL
const standardPayment = p2pkh.unlock(privateKey, 'all', false);

// Crowdfunding: SIGHASH_ALL | ANYONECANPAY
const crowdfundInput = p2pkh.unlock(privateKey, 'all', true);

// Donation box: SIGHASH_NONE | ANYONECANPAY
const donationInput = p2pkh.unlock(privateKey, 'none', true);

// Linked payments: SIGHASH_SINGLE
const linkedPayment = p2pkh.unlock(privateKey, 'single', false);
```

## Troubleshooting

### Invalid Signature Errors

**Problem:** Transaction fails verification with invalid signature error.

**Solution:**
```typescript
// Check that private key matches locking script
const privateKey = PrivateKey.fromWif('L5EY...');
const address = privateKey.toPublicKey().toAddress();
console.log('Your address:', address);

// Verify source output is locked to your address
const sourceOutput = sourceTransaction.outputs[outputIndex];
const expectedLocking = p2pkh.lock(address);
console.log('Locking scripts match:',
  sourceOutput.lockingScript.toHex() === expectedLocking.toHex()
);
```

### Insufficient Funds for Fee

**Problem:** Cannot create transaction due to insufficient funds.

**Solution:**
```typescript
// Calculate required amount including fee
const estimatedFee = Math.ceil((tx.toBinary().length + 150) * feeRate / 1000);
const required = amount + estimatedFee;

console.log(`Amount: ${amount}`);
console.log(`Estimated fee: ${estimatedFee}`);
console.log(`Total required: ${required}`);
console.log(`Available: ${totalInputSatoshis}`);

if (totalInputSatoshis < required) {
  throw new Error(`Need ${required - totalInputSatoshis} more satoshis`);
}
```

### Address Format Errors

**Problem:** Invalid address format when creating locking script.

**Solution:**
```typescript
// Validate address format
function isValidAddress(address: string): boolean {
  try {
    // Bitcoin addresses are 25-34 characters, base58check encoded
    if (address.length < 25 || address.length > 34) {
      return false;
    }

    // Should start with 1 (mainnet) or m/n (testnet)
    if (!address.match(/^[1mn]/)) {
      return false;
    }

    // Try to create locking script
    p2pkh.lock(address);
    return true;
  } catch {
    return false;
  }
}

if (!isValidAddress(recipientAddress)) {
  throw new Error('Invalid Bitcoin address format');
}
```

## Further Reading

- **[BRC-29: Simple Payment Protocol](https://github.com/bitcoin-sv/BRCs/blob/master/payments/0029.md)** - P2PKH-based payment protocol
- **[Bitcoin Script](https://wiki.bitcoinsv.io/index.php/Script)** - Understanding Bitcoin script
- **[SIGHASH Flags](https://wiki.bitcoinsv.io/index.php/SIGHASH_flags)** - Signature hash types
- **[BRC-42: Key Derivation](https://github.com/bitcoin-sv/BRCs/blob/master/key-derivation/0042.md)** - Derive unique addresses
- **[BSV SDK Documentation](https://docs.bsvblockchain.org/)** - Official SDK documentation

## Status

✅ **Complete** - Comprehensive documentation with P2PKH template, SIGHASH types, and payment patterns.
