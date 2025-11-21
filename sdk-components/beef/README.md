# BEEF (Background Evaluation Extended Format)

## Overview

The `Beef` class in the BSV TypeScript SDK implements BRC-62, the Background Evaluation Extended Format for Bitcoin transaction packages. BEEF provides a standardized way to bundle transactions with their dependencies and merkle proofs, enabling efficient SPV validation without requiring access to the full blockchain. It creates atomic transaction envelopes that include all necessary proof data for independent verification.

## Purpose

- Bundle transactions with their input dependencies into atomic packages
- Include merkle proofs for SPV validation of transaction ancestry
- Enable offline transaction verification without blockchain access
- Reduce data transmission overhead for transaction chains
- Standardize transaction package format for interoperability
- Support both Version 1 (simple) and Version 2 (optimized) BEEF formats

## Basic Usage

### Creating a BEEF Package

```typescript
import { Beef, Transaction, MerklePath } from '@bsv/sdk';

// Create a BEEF package
const beef = new Beef();

// Add transactions to the package
const parentTx = Transaction.fromHex('0100000001...');
const childTx = Transaction.fromHex('0100000001...');

beef.addTransaction(parentTx);
beef.addTransaction(childTx);

// Add merkle proof for SPV validation
const merklePath = MerklePath.fromHex('fed7c509000a02fddd01...');
beef.addMerklePath(merklePath);

// Serialize to hex for transmission
const beefHex = beef.toHex();
console.log('BEEF package:', beefHex);

// Parse BEEF package
const parsedBeef = Beef.fromHex(beefHex);
console.log('Transactions:', parsedBeef.getTransactions());
```

### Verifying a BEEF Package

```typescript
import { Beef, ChainTracker } from '@bsv/sdk';

// Custom chain tracker for merkle root verification
class WhatsOnChainTracker implements ChainTracker {
  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    const response = await fetch(
      `https://api.whatsonchain.com/v1/bsv/main/block/${height}/header`
    );
    const data = await response.json();
    return data.merkleroot === root;
  }
}

// Verify BEEF package
const beef = Beef.fromHex(beefHex);
const chainTracker = new WhatsOnChainTracker();

const isValid = await beef.verify(chainTracker);
console.log('BEEF package valid:', isValid);
```

## Key Features

### 1. Transaction Bundle Creation

BEEF packages bundle transactions with their dependencies for atomic transmission:

```typescript
import { Beef, Transaction, PrivateKey, P2PKH } from '@bsv/sdk';

// Create a chain of transactions
const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');

// Parent transaction (already confirmed)
const parentTx = Transaction.fromHex('0100000001...');

// Create child transaction spending parent
const childTx = new Transaction();
childTx.addInput({
  sourceTransaction: parentTx,
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(privKey)
});

childTx.addOutput({
  lockingScript: new P2PKH().lock('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa'),
  satoshis: 1000
});

await childTx.fee();
await childTx.sign();

// Create BEEF package with both transactions
const beef = new Beef();

// Add transactions in dependency order
beef.addTransaction(parentTx);
beef.addTransaction(childTx);

// BEEF automatically handles:
// - Transaction ordering
// - Dependency tracking
// - Binary serialization
console.log('BEEF size:', beef.toBinary().length, 'bytes');
console.log('Transactions included:', beef.getTransactions().length);
```

### 2. Merkle Proof Integration

BEEF includes merkle proofs for SPV validation of transaction ancestry:

```typescript
import { Beef, Transaction, MerklePath } from '@bsv/sdk';

// Create BEEF with merkle proof
const beef = new Beef();

// Add confirmed transaction
const confirmedTx = Transaction.fromHex('0100000001...');
beef.addTransaction(confirmedTx);

// Add merkle proof for the confirmed transaction
const merklePath = MerklePath.fromHex('fed7c509000a02fddd01...');
beef.addMerklePath(merklePath);

// The merkle path contains:
// - Block height
// - Transaction index in block
// - Merkle tree path to root
// - All sibling hashes needed for verification

// Get merkle root from path
const merkleRoot = merklePath.computeRoot(confirmedTx.id('hex'));
console.log('Merkle root:', merkleRoot);
console.log('Block height:', merklePath.blockHeight);

// Add unconfirmed transaction spending confirmed UTXO
const unconfirmedTx = new Transaction();
unconfirmedTx.addInput({
  sourceTransaction: confirmedTx,
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(privKey)
});
unconfirmedTx.addOutput({
  lockingScript: new P2PKH().lock(recipientAddress),
  satoshis: 800
});

await unconfirmedTx.fee();
await unconfirmedTx.sign();

beef.addTransaction(unconfirmedTx);

// BEEF now contains full proof chain:
// - Confirmed parent with merkle proof
// - Unconfirmed child with parent reference
console.log('Complete BEEF package ready for SPV validation');
```

### 3. Version 1 and Version 2 Formats

BEEF supports two formats for different optimization levels:

```typescript
import { Beef, Transaction } from '@bsv/sdk';

// Version 1 BEEF (simple format)
const beefV1 = new Beef();
beefV1.version = 1;

// Version 1 format:
// - Simple transaction list
// - Straightforward merkle proof attachment
// - Easier to parse and validate
// - Slightly larger size

beefV1.addTransaction(tx1);
beefV1.addTransaction(tx2);

const v1Binary = beefV1.toBinary();
console.log('Version 1 size:', v1Binary.length);

// Version 2 BEEF (optimized format)
const beefV2 = new Beef();
beefV2.version = 2;

// Version 2 format:
// - Optimized transaction encoding
// - Deduplication of common data
// - Compressed merkle proof storage
// - Smaller size for large packages

beefV2.addTransaction(tx1);
beefV2.addTransaction(tx2);

const v2Binary = beefV2.toBinary();
console.log('Version 2 size:', v2Binary.length);
console.log('Size reduction:', ((1 - v2Binary.length / v1Binary.length) * 100).toFixed(2) + '%');

// Both versions are fully compatible
const parsedV1 = Beef.fromBinary(v1Binary);
const parsedV2 = Beef.fromBinary(v2Binary);

console.log('V1 valid:', await parsedV1.verify(chainTracker));
console.log('V2 valid:', await parsedV2.verify(chainTracker));
```

### 4. Atomic Transaction Validation

BEEF enables atomic validation of entire transaction chains:

```typescript
import { Beef, Transaction, ChainTracker } from '@bsv/sdk';

class BeefValidator {
  /**
   * Validate entire BEEF package atomically
   */
  static async validateBeef(
    beef: Beef,
    chainTracker: ChainTracker
  ): Promise<{ valid: boolean; errors: string[] }> {
    const errors: string[] = [];

    try {
      // 1. Verify merkle proofs
      const isValid = await beef.verify(chainTracker);
      if (!isValid) {
        errors.push('Merkle proof verification failed');
        return { valid: false, errors };
      }

      // 2. Verify transaction chain integrity
      const transactions = beef.getTransactions();
      const txMap = new Map<string, Transaction>();

      for (const tx of transactions) {
        txMap.set(tx.id('hex'), tx);
      }

      // 3. Verify each transaction spends valid inputs
      for (const tx of transactions) {
        for (let i = 0; i < tx.inputs.length; i++) {
          const input = tx.inputs[i];

          // Check if input references transaction in package
          if (input.sourceTXID) {
            const sourceTx = txMap.get(input.sourceTXID);
            if (!sourceTx) {
              errors.push(
                `Transaction ${tx.id('hex')} input ${i} ` +
                `references missing transaction ${input.sourceTXID}`
              );
            } else {
              // Verify output exists
              if (!sourceTx.outputs[input.sourceOutputIndex]) {
                errors.push(
                  `Transaction ${tx.id('hex')} input ${i} ` +
                  `references non-existent output ${input.sourceOutputIndex}`
                );
              }
            }
          }
        }
      }

      // 4. Verify transaction scripts
      for (const tx of transactions) {
        try {
          // Verify all inputs have valid unlocking scripts
          for (let i = 0; i < tx.inputs.length; i++) {
            if (!tx.inputs[i].unlockingScript) {
              errors.push(
                `Transaction ${tx.id('hex')} input ${i} missing unlocking script`
              );
            }
          }
        } catch (e) {
          errors.push(`Transaction ${tx.id('hex')} validation failed: ${e.message}`);
        }
      }

      return {
        valid: errors.length === 0,
        errors
      };

    } catch (e) {
      errors.push(`BEEF validation error: ${e.message}`);
      return { valid: false, errors };
    }
  }
}

// Usage
const beef = Beef.fromHex(beefHex);
const chainTracker = new WhatsOnChainTracker();

const result = await BeefValidator.validateBeef(beef, chainTracker);

if (result.valid) {
  console.log('BEEF package is valid and complete');
} else {
  console.error('BEEF validation errors:');
  result.errors.forEach(err => console.error('  -', err));
}
```

## API Reference

### Constructor

```typescript
constructor(version?: number)
```

Creates a new BEEF package.

**Parameters:**
- `version?: number` - BEEF format version (1 or 2, default: 2)

### Static Methods

#### `Beef.fromBinary(binary: number[]): Beef`

Parses a BEEF package from binary format.

**Parameters:**
- `binary: number[]` - The binary BEEF data

**Returns:** `Beef` - The parsed BEEF package

**Example:**
```typescript
const beef = Beef.fromBinary(binaryData);
```

#### `Beef.fromHex(hex: string): Beef`

Parses a BEEF package from hexadecimal string.

**Parameters:**
- `hex: string` - The hexadecimal BEEF string

**Returns:** `Beef` - The parsed BEEF package

**Example:**
```typescript
const beef = Beef.fromHex('0100beef01...');
```

### Instance Methods

#### `addTransaction(tx: Transaction): void`

Adds a transaction to the BEEF package.

**Parameters:**
- `tx: Transaction` - The transaction to add

**Example:**
```typescript
beef.addTransaction(transaction);
```

#### `addMerklePath(path: MerklePath): void`

Adds a merkle proof to the BEEF package.

**Parameters:**
- `path: MerklePath` - The merkle path for SPV validation

**Example:**
```typescript
beef.addMerklePath(merklePath);
```

#### `getTransactions(): Transaction[]`

Returns all transactions in the BEEF package.

**Returns:** `Transaction[]` - Array of transactions

#### `verify(chainTracker: ChainTracker): Promise<boolean>`

Verifies the BEEF package merkle proofs.

**Parameters:**
- `chainTracker: ChainTracker` - Interface for verifying merkle roots

**Returns:** `Promise<boolean>` - True if all proofs are valid

#### `toBinary(): number[]`

Serializes the BEEF package to binary format.

**Returns:** `number[]` - Binary BEEF data

#### `toHex(): string`

Serializes the BEEF package to hexadecimal string.

**Returns:** `string` - Hexadecimal BEEF string

### Instance Properties

```typescript
beef.version        // number - BEEF format version (1 or 2)
beef.transactions   // Transaction[] - Transactions in the package
beef.merklePaths    // MerklePath[] - Merkle proofs for validation
```

## Common Patterns

### Pattern 1: Payment Channel Transaction Bundle

Creating BEEF packages for payment channel updates:

```typescript
import { Beef, Transaction, PrivateKey, P2PKH, MerklePath } from '@bsv/sdk';

class PaymentChannelBeef {
  /**
   * Create BEEF package for payment channel update
   */
  static async createChannelUpdate(
    fundingTx: Transaction,
    fundingMerklePath: MerklePath,
    channelState: number,
    alicePrivKey: PrivateKey,
    bobPrivKey: PrivateKey,
    aliceAmount: number,
    bobAmount: number
  ): Promise<Beef> {
    const beef = new Beef();

    // Add funding transaction with proof
    beef.addTransaction(fundingTx);
    beef.addMerklePath(fundingMerklePath);

    // Create channel update transaction
    const updateTx = new Transaction();

    // Spend from 2-of-2 multisig funding output
    updateTx.addInput({
      sourceTransaction: fundingTx,
      sourceOutputIndex: 0,
      sequence: channelState // Use sequence for state tracking
    });

    // Output to Alice
    updateTx.addOutput({
      lockingScript: new P2PKH().lock(alicePrivKey.toPublicKey().toAddress()),
      satoshis: aliceAmount
    });

    // Output to Bob
    updateTx.addOutput({
      lockingScript: new P2PKH().lock(bobPrivKey.toPublicKey().toAddress()),
      satoshis: bobAmount
    });

    await updateTx.fee();

    // Both parties sign
    await updateTx.sign();

    // Add update transaction
    beef.addTransaction(updateTx);

    return beef;
  }

  /**
   * Verify and extract channel state
   */
  static async verifyChannelUpdate(
    beefHex: string,
    chainTracker: ChainTracker,
    expectedFundingTxid: string
  ): Promise<{ valid: boolean; state: number; aliceAmount: number; bobAmount: number }> {
    const beef = Beef.fromHex(beefHex);

    // Verify BEEF package
    const isValid = await beef.verify(chainTracker);
    if (!isValid) {
      throw new Error('Invalid BEEF package');
    }

    const transactions = beef.getTransactions();

    // Find funding and update transactions
    const fundingTx = transactions.find(tx => tx.id('hex') === expectedFundingTxid);
    if (!fundingTx) {
      throw new Error('Funding transaction not found');
    }

    const updateTx = transactions.find(tx =>
      tx.inputs.some(input => input.sourceTXID === expectedFundingTxid)
    );
    if (!updateTx) {
      throw new Error('Update transaction not found');
    }

    // Extract state and amounts
    const state = updateTx.inputs[0].sequence;
    const aliceAmount = updateTx.outputs[0].satoshis || 0;
    const bobAmount = updateTx.outputs[1].satoshis || 0;

    return {
      valid: true,
      state,
      aliceAmount,
      bobAmount
    };
  }
}

// Usage
const fundingTx = Transaction.fromHex('...');
const fundingProof = MerklePath.fromHex('...');
const aliceKey = PrivateKey.fromRandom();
const bobKey = PrivateKey.fromRandom();

// Create channel update
const beef = await PaymentChannelBeef.createChannelUpdate(
  fundingTx,
  fundingProof,
  42,      // State number
  aliceKey,
  bobKey,
  7000,    // Alice gets 7000 sats
  3000     // Bob gets 3000 sats
);

console.log('Channel update BEEF:', beef.toHex());

// Verify channel update
const chainTracker = new WhatsOnChainTracker();
const result = await PaymentChannelBeef.verifyChannelUpdate(
  beef.toHex(),
  chainTracker,
  fundingTx.id('hex')
);

console.log('Channel state:', result.state);
console.log('Alice balance:', result.aliceAmount);
console.log('Bob balance:', result.bobAmount);
```

### Pattern 2: Atomic Multi-Party Transaction

Using BEEF for atomic multi-party transactions:

```typescript
import { Beef, Transaction, PrivateKey, P2PKH } from '@bsv/sdk';

class AtomicSwapBeef {
  /**
   * Create atomic swap BEEF package
   */
  static async createAtomicSwap(
    aliceUtxo: { tx: Transaction; outputIndex: number; privKey: PrivateKey },
    bobUtxo: { tx: Transaction; outputIndex: number; privKey: PrivateKey },
    aliceReceiveAddress: string,
    bobReceiveAddress: string,
    aliceAmount: number,
    bobAmount: number
  ): Promise<Beef> {
    const beef = new Beef();

    // Add input transactions
    beef.addTransaction(aliceUtxo.tx);
    beef.addTransaction(bobUtxo.tx);

    // Create swap transaction with both inputs
    const swapTx = new Transaction();

    // Alice's input
    swapTx.addInput({
      sourceTransaction: aliceUtxo.tx,
      sourceOutputIndex: aliceUtxo.outputIndex,
      unlockingScriptTemplate: new P2PKH().unlock(aliceUtxo.privKey)
    });

    // Bob's input
    swapTx.addInput({
      sourceTransaction: bobUtxo.tx,
      sourceOutputIndex: bobUtxo.outputIndex,
      unlockingScriptTemplate: new P2PKH().unlock(bobUtxo.privKey)
    });

    // Alice receives Bob's amount
    swapTx.addOutput({
      lockingScript: new P2PKH().lock(aliceReceiveAddress),
      satoshis: bobAmount
    });

    // Bob receives Alice's amount
    swapTx.addOutput({
      lockingScript: new P2PKH().lock(bobReceiveAddress),
      satoshis: aliceAmount
    });

    // Add change outputs if needed
    swapTx.addOutput({
      lockingScript: new P2PKH().lock(aliceUtxo.privKey.toPublicKey().toAddress()),
      change: true
    });

    await swapTx.fee();
    await swapTx.sign();

    beef.addTransaction(swapTx);

    return beef;
  }

  /**
   * Verify atomic swap fairness
   */
  static async verifyAtomicSwap(
    beefHex: string,
    chainTracker: ChainTracker,
    aliceExpectedReceive: number,
    bobExpectedReceive: number,
    aliceAddress: string,
    bobAddress: string
  ): Promise<boolean> {
    const beef = Beef.fromHex(beefHex);

    // Verify BEEF
    const isValid = await beef.verify(chainTracker);
    if (!isValid) {
      return false;
    }

    const transactions = beef.getTransactions();

    // Find swap transaction (has inputs from both parties)
    const swapTx = transactions.find(tx => tx.inputs.length >= 2);
    if (!swapTx) {
      return false;
    }

    // Verify Alice receives correct amount
    const aliceOutput = swapTx.outputs.find(output => {
      const script = output.lockingScript;
      return script && script.toASM().includes(aliceAddress);
    });

    if (!aliceOutput || aliceOutput.satoshis !== aliceExpectedReceive) {
      return false;
    }

    // Verify Bob receives correct amount
    const bobOutput = swapTx.outputs.find(output => {
      const script = output.lockingScript;
      return script && script.toASM().includes(bobAddress);
    });

    if (!bobOutput || bobOutput.satoshis !== bobExpectedReceive) {
      return false;
    }

    return true;
  }
}

// Usage
const aliceUtxo = {
  tx: Transaction.fromHex('...'),
  outputIndex: 0,
  privKey: PrivateKey.fromRandom()
};

const bobUtxo = {
  tx: Transaction.fromHex('...'),
  outputIndex: 0,
  privKey: PrivateKey.fromRandom()
};

// Create atomic swap
const swapBeef = await AtomicSwapBeef.createAtomicSwap(
  aliceUtxo,
  bobUtxo,
  '1AliceAddress...',
  '1BobAddress...',
  5000,  // Alice gives 5000 sats
  8000   // Bob gives 8000 sats
);

console.log('Atomic swap BEEF:', swapBeef.toHex());

// Verify fairness
const isFair = await AtomicSwapBeef.verifyAtomicSwap(
  swapBeef.toHex(),
  chainTracker,
  8000,  // Alice should receive 8000
  5000,  // Bob should receive 5000
  '1AliceAddress...',
  '1BobAddress...'
);

console.log('Swap is fair:', isFair);
```

### Pattern 3: Transaction Chain Archival

Using BEEF for long-term transaction chain storage:

```typescript
import { Beef, Transaction, MerklePath } from '@bsv/sdk';
import * as fs from 'fs';
import * as zlib from 'zlib';

class BeefArchive {
  /**
   * Archive transaction chain with proofs
   */
  static async archiveTransactionChain(
    transactions: Transaction[],
    merklePaths: Map<string, MerklePath>,
    outputPath: string
  ): Promise<void> {
    const beef = new Beef();

    // Add all transactions in dependency order
    const sortedTxs = this.topologicalSort(transactions);

    for (const tx of sortedTxs) {
      beef.addTransaction(tx);

      // Add merkle path if available
      const txid = tx.id('hex');
      const merklePath = merklePaths.get(txid);
      if (merklePath) {
        beef.addMerklePath(merklePath);
      }
    }

    // Serialize and compress
    const beefBinary = beef.toBinary();
    const compressed = zlib.gzipSync(Buffer.from(beefBinary));

    // Write to file
    fs.writeFileSync(outputPath, compressed);

    console.log('Archived', transactions.length, 'transactions');
    console.log('Original size:', beefBinary.length, 'bytes');
    console.log('Compressed size:', compressed.length, 'bytes');
    console.log('Compression ratio:',
      ((1 - compressed.length / beefBinary.length) * 100).toFixed(2) + '%'
    );
  }

  /**
   * Restore transaction chain from archive
   */
  static async restoreTransactionChain(
    inputPath: string
  ): Promise<{ transactions: Transaction[]; merklePaths: MerklePath[] }> {
    // Read and decompress
    const compressed = fs.readFileSync(inputPath);
    const beefBinary = zlib.gunzipSync(compressed);

    // Parse BEEF
    const beef = Beef.fromBinary(Array.from(beefBinary));

    return {
      transactions: beef.getTransactions(),
      merklePaths: beef.merklePaths || []
    };
  }

  /**
   * Topological sort for dependency ordering
   */
  private static topologicalSort(transactions: Transaction[]): Transaction[] {
    const txMap = new Map<string, Transaction>();
    const graph = new Map<string, Set<string>>();
    const inDegree = new Map<string, number>();

    // Build transaction map and initialize
    for (const tx of transactions) {
      const txid = tx.id('hex');
      txMap.set(txid, tx);
      graph.set(txid, new Set());
      inDegree.set(txid, 0);
    }

    // Build dependency graph
    for (const tx of transactions) {
      const txid = tx.id('hex');
      for (const input of tx.inputs) {
        if (input.sourceTXID && txMap.has(input.sourceTXID)) {
          graph.get(input.sourceTXID)!.add(txid);
          inDegree.set(txid, (inDegree.get(txid) || 0) + 1);
        }
      }
    }

    // Kahn's algorithm
    const queue: string[] = [];
    for (const [txid, degree] of inDegree) {
      if (degree === 0) {
        queue.push(txid);
      }
    }

    const sorted: Transaction[] = [];
    while (queue.length > 0) {
      const txid = queue.shift()!;
      sorted.push(txMap.get(txid)!);

      for (const neighbor of graph.get(txid)!) {
        inDegree.set(neighbor, inDegree.get(neighbor)! - 1);
        if (inDegree.get(neighbor) === 0) {
          queue.push(neighbor);
        }
      }
    }

    return sorted;
  }
}

// Usage
const transactions = [
  Transaction.fromHex('...'),
  Transaction.fromHex('...'),
  Transaction.fromHex('...')
];

const merklePaths = new Map<string, MerklePath>([
  ['txid1', MerklePath.fromHex('...')],
  ['txid2', MerklePath.fromHex('...')]
]);

// Archive
await BeefArchive.archiveTransactionChain(
  transactions,
  merklePaths,
  'archive.beef.gz'
);

// Restore
const restored = await BeefArchive.restoreTransactionChain('archive.beef.gz');
console.log('Restored', restored.transactions.length, 'transactions');
console.log('With', restored.merklePaths.length, 'merkle proofs');
```

## Security Considerations

1. **Merkle Proof Validation**: Always verify merkle proofs with a trusted chain tracker before accepting BEEF packages. Invalid proofs indicate fraudulent or unconfirmed transactions.

2. **Transaction Dependency Verification**: Verify that all referenced input transactions are included in the BEEF package or have valid merkle proofs.

3. **Double-Spend Detection**: BEEF packages don't prevent double-spends. Always verify transactions against your UTXO set and check for conflicts.

4. **Size Limits**: Implement reasonable size limits for BEEF packages to prevent DoS attacks through oversized packages.

5. **Chain Tracker Trust**: The security of BEEF verification depends on the reliability of your chain tracker implementation. Use trusted data sources.

6. **Package Integrity**: Verify the integrity of BEEF packages during transmission using checksums or signatures to detect tampering.

## Performance Considerations

1. **Version Selection**: Use Version 2 BEEF for large transaction packages to benefit from optimizations. Version 1 is simpler for small packages.

2. **Binary Format**: Use binary serialization for storage and transmission. Hex encoding adds 100% size overhead.

3. **Compression**: BEEF packages compress well with gzip. Implement compression for long-term storage or network transmission.

4. **Incremental Parsing**: For large BEEF packages, parse transactions incrementally rather than loading the entire package into memory.

5. **Caching**: Cache merkle proof verifications to avoid redundant chain tracker queries for the same block heights.

6. **Batch Processing**: When processing multiple BEEF packages, batch chain tracker queries for better performance.

## Related Components

- [Transaction](../transaction/README.md) - Create and manage transactions
- [MerklePath](../merkle-proofs/README.md) - Work with merkle proofs
- [SPV](../spv/README.md) - Simplified Payment Verification
- [TransactionInput](../transaction-input/README.md) - Handle transaction inputs

## Code Examples

See complete working examples in:
- [SPV Validation](../../code-features/spv-validation/README.md)
- [Transaction Chains](../../code-features/transaction-chains/README.md)
- [Payment Channels](../../code-features/payment-channels/README.md)
- [Atomic Swaps](../../code-features/atomic-swaps/README.md)

## Best Practices

1. **Always include merkle proofs** for confirmed transactions to enable SPV validation
2. **Order transactions by dependency** (parents before children) for easier validation
3. **Use Version 2 BEEF** for production systems with large transaction packages
4. **Implement size limits** (e.g., 10MB) to prevent resource exhaustion
5. **Compress BEEF packages** for storage and network transmission
6. **Verify package integrity** before processing to detect tampering
7. **Cache merkle root verifications** to improve performance
8. **Use trusted chain trackers** with fallback options for reliability
9. **Implement atomic validation** - accept or reject entire packages, not partial
10. **Archive BEEF packages** for audit trails and dispute resolution

## Troubleshooting

### Issue: BEEF verification fails with valid transactions

**Solution**: Ensure chain tracker has access to correct merkle roots.

```typescript
// Implement robust chain tracker with fallbacks
class RobustChainTracker implements ChainTracker {
  private sources = [
    'https://api.whatsonchain.com/v1/bsv/main',
    'https://api.taal.com/api/v1'
  ];

  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    for (const source of this.sources) {
      try {
        const response = await fetch(`${source}/block/${height}/header`);
        const data = await response.json();
        if (data.merkleroot === root) {
          return true;
        }
      } catch (e) {
        continue; // Try next source
      }
    }
    return false;
  }
}
```

### Issue: BEEF package too large

**Solution**: Use Version 2 format and compression.

```typescript
// Create optimized BEEF
const beef = new Beef();
beef.version = 2; // Use optimized format

// Serialize and compress
const binary = beef.toBinary();
const compressed = zlib.gzipSync(Buffer.from(binary));

console.log('Size reduction:',
  ((1 - compressed.length / binary.length) * 100).toFixed(2) + '%'
);
```

### Issue: Transaction ordering errors

**Solution**: Add transactions in dependency order (parents first).

```typescript
// Sort transactions topologically
function sortTransactions(txs: Transaction[]): Transaction[] {
  const sorted: Transaction[] = [];
  const added = new Set<string>();

  function addWithDependencies(tx: Transaction) {
    const txid = tx.id('hex');
    if (added.has(txid)) return;

    // Add parent transactions first
    for (const input of tx.inputs) {
      if (input.sourceTransaction) {
        addWithDependencies(input.sourceTransaction);
      }
    }

    sorted.push(tx);
    added.add(txid);
  }

  txs.forEach(tx => addWithDependencies(tx));
  return sorted;
}
```

### Issue: Missing merkle proofs

**Solution**: Fetch proofs from blockchain services before creating BEEF.

```typescript
// Fetch merkle proof
async function getMerkleProof(txid: string): Promise<MerklePath> {
  const response = await fetch(
    `https://api.whatsonchain.com/v1/bsv/main/tx/${txid}/proof`
  );
  const proofData = await response.json();
  return MerklePath.fromHex(proofData.proof);
}

// Add to BEEF
const proof = await getMerkleProof(tx.id('hex'));
beef.addMerklePath(proof);
```

## Further Reading

- [BRC-62: BEEF Specification](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0062.md) - Official BEEF format specification
- [BRC-8: Transaction Envelopes](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0008.md) - Transaction envelope format
- [BRC-67: SPV Validation](https://github.com/bitcoin-sv/BRCs/blob/master/payments/0067.md) - SPV validation rules
- [Merkle Trees](https://wiki.bitcoinsv.io/index.php/Merkle_tree) - Understanding merkle trees in Bitcoin
- [SPV](https://wiki.bitcoinsv.io/index.php/Simplified_Payment_Verification) - Simplified Payment Verification overview
- [BSV SDK Documentation](https://bsv-blockchain.github.io/ts-sdk) - Official SDK documentation

## Status

✅ Complete
