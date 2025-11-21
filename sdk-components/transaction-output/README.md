# Transaction Output

## Overview

The `TransactionOutput` class in the BSV TypeScript SDK represents an output in a Bitcoin transaction. Each output specifies an amount of satoshis and a locking script that defines the conditions under which those coins can be spent. Outputs create new UTXOs (Unspent Transaction Outputs) that can be referenced as inputs in future transactions.

## Purpose

- Define where coins are sent in a transaction with locking scripts
- Specify the amount of satoshis for each output
- Create spendable UTXOs for future transactions
- Support standard payment types (P2PKH, P2PK, OP_RETURN)
- Enable change outputs with automatic value calculation
- Handle custom locking conditions for smart contracts
- Serialize and deserialize outputs for transaction broadcasting

## Basic Usage

```typescript
import { Transaction, P2PKH, Script, OP } from '@bsv/sdk';

const tx = new Transaction();

// Add a standard P2PKH payment output
tx.addOutput({
  lockingScript: new P2PKH().lock('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa'),
  satoshis: 10000
});

// Add a change output (amount calculated automatically)
tx.addOutput({
  lockingScript: new P2PKH().lock(changeAddress),
  change: true
});

// Add an OP_RETURN data output
tx.addOutput({
  lockingScript: new Script()
    .writeOpCode(OP.OP_FALSE)
    .writeOpCode(OP.OP_RETURN)
    .writeBin(Buffer.from('Hello BSV')),
  satoshis: 0
});

console.log('Outputs:', tx.outputs.length);
console.log('Total output value:', tx.outputs.reduce((sum, o) => sum + o.satoshis, 0));
```

## Key Features

### 1. Standard Payment Outputs

Create outputs for sending payments to addresses:

```typescript
import { Transaction, P2PKH, PrivateKey } from '@bsv/sdk';

const tx = new Transaction();

// P2PKH output (most common - pay to public key hash)
const recipientAddress = '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa';
tx.addOutput({
  lockingScript: new P2PKH().lock(recipientAddress),
  satoshis: 50000 // 0.0005 BSV
});

// P2PK output (pay to public key directly)
const recipientPubKey = PrivateKey.fromRandom().toPublicKey();
tx.addOutput({
  lockingScript: new Script()
    .writeBin(recipientPubKey.encode(true))
    .writeOpCode(OP.OP_CHECKSIG),
  satoshis: 25000
});

// Multiple outputs to different recipients
const recipients = [
  { address: '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa', satoshis: 10000 },
  { address: '1B2Y3Z...', satoshis: 20000 },
  { address: '1C3X4W...', satoshis: 30000 }
];

recipients.forEach(recipient => {
  tx.addOutput({
    lockingScript: new P2PKH().lock(recipient.address),
    satoshis: recipient.satoshis
  });
});

console.log('Created', tx.outputs.length, 'payment outputs');
```

### 2. Change Outputs

Automatically calculate change from transaction inputs:

```typescript
import { Transaction, PrivateKey, P2PKH } from '@bsv/sdk';

const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');
const changeAddress = privKey.toPublicKey().toAddress();

const tx = new Transaction();

// Add input (100,000 satoshis)
tx.addInput({
  sourceTXID: '4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b',
  sourceOutputIndex: 0,
  sourceSatoshis: 100000,
  unlockingScriptTemplate: new P2PKH().unlock(privKey, 'all', false, 100000)
});

// Add payment output (60,000 satoshis)
tx.addOutput({
  lockingScript: new P2PKH().lock('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa'),
  satoshis: 60000
});

// Add change output - NEVER reuse the input address!
tx.addOutput({
  lockingScript: new P2PKH().lock(changeAddress),
  change: true // Amount calculated automatically
});

// Calculate fee (deducts from change output)
await tx.fee();

console.log('Change output value:', tx.outputs[1].satoshis);
// Example: 39800 satoshis (100000 - 60000 - 200 fee)

await tx.sign();
```

### 3. OP_RETURN Data Outputs

Store arbitrary data on the blockchain:

```typescript
import { Transaction, Script, OP } from '@bsv/sdk';

const tx = new Transaction();

// Simple OP_RETURN with text
tx.addOutput({
  lockingScript: new Script()
    .writeOpCode(OP.OP_FALSE)
    .writeOpCode(OP.OP_RETURN)
    .writeBin(Buffer.from('Hello BSV')),
  satoshis: 0 // OP_RETURN outputs typically have 0 value
});

// Multi-field OP_RETURN (protocol data)
tx.addOutput({
  lockingScript: new Script()
    .writeOpCode(OP.OP_FALSE)
    .writeOpCode(OP.OP_RETURN)
    .writeBin(Buffer.from('19HxigV4QyBv3tHpQVcUEQyq1pzZVdoAut')) // B:// protocol
    .writeBin(Buffer.from('image/png', 'utf8'))
    .writeBin(Buffer.from('filename.png', 'utf8'))
    .writeBin(Buffer.from([0x89, 0x50, 0x4e, 0x47])), // PNG header
  satoshis: 0
});

// Large data output (BSV supports large OP_RETURN)
const largeData = Buffer.alloc(100000); // 100 KB
largeData.fill('BSV');

tx.addOutput({
  lockingScript: new Script()
    .writeOpCode(OP.OP_FALSE)
    .writeOpCode(OP.OP_RETURN)
    .writeBin(largeData),
  satoshis: 0
});

console.log('OP_RETURN data size:', largeData.length, 'bytes');
```

### 4. Custom Locking Scripts

Create outputs with custom spending conditions:

```typescript
import { Transaction, Script, OP, Hash } from '@bsv/sdk';

const tx = new Transaction();

// Hash puzzle output (anyone with secret can spend)
const secret = Buffer.from('my_secret');
const secretHash = Hash.sha256(secret);

tx.addOutput({
  lockingScript: new Script()
    .writeOpCode(OP.OP_HASH256)
    .writeBin(Array.from(secretHash))
    .writeOpCode(OP.OP_EQUAL),
  satoshis: 10000
});

// Time-locked output (can only spend after block height/time)
const lockTime = 1700000000; // Unix timestamp
tx.addOutput({
  lockingScript: new Script()
    .writeBin(Buffer.from(lockTime.toString(16), 'hex'))
    .writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
    .writeOpCode(OP.OP_DROP)
    .writeOpCode(OP.OP_DUP)
    .writeOpCode(OP.OP_HASH160)
    .writeBin(Buffer.from('pubkeyhash', 'hex'))
    .writeOpCode(OP.OP_EQUALVERIFY)
    .writeOpCode(OP.OP_CHECKSIG),
  satoshis: 50000
});

// Multi-signature output (2-of-3)
const pubKey1 = Buffer.from('02abc...', 'hex');
const pubKey2 = Buffer.from('03def...', 'hex');
const pubKey3 = Buffer.from('02123...', 'hex');

tx.addOutput({
  lockingScript: new Script()
    .writeOpCode(OP.OP_2) // Require 2 signatures
    .writeBin(pubKey1)
    .writeBin(pubKey2)
    .writeBin(pubKey3)
    .writeOpCode(OP.OP_3) // Out of 3 keys
    .writeOpCode(OP.OP_CHECKMULTISIG),
  satoshis: 100000
});

console.log('Created custom locking outputs');
```

## API Reference

### TransactionOutput Interface

```typescript
interface TransactionOutput {
  // Amount in satoshis
  satoshis: number;

  // Locking script (defines spending conditions)
  lockingScript: Script;

  // Whether this is a change output (amount calculated automatically)
  change?: boolean;
}
```

### Creating Outputs

```typescript
// Add output to transaction
tx.addOutput(output: TransactionOutput): Transaction

// Access outputs
tx.outputs: TransactionOutput[]

// Get specific output
const output = tx.outputs[0];
```

### Output Properties

```typescript
output.satoshis       // number - Amount in satoshis
output.lockingScript  // Script - Locking script
output.change         // boolean | undefined - Is change output
```

### Serialization

```typescript
// Transaction outputs are serialized as part of transaction
const txHex = tx.toHex();
const txBinary = tx.toBinary();

// Parse transaction with outputs
const parsedTx = Transaction.fromHex(txHex);
console.log('Outputs:', parsedTx.outputs.length);
console.log('First output value:', parsedTx.outputs[0].satoshis);
```

## Common Patterns

### Pattern 1: Payment Distribution

Split payment among multiple recipients:

```typescript
import { Transaction, PrivateKey, P2PKH } from '@bsv/sdk';

async function distributePayment(
  sourceUTXO: { txid: string; vout: number; satoshis: number },
  recipients: Array<{ address: string; amount: number }>,
  privKey: PrivateKey,
  changeAddress: string
): Promise<Transaction> {
  const tx = new Transaction();

  // Add input
  tx.addInput({
    sourceTXID: sourceUTXO.txid,
    sourceOutputIndex: sourceUTXO.vout,
    sourceSatoshis: sourceUTXO.satoshis,
    unlockingScriptTemplate: new P2PKH().unlock(
      privKey,
      'all',
      false,
      sourceUTXO.satoshis
    )
  });

  // Add output for each recipient
  recipients.forEach(recipient => {
    tx.addOutput({
      lockingScript: new P2PKH().lock(recipient.address),
      satoshis: recipient.amount
    });
  });

  // Add change output
  tx.addOutput({
    lockingScript: new P2PKH().lock(changeAddress),
    change: true
  });

  // Calculate fees and sign
  await tx.fee();
  await tx.sign();

  console.log('Distributed to', recipients.length, 'recipients');
  return tx;
}

// Usage
const utxo = { txid: '4a5e...', vout: 0, satoshis: 100000 };
const recipients = [
  { address: '1A1z...', amount: 10000 },
  { address: '1B2y...', amount: 20000 },
  { address: '1C3x...', amount: 30000 }
];

const privKey = PrivateKey.fromWif('L5EY...');
const tx = await distributePayment(utxo, recipients, privKey, '1D4w...');
```

### Pattern 2: Data and Payment Combined

Store data while making payments:

```typescript
import { Transaction, P2PKH, Script, OP } from '@bsv/sdk';

class DataPaymentTransaction {
  static create(
    paymentAddress: string,
    paymentSatoshis: number,
    data: Buffer[],
    changeAddress: string
  ): Transaction {
    const tx = new Transaction();

    // Output 0: Payment to recipient
    tx.addOutput({
      lockingScript: new P2PKH().lock(paymentAddress),
      satoshis: paymentSatoshis
    });

    // Output 1: OP_RETURN data
    const dataScript = new Script()
      .writeOpCode(OP.OP_FALSE)
      .writeOpCode(OP.OP_RETURN);

    data.forEach(chunk => dataScript.writeBin(chunk));

    tx.addOutput({
      lockingScript: dataScript,
      satoshis: 0
    });

    // Output 2: Change
    tx.addOutput({
      lockingScript: new P2PKH().lock(changeAddress),
      change: true
    });

    return tx;
  }
}

// Usage: Invoice with payment and data
const tx = DataPaymentTransaction.create(
  '1A1z...', // Payment recipient
  50000,     // Payment amount
  [
    Buffer.from('INVOICE'),
    Buffer.from('INV-2024-001'),
    Buffer.from(JSON.stringify({ items: ['item1', 'item2'], total: 50000 }))
  ],
  '1D4w...' // Change address
);

// Add input and finalize
// ... add inputs, calculate fee, sign ...
```

### Pattern 3: Atomic Swap Output Structure

Create outputs for atomic swap transactions:

```typescript
import { Transaction, Script, OP, Hash } from '@bsv/sdk';

class AtomicSwapOutput {
  /**
   * Create Hash Time Locked Contract (HTLC) output for atomic swap
   */
  static createHTLC(
    payeeHash: Buffer,
    payerHash: Buffer,
    secret: Buffer,
    lockTime: number,
    amount: number
  ): TransactionOutput {
    const secretHash = Hash.sha256(secret);

    const lockingScript = new Script()
      // Path 1: Payee provides secret
      .writeOpCode(OP.OP_IF)
        .writeOpCode(OP.OP_HASH256)
        .writeBin(Array.from(secretHash))
        .writeOpCode(OP.OP_EQUALVERIFY)
        .writeOpCode(OP.OP_DUP)
        .writeOpCode(OP.OP_HASH160)
        .writeBin(payeeHash)
      .writeOpCode(OP.OP_ELSE)
        // Path 2: Payer reclaims after timeout
        .writeBin(Buffer.from(lockTime.toString(16), 'hex'))
        .writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
        .writeOpCode(OP.OP_DROP)
        .writeOpCode(OP.OP_DUP)
        .writeOpCode(OP.OP_HASH160)
        .writeBin(payerHash)
      .writeOpCode(OP.OP_ENDIF)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG);

    return {
      lockingScript,
      satoshis: amount
    };
  }

  /**
   * Create both sides of atomic swap
   */
  static createSwapTransactions(
    party1: { hash: Buffer; amount: number },
    party2: { hash: Buffer; amount: number },
    secret: Buffer,
    lockTime1: number,
    lockTime2: number
  ): { tx1: Transaction; tx2: Transaction } {
    // Transaction 1: Party 1 locks funds
    const tx1 = new Transaction();
    tx1.addOutput(
      this.createHTLC(party2.hash, party1.hash, secret, lockTime1, party1.amount)
    );

    // Transaction 2: Party 2 locks funds (same secret hash)
    const tx2 = new Transaction();
    tx2.addOutput(
      this.createHTLC(party1.hash, party2.hash, secret, lockTime2, party2.amount)
    );

    return { tx1, tx2 };
  }
}

// Usage: Create atomic swap between two parties
const secret = Buffer.from('shared_secret_12345');
const party1Hash = Buffer.from('party1_pubkey_hash', 'hex');
const party2Hash = Buffer.from('party2_pubkey_hash', 'hex');

const { tx1, tx2 } = AtomicSwapOutput.createSwapTransactions(
  { hash: party1Hash, amount: 100000 },
  { hash: party2Hash, amount: 200000 },
  secret,
  Math.floor(Date.now() / 1000) + 86400, // 24 hours
  Math.floor(Date.now() / 1000) + 86400
);

console.log('Atomic swap transactions created');
```

## Security Considerations

1. **Output Value Validation**: Ensure output values don't exceed input values (accounting for fees).

2. **Change Address Security**: Never reuse addresses. Generate new addresses for change outputs.

3. **Dust Outputs**: Very small outputs (below dust threshold) may not be relayed by nodes. Minimum is typically 546 satoshis for P2PKH.

4. **OP_RETURN Data**: While BSV supports large OP_RETURN, verify miner policies for data size acceptance.

5. **Locking Script Validation**: Test custom locking scripts thoroughly before using with real funds.

6. **Zero-Value Outputs**: Only OP_RETURN outputs should have zero satoshis. Payment outputs must have positive values.

## Performance Considerations

1. **Output Count**: More outputs increase transaction size and fees. Consolidate when possible.

2. **Script Complexity**: Complex locking scripts increase size. Keep scripts simple when possible.

3. **OP_RETURN Size**: Large OP_RETURN data increases transaction size proportionally. Consider overlay networks for very large data.

4. **Change Output Management**: Always include change output to avoid losing funds to fees.

5. **Output Ordering**: Output order doesn't affect validity but may affect privacy. Consider randomizing output order.

## Related Components

- [Transaction](../transaction/README.md) - Build complete transactions with outputs
- [TransactionInput](../transaction-input/README.md) - Inputs that reference outputs
- [Script](../script/README.md) - Create locking scripts
- [ScriptTemplate](../script-templates/README.md) - Templates for common locking patterns
- [P2PKH](../script-templates/README.md#p2pkh) - Standard payment template

## Code Examples

See complete working examples in:
- [Transaction Building](../../code-features/transaction-building/README.md)
- [OP_RETURN Data](../../code-features/op-return/README.md)
- [Payment Distribution](../../code-features/payment-distribution/README.md)
- [Smart Contracts](../../code-features/smart-contracts/README.md)

## Best Practices

1. **Always include change output** to avoid losing funds to fees
2. **Never reuse addresses** - generate new address for change
3. **Validate total output value** doesn't exceed input value
4. **Use appropriate output types** - P2PKH for payments, OP_RETURN for data
5. **Set OP_RETURN outputs to 0 satoshis** unless protocol requires otherwise
6. **Check dust limits** - ensure outputs meet minimum value requirements
7. **Test custom scripts** thoroughly before production use
8. **Document OP_RETURN data format** for protocol interoperability
9. **Consider miner policies** for large data outputs
10. **Implement proper error handling** for output creation failures

## Troubleshooting

### Issue: Output value exceeds input value

**Solution**: Ensure total outputs (including fees) don't exceed inputs.

```typescript
const totalInput = tx.inputs.reduce((sum, i) => sum + (i.sourceSatoshis || 0), 0);
const totalOutput = tx.outputs.reduce((sum, o) => sum + o.satoshis, 0);

if (totalOutput > totalInput) {
  throw new Error(`Output (${totalOutput}) exceeds input (${totalInput})`);
}
```

### Issue: Dust output rejected

**Solution**: Ensure outputs meet minimum dust threshold (546 satoshis for P2PKH).

```typescript
const DUST_LIMIT = 546;

tx.outputs.forEach((output, index) => {
  if (output.satoshis > 0 && output.satoshis < DUST_LIMIT) {
    console.warn(`Output ${index} is dust: ${output.satoshis} satoshis`);
  }
});
```

### Issue: Change output has negative value

**Solution**: Ensure sufficient input value to cover outputs and fees.

```typescript
await tx.fee(); // Calculate fees

// Check if change output is negative
const changeOutput = tx.outputs.find(o => o.change);
if (changeOutput && changeOutput.satoshis < 0) {
  throw new Error('Insufficient funds for transaction');
}
```

### Issue: OP_RETURN too large

**Solution**: Check miner policies and consider splitting data or using overlay networks.

```typescript
const MAX_OP_RETURN = 100000; // Example limit

const opReturnOutput = tx.outputs.find(o =>
  o.lockingScript.chunks.some(c => c.op === OP.OP_RETURN)
);

if (opReturnOutput) {
  const size = opReturnOutput.lockingScript.toBinary().length;
  if (size > MAX_OP_RETURN) {
    console.warn(`OP_RETURN size (${size}) exceeds recommended limit`);
  }
}
```

## Further Reading

- [Bitcoin Outputs](https://wiki.bitcoinsv.io/index.php/Transaction#Outputs) - Transaction output structure
- [OP_RETURN](https://wiki.bitcoinsv.io/index.php/OP_RETURN) - Data storage on blockchain
- [Dust](https://wiki.bitcoinsv.io/index.php/Dust) - Understanding dust limits
- [Locking Scripts](https://wiki.bitcoinsv.io/index.php/Script) - Bitcoin Script for outputs
- [BRC-8](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0008.md) - Transaction envelopes
- [BSV SDK Documentation](https://bsv-blockchain.github.io/ts-sdk) - Official SDK docs

## Status

✅ Complete
