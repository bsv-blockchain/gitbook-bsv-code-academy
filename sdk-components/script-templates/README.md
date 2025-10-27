# Script Templates

## Overview

Script Templates in the BSV TypeScript SDK provide standardized, reusable patterns for creating locking and unlocking scripts. The `ScriptTemplate` interface and its implementations (like `P2PKH`) abstract away the complexity of Bitcoin Script construction, making it easy to create secure, standard transaction types. Templates ensure correct script generation and reduce errors in transaction building.

## Purpose

- Provide pre-built templates for standard transaction types (P2PKH, P2PK, etc.)
- Abstract Bitcoin Script complexity with simple, type-safe interfaces
- Enable creation of both locking scripts (outputs) and unlocking script templates (inputs)
- Support custom template creation for advanced use cases
- Ensure BRC compliance and interoperability
- Simplify transaction signing with automatic script generation
- Reduce errors through tested, standard implementations

## Basic Usage

```typescript
import { Transaction, P2PKH, PrivateKey } from '@bsv/sdk';

const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');
const recipientAddress = '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa';

// Create P2PKH template instance
const p2pkh = new P2PKH();

const tx = new Transaction();

// Create locking script for output
const lockingScript = p2pkh.lock(recipientAddress);
tx.addOutput({
  lockingScript,
  satoshis: 10000
});

// Create unlocking script template for input
const unlockingTemplate = p2pkh.unlock(privKey);
tx.addInput({
  sourceTXID: '4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b',
  sourceOutputIndex: 0,
  unlockingScriptTemplate: unlockingTemplate
});

// Sign transaction (unlocking script generated automatically)
await tx.sign();

console.log('Transaction created with P2PKH template');
```

## Key Features

### 1. P2PKH Template (Pay to Public Key Hash)

The most common Bitcoin transaction template:

```typescript
import { Transaction, P2PKH, PrivateKey, PublicKey } from '@bsv/sdk';

const p2pkh = new P2PKH();

// Create locking script from address
const address = '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa';
const lockingScript = p2pkh.lock(address);
console.log('Locking script:', lockingScript.toASM());
// Output: "OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG"

// Create locking script from public key
const pubKey = PrivateKey.fromRandom().toPublicKey();
const lockingScriptFromPubKey = p2pkh.lock(pubKey.toAddress());

// Create unlocking script template (default: SIGHASH_ALL)
const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');
const unlockingTemplate = p2pkh.unlock(privKey);

// Create unlocking template with custom sighash
const customUnlocking = p2pkh.unlock(
  privKey,
  'single',  // signOutputs: 'all' | 'none' | 'single'
  true,      // anyoneCanPay: allows others to add inputs
  10000,     // sourceSatoshis (optional)
  lockingScript // lockingScript (optional)
);

// Use in transaction
const tx = new Transaction();

tx.addInput({
  sourceTXID: '4a5e...',
  sourceOutputIndex: 0,
  sourceSatoshis: 10000,
  unlockingScriptTemplate: unlockingTemplate
});

tx.addOutput({
  lockingScript: p2pkh.lock(address),
  satoshis: 9000
});

tx.addOutput({
  lockingScript: p2pkh.lock(privKey.toPublicKey().toAddress()),
  change: true
});

await tx.fee();
await tx.sign();

console.log('P2PKH transaction signed');
```

### 2. Custom Script Templates

Create your own templates for advanced use cases:

```typescript
import {
  ScriptTemplate,
  LockingScript,
  UnlockingScript,
  Script,
  OP,
  Transaction,
  Hash
} from '@bsv/sdk';

/**
 * Hash Puzzle Template
 * Anyone who knows the secret can unlock
 */
class HashPuzzle implements ScriptTemplate {
  /**
   * Create locking script with hash of secret
   */
  lock(secretHash: Buffer): LockingScript {
    return new Script()
      .writeOpCode(OP.OP_HASH256)
      .writeBin(secretHash)
      .writeOpCode(OP.OP_EQUAL);
  }

  /**
   * Create unlocking script template with secret
   */
  unlock(secret: Buffer): {
    sign: (tx: Transaction, inputIndex: number) => Promise<UnlockingScript>;
    estimateLength: (tx: Transaction, inputIndex: number) => Promise<number>;
  } {
    return {
      sign: async (tx: Transaction, inputIndex: number) => {
        // Simply push the secret onto the stack
        return new Script().writeBin(secret);
      },
      estimateLength: async () => {
        // Length of secret + push opcode
        return secret.length + 2;
      }
    };
  }
}

// Usage
const secret = Buffer.from('my_secret_value');
const secretHash = Hash.sha256(secret);

const hashPuzzle = new HashPuzzle();

const tx = new Transaction();

// Create output with hash puzzle
tx.addOutput({
  lockingScript: hashPuzzle.lock(secretHash),
  satoshis: 10000
});

// Later, unlock with secret
tx.addInput({
  sourceTXID: '...',
  sourceOutputIndex: 0,
  unlockingScriptTemplate: hashPuzzle.unlock(secret)
});

await tx.sign();
console.log('Hash puzzle unlocked');
```

### 3. Time-Locked Template

Create template with time-based conditions:

```typescript
import {
  ScriptTemplate,
  Script,
  OP,
  Transaction,
  PrivateKey,
  Signature,
  Hash
} from '@bsv/sdk';

/**
 * CheckLockTimeVerify (CLTV) Template
 * Can only spend after specified time/block height
 */
class TimeLock implements ScriptTemplate {
  /**
   * Create time-locked locking script
   */
  lock(lockTime: number, pubKeyHash: Buffer): LockingScript {
    const lockTimeBuffer = Buffer.alloc(4);
    lockTimeBuffer.writeUInt32LE(lockTime, 0);

    return new Script()
      // Check locktime
      .writeBin(lockTimeBuffer)
      .writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
      .writeOpCode(OP.OP_DROP)
      // Standard P2PKH after locktime check
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(pubKeyHash)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG);
  }

  /**
   * Create unlocking template
   */
  unlock(
    privKey: PrivateKey,
    lockTime: number
  ): {
    sign: (tx: Transaction, inputIndex: number) => Promise<UnlockingScript>;
    estimateLength: (tx: Transaction, inputIndex: number) => Promise<number>;
  } {
    return {
      sign: async (tx: Transaction, inputIndex: number) => {
        // Set transaction locktime
        tx.lockTime = lockTime;

        // Get sighash and sign
        const preimage = tx.getPreimage(inputIndex);
        const hash = Hash.sha256sha256(Array.from(preimage));
        const signature = Signature.sign(hash, privKey);

        // Create unlocking script: <sig> <pubkey>
        return new Script()
          .writeBin(signature.toChecksigFormat())
          .writeBin(privKey.toPublicKey().encode(true));
      },
      estimateLength: async () => {
        return 107; // Typical signature + pubkey size
      }
    };
  }
}

// Usage: Lock coins until specific block height
const lockTime = 800000; // Block height
const pubKeyHash = Hash.hash160(privKey.toPublicKey().encode(true));

const timeLock = new TimeLock();

const tx = new Transaction();

tx.addOutput({
  lockingScript: timeLock.lock(lockTime, pubKeyHash),
  satoshis: 100000
});

// Later, after block 800000
const unlockTx = new Transaction();
unlockTx.addInput({
  sourceTXID: tx.id('hex'),
  sourceOutputIndex: 0,
  unlockingScriptTemplate: timeLock.unlock(privKey, lockTime)
});

await unlockTx.sign();
console.log('Time-locked funds unlocked');
```

### 4. Multi-Signature Template

M-of-N multisig template:

```typescript
import {
  ScriptTemplate,
  Script,
  OP,
  Transaction,
  PrivateKey,
  PublicKey,
  Signature,
  Hash
} from '@bsv/sdk';

/**
 * Multi-Signature Template (M-of-N)
 */
class MultiSig implements ScriptTemplate {
  /**
   * Create M-of-N multisig locking script
   */
  lock(requiredSigs: number, publicKeys: PublicKey[]): LockingScript {
    const script = new Script()
      .writeNumber(requiredSigs);

    // Add all public keys
    publicKeys.forEach(pubKey => {
      script.writeBin(pubKey.encode(true));
    });

    script
      .writeNumber(publicKeys.length)
      .writeOpCode(OP.OP_CHECKMULTISIG);

    return script;
  }

  /**
   * Create unlocking template with M private keys
   */
  unlock(
    privateKeys: PrivateKey[],
    requiredSigs: number
  ): {
    sign: (tx: Transaction, inputIndex: number) => Promise<UnlockingScript>;
    estimateLength: (tx: Transaction, inputIndex: number) => Promise<number>;
  } {
    if (privateKeys.length < requiredSigs) {
      throw new Error(`Need at least ${requiredSigs} private keys`);
    }

    return {
      sign: async (tx: Transaction, inputIndex: number) => {
        // Get sighash
        const preimage = tx.getPreimage(inputIndex);
        const hash = Hash.sha256sha256(Array.from(preimage));

        // Create signatures
        const signatures = privateKeys
          .slice(0, requiredSigs)
          .map(privKey => Signature.sign(hash, privKey));

        // Create unlocking script: OP_0 <sig1> <sig2> ... <sigM>
        const script = new Script()
          .writeOpCode(OP.OP_0); // Bug in CHECKMULTISIG requires extra value

        signatures.forEach(sig => {
          script.writeBin(sig.toChecksigFormat());
        });

        return script;
      },
      estimateLength: async () => {
        // OP_0 + M signatures (73 bytes each average)
        return 1 + (requiredSigs * 73);
      }
    };
  }
}

// Usage: 2-of-3 multisig
const privKeys = [
  PrivateKey.fromRandom(),
  PrivateKey.fromRandom(),
  PrivateKey.fromRandom()
];

const pubKeys = privKeys.map(pk => pk.toPublicKey());

const multiSig = new MultiSig();

// Create 2-of-3 multisig output
const tx = new Transaction();
tx.addOutput({
  lockingScript: multiSig.lock(2, pubKeys),
  satoshis: 100000
});

// Later, unlock with 2 of the 3 keys
const unlockTx = new Transaction();
unlockTx.addInput({
  sourceTXID: tx.id('hex'),
  sourceOutputIndex: 0,
  unlockingScriptTemplate: multiSig.unlock([privKeys[0], privKeys[2]], 2)
});

await unlockTx.sign();
console.log('2-of-3 multisig unlocked');
```

## API Reference

### ScriptTemplate Interface

```typescript
interface ScriptTemplate {
  /**
   * Create locking script (for outputs)
   */
  lock(...args: any[]): LockingScript;

  /**
   * Create unlocking script template (for inputs)
   */
  unlock(...args: any[]): {
    sign: (tx: Transaction, inputIndex: number) => Promise<UnlockingScript>;
    estimateLength: (tx: Transaction, inputIndex: number) => Promise<number>;
  };
}
```

### P2PKH Template

```typescript
class P2PKH implements ScriptTemplate {
  // Create locking script from address or public key
  lock(address: string): LockingScript;

  // Create unlocking script template
  unlock(
    privateKey: PrivateKey,
    signOutputs?: 'all' | 'none' | 'single',
    anyoneCanPay?: boolean,
    sourceSatoshis?: number,
    lockingScript?: Script
  ): UnlockingScriptTemplate;
}
```

### Creating Custom Templates

```typescript
class CustomTemplate implements ScriptTemplate {
  lock(param1: any, param2: any): LockingScript {
    // Return a Script that defines spending conditions
    return new Script()
      .writeOpCode(OP.OP_...)
      .writeBin(data);
  }

  unlock(param1: any): UnlockingScriptTemplate {
    return {
      sign: async (tx: Transaction, inputIndex: number) => {
        // Generate unlocking script based on transaction context
        // Typically involves creating signatures
        return new Script()
          .writeBin(signature)
          .writeBin(data);
      },
      estimateLength: async (tx: Transaction, inputIndex: number) => {
        // Return estimated byte length of unlocking script
        return estimatedLength;
      }
    };
  }
}
```

## Common Patterns

### Pattern 1: Reusable Template Factory

Create template instances with configuration:

```typescript
import { P2PKH, PrivateKey, Transaction } from '@bsv/sdk';

class TemplateFactory {
  /**
   * Create configured P2PKH template
   */
  static createP2PKH(options?: {
    signOutputs?: 'all' | 'none' | 'single';
    anyoneCanPay?: boolean;
  }) {
    return {
      template: new P2PKH(),
      options: {
        signOutputs: options?.signOutputs || 'all',
        anyoneCanPay: options?.anyoneCanPay || false
      }
    };
  }

  /**
   * Create payment transaction with template
   */
  static async createPayment(
    from: {
      txid: string;
      vout: number;
      satoshis: number;
      privKey: PrivateKey
    },
    to: { address: string; satoshis: number },
    change: { address: string }
  ): Promise<Transaction> {
    const p2pkh = new P2PKH();
    const tx = new Transaction();

    // Add input with P2PKH unlocking
    tx.addInput({
      sourceTXID: from.txid,
      sourceOutputIndex: from.vout,
      sourceSatoshis: from.satoshis,
      unlockingScriptTemplate: p2pkh.unlock(from.privKey)
    });

    // Add payment output
    tx.addOutput({
      lockingScript: p2pkh.lock(to.address),
      satoshis: to.satoshis
    });

    // Add change output
    tx.addOutput({
      lockingScript: p2pkh.lock(change.address),
      change: true
    });

    await tx.fee();
    await tx.sign();

    return tx;
  }
}

// Usage
const payment = await TemplateFactory.createPayment(
  {
    txid: '4a5e...',
    vout: 0,
    satoshis: 100000,
    privKey: PrivateKey.fromWif('L5EY...')
  },
  { address: '1A1z...', satoshis: 50000 },
  { address: '1B2y...' }
);
```

### Pattern 2: Template Composition

Combine multiple templates:

```typescript
import { ScriptTemplate, Script, OP, Transaction } from '@bsv/sdk';

/**
 * Composite template combining multiple conditions
 */
class CompositeTemplate implements ScriptTemplate {
  constructor(
    private templates: ScriptTemplate[],
    private mode: 'AND' | 'OR' = 'AND'
  ) {}

  lock(...args: any[]): LockingScript {
    const scripts = this.templates.map((t, i) => t.lock(args[i]));

    if (this.mode === 'OR') {
      // IF condition1 ELSE IF condition2 ELSE ... ENDIF
      const composite = new Script();

      scripts.forEach((script, index) => {
        if (index > 0) composite.writeOpCode(OP.OP_ELSE);
        composite.writeOpCode(OP.OP_IF);
        scripts.forEach(chunk => composite.writeOpCode(chunk.op));
      });

      composite.writeOpCode(OP.OP_ENDIF);
      return composite;
    } else {
      // AND: All conditions must be met
      const composite = new Script();
      scripts.forEach(script => {
        script.chunks.forEach(chunk => {
          if (chunk.data) {
            composite.writeBin(chunk.data);
          } else {
            composite.writeOpCode(chunk.op);
          }
        });
      });
      return composite;
    }
  }

  unlock(...args: any[]): UnlockingScriptTemplate {
    // Implementation depends on composition logic
    throw new Error('Composite unlock not implemented');
  }
}

// Usage: Combine hash puzzle with time lock
const composite = new CompositeTemplate(
  [
    new HashPuzzle(),
    new TimeLock()
  ],
  'AND'
);
```

### Pattern 3: Protocol-Specific Templates

Create templates for specific protocols:

```typescript
import { ScriptTemplate, Script, OP, P2PKH } from '@bsv/sdk';

/**
 * BRC-20 Token Template (simplified example)
 */
class TokenTemplate {
  private p2pkh = new P2PKH();

  /**
   * Create token output with embedded token data
   */
  createTokenOutput(
    tokenId: Buffer,
    amount: bigint,
    ownerAddress: string,
    satoshis: number
  ): TransactionOutput {
    // Token metadata in OP_RETURN
    const tokenData = new Script()
      .writeOpCode(OP.OP_FALSE)
      .writeOpCode(OP.OP_RETURN)
      .writeBin(Buffer.from('TOKEN'))
      .writeBin(tokenId)
      .writeBin(Buffer.from(amount.toString()));

    // Standard P2PKH for ownership
    const ownershipScript = this.p2pkh.lock(ownerAddress);

    return {
      lockingScript: ownershipScript,
      satoshis
    };
  }

  /**
   * Create token transfer transaction
   */
  async transferToken(
    tokenUTXO: {
      txid: string;
      vout: number;
      satoshis: number;
      tokenId: Buffer;
      amount: bigint;
    },
    fromPrivKey: PrivateKey,
    toAddress: string
  ): Promise<Transaction> {
    const tx = new Transaction();

    // Input: Spend token UTXO
    tx.addInput({
      sourceTXID: tokenUTXO.txid,
      sourceOutputIndex: tokenUTXO.vout,
      sourceSatoshis: tokenUTXO.satoshis,
      unlockingScriptTemplate: this.p2pkh.unlock(fromPrivKey)
    });

    // Output: Token to new owner
    tx.addOutput(
      this.createTokenOutput(
        tokenUTXO.tokenId,
        tokenUTXO.amount,
        toAddress,
        tokenUTXO.satoshis
      )
    );

    await tx.fee();
    await tx.sign();

    return tx;
  }
}

// Usage
const tokenTemplate = new TokenTemplate();
const tokenId = Buffer.from('TOKEN_ID_123');

// Create token
const output = tokenTemplate.createTokenOutput(
  tokenId,
  1000n,
  '1A1z...',
  1000
);

// Transfer token
const transferTx = await tokenTemplate.transferToken(
  {
    txid: '4a5e...',
    vout: 0,
    satoshis: 1000,
    tokenId,
    amount: 1000n
  },
  PrivateKey.fromWif('L5EY...'),
  '1B2y...'
);
```

## Security Considerations

1. **Template Validation**: Always validate template parameters before using in production.

2. **Sighash Type Selection**: Choose appropriate sighash types for your use case. SIGHASH_ALL is most secure.

3. **Private Key Protection**: Never log or expose private keys used in unlock templates.

4. **Custom Template Testing**: Thoroughly test custom templates with small amounts before production use.

5. **Script Size Limits**: Ensure generated scripts stay within protocol limits.

6. **Standard Compliance**: Use standard templates (P2PKH) when possible for maximum compatibility.

## Performance Considerations

1. **Template Reuse**: Create template instances once and reuse them across multiple transactions.

2. **Unlocking Script Size**: Simpler unlock templates result in smaller transactions and lower fees.

3. **Estimation Accuracy**: Accurate `estimateLength` implementations help with fee calculation.

4. **Signature Caching**: For repeated signatures, consider caching when safe to do so.

## Related Components

- [Transaction](../transaction/README.md) - Use templates in transactions
- [Script](../script/README.md) - Low-level script building
- [TransactionInput](../transaction-input/README.md) - Use unlock templates
- [TransactionOutput](../transaction-output/README.md) - Use lock templates
- [Signature](../signatures/README.md) - Generate signatures for unlocking

## Code Examples

See complete working examples in:
- [Standard Transactions](../../code-features/standard-transactions/README.md)
- [Custom Templates](../../code-features/custom-templates/README.md)
- [Multi-Signature](../../code-features/multi-signature/README.md)
- [Smart Contracts](../../code-features/smart-contracts/README.md)

## Best Practices

1. **Use standard templates** (P2PKH) for maximum compatibility
2. **Test custom templates thoroughly** before production use
3. **Document template parameters** clearly for maintainability
4. **Implement proper error handling** in unlock functions
5. **Provide accurate length estimates** for fee calculation
6. **Follow BRC standards** for protocol-specific templates
7. **Keep templates simple** to minimize errors and fees
8. **Reuse template instances** across transactions
9. **Validate all inputs** before script generation
10. **Include comprehensive unit tests** for custom templates

## Troubleshooting

### Issue: Template unlock fails during signing

**Solution**: Verify template parameters match the locking script.

```typescript
// Ensure unlock template matches lock type
const p2pkh = new P2PKH();
const lockingScript = p2pkh.lock(address);

// Use same template for unlocking
const unlockingTemplate = p2pkh.unlock(privKey);
```

### Issue: Custom template generates invalid script

**Solution**: Validate script structure and opcodes.

```typescript
class CustomTemplate implements ScriptTemplate {
  lock(data: Buffer): LockingScript {
    const script = new Script()
      .writeOpCode(OP.OP_HASH256)
      .writeBin(data)
      .writeOpCode(OP.OP_EQUAL);

    // Validate script
    const asm = script.toASM();
    console.log('Generated script:', asm);

    return script;
  }
}
```

### Issue: Unlock template estimation inaccurate

**Solution**: Implement precise length estimation.

```typescript
estimateLength: async (tx: Transaction, inputIndex: number) => {
  // Account for all elements
  const sigLength = 73; // Max DER signature length
  const pubKeyLength = 33; // Compressed public key
  const pushOpcodes = 2; // OP_PUSHDATA opcodes

  return sigLength + pubKeyLength + pushOpcodes;
}
```

### Issue: Template not working with BRC protocol

**Solution**: Ensure compliance with BRC specifications.

```typescript
// Follow BRC-29 for simple payment protocol
class BRC29Template extends P2PKH {
  // Implement BRC-29 specific features
}
```

## Further Reading

- [Script Templates](https://wiki.bitcoinsv.io/index.php/Script#Standard_scripts) - Standard Bitcoin scripts
- [P2PKH](https://wiki.bitcoinsv.io/index.php/Pay_to_Public_Key_Hash) - Pay to public key hash
- [BRC-29](https://github.com/bitcoin-sv/BRCs/blob/master/payments/0029.md) - Simple payment protocol
- [Custom Scripts](https://wiki.bitcoinsv.io/index.php/Script_examples) - Script examples
- [BSV SDK Documentation](https://bsv-blockchain.github.io/ts-sdk) - Official SDK docs

## Status

✅ Complete
