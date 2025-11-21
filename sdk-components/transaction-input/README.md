# Transaction Input

## Overview

The `TransactionInput` class in the BSV TypeScript SDK represents an input to a Bitcoin transaction. Each input references an unspent transaction output (UTXO) from a previous transaction and includes an unlocking script (scriptSig) that proves the right to spend those coins. Understanding transaction inputs is fundamental to building and validating Bitcoin transactions.

## Purpose

- Reference unspent transaction outputs (UTXOs) from previous transactions
- Store unlocking scripts (formerly called scriptSig) that prove spending authority
- Maintain sequence numbers for transaction replacement and timelocks
- Support SPV (Simplified Payment Verification) with merkle proofs
- Handle various input types (P2PKH, P2PK, custom scripts, etc.)
- Serialize and deserialize inputs for transaction broadcasting
- Enable transaction signing and validation workflows

## Basic Usage

```typescript
import { Transaction, TransactionInput, PrivateKey, P2PKH } from '@bsv/sdk';

// Create a transaction with an input
const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');

// Reference to previous transaction
const sourceTransaction = Transaction.fromHex('0100000001...');
const sourceOutputIndex = 0;

// Create input with unlocking script template
const input: TransactionInput = {
  sourceTransaction,
  sourceOutputIndex,
  unlockingScriptTemplate: new P2PKH().unlock(privKey),
  sequence: 0xffffffff // Default sequence number
};

// Create transaction and add input
const tx = new Transaction();
tx.addInput(input);

// The unlocking script will be populated during signing
await tx.sign();

console.log('Input added:', tx.inputs[0]);
console.log('Unlocking script:', tx.inputs[0].unlockingScript?.toHex());
```

## Key Features

### 1. Creating Transaction Inputs with Source References

Inputs reference previous transaction outputs (UTXOs):

```typescript
import { Transaction, TransactionInput, PrivateKey, P2PKH } from '@bsv/sdk';

// Create input referencing a specific UTXO
const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');

// Option 1: Full source transaction (for SPV)
const sourceTransaction = Transaction.fromHex('0100000001...');
const input1: TransactionInput = {
  sourceTransaction,
  sourceOutputIndex: 0, // Which output of the source transaction
  unlockingScriptTemplate: new P2PKH().unlock(privKey)
};

// Option 2: Just the TXID (requires trusted validation)
const input2: TransactionInput = {
  sourceTXID: '4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b',
  sourceOutputIndex: 1,
  unlockingScriptTemplate: new P2PKH().unlock(privKey)
};

// Option 3: With source satoshis and locking script (lightweight SPV)
const input3: TransactionInput = {
  sourceTXID: '4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b',
  sourceOutputIndex: 0,
  sourceSatoshis: 10000,
  lockingScript: new P2PKH().lock('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa'),
  unlockingScriptTemplate: new P2PKH().unlock(
    privKey,
    'all',
    false,
    10000
  )
};

// Add inputs to transaction
const tx = new Transaction();
tx.addInput(input1);
tx.addInput(input2);
tx.addInput(input3);

console.log('Transaction has', tx.inputs.length, 'inputs');
```

### 2. Unlocking Script Templates

Unlocking scripts prove the right to spend UTXOs:

```typescript
import { Transaction, PrivateKey, P2PKH, Script, OP } from '@bsv/sdk';

const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');
const sourceTransaction = Transaction.fromHex('...');

// Standard P2PKH unlocking
const p2pkhInput = {
  sourceTransaction,
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(privKey)
};

// P2PKH with custom sighash flags
const customSighashInput = {
  sourceTransaction,
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(
    privKey,
    'single', // signOutputs: 'all' | 'none' | 'single'
    true,     // anyoneCanPay: allows others to add inputs
    10000,    // sourceSatoshis
    new P2PKH().lock('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa') // lockingScript
  )
};

// Custom unlocking script template
import { ScriptTemplate, UnlockingScript, LockingScript } from '@bsv/sdk';

class CustomUnlock implements Partial<ScriptTemplate> {
  unlock(secret: Buffer) {
    return {
      sign: async (tx: Transaction, inputIndex: number) => {
        // Create custom unlocking script
        return new Script()
          .writeBin(secret)
          .writeBin(Buffer.from('additional_data'));
      },
      estimateLength: async () => {
        return 100; // Estimated length in bytes
      }
    };
  }
}

const customInput = {
  sourceTransaction,
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new CustomUnlock().unlock(Buffer.from('my_secret'))
};

// Add inputs to transaction
const tx = new Transaction();
tx.addInput(p2pkhInput);
tx.addInput(customSighashInput);
tx.addInput(customInput);

// Sign transaction (populates unlocking scripts)
await tx.sign();

console.log('All inputs signed');
```

### 3. Sequence Numbers and Time Locks

Sequence numbers enable transaction replacement and relative time locks:

```typescript
import { Transaction, PrivateKey, P2PKH } from '@bsv/sdk';

const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');
const sourceTransaction = Transaction.fromHex('...');

// Default sequence (0xffffffff) - final, no replacement
const finalInput = {
  sourceTransaction,
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(privKey),
  sequence: 0xffffffff
};

// Sequence for RBF (Replace-By-Fee)
const rbfInput = {
  sourceTransaction,
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(privKey),
  sequence: 0xfffffffe // Signals RBF enabled
};

// Relative time lock (BIP 68)
// Lock for 144 blocks (approximately 1 day)
const relativeTimeLockBlocks = {
  sourceTransaction,
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(privKey),
  sequence: 144 // Number of blocks
};

// Relative time lock in seconds (512 second intervals)
// Lock for approximately 1 day (169 * 512 seconds)
const relativeTimeLockSeconds = {
  sourceTransaction,
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(privKey),
  sequence: (1 << 22) | 169 // Type flag + time units
};

const tx = new Transaction();
tx.addInput(finalInput);

console.log('Input sequence:', tx.inputs[0].sequence);
```

### 4. SPV with Merkle Proofs

Include merkle proofs for SPV validation:

```typescript
import { Transaction, MerklePath, TransactionInput } from '@bsv/sdk';

// Parse merkle path from network
const merklePath = MerklePath.fromHex('fed7c509000a02fddd01...');

// Attach merkle proof to source transaction
const sourceTransaction = Transaction.fromHex('0100000001...');
sourceTransaction.merklePath = merklePath;

// Create input with SPV proof
const spvInput: TransactionInput = {
  sourceTransaction, // Includes merkle proof
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(privKey)
};

// The transaction can now be verified with SPV
const tx = new Transaction();
tx.addInput(spvInput);

// Verify merkle proof
const isValidProof = sourceTransaction.merklePath.verify(
  sourceTransaction.id('hex'),
  chainTracker // ChainTracker instance
);

console.log('SPV proof valid:', isValidProof);

// Transaction with SPV proof can be serialized for transmission
const txWithProof = tx.toHex();

// Or use BEEF format for efficient transmission
import { Beef } from '@bsv/sdk';
const beef = new Beef();
beef.addTransaction(tx);
beef.addTransaction(sourceTransaction);
beef.addMerklePath(merklePath);

const beefHex = beef.toHex();
console.log('BEEF transaction package:', beefHex);
```

## API Reference

### TransactionInput Interface

```typescript
interface TransactionInput {
  // Source transaction reference (one of these is required)
  sourceTransaction?: Transaction;  // Full transaction (for SPV)
  sourceTXID?: string;              // Transaction ID only

  // Which output of the source transaction
  sourceOutputIndex: number;

  // Unlocking script (populated during signing)
  unlockingScript?: Script;

  // Unlocking script template (provides signing logic)
  unlockingScriptTemplate?: {
    sign: (tx: Transaction, inputIndex: number) => Promise<UnlockingScript>;
    estimateLength: (tx: Transaction, inputIndex: number) => Promise<number>;
  };

  // Optional fields for lightweight SPV
  sourceSatoshis?: number;          // Amount in satoshis
  lockingScript?: Script;           // Locking script from source output

  // Sequence number (default: 0xffffffff)
  sequence?: number;
}
```

### Creating Inputs

```typescript
// Add input to transaction
tx.addInput(input: TransactionInput): Transaction

// Access inputs
tx.inputs: TransactionInput[]

// Get specific input
const input = tx.inputs[0];
```

### Input Properties

```typescript
input.sourceTransaction    // Transaction | undefined - Source transaction
input.sourceTXID          // string | undefined - Source transaction ID
input.sourceOutputIndex   // number - Output index in source transaction
input.unlockingScript     // Script | undefined - Unlocking script (after signing)
input.unlockingScriptTemplate  // Template for creating unlocking script
input.sourceSatoshis      // number | undefined - Amount from source output
input.lockingScript       // Script | undefined - Locking script from source
input.sequence            // number | undefined - Sequence number
```

### Serialization

```typescript
// Transaction inputs are serialized as part of transaction
const txHex = tx.toHex();
const txBinary = tx.toBinary();

// Parse transaction with inputs
const parsedTx = Transaction.fromHex(txHex);
console.log('Inputs:', parsedTx.inputs.length);
```

## Common Patterns

### Pattern 1: Spending Multiple UTXOs

Consolidate multiple UTXOs into a single transaction:

```typescript
import { Transaction, PrivateKey, P2PKH, ARC } from '@bsv/sdk';

async function consolidateUTXOs(
  utxos: Array<{ txid: string; vout: number; satoshis: number }>,
  privKey: PrivateKey,
  destinationAddress: string
): Promise<Transaction> {
  const tx = new Transaction();

  // Add all UTXOs as inputs
  for (const utxo of utxos) {
    tx.addInput({
      sourceTXID: utxo.txid,
      sourceOutputIndex: utxo.vout,
      sourceSatoshis: utxo.satoshis,
      unlockingScriptTemplate: new P2PKH().unlock(
        privKey,
        'all',
        false,
        utxo.satoshis
      )
    });
  }

  // Calculate total input value
  const totalSatoshis = utxos.reduce((sum, utxo) => sum + utxo.satoshis, 0);

  // Add single output to destination
  tx.addOutput({
    lockingScript: new P2PKH().lock(destinationAddress),
    satoshis: totalSatoshis
  });

  // Calculate fee and adjust output
  await tx.fee();
  await tx.sign();

  // Broadcast
  await tx.broadcast();

  console.log('Consolidated', utxos.length, 'UTXOs');
  console.log('Transaction ID:', tx.id('hex'));

  return tx;
}

// Usage
const utxos = [
  { txid: '4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b', vout: 0, satoshis: 10000 },
  { txid: '3b5e2c8baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda44c', vout: 1, satoshis: 20000 },
  { txid: '1f6d3d9caab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda55d', vout: 0, satoshis: 15000 }
];

const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');
const destination = '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa';

const consolidatedTx = await consolidateUTXOs(utxos, privKey, destination);
```

### Pattern 2: Multi-Signature Input

Create input requiring multiple signatures:

```typescript
import { Transaction, PrivateKey, Script, OP, Signature, Hash } from '@bsv/sdk';

class MultiSigInput {
  /**
   * Create 2-of-3 multisig input
   */
  static create(
    sourceTransaction: Transaction,
    sourceOutputIndex: number,
    privateKeys: PrivateKey[], // Array of 2 private keys
    publicKeys: Buffer[]        // Array of all 3 public keys
  ) {
    return {
      sourceTransaction,
      sourceOutputIndex,
      unlockingScriptTemplate: {
        sign: async (tx: Transaction, inputIndex: number) => {
          // Get the locking script from source output
          const lockingScript = sourceTransaction.outputs[sourceOutputIndex].lockingScript;

          // Calculate sighash
          const preimage = tx.getPreimage(inputIndex);
          const hash = Hash.sha256sha256(Array.from(preimage));

          // Create signatures
          const signatures = privateKeys.map(privKey =>
            Signature.sign(hash, privKey)
          );

          // Create unlocking script: OP_0 <sig1> <sig2>
          const unlockingScript = new Script()
            .writeOpCode(OP.OP_0); // Dummy value for CHECKMULTISIG bug

          signatures.forEach(sig => {
            unlockingScript.writeBin(sig.toChecksigFormat());
          });

          return unlockingScript;
        },
        estimateLength: async () => {
          // OP_0 + 2 signatures (72 bytes each)
          return 1 + (72 * 2);
        }
      }
    };
  }
}

// Usage: Create 2-of-3 multisig transaction
const privKeys = [
  PrivateKey.fromRandom(),
  PrivateKey.fromRandom()
];

const allPubKeys = [
  PrivateKey.fromRandom().toPublicKey().encode(true),
  PrivateKey.fromRandom().toPublicKey().encode(true),
  PrivateKey.fromRandom().toPublicKey().encode(true)
];

const sourceTransaction = Transaction.fromHex('...');

const tx = new Transaction();
tx.addInput(MultiSigInput.create(
  sourceTransaction,
  0,
  privKeys,
  allPubKeys
));

await tx.sign();
console.log('Multisig input created');
```

### Pattern 3: UTXO Management and Coin Selection

Implement coin selection for optimal transaction building:

```typescript
import { Transaction, PrivateKey, P2PKH } from '@bsv/sdk';

interface UTXO {
  txid: string;
  vout: number;
  satoshis: number;
  address: string;
}

class CoinSelector {
  /**
   * Largest-first coin selection
   */
  static selectLargestFirst(
    utxos: UTXO[],
    targetSatoshis: number
  ): UTXO[] {
    // Sort by satoshis descending
    const sorted = [...utxos].sort((a, b) => b.satoshis - a.satoshis);

    const selected: UTXO[] = [];
    let totalSatoshis = 0;

    for (const utxo of sorted) {
      selected.push(utxo);
      totalSatoshis += utxo.satoshis;

      if (totalSatoshis >= targetSatoshis) {
        break;
      }
    }

    return selected;
  }

  /**
   * Smallest-first coin selection (for UTXO consolidation)
   */
  static selectSmallestFirst(
    utxos: UTXO[],
    targetSatoshis: number
  ): UTXO[] {
    // Sort by satoshis ascending
    const sorted = [...utxos].sort((a, b) => a.satoshis - b.satoshis);

    const selected: UTXO[] = [];
    let totalSatoshis = 0;

    for (const utxo of sorted) {
      selected.push(utxo);
      totalSatoshis += utxo.satoshis;

      if (totalSatoshis >= targetSatoshis) {
        break;
      }
    }

    return selected;
  }

  /**
   * Branch and bound coin selection (optimal)
   */
  static selectOptimal(
    utxos: UTXO[],
    targetSatoshis: number
  ): UTXO[] {
    // Simplified version - find exact match or minimal overpay
    const sorted = [...utxos].sort((a, b) => a.satoshis - b.satoshis);

    // Try to find exact match
    let totalSatoshis = 0;
    const selected: UTXO[] = [];

    for (const utxo of sorted) {
      if (totalSatoshis + utxo.satoshis <= targetSatoshis + 1000) {
        selected.push(utxo);
        totalSatoshis += utxo.satoshis;
      }

      if (totalSatoshis >= targetSatoshis) {
        break;
      }
    }

    return selected;
  }

  /**
   * Build transaction with coin selection
   */
  static async buildTransaction(
    utxos: UTXO[],
    outputs: Array<{ address: string; satoshis: number }>,
    privateKeys: Map<string, PrivateKey>,
    changeAddress: string
  ): Promise<Transaction> {
    // Calculate target amount
    const targetSatoshis = outputs.reduce((sum, out) => sum + out.satoshis, 0);

    // Select UTXOs
    const selectedUTXOs = this.selectOptimal(utxos, targetSatoshis + 1000); // +1000 for fees

    if (selectedUTXOs.length === 0) {
      throw new Error('Insufficient funds');
    }

    // Create transaction
    const tx = new Transaction();

    // Add inputs
    for (const utxo of selectedUTXOs) {
      const privKey = privateKeys.get(utxo.address);
      if (!privKey) {
        throw new Error(`No private key for address ${utxo.address}`);
      }

      tx.addInput({
        sourceTXID: utxo.txid,
        sourceOutputIndex: utxo.vout,
        sourceSatoshis: utxo.satoshis,
        unlockingScriptTemplate: new P2PKH().unlock(
          privKey,
          'all',
          false,
          utxo.satoshis
        )
      });
    }

    // Add outputs
    for (const output of outputs) {
      tx.addOutput({
        lockingScript: new P2PKH().lock(output.address),
        satoshis: output.satoshis
      });
    }

    // Add change output
    tx.addOutput({
      lockingScript: new P2PKH().lock(changeAddress),
      change: true
    });

    // Calculate fees and sign
    await tx.fee();
    await tx.sign();

    return tx;
  }
}

// Usage
const utxos: UTXO[] = [
  { txid: '4a5e...', vout: 0, satoshis: 10000, address: '1A1z...' },
  { txid: '3b5e...', vout: 1, satoshis: 20000, address: '1A1z...' },
  { txid: '1f6d...', vout: 0, satoshis: 15000, address: '1B2y...' }
];

const privateKeys = new Map<string, PrivateKey>([
  ['1A1z...', PrivateKey.fromWif('L5EY...')],
  ['1B2y...', PrivateKey.fromWif('K9XM...')]
]);

const outputs = [
  { address: '1C3x...', satoshis: 25000 }
];

const tx = await CoinSelector.buildTransaction(
  utxos,
  outputs,
  privateKeys,
  '1A1z...' // Change address
);

console.log('Transaction built with', tx.inputs.length, 'inputs');
```

## Security Considerations

1. **UTXO Verification**: Always verify that referenced UTXOs exist and are unspent before creating transactions.

2. **Source Transaction Validation**: When using full source transactions, validate the transaction before referencing it.

3. **SPV Security**: Merkle proofs provide SPV but require trusted chain tracker. Validate merkle roots against known block headers.

4. **Private Key Management**: Never expose private keys. Use secure key derivation and storage.

5. **Sequence Number Usage**: Be aware that non-final sequence numbers enable transaction replacement. Use 0xffffffff for final transactions.

6. **Double Spend Prevention**: Ensure UTXOs haven't been spent in another transaction before broadcasting.

## Performance Considerations

1. **Input Count**: More inputs increase transaction size and fees. Use coin selection to minimize inputs.

2. **UTXO Consolidation**: Periodically consolidate small UTXOs to reduce future transaction costs.

3. **SPV vs Full Transaction**: Including full source transactions increases size. Use SPV proofs or BEEF format for efficiency.

4. **Unlocking Script Size**: Custom unlocking scripts should be as compact as possible to minimize fees.

5. **Parallel Signing**: When signing multiple inputs, consider parallel signature generation for better performance.

## Related Components

- [Transaction](../transaction/README.md) - Build complete transactions with inputs
- [TransactionOutput](../transaction-output/README.md) - Outputs that become inputs
- [Script](../script/README.md) - Create unlocking scripts
- [ScriptTemplate](../script-templates/README.md) - Templates for common unlock patterns
- [Signature](../signatures/README.md) - Sign transaction inputs

## Code Examples

See complete working examples in:
- [Transaction Building](../../code-features/transaction-building/README.md)
- [UTXO Management](../../code-features/utxo-management/README.md)
- [Multi-Signature](../../code-features/multi-signature/README.md)
- [SPV Validation](../../code-features/spv/README.md)

## Best Practices

1. **Always include source transaction or TXID** - Required for input validation
2. **Use SPV proofs** when possible for lighter weight transactions
3. **Set appropriate sequence numbers** - Use 0xffffffff for final transactions
4. **Implement coin selection** to optimize transaction fees and UTXO management
5. **Verify UTXOs before spending** - Ensure they exist and are unspent
6. **Use BEEF format** for efficient transaction package transmission
7. **Handle errors gracefully** - Invalid UTXOs should not crash your application
8. **Never reuse addresses** - Generate new change addresses for each transaction
9. **Consolidate small UTXOs** periodically to reduce future transaction costs
10. **Test with small amounts** before using real funds

## Troubleshooting

### Issue: Input references non-existent UTXO

**Solution**: Verify the source transaction exists and the output index is valid.

```typescript
// Verify UTXO exists
const sourceTransaction = await fetchTransaction(sourceTXID);
if (!sourceTransaction.outputs[sourceOutputIndex]) {
  throw new Error('Invalid output index');
}
```

### Issue: Insufficient funds

**Solution**: Ensure total input value exceeds output value plus fees.

```typescript
const totalInput = tx.inputs.reduce((sum, input) =>
  sum + (input.sourceSatoshis || 0), 0
);

const totalOutput = tx.outputs.reduce((sum, output) =>
  sum + output.satoshis, 0
);

if (totalInput < totalOutput) {
  throw new Error('Insufficient funds');
}
```

### Issue: Unlocking script doesn't match locking script

**Solution**: Ensure the unlocking script template matches the locking script type.

```typescript
// For P2PKH locking script, use P2PKH unlock template
const input = {
  sourceTransaction,
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(privKey)
};
```

### Issue: Sequence number prevents replacement

**Solution**: Use appropriate sequence number for your use case.

```typescript
// For RBF (Replace-By-Fee)
input.sequence = 0xfffffffe;

// For final transaction (no replacement)
input.sequence = 0xffffffff;
```

## Further Reading

- [Bitcoin Transactions](https://wiki.bitcoinsv.io/index.php/Transaction) - Complete transaction documentation
- [UTXOs](https://wiki.bitcoinsv.io/index.php/UTXO) - Understanding unspent transaction outputs
- [Sequence Numbers](https://wiki.bitcoinsv.io/index.php/NSequence) - Transaction replacement and timelocks
- [SPV](https://wiki.bitcoinsv.io/index.php/Simplified_Payment_Verification) - Simplified Payment Verification
- [BEEF Format](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0062.md) - BRC-62: Background Evaluation Extended Format
- [BSV SDK Documentation](https://bsv-blockchain.github.io/ts-sdk) - Official SDK docs

## Status

✅ Complete
