# UTXO Management

## Overview

The UTXO (Unspent Transaction Output) Management component provides essential tools for tracking, selecting, and managing unspent outputs in BSV blockchain applications. While the BSV SDK doesn't include a built-in UTXO manager, it provides all the primitives needed to implement robust UTXO tracking systems.

This documentation demonstrates how to build efficient UTXO management using the SDK's `Transaction`, `UnspentOutput`, and wallet interfaces, including coin selection strategies, balance calculation, and UTXO lifecycle management.

## Purpose

UTXO Management solves several critical challenges in BSV application development:

- **UTXO Tracking**: Monitor all unspent outputs across multiple addresses and protocols
- **Coin Selection**: Choose optimal UTXOs for transaction creation based on various strategies
- **Balance Calculation**: Calculate spendable balances accurately across different contexts
- **UTXO Consolidation**: Merge small UTXOs to reduce transaction size and fees
- **State Management**: Track UTXO states (unspent, pending, spent) throughout lifecycle
- **Multi-Protocol Support**: Manage UTXOs separately for different application protocols

Proper UTXO management is essential for building efficient, scalable BSV applications that minimize fees and maximize transaction throughput.

## Basic Usage

### Define UTXO Structure

```typescript
import { Transaction } from '@bsv/sdk';

// UTXO representation
interface UTXO {
  txid: string;                    // Transaction ID
  outputIndex: number;             // Output index (vout)
  satoshis: number;                // Amount in satoshis
  lockingScript: string;           // Locking script hex
  address?: string;                // Associated address (if P2PKH)
  scriptType?: string;             // Script template type
  sequenceNumber?: number;         // Sequence number for RBF
}

// Example UTXO
const utxo: UTXO = {
  txid: '7f3b4c9a1e2d5f8a6b3c9d2e4f1a8b5c6d7e9f0a1b2c3d4e5f6a7b8c9d0e1f2',
  outputIndex: 0,
  satoshis: 10000,
  lockingScript: '76a914...',
  address: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
  scriptType: 'P2PKH'
};
```

### Basic UTXO Tracker

```typescript
import { Transaction, P2PKH, PrivateKey } from '@bsv/sdk';

class BasicUTXOTracker {
  private utxos: Map<string, UTXO> = new Map();

  // Add UTXO to tracking
  addUTXO(utxo: UTXO): void {
    const key = `${utxo.txid}:${utxo.outputIndex}`;
    this.utxos.set(key, utxo);
  }

  // Remove spent UTXO
  removeUTXO(txid: string, outputIndex: number): void {
    const key = `${txid}:${outputIndex}`;
    this.utxos.delete(key);
  }

  // Get all UTXOs
  getAllUTXOs(): UTXO[] {
    return Array.from(this.utxos.values());
  }

  // Calculate total balance
  getBalance(): number {
    return this.getAllUTXOs().reduce((sum, utxo) => sum + utxo.satoshis, 0);
  }

  // Get UTXOs by address
  getUTXOsByAddress(address: string): UTXO[] {
    return this.getAllUTXOs().filter(utxo => utxo.address === address);
  }
}

// Usage
const tracker = new BasicUTXOTracker();

// Add UTXOs
tracker.addUTXO({
  txid: 'abc123...',
  outputIndex: 0,
  satoshis: 50000,
  lockingScript: '76a914...',
  address: '1A1zP1eP...'
});

// Get balance
const balance = tracker.getBalance();
console.log(`Total balance: ${balance} satoshis`);
```

### Simple Coin Selection

```typescript
import { Transaction, P2PKH, PrivateKey } from '@bsv/sdk';

class SimpleCoinSelector {
  private utxos: UTXO[];

  constructor(utxos: UTXO[]) {
    this.utxos = utxos;
  }

  // Select UTXOs for amount (first-fit strategy)
  selectCoins(targetAmount: number): UTXO[] {
    const selected: UTXO[] = [];
    let total = 0;

    // Sort by size descending (use largest first)
    const sorted = [...this.utxos].sort((a, b) => b.satoshis - a.satoshis);

    for (const utxo of sorted) {
      selected.push(utxo);
      total += utxo.satoshis;

      if (total >= targetAmount) {
        break;
      }
    }

    if (total < targetAmount) {
      throw new Error(`Insufficient funds: need ${targetAmount}, have ${total}`);
    }

    return selected;
  }
}

// Usage
const selector = new SimpleCoinSelector([
  { txid: 'tx1', outputIndex: 0, satoshis: 10000, lockingScript: '...' },
  { txid: 'tx2', outputIndex: 0, satoshis: 25000, lockingScript: '...' },
  { txid: 'tx3', outputIndex: 1, satoshis: 5000, lockingScript: '...' }
]);

const selectedUTXOs = selector.selectCoins(30000);
console.log(`Selected ${selectedUTXOs.length} UTXOs`);
```

### Create Transaction from UTXOs

```typescript
import { Transaction, P2PKH, PrivateKey } from '@bsv/sdk';

async function createTransactionFromUTXOs(
  utxos: UTXO[],
  privateKey: PrivateKey,
  recipientAddress: string,
  amount: number,
  changeAddress: string
): Promise<Transaction> {
  const tx = new Transaction();

  // Add inputs from selected UTXOs
  for (const utxo of utxos) {
    // Create source transaction (simplified - in production, fetch from network)
    const sourceTx = Transaction.fromHex(await fetchTransactionHex(utxo.txid));

    tx.addInput({
      sourceTransaction: sourceTx,
      sourceOutputIndex: utxo.outputIndex,
      unlockingScriptTemplate: new P2PKH().unlock(privateKey)
    });
  }

  // Add payment output
  tx.addOutput({
    lockingScript: new P2PKH().lock(recipientAddress),
    satoshis: amount
  });

  // Add change output
  tx.addOutput({
    lockingScript: new P2PKH().lock(changeAddress),
    change: true
  });

  // Calculate fee and sign
  await tx.fee();
  await tx.sign();

  return tx;
}

async function fetchTransactionHex(txid: string): Promise<string> {
  // Fetch from API (WhatsOnChain, etc.)
  return '...transaction hex...';
}
```

## Key Features

### 1. UTXO State Management

Track UTXO lifecycle states from creation to spending:

```typescript
enum UTXOState {
  UNCONFIRMED = 'unconfirmed',  // In mempool
  CONFIRMED = 'confirmed',       // In block
  PENDING = 'pending',           // Being spent (tx in mempool)
  SPENT = 'spent',               // Spent and confirmed
  INVALID = 'invalid'            // Invalid/double-spent
}

interface ManagedUTXO extends UTXO {
  state: UTXOState;
  confirmations: number;
  createdAt: Date;
  spentAt?: Date;
  spentInTx?: string;
}

class UTXOStateManager {
  private utxos: Map<string, ManagedUTXO> = new Map();

  addUTXO(utxo: UTXO, confirmations: number = 0): void {
    const key = `${utxo.txid}:${utxo.outputIndex}`;

    const managed: ManagedUTXO = {
      ...utxo,
      state: confirmations > 0 ? UTXOState.CONFIRMED : UTXOState.UNCONFIRMED,
      confirmations,
      createdAt: new Date()
    };

    this.utxos.set(key, managed);
  }

  markAsPending(txid: string, outputIndex: number): void {
    const key = `${txid}:${outputIndex}`;
    const utxo = this.utxos.get(key);

    if (utxo) {
      utxo.state = UTXOState.PENDING;
    }
  }

  markAsSpent(txid: string, outputIndex: number, spentInTx: string): void {
    const key = `${txid}:${outputIndex}`;
    const utxo = this.utxos.get(key);

    if (utxo) {
      utxo.state = UTXOState.SPENT;
      utxo.spentAt = new Date();
      utxo.spentInTx = spentInTx;
    }
  }

  updateConfirmations(txid: string, outputIndex: number, confirmations: number): void {
    const key = `${txid}:${outputIndex}`;
    const utxo = this.utxos.get(key);

    if (utxo) {
      utxo.confirmations = confirmations;

      if (confirmations > 0 && utxo.state === UTXOState.UNCONFIRMED) {
        utxo.state = UTXOState.CONFIRMED;
      }
    }
  }

  // Get spendable UTXOs (confirmed and not pending/spent)
  getSpendableUTXOs(minConfirmations: number = 1): ManagedUTXO[] {
    return Array.from(this.utxos.values()).filter(utxo =>
      utxo.confirmations >= minConfirmations &&
      (utxo.state === UTXOState.CONFIRMED || utxo.state === UTXOState.UNCONFIRMED)
    );
  }

  // Get pending UTXOs
  getPendingUTXOs(): ManagedUTXO[] {
    return Array.from(this.utxos.values()).filter(
      utxo => utxo.state === UTXOState.PENDING
    );
  }

  // Calculate spendable balance
  getSpendableBalance(minConfirmations: number = 1): number {
    return this.getSpendableUTXOs(minConfirmations).reduce(
      (sum, utxo) => sum + utxo.satoshis,
      0
    );
  }
}

// Usage
const stateManager = new UTXOStateManager();

// Add confirmed UTXO
stateManager.addUTXO({
  txid: 'abc123',
  outputIndex: 0,
  satoshis: 100000,
  lockingScript: '76a914...'
}, 6); // 6 confirmations

// Mark as pending when creating transaction
stateManager.markAsPending('abc123', 0);

// After transaction confirms
stateManager.markAsSpent('abc123', 0, 'def456');

// Get spendable balance
const balance = stateManager.getSpendableBalance(1);
console.log(`Spendable: ${balance} satoshis`);
```

### 2. Advanced Coin Selection Strategies

Implement multiple coin selection algorithms:

```typescript
import { Transaction } from '@bsv/sdk';

interface CoinSelectionStrategy {
  select(utxos: UTXO[], targetAmount: number, feeRate: number): UTXO[];
}

// Strategy 1: Largest First (minimize number of inputs)
class LargestFirstStrategy implements CoinSelectionStrategy {
  select(utxos: UTXO[], targetAmount: number, feeRate: number): UTXO[] {
    const sorted = [...utxos].sort((a, b) => b.satoshis - a.satoshis);
    return this.selectFromSorted(sorted, targetAmount, feeRate);
  }

  private selectFromSorted(sorted: UTXO[], target: number, feeRate: number): UTXO[] {
    const selected: UTXO[] = [];
    let total = 0;

    for (const utxo of sorted) {
      selected.push(utxo);
      total += utxo.satoshis;

      // Estimate fee for current input count
      const estimatedFee = this.estimateFee(selected.length, 2, feeRate);

      if (total >= target + estimatedFee) {
        return selected;
      }
    }

    throw new Error('Insufficient funds');
  }

  private estimateFee(inputs: number, outputs: number, feeRate: number): number {
    // Rough estimate: 148 bytes per input, 34 bytes per output, 10 bytes overhead
    const estimatedSize = (inputs * 148) + (outputs * 34) + 10;
    return Math.ceil(estimatedSize * feeRate);
  }
}

// Strategy 2: Smallest First (maximize UTXO consolidation)
class SmallestFirstStrategy implements CoinSelectionStrategy {
  select(utxos: UTXO[], targetAmount: number, feeRate: number): UTXO[] {
    const sorted = [...utxos].sort((a, b) => a.satoshis - b.satoshis);
    const selected: UTXO[] = [];
    let total = 0;

    for (const utxo of sorted) {
      selected.push(utxo);
      total += utxo.satoshis;

      const estimatedFee = this.estimateFee(selected.length, 2, feeRate);

      if (total >= targetAmount + estimatedFee) {
        return selected;
      }
    }

    throw new Error('Insufficient funds');
  }

  private estimateFee(inputs: number, outputs: number, feeRate: number): number {
    const estimatedSize = (inputs * 148) + (outputs * 34) + 10;
    return Math.ceil(estimatedSize * feeRate);
  }
}

// Strategy 3: Branch and Bound (optimal selection)
class BranchAndBoundStrategy implements CoinSelectionStrategy {
  select(utxos: UTXO[], targetAmount: number, feeRate: number): UTXO[] {
    // Sort descending by value
    const sorted = [...utxos].sort((a, b) => b.satoshis - a.satoshis);

    // Try to find exact match or minimal waste
    const result = this.branchAndBound(sorted, targetAmount, feeRate);

    if (result) {
      return result;
    }

    // Fallback to largest first if no optimal solution
    return new LargestFirstStrategy().select(utxos, targetAmount, feeRate);
  }

  private branchAndBound(
    utxos: UTXO[],
    target: number,
    feeRate: number,
    depth: number = 0,
    selected: UTXO[] = [],
    currentTotal: number = 0
  ): UTXO[] | null {
    // Limit search depth to prevent timeout
    if (depth > 20) return null;

    const estimatedFee = this.estimateFee(selected.length, 2, feeRate);
    const needed = target + estimatedFee;

    // Found solution
    if (currentTotal >= needed && currentTotal - needed < 1000) {
      return selected;
    }

    // Exceeded target too much
    if (currentTotal > needed + 10000) {
      return null;
    }

    // No more UTXOs to try
    if (depth >= utxos.length) {
      return null;
    }

    // Try including current UTXO
    const withCurrent = this.branchAndBound(
      utxos,
      target,
      feeRate,
      depth + 1,
      [...selected, utxos[depth]],
      currentTotal + utxos[depth].satoshis
    );

    if (withCurrent) return withCurrent;

    // Try excluding current UTXO
    return this.branchAndBound(
      utxos,
      target,
      feeRate,
      depth + 1,
      selected,
      currentTotal
    );
  }

  private estimateFee(inputs: number, outputs: number, feeRate: number): number {
    const estimatedSize = (inputs * 148) + (outputs * 34) + 10;
    return Math.ceil(estimatedSize * feeRate);
  }
}

// Strategy selector
class CoinSelector {
  private strategies: Map<string, CoinSelectionStrategy> = new Map();

  constructor() {
    this.strategies.set('largest-first', new LargestFirstStrategy());
    this.strategies.set('smallest-first', new SmallestFirstStrategy());
    this.strategies.set('branch-and-bound', new BranchAndBoundStrategy());
  }

  select(
    strategyName: string,
    utxos: UTXO[],
    targetAmount: number,
    feeRate: number = 0.5
  ): UTXO[] {
    const strategy = this.strategies.get(strategyName);

    if (!strategy) {
      throw new Error(`Unknown strategy: ${strategyName}`);
    }

    return strategy.select(utxos, targetAmount, feeRate);
  }
}

// Usage
const selector = new CoinSelector();

const utxos: UTXO[] = [
  { txid: 'tx1', outputIndex: 0, satoshis: 5000, lockingScript: '...' },
  { txid: 'tx2', outputIndex: 0, satoshis: 10000, lockingScript: '...' },
  { txid: 'tx3', outputIndex: 0, satoshis: 25000, lockingScript: '...' },
  { txid: 'tx4', outputIndex: 0, satoshis: 50000, lockingScript: '...' }
];

// Use largest-first for minimal inputs
const selected1 = selector.select('largest-first', utxos, 30000, 0.5);
console.log(`Largest-first selected ${selected1.length} UTXOs`);

// Use smallest-first for UTXO consolidation
const selected2 = selector.select('smallest-first', utxos, 30000, 0.5);
console.log(`Smallest-first selected ${selected2.length} UTXOs`);

// Use branch-and-bound for optimal selection
const selected3 = selector.select('branch-and-bound', utxos, 30000, 0.5);
console.log(`Branch-and-bound selected ${selected3.length} UTXOs`);
```

### 3. UTXO Consolidation

Merge small UTXOs to reduce fragmentation:

```typescript
import { Transaction, P2PKH, PrivateKey } from '@bsv/sdk';

class UTXOConsolidator {
  private privateKey: PrivateKey;
  private feeRate: number;

  constructor(privateKey: PrivateKey, feeRate: number = 0.5) {
    this.privateKey = privateKey;
    this.feeRate = feeRate;
  }

  // Determine if consolidation is beneficial
  shouldConsolidate(utxos: UTXO[], threshold: number = 10000): boolean {
    const smallUTXOs = utxos.filter(u => u.satoshis < threshold);

    // Consolidate if we have many small UTXOs
    return smallUTXOs.length >= 10;
  }

  // Create consolidation transaction
  async consolidateUTXOs(
    utxos: UTXO[],
    destinationAddress: string,
    maxInputs: number = 100
  ): Promise<Transaction> {
    // Sort by size ascending (consolidate smallest first)
    const sorted = [...utxos].sort((a, b) => a.satoshis - b.satoshis);

    // Take up to maxInputs
    const toConsolidate = sorted.slice(0, maxInputs);

    const tx = new Transaction();

    // Add all inputs
    for (const utxo of toConsolidate) {
      const sourceTx = Transaction.fromHex(await this.fetchTxHex(utxo.txid));

      tx.addInput({
        sourceTransaction: sourceTx,
        sourceOutputIndex: utxo.outputIndex,
        unlockingScriptTemplate: new P2PKH().unlock(this.privateKey)
      });
    }

    // Single output to consolidate into
    const totalInput = toConsolidate.reduce((sum, u) => sum + u.satoshis, 0);
    const estimatedFee = this.estimateConsolidationFee(toConsolidate.length);

    tx.addOutput({
      lockingScript: new P2PKH().lock(destinationAddress),
      satoshis: totalInput - estimatedFee
    });

    await tx.fee();
    await tx.sign();

    return tx;
  }

  // Estimate fee for consolidation
  private estimateConsolidationFee(inputCount: number): number {
    // Size: inputs * 148 bytes + 1 output * 34 bytes + 10 bytes overhead
    const estimatedSize = (inputCount * 148) + 34 + 10;
    return Math.ceil(estimatedSize * this.feeRate);
  }

  // Calculate consolidation savings
  calculateSavings(utxos: UTXO[]): {
    currentCost: number;
    afterConsolidation: number;
    savings: number;
  } {
    // Cost to spend all UTXOs individually in future transactions
    const currentCost = utxos.length * 148 * this.feeRate;

    // Cost to spend 1 UTXO after consolidation
    const afterConsolidation = 148 * this.feeRate;

    // Cost to consolidate now
    const consolidationCost = this.estimateConsolidationFee(utxos.length);

    const savings = currentCost - afterConsolidation - consolidationCost;

    return {
      currentCost: Math.ceil(currentCost),
      afterConsolidation: Math.ceil(afterConsolidation),
      savings: Math.ceil(savings)
    };
  }

  private async fetchTxHex(txid: string): Promise<string> {
    // Fetch from API
    return '...';
  }
}

// Usage
const consolidator = new UTXOConsolidator(PrivateKey.fromWif('...'), 0.5);

const fragmentedUTXOs: UTXO[] = [
  { txid: 'tx1', outputIndex: 0, satoshis: 1000, lockingScript: '...' },
  { txid: 'tx2', outputIndex: 0, satoshis: 1500, lockingScript: '...' },
  { txid: 'tx3', outputIndex: 0, satoshis: 2000, lockingScript: '...' },
  // ... 20 more small UTXOs
];

// Check if consolidation is beneficial
if (consolidator.shouldConsolidate(fragmentedUTXOs, 5000)) {
  const savings = consolidator.calculateSavings(fragmentedUTXOs);
  console.log(`Consolidation savings: ${savings.savings} satoshis`);

  // Perform consolidation
  const consolidationTx = await consolidator.consolidateUTXOs(
    fragmentedUTXOs,
    '1A1zP1eP...',
    50 // Max 50 inputs per transaction
  );

  console.log(`Consolidated ${fragmentedUTXOs.length} UTXOs`);
}
```

### 4. Multi-Address UTXO Tracking

Track UTXOs across multiple addresses and key derivation paths:

```typescript
import { KeyDeriver, PrivateKey, PublicKey } from '@bsv/sdk';

class MultiAddressUTXOTracker {
  private keyDeriver: KeyDeriver;
  private utxosByAddress: Map<string, UTXO[]> = new Map();
  private addressToKeyPath: Map<string, {
    protocolID: [number, string];
    keyID: string;
    counterparty: PublicKey | 'self' | 'anyone';
  }> = new Map();

  constructor(masterKey: PrivateKey) {
    this.keyDeriver = new KeyDeriver(masterKey);
  }

  // Register address with key derivation path
  registerAddress(
    address: string,
    protocolID: [number, string],
    keyID: string,
    counterparty: PublicKey | 'self' | 'anyone' = 'anyone'
  ): void {
    this.addressToKeyPath.set(address, { protocolID, keyID, counterparty });
    this.utxosByAddress.set(address, []);
  }

  // Add UTXO to specific address
  addUTXO(address: string, utxo: UTXO): void {
    const utxos = this.utxosByAddress.get(address) || [];
    utxos.push({ ...utxo, address });
    this.utxosByAddress.set(address, utxos);
  }

  // Get all UTXOs across all addresses
  getAllUTXOs(): UTXO[] {
    const allUTXOs: UTXO[] = [];

    for (const utxos of this.utxosByAddress.values()) {
      allUTXOs.push(...utxos);
    }

    return allUTXOs;
  }

  // Get UTXOs for specific address
  getUTXOsForAddress(address: string): UTXO[] {
    return this.utxosByAddress.get(address) || [];
  }

  // Get private key for address
  getPrivateKeyForAddress(address: string): PrivateKey {
    const keyPath = this.addressToKeyPath.get(address);

    if (!keyPath) {
      throw new Error(`No key path registered for address: ${address}`);
    }

    return this.keyDeriver.derivePrivateKey(
      keyPath.protocolID,
      keyPath.keyID,
      keyPath.counterparty
    );
  }

  // Calculate balance by address
  getBalanceByAddress(): Map<string, number> {
    const balances = new Map<string, number>();

    for (const [address, utxos] of this.utxosByAddress.entries()) {
      const balance = utxos.reduce((sum, utxo) => sum + utxo.satoshis, 0);
      balances.set(address, balance);
    }

    return balances;
  }

  // Get total balance across all addresses
  getTotalBalance(): number {
    let total = 0;

    for (const utxos of this.utxosByAddress.values()) {
      total += utxos.reduce((sum, utxo) => sum + utxo.satoshis, 0);
    }

    return total;
  }

  // Select UTXOs across multiple addresses
  selectUTXOsAcrossAddresses(targetAmount: number): {
    utxos: UTXO[];
    addresses: Set<string>;
  } {
    const allUTXOs = this.getAllUTXOs();
    const sorted = allUTXOs.sort((a, b) => b.satoshis - a.satoshis);

    const selected: UTXO[] = [];
    const addresses = new Set<string>();
    let total = 0;

    for (const utxo of sorted) {
      selected.push(utxo);
      if (utxo.address) addresses.add(utxo.address);
      total += utxo.satoshis;

      if (total >= targetAmount) {
        break;
      }
    }

    if (total < targetAmount) {
      throw new Error('Insufficient funds across all addresses');
    }

    return { utxos: selected, addresses };
  }
}

// Usage
const multiTracker = new MultiAddressUTXOTracker(PrivateKey.fromRandom());

const protocolID: [number, string] = [2, 'payment-v1'];

// Register multiple addresses
multiTracker.registerAddress('1Address1...', protocolID, 'key-0', 'anyone');
multiTracker.registerAddress('1Address2...', protocolID, 'key-1', 'anyone');
multiTracker.registerAddress('1Address3...', protocolID, 'change-0', 'self');

// Add UTXOs to different addresses
multiTracker.addUTXO('1Address1...', {
  txid: 'tx1',
  outputIndex: 0,
  satoshis: 50000,
  lockingScript: '...'
});

multiTracker.addUTXO('1Address2...', {
  txid: 'tx2',
  outputIndex: 0,
  satoshis: 75000,
  lockingScript: '...'
});

// Get balance per address
const balances = multiTracker.getBalanceByAddress();
for (const [address, balance] of balances.entries()) {
  console.log(`${address}: ${balance} satoshis`);
}

// Select UTXOs across addresses
const { utxos, addresses } = multiTracker.selectUTXOsAcrossAddresses(100000);
console.log(`Selected ${utxos.length} UTXOs from ${addresses.size} addresses`);
```

## API Reference

### UTXO Interface

```typescript
interface UTXO {
  txid: string;                    // Transaction ID
  outputIndex: number;             // Output index (vout)
  satoshis: number;                // Amount in satoshis
  lockingScript: string;           // Locking script hex
  address?: string;                // Associated address
  scriptType?: string;             // Script template type
  sequenceNumber?: number;         // Sequence number
}
```

### ManagedUTXO Interface

```typescript
interface ManagedUTXO extends UTXO {
  state: UTXOState;               // Current state
  confirmations: number;          // Confirmation count
  createdAt: Date;                // Creation timestamp
  spentAt?: Date;                 // Spent timestamp
  spentInTx?: string;             // Spending transaction ID
}
```

### UTXOState Enum

```typescript
enum UTXOState {
  UNCONFIRMED = 'unconfirmed',   // In mempool
  CONFIRMED = 'confirmed',        // In block
  PENDING = 'pending',            // Being spent
  SPENT = 'spent',                // Spent and confirmed
  INVALID = 'invalid'             // Invalid/double-spent
}
```

### CoinSelectionStrategy Interface

```typescript
interface CoinSelectionStrategy {
  select(
    utxos: UTXO[],
    targetAmount: number,
    feeRate: number
  ): UTXO[];
}
```

## Common Patterns

### Pattern 1: UTXO Discovery and Syncing

Discover and sync UTXOs from blockchain:

```typescript
import { Transaction, P2PKH } from '@bsv/sdk';

class UTXOSynchronizer {
  private tracker: BasicUTXOTracker;
  private addresses: string[];

  constructor(addresses: string[]) {
    this.tracker = new BasicUTXOTracker();
    this.addresses = addresses;
  }

  // Sync UTXOs from blockchain API
  async syncUTXOs(): Promise<void> {
    for (const address of this.addresses) {
      const utxos = await this.fetchUTXOsFromAPI(address);

      for (const utxo of utxos) {
        this.tracker.addUTXO(utxo);
      }
    }
  }

  // Fetch UTXOs from WhatsOnChain API
  private async fetchUTXOsFromAPI(address: string): Promise<UTXO[]> {
    try {
      const response = await fetch(
        `https://api.whatsonchain.com/v1/bsv/main/address/${address}/unspent`
      );

      const data = await response.json();

      return data.map((item: any) => ({
        txid: item.tx_hash,
        outputIndex: item.tx_pos,
        satoshis: item.value,
        lockingScript: item.script || '',
        address: address
      }));
    } catch (error) {
      console.error(`Error fetching UTXOs for ${address}:`, error);
      return [];
    }
  }

  // Get synced UTXOs
  getUTXOs(): UTXO[] {
    return this.tracker.getAllUTXOs();
  }

  // Get balance
  getBalance(): number {
    return this.tracker.getBalance();
  }
}

// Usage
const synchronizer = new UTXOSynchronizer([
  '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',
  '1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2'
]);

await synchronizer.syncUTXOs();
const balance = synchronizer.getBalance();
console.log(`Synced balance: ${balance} satoshis`);
```

### Pattern 2: Optimistic UTXO Updates

Update UTXO set optimistically before confirmation:

```typescript
class OptimisticUTXOManager {
  private stateManager: UTXOStateManager;
  private pendingTransactions: Map<string, Transaction> = new Map();

  constructor() {
    this.stateManager = new UTXOStateManager();
  }

  // Create and broadcast transaction with optimistic update
  async createAndBroadcastTransaction(
    utxos: UTXO[],
    outputs: Array<{ address: string; satoshis: number }>,
    changeAddress: string,
    privateKey: PrivateKey
  ): Promise<Transaction> {
    const tx = new Transaction();

    // Add inputs
    for (const utxo of utxos) {
      // Mark as pending immediately
      this.stateManager.markAsPending(utxo.txid, utxo.outputIndex);

      const sourceTx = Transaction.fromHex(await this.fetchTxHex(utxo.txid));

      tx.addInput({
        sourceTransaction: sourceTx,
        sourceOutputIndex: utxo.outputIndex,
        unlockingScriptTemplate: new P2PKH().unlock(privateKey)
      });
    }

    // Add outputs
    for (const output of outputs) {
      tx.addOutput({
        lockingScript: new P2PKH().lock(output.address),
        satoshis: output.satoshis
      });
    }

    // Add change
    tx.addOutput({
      lockingScript: new P2PKH().lock(changeAddress),
      change: true
    });

    await tx.fee();
    await tx.sign();

    // Broadcast
    const txid = await this.broadcastTransaction(tx);

    // Store pending transaction
    this.pendingTransactions.set(txid, tx);

    // Optimistically add change UTXO
    const changeOutput = tx.outputs.find(o => o.change);
    if (changeOutput) {
      this.stateManager.addUTXO({
        txid: txid,
        outputIndex: tx.outputs.indexOf(changeOutput),
        satoshis: changeOutput.satoshis || 0,
        lockingScript: changeOutput.lockingScript.toHex(),
        address: changeAddress
      }, 0); // 0 confirmations
    }

    return tx;
  }

  // Handle transaction confirmation
  async handleConfirmation(txid: string, blockHeight: number): Promise<void> {
    const tx = this.pendingTransactions.get(txid);

    if (!tx) return;

    // Mark spent inputs as spent
    for (const input of tx.inputs) {
      const sourceTxid = input.sourceTransaction?.id('hex');
      const sourceIndex = input.sourceOutputIndex;

      if (sourceTxid !== undefined && sourceIndex !== undefined) {
        this.stateManager.markAsSpent(sourceTxid, sourceIndex, txid);
      }
    }

    // Update change UTXO confirmations
    const changeOutput = tx.outputs.find(o => o.change);
    if (changeOutput) {
      this.stateManager.updateConfirmations(
        txid,
        tx.outputs.indexOf(changeOutput),
        1
      );
    }

    this.pendingTransactions.delete(txid);
  }

  // Rollback on failure
  async handleTransactionFailure(txid: string): Promise<void> {
    const tx = this.pendingTransactions.get(txid);

    if (!tx) return;

    // Restore inputs to spendable state
    for (const input of tx.inputs) {
      const sourceTxid = input.sourceTransaction?.id('hex');
      const sourceIndex = input.sourceOutputIndex;

      if (sourceTxid !== undefined && sourceIndex !== undefined) {
        // Reset to confirmed state
        this.stateManager.updateConfirmations(sourceTxid, sourceIndex, 1);
      }
    }

    this.pendingTransactions.delete(txid);
  }

  private async fetchTxHex(txid: string): Promise<string> {
    return '...';
  }

  private async broadcastTransaction(tx: Transaction): Promise<string> {
    // Broadcast and return txid
    return tx.id('hex');
  }
}
```

### Pattern 3: UTXO-Based Balance Calculation

Calculate balances with different confirmation requirements:

```typescript
class BalanceCalculator {
  private stateManager: UTXOStateManager;

  constructor(stateManager: UTXOStateManager) {
    this.stateManager = stateManager;
  }

  // Get confirmed balance (6+ confirmations)
  getConfirmedBalance(): number {
    return this.stateManager.getSpendableBalance(6);
  }

  // Get unconfirmed balance (0+ confirmations)
  getUnconfirmedBalance(): number {
    return this.stateManager.getSpendableBalance(0);
  }

  // Get pending balance (being spent)
  getPendingBalance(): number {
    return this.stateManager.getPendingUTXOs().reduce(
      (sum, utxo) => sum + utxo.satoshis,
      0
    );
  }

  // Get total balance (all states)
  getTotalBalance(): number {
    return this.getUnconfirmedBalance() + this.getPendingBalance();
  }

  // Get balance breakdown
  getBalanceBreakdown(): {
    confirmed: number;
    unconfirmed: number;
    pending: number;
    total: number;
  } {
    return {
      confirmed: this.getConfirmedBalance(),
      unconfirmed: this.getUnconfirmedBalance(),
      pending: this.getPendingBalance(),
      total: this.getTotalBalance()
    };
  }

  // Get spendable balance with custom confirmation requirement
  getSpendableBalance(minConfirmations: number = 1): number {
    return this.stateManager.getSpendableBalance(minConfirmations);
  }
}

// Usage
const calculator = new BalanceCalculator(stateManager);

const breakdown = calculator.getBalanceBreakdown();
console.log('Balance breakdown:', {
  confirmed: `${breakdown.confirmed} sats`,
  unconfirmed: `${breakdown.unconfirmed} sats`,
  pending: `${breakdown.pending} sats`,
  total: `${breakdown.total} sats`
});
```

## Security Considerations

### Double-Spend Protection

Prevent double-spending by tracking UTXO states:

```typescript
class DoubleSpendProtection {
  private usedUTXOs: Set<string> = new Set();

  // Check if UTXO is already being used
  isUTXOAvailable(txid: string, outputIndex: number): boolean {
    const key = `${txid}:${outputIndex}`;
    return !this.usedUTXOs.has(key);
  }

  // Mark UTXO as being used
  markUTXOAsUsed(txid: string, outputIndex: number): void {
    const key = `${txid}:${outputIndex}`;

    if (this.usedUTXOs.has(key)) {
      throw new Error(`UTXO ${key} is already being spent`);
    }

    this.usedUTXOs.add(key);
  }

  // Release UTXO if transaction fails
  releaseUTXO(txid: string, outputIndex: number): void {
    const key = `${txid}:${outputIndex}`;
    this.usedUTXOs.delete(key);
  }

  // Select UTXOs safely
  selectSafeUTXOs(utxos: UTXO[], targetAmount: number): UTXO[] {
    const available = utxos.filter(utxo =>
      this.isUTXOAvailable(utxo.txid, utxo.outputIndex)
    );

    const sorted = available.sort((a, b) => b.satoshis - a.satoshis);
    const selected: UTXO[] = [];
    let total = 0;

    for (const utxo of sorted) {
      try {
        this.markUTXOAsUsed(utxo.txid, utxo.outputIndex);
        selected.push(utxo);
        total += utxo.satoshis;

        if (total >= targetAmount) {
          return selected;
        }
      } catch (error) {
        // UTXO already being used, skip
        continue;
      }
    }

    // Rollback if insufficient funds
    for (const utxo of selected) {
      this.releaseUTXO(utxo.txid, utxo.outputIndex);
    }

    throw new Error('Insufficient available funds');
  }
}
```

## Performance Considerations

### UTXO Set Pruning

Keep UTXO set size manageable:

```typescript
class UTXOSetPruner {
  private static readonly MAX_SPENT_AGE_DAYS = 30;
  private static readonly MAX_UTXO_COUNT = 10000;

  // Prune old spent UTXOs
  pruneSpentUTXOs(utxos: Map<string, ManagedUTXO>): void {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - UTXOSetPruner.MAX_SPENT_AGE_DAYS);

    for (const [key, utxo] of utxos.entries()) {
      if (
        utxo.state === UTXOState.SPENT &&
        utxo.spentAt &&
        utxo.spentAt < cutoffDate
      ) {
        utxos.delete(key);
      }
    }
  }

  // Limit UTXO set size
  limitUTXOSetSize(utxos: Map<string, ManagedUTXO>): void {
    if (utxos.size <= UTXOSetPruner.MAX_UTXO_COUNT) {
      return;
    }

    // Convert to array and sort by creation date
    const sorted = Array.from(utxos.entries()).sort(
      ([, a], [, b]) => a.createdAt.getTime() - b.createdAt.getTime()
    );

    // Remove oldest spent UTXOs first
    const toRemove = sorted
      .filter(([, utxo]) => utxo.state === UTXOState.SPENT)
      .slice(0, utxos.size - UTXOSetPruner.MAX_UTXO_COUNT);

    for (const [key] of toRemove) {
      utxos.delete(key);
    }
  }
}
```

## Related Components

- **[Transactions](../transactions/README.md)**: Create transactions from UTXOs
- **[HD Wallets](../hd-wallets/README.md)**: Derive addresses for UTXO tracking
- **[P2PKH](../p2pkh/README.md)**: Most common locking script for UTXOs
- **[SPV](../spv/README.md)**: Verify UTXO validity with SPV

## Best Practices

1. **Always track UTXO states** to prevent double-spending
2. **Use confirmation thresholds** appropriate for transaction value
3. **Implement UTXO consolidation** when fragmentation increases
4. **Choose coin selection strategy** based on use case
5. **Sync UTXOs regularly** from blockchain to stay current
6. **Prune old spent UTXOs** to keep memory usage reasonable
7. **Handle optimistic updates carefully** with rollback capability
8. **Validate UTXO ownership** before spending

## Troubleshooting

### Issue: Insufficient Funds Despite Balance

**Problem:** Balance shows funds but coin selection fails.

**Solution:** Check for pending UTXOs:
```typescript
const spendable = stateManager.getSpendableUTXOs(1);
const pending = stateManager.getPendingUTXOs();

console.log(`Spendable: ${spendable.length}, Pending: ${pending.length}`);
```

### Issue: UTXO Already Spent Error

**Problem:** Attempting to spend already-used UTXO.

**Solution:** Implement double-spend protection and sync UTXOs:
```typescript
const protection = new DoubleSpendProtection();
const safeUTXOs = protection.selectSafeUTXOs(utxos, targetAmount);
```

## Further Reading

- **[BRC-8](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0008.md)**: Transaction Envelopes
- **[BRC-29](https://github.com/bitcoin-sv/BRCs/blob/master/payments/0029.md)**: Simple Payment Protocol
- **[Bitcoin UTXO Model](https://wiki.bitcoinsv.io/index.php/UTXO)**: Understanding UTXO architecture

## Status

✅ Complete - Comprehensive UTXO management documentation with tracking, selection strategies, and lifecycle management.
