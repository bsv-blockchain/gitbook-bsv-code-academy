# Script

## Overview

The `Script` class in the BSV TypeScript SDK provides a comprehensive implementation of Bitcoin Script, the stack-based programming language used to define spending conditions for Bitcoin transactions. It enables creation, manipulation, parsing, and execution of Bitcoin scripts with support for all opcodes, script templates, and advanced scripting patterns.

## Purpose

- Create and parse Bitcoin scripts (locking and unlocking scripts)
- Execute Bitcoin Script operations with a stack-based virtual machine
- Support all Bitcoin opcodes including arithmetic, cryptographic, and flow control operations
- Serialize scripts to binary, hexadecimal, and ASM (assembly) formats
- Parse scripts from various formats (hex, binary, ASM)
- Build custom spending conditions and smart contracts
- Implement standard script templates (P2PKH, P2PK, OP_RETURN, etc.)

## Basic Usage

```typescript
import { Script, OP } from '@bsv/sdk';

// Create a simple script using opcodes
const script = new Script()
  .writeOpCode(OP.OP_DUP)
  .writeOpCode(OP.OP_HASH160)
  .writeBin(Buffer.from('89abcdefabbaabbaabbaabbaabbaabbaabbaabba', 'hex'))
  .writeOpCode(OP.OP_EQUALVERIFY)
  .writeOpCode(OP.OP_CHECKSIG);

// Serialize to hex
console.log('Script hex:', script.toHex());

// Parse from hex
const parsedScript = Script.fromHex('76a91489abcdefabbaabbaabbaabbaabbaabbaabbaabba88ac');

// Convert to ASM (assembly format)
console.log('Script ASM:', parsedScript.toASM());
// Output: "OP_DUP OP_HASH160 89abcdefabbaabbaabbaabbaabbaabbaabbaabba OP_EQUALVERIFY OP_CHECKSIG"

// Get script binary
const binary = script.toBinary();
console.log('Script length:', binary.length);
```

## Key Features

### 1. Script Construction with Opcodes

Build scripts using the complete set of Bitcoin opcodes:

```typescript
import { Script, OP } from '@bsv/sdk';

// Arithmetic script: Add two numbers
const arithmeticScript = new Script()
  .writeOpCode(OP.OP_2)      // Push 2 onto stack
  .writeOpCode(OP.OP_3)      // Push 3 onto stack
  .writeOpCode(OP.OP_ADD)    // Add top two stack items
  .writeOpCode(OP.OP_5)      // Push 5 onto stack
  .writeOpCode(OP.OP_EQUAL); // Check if equal

// Data script: OP_RETURN for storing data
const dataScript = new Script()
  .writeOpCode(OP.OP_FALSE)
  .writeOpCode(OP.OP_RETURN)
  .writeBin(Buffer.from('Hello BSV'))
  .writeBin(Buffer.from('Protocol ID: BSV20'));

// P2PKH locking script (pay to public key hash)
const pubKeyHash = Buffer.from('89abcdefabbaabbaabbaabbaabbaabbaabbaabba', 'hex');
const p2pkhLockingScript = new Script()
  .writeOpCode(OP.OP_DUP)
  .writeOpCode(OP.OP_HASH160)
  .writeBin(pubKeyHash)
  .writeOpCode(OP.OP_EQUALVERIFY)
  .writeOpCode(OP.OP_CHECKSIG);

// Conditional script: IF/ELSE/ENDIF
const conditionalScript = new Script()
  .writeOpCode(OP.OP_IF)
    .writeBin(Buffer.from('Path A'))
    .writeOpCode(OP.OP_DROP)
    .writeOpCode(OP.OP_1)
  .writeOpCode(OP.OP_ELSE)
    .writeBin(Buffer.from('Path B'))
    .writeOpCode(OP.OP_DROP)
    .writeOpCode(OP.OP_0)
  .writeOpCode(OP.OP_ENDIF);

// Multi-signature script: 2-of-3
const pubKey1 = Buffer.from('02abc...', 'hex');
const pubKey2 = Buffer.from('03def...', 'hex');
const pubKey3 = Buffer.from('02123...', 'hex');

const multiSigScript = new Script()
  .writeOpCode(OP.OP_2)       // Require 2 signatures
  .writeBin(pubKey1)
  .writeBin(pubKey2)
  .writeBin(pubKey3)
  .writeOpCode(OP.OP_3)       // Out of 3 public keys
  .writeOpCode(OP.OP_CHECKMULTISIG);
```

### 2. Script Parsing and Serialization

Convert scripts between different formats:

```typescript
import { Script } from '@bsv/sdk';

// Parse from hexadecimal
const hexScript = '76a91489abcdefabbaabbaabbaabbaabbaabbaabbaabba88ac';
const fromHex = Script.fromHex(hexScript);

// Parse from binary
const binary = Buffer.from(hexScript, 'hex');
const fromBinary = Script.fromBinary(Array.from(binary));

// Parse from ASM (assembly format)
const asmScript = 'OP_DUP OP_HASH160 89abcdefabbaabbaabbaabbaabbaabbaabbaabba OP_EQUALVERIFY OP_CHECKSIG';
const fromASM = Script.fromASM(asmScript);

// Convert to different formats
console.log('Hex:', fromASM.toHex());
console.log('Binary:', fromASM.toBinary());
console.log('ASM:', fromASM.toASM());

// Parse OP_RETURN data
const opReturnScript = Script.fromHex('006a0b48656c6c6f20576f726c64');
const chunks = opReturnScript.chunks;
console.log('OP_RETURN data:', chunks[2].data?.toString('utf8')); // "Hello World"

// Serialize with human-readable output
const detailedASM = script.toASM();
console.log('Detailed script:', detailedASM);
```

### 3. Script Execution and Validation

Execute scripts with the Bitcoin Script interpreter:

```typescript
import { Script, OP, Interpreter } from '@bsv/sdk';

// Simple script execution
const unlockingScript = new Script()
  .writeOpCode(OP.OP_5);

const lockingScript = new Script()
  .writeOpCode(OP.OP_5)
  .writeOpCode(OP.OP_EQUAL);

// Execute combined script
const interpreter = new Interpreter();
const combinedScript = new Script()
  .fromBinary(unlockingScript.toBinary())
  .fromBinary(lockingScript.toBinary());

const result = interpreter.run(combinedScript);
console.log('Script valid:', result); // true

// More complex: Verify signature
import { PrivateKey, Signature, Hash, Transaction } from '@bsv/sdk';

const privKey = PrivateKey.fromRandom();
const pubKey = privKey.toPublicKey();
const message = Hash.sha256(Buffer.from('message'));
const signature = Signature.sign(Array.from(message), privKey);

// Unlocking script: <signature> <pubkey>
const sigUnlockingScript = new Script()
  .writeBin(signature.toDER())
  .writeBin(pubKey.encode(true));

// Locking script: OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG
const pubKeyHash = Hash.hash160(pubKey.encode(true));
const sigLockingScript = new Script()
  .writeOpCode(OP.OP_DUP)
  .writeOpCode(OP.OP_HASH160)
  .writeBin(Array.from(pubKeyHash))
  .writeOpCode(OP.OP_EQUALVERIFY)
  .writeOpCode(OP.OP_CHECKSIG);

// Validation happens during transaction verification
```

### 4. Advanced Script Patterns

Implement complex spending conditions:

```typescript
import { Script, OP } from '@bsv/sdk';

// Time-locked script (CheckLockTimeVerify)
const lockTime = 1700000000; // Unix timestamp
const timeLockScript = new Script()
  .writeBin(Buffer.from(lockTime.toString(16).padStart(8, '0'), 'hex'))
  .writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
  .writeOpCode(OP.OP_DROP)
  // Then require signature
  .writeOpCode(OP.OP_DUP)
  .writeOpCode(OP.OP_HASH160)
  .writeBin(Buffer.from('pubkeyhash', 'hex'))
  .writeOpCode(OP.OP_EQUALVERIFY)
  .writeOpCode(OP.OP_CHECKSIG);

// Hash puzzle script
const secretHash = Hash.sha256(Buffer.from('secret'));
const hashPuzzleScript = new Script()
  .writeOpCode(OP.OP_HASH256)
  .writeBin(Array.from(secretHash))
  .writeOpCode(OP.OP_EQUAL);

// Unlocking script for hash puzzle
const hashPuzzleUnlock = new Script()
  .writeBin(Buffer.from('secret'));

// Escrow script with multiple conditions
const arbiterPubKeyHash = Buffer.from('arbiter_hash', 'hex');
const buyerPubKeyHash = Buffer.from('buyer_hash', 'hex');
const sellerPubKeyHash = Buffer.from('seller_hash', 'hex');

const escrowScript = new Script()
  // Option 1: Arbiter decides
  .writeOpCode(OP.OP_IF)
    .writeOpCode(OP.OP_DUP)
    .writeOpCode(OP.OP_HASH160)
    .writeBin(arbiterPubKeyHash)
  .writeOpCode(OP.OP_ELSE)
    // Option 2: Both parties agree
    .writeOpCode(OP.OP_2)
    .writeOpCode(OP.OP_DUP)
    .writeOpCode(OP.OP_HASH160)
    .writeBin(buyerPubKeyHash)
    .writeOpCode(OP.OP_EQUALVERIFY)
    .writeOpCode(OP.OP_CHECKSIGVERIFY)
    .writeOpCode(OP.OP_DUP)
    .writeOpCode(OP.OP_HASH160)
    .writeBin(sellerPubKeyHash)
  .writeOpCode(OP.OP_ENDIF)
  .writeOpCode(OP.OP_EQUALVERIFY)
  .writeOpCode(OP.OP_CHECKSIG);

// R-puzzle script (for atomic swaps)
const rValue = Buffer.from('r_value_32_bytes_here_xxxxxxxxxxxx', 'hex');
const rPuzzleScript = new Script()
  .writeOpCode(OP.OP_OVER)           // Duplicate signature
  .writeOpCode(OP.OP_3)
  .writeOpCode(OP.OP_SPLIT)          // Split signature at byte 3
  .writeOpCode(OP.OP_NIP)            // Remove first 3 bytes
  .writeOpCode(OP.OP_1)
  .writeOpCode(OP.OP_SPLIT)          // Get R value from signature
  .writeOpCode(OP.OP_SWAP)
  .writeOpCode(OP.OP_SPLIT)
  .writeOpCode(OP.OP_DROP)
  .writeBin(rValue)
  .writeOpCode(OP.OP_EQUALVERIFY)    // Verify R value matches
  // Continue with normal P2PKH...
  .writeOpCode(OP.OP_DUP)
  .writeOpCode(OP.OP_HASH160)
  .writeBin(Buffer.from('pubkeyhash', 'hex'))
  .writeOpCode(OP.OP_EQUALVERIFY)
  .writeOpCode(OP.OP_CHECKSIG);
```

## API Reference

### Constructor

```typescript
constructor()
```

Creates a new empty Script instance.

### Static Methods

#### `Script.fromHex(hex: string): Script`

Parses a script from hexadecimal string.

**Parameters:**
- `hex: string` - Hexadecimal representation of the script

**Returns:** `Script` - The parsed script

**Example:**
```typescript
const script = Script.fromHex('76a91489abcd...88ac');
```

#### `Script.fromBinary(bin: number[]): Script`

Parses a script from binary array.

**Parameters:**
- `bin: number[]` - Binary representation of the script

**Returns:** `Script` - The parsed script

#### `Script.fromASM(asm: string): Script`

Parses a script from ASM (assembly) string.

**Parameters:**
- `asm: string` - ASM representation (e.g., "OP_DUP OP_HASH160 ... OP_CHECKSIG")

**Returns:** `Script` - The parsed script

**Example:**
```typescript
const script = Script.fromASM('OP_DUP OP_HASH160 89abcd OP_EQUALVERIFY OP_CHECKSIG');
```

### Instance Methods

#### `writeOpCode(opcode: number): Script`

Appends an opcode to the script.

**Parameters:**
- `opcode: number` - The opcode to append (from OP enum)

**Returns:** `Script` - The script instance (for chaining)

**Example:**
```typescript
script.writeOpCode(OP.OP_DUP).writeOpCode(OP.OP_HASH160);
```

#### `writeBin(data: number[] | Buffer): Script`

Appends data to the script with appropriate push operation.

**Parameters:**
- `data: number[] | Buffer` - The data to append

**Returns:** `Script` - The script instance (for chaining)

**Example:**
```typescript
script.writeBin(Buffer.from('Hello BSV'));
```

#### `writeNumber(num: number): Script`

Appends a number to the script.

**Parameters:**
- `num: number` - The number to append

**Returns:** `Script` - The script instance (for chaining)

**Example:**
```typescript
script.writeNumber(42);
```

#### `toHex(): string`

Converts the script to hexadecimal string.

**Returns:** `string` - Hexadecimal representation

#### `toBinary(): number[]`

Converts the script to binary array.

**Returns:** `number[]` - Binary representation

#### `toASM(): string`

Converts the script to ASM (assembly) format.

**Returns:** `string` - Human-readable ASM representation

**Example:**
```typescript
const asm = script.toASM();
console.log(asm); // "OP_DUP OP_HASH160 89abcd OP_EQUALVERIFY OP_CHECKSIG"
```

#### `isPublicKeyHashOutput(): boolean`

Checks if the script is a standard P2PKH output.

**Returns:** `boolean` - True if P2PKH format

#### `isPublicKeyHashInput(): boolean`

Checks if the script is a standard P2PKH input.

**Returns:** `boolean` - True if P2PKH input format

#### `isPushOnly(): boolean`

Checks if the script contains only data push operations.

**Returns:** `boolean` - True if push-only

### Instance Properties

```typescript
script.chunks    // ScriptChunk[] - Array of script operations
script.length    // number - Length of serialized script in bytes
```

### Script Chunk Structure

```typescript
interface ScriptChunk {
  op: number;           // Opcode value
  data?: number[];      // Data if this is a push operation
}
```

## Common Patterns

### Pattern 1: Creating Standard Transaction Scripts

Build standard Bitcoin transaction scripts:

```typescript
import { Script, OP, Hash, PrivateKey } from '@bsv/sdk';

class StandardScripts {
  /**
   * Create P2PKH (Pay to Public Key Hash) locking script
   */
  static createP2PKHLockingScript(address: string): Script {
    // Decode address to get pubkey hash
    // (simplified - actual implementation needs base58 decoding)
    const pubKeyHash = Buffer.from(address, 'hex');

    return new Script()
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(pubKeyHash)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG);
  }

  /**
   * Create P2PKH unlocking script
   */
  static createP2PKHUnlockingScript(
    signature: Buffer,
    publicKey: Buffer
  ): Script {
    return new Script()
      .writeBin(signature)
      .writeBin(publicKey);
  }

  /**
   * Create OP_RETURN data output
   */
  static createOpReturnScript(data: Buffer[]): Script {
    const script = new Script()
      .writeOpCode(OP.OP_FALSE)
      .writeOpCode(OP.OP_RETURN);

    data.forEach(chunk => script.writeBin(chunk));

    return script;
  }

  /**
   * Create P2PK (Pay to Public Key) locking script
   */
  static createP2PKLockingScript(publicKey: Buffer): Script {
    return new Script()
      .writeBin(publicKey)
      .writeOpCode(OP.OP_CHECKSIG);
  }

  /**
   * Create multisig locking script (M-of-N)
   */
  static createMultiSigScript(
    requiredSigs: number,
    publicKeys: Buffer[]
  ): Script {
    const script = new Script()
      .writeNumber(requiredSigs);

    publicKeys.forEach(pubKey => script.writeBin(pubKey));

    return script
      .writeNumber(publicKeys.length)
      .writeOpCode(OP.OP_CHECKMULTISIG);
  }
}

// Usage
const address = '89abcdefabbaabbaabbaabbaabbaabbaabbaabba';
const lockingScript = StandardScripts.createP2PKHLockingScript(address);
console.log('P2PKH script:', lockingScript.toASM());

// Create OP_RETURN with protocol data
const opReturn = StandardScripts.createOpReturnScript([
  Buffer.from('19HxigV4QyBv3tHpQVcUEQyq1pzZVdoAut'), // B:// protocol
  Buffer.from('image/png', 'utf8'),
  Buffer.from('filename.png', 'utf8'),
  Buffer.from([0x89, 0x50, 0x4e, 0x47]) // PNG header
]);
console.log('OP_RETURN hex:', opReturn.toHex());
```

### Pattern 2: Script Templates for Smart Contracts

Create reusable script templates:

```typescript
import { Script, OP } from '@bsv/sdk';

class SmartContractTemplates {
  /**
   * Hash Time Locked Contract (HTLC)
   */
  static createHTLC(
    payeeHash: Buffer,
    payerHash: Buffer,
    secretHash: Buffer,
    lockTime: number
  ): Script {
    return new Script()
      // If secret is revealed
      .writeOpCode(OP.OP_IF)
        .writeOpCode(OP.OP_HASH256)
        .writeBin(secretHash)
        .writeOpCode(OP.OP_EQUALVERIFY)
        .writeOpCode(OP.OP_DUP)
        .writeOpCode(OP.OP_HASH160)
        .writeBin(payeeHash)
      .writeOpCode(OP.OP_ELSE)
        // Or if locktime expires
        .writeBin(Buffer.from(lockTime.toString(16), 'hex'))
        .writeOpCode(OP.OP_CHECKLOCKTIMEVERIFY)
        .writeOpCode(OP.OP_DROP)
        .writeOpCode(OP.OP_DUP)
        .writeOpCode(OP.OP_HASH160)
        .writeBin(payerHash)
      .writeOpCode(OP.OP_ENDIF)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG);
  }

  /**
   * Token-colored coins script
   */
  static createTokenScript(
    tokenId: Buffer,
    amount: number,
    ownerHash: Buffer
  ): Script {
    return new Script()
      // Token validation
      .writeBin(tokenId)
      .writeOpCode(OP.OP_DROP)
      .writeNumber(amount)
      .writeOpCode(OP.OP_DROP)
      // Standard P2PKH
      .writeOpCode(OP.OP_DUP)
      .writeOpCode(OP.OP_HASH160)
      .writeBin(ownerHash)
      .writeOpCode(OP.OP_EQUALVERIFY)
      .writeOpCode(OP.OP_CHECKSIG);
  }

  /**
   * Threshold signature script
   */
  static createThresholdScript(
    threshold: number,
    conditions: Script[]
  ): Script {
    const script = new Script();

    // Push each condition
    conditions.forEach(condition => {
      script.fromBinary(condition.toBinary());
      script.writeOpCode(OP.OP_IF);
      script.writeOpCode(OP.OP_1);
      script.writeOpCode(OP.OP_ENDIF);
    });

    // Sum the results
    for (let i = 1; i < conditions.length; i++) {
      script.writeOpCode(OP.OP_ADD);
    }

    // Check threshold
    script.writeNumber(threshold);
    script.writeOpCode(OP.OP_GREATERTHANOREQUAL);

    return script;
  }

  /**
   * Commit-reveal scheme
   */
  static createCommitScript(commitmentHash: Buffer): Script {
    return new Script()
      .writeOpCode(OP.OP_HASH256)
      .writeBin(commitmentHash)
      .writeOpCode(OP.OP_EQUAL);
  }

  static createRevealUnlockingScript(
    secret: Buffer,
    randomness: Buffer
  ): Script {
    return new Script()
      .writeBin(Buffer.concat([secret, randomness]));
  }
}

// Usage: Create HTLC
const htlc = SmartContractTemplates.createHTLC(
  Buffer.from('payee_hash', 'hex'),
  Buffer.from('payer_hash', 'hex'),
  Hash.sha256(Buffer.from('secret')),
  1700000000 // Lock time
);

console.log('HTLC script:', htlc.toASM());
console.log('HTLC size:', htlc.toBinary().length, 'bytes');
```

### Pattern 3: Script Analysis and Validation

Analyze and validate scripts:

```typescript
import { Script, OP } from '@bsv/sdk';

class ScriptAnalyzer {
  /**
   * Extract data from OP_RETURN script
   */
  static extractOpReturnData(script: Script): Buffer[] {
    const chunks = script.chunks;
    const data: Buffer[] = [];

    let inOpReturn = false;
    for (const chunk of chunks) {
      if (chunk.op === OP.OP_RETURN) {
        inOpReturn = true;
        continue;
      }
      if (inOpReturn && chunk.data) {
        data.push(Buffer.from(chunk.data));
      }
    }

    return data;
  }

  /**
   * Identify script type
   */
  static identifyScriptType(script: Script): string {
    const chunks = script.chunks;

    if (chunks.length === 0) {
      return 'empty';
    }

    // Check for OP_RETURN
    if (chunks.some(c => c.op === OP.OP_RETURN)) {
      return 'OP_RETURN';
    }

    // Check for P2PKH (25 bytes)
    if (chunks.length === 5 &&
        chunks[0].op === OP.OP_DUP &&
        chunks[1].op === OP.OP_HASH160 &&
        chunks[3].op === OP.OP_EQUALVERIFY &&
        chunks[4].op === OP.OP_CHECKSIG) {
      return 'P2PKH';
    }

    // Check for P2PK
    if (chunks.length === 2 &&
        chunks[1].op === OP.OP_CHECKSIG) {
      return 'P2PK';
    }

    // Check for multisig
    if (chunks[chunks.length - 1].op === OP.OP_CHECKMULTISIG) {
      return 'MULTISIG';
    }

    return 'custom';
  }

  /**
   * Calculate script complexity
   */
  static calculateComplexity(script: Script): {
    opcodeCount: number;
    dataSize: number;
    hasBranching: boolean;
    hasCrypto: boolean;
  } {
    const chunks = script.chunks;
    let opcodeCount = 0;
    let dataSize = 0;
    let hasBranching = false;
    let hasCrypto = false;

    for (const chunk of chunks) {
      if (chunk.data) {
        dataSize += chunk.data.length;
      } else {
        opcodeCount++;

        // Check for branching opcodes
        if ([OP.OP_IF, OP.OP_NOTIF, OP.OP_ELSE, OP.OP_ENDIF].includes(chunk.op)) {
          hasBranching = true;
        }

        // Check for crypto opcodes
        if ([
          OP.OP_RIPEMD160,
          OP.OP_SHA1,
          OP.OP_SHA256,
          OP.OP_HASH160,
          OP.OP_HASH256,
          OP.OP_CHECKSIG,
          OP.OP_CHECKSIGVERIFY,
          OP.OP_CHECKMULTISIG,
          OP.OP_CHECKMULTISIGVERIFY
        ].includes(chunk.op)) {
          hasCrypto = true;
        }
      }
    }

    return { opcodeCount, dataSize, hasBranching, hasCrypto };
  }

  /**
   * Validate script format
   */
  static validateScript(script: Script): {
    valid: boolean;
    errors: string[];
  } {
    const errors: string[] = [];
    const chunks = script.chunks;

    // Check size limit (10,000 bytes in BSV)
    if (script.toBinary().length > 10000) {
      errors.push('Script exceeds maximum size');
    }

    // Check for invalid opcodes
    const disabledOps = [
      OP.OP_CAT, OP.OP_SUBSTR, OP.OP_LEFT, OP.OP_RIGHT,
      OP.OP_INVERT, OP.OP_AND, OP.OP_OR, OP.OP_XOR,
      OP.OP_2MUL, OP.OP_2DIV, OP.OP_MUL, OP.OP_DIV, OP.OP_MOD,
      OP.OP_LSHIFT, OP.OP_RSHIFT
    ];

    for (const chunk of chunks) {
      if (disabledOps.includes(chunk.op)) {
        errors.push(`Disabled opcode: ${chunk.op}`);
      }
    }

    return {
      valid: errors.length === 0,
      errors
    };
  }
}

// Usage
const script = Script.fromASM('OP_DUP OP_HASH160 89abcd OP_EQUALVERIFY OP_CHECKSIG');

const scriptType = ScriptAnalyzer.identifyScriptType(script);
console.log('Script type:', scriptType); // "P2PKH"

const complexity = ScriptAnalyzer.calculateComplexity(script);
console.log('Complexity:', complexity);

const validation = ScriptAnalyzer.validateScript(script);
console.log('Valid:', validation.valid);
```

## Security Considerations

1. **Script Size Limits**: BSV allows larger scripts than BTC, but validate reasonable limits for your use case.

2. **Disabled Opcodes**: Some opcodes were disabled in early Bitcoin but are being re-enabled in BSV. Verify which opcodes are available on your target network.

3. **Stack Size**: The maximum stack size is 1,000 elements. Design scripts that stay within this limit.

4. **Push Data Size**: Individual data pushes are limited to 520 bytes in standard transactions.

5. **Script Validation**: Always validate scripts before broadcasting to prevent rejection and fee loss.

6. **OP_RETURN Data**: While BSV allows large OP_RETURN outputs, miners may have different policies. Verify with your mining pool.

## Performance Considerations

1. **Script Complexity**: More complex scripts require more computation. Keep scripts as simple as possible.

2. **Data Size**: Large data outputs increase transaction size and fees. Consider using overlay networks for large data.

3. **Branching**: IF/ELSE statements add complexity. Minimize branching when possible.

4. **Push Operations**: Multiple small pushes are less efficient than fewer larger pushes.

5. **Parsing Overhead**: ASM parsing is slower than binary parsing. Use binary format when performance matters.

## Related Components

- [Transaction](../transaction/README.md) - Use scripts in transaction inputs and outputs
- [ScriptTemplate](../script-templates/README.md) - Pre-built script templates
- [Signature](../signatures/README.md) - Create signatures for scripts
- [TransactionInput](../transaction-input/README.md) - Scripts in transaction inputs
- [TransactionOutput](../transaction-output/README.md) - Scripts in transaction outputs

## Code Examples

See complete working examples in:
- [Custom Scripts](../../code-features/custom-scripts/README.md)
- [Smart Contracts](../../code-features/smart-contracts/README.md)
- [OP_RETURN Data](../../code-features/op-return/README.md)
- [Multi-Signature](../../code-features/multi-signature/README.md)

## Best Practices

1. **Use standard templates** when possible (P2PKH, P2PK) for better compatibility
2. **Test scripts thoroughly** before deploying with real funds
3. **Validate script size** before broadcasting transactions
4. **Use OP_RETURN** for data storage, not other opcodes
5. **Document custom scripts** with clear comments explaining logic
6. **Minimize script complexity** to reduce validation time and potential bugs
7. **Use ASM format** for human readability during development
8. **Store scripts as hex** in production for efficiency
9. **Implement proper error handling** for script parsing and execution
10. **Follow BRC standards** for script templates and protocols

## Troubleshooting

### Issue: Script parsing fails

**Solution**: Ensure the hex/binary format is valid and complete.

```typescript
try {
  const script = Script.fromHex(hexString);
} catch (error) {
  console.error('Invalid script hex:', error);
  // Validate hex string is proper format
}
```

### Issue: Script too large

**Solution**: Optimize script or split data across multiple outputs.

```typescript
const scriptSize = script.toBinary().length;
if (scriptSize > 10000) {
  console.warn('Script exceeds recommended size');
  // Consider using overlay network for data storage
}
```

### Issue: OP_RETURN data not extracted correctly

**Solution**: Ensure proper parsing of OP_RETURN chunks.

```typescript
const opReturnData = ScriptAnalyzer.extractOpReturnData(script);
if (opReturnData.length === 0) {
  console.log('No OP_RETURN data found');
}
```

### Issue: Script execution fails

**Solution**: Verify all required data is on the stack and opcodes are valid.

```typescript
// Ensure unlocking script provides all required data
const unlockingScript = new Script()
  .writeBin(signature)
  .writeBin(publicKey);

// Verify script is standard format
if (!lockingScript.isPublicKeyHashOutput()) {
  console.warn('Non-standard locking script');
}
```

## Further Reading

- [Bitcoin Script](https://wiki.bitcoinsv.io/index.php/Script) - Complete Bitcoin Script documentation
- [Opcodes](https://wiki.bitcoinsv.io/index.php/Opcodes_used_in_Bitcoin_Script) - All Bitcoin Script opcodes
- [Script Examples](https://wiki.bitcoinsv.io/index.php/Script_examples) - Common script patterns
- [Genesis Upgrade](https://wiki.bitcoinsv.io/index.php/Genesis_upgrade) - Restored opcodes in BSV
- [BSV SDK Documentation](https://bsv-blockchain.github.io/ts-sdk) - Official SDK docs
- [OP_RETURN Protocol](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0008.md) - BRC-8: Transaction envelopes

## Status

✅ Complete
