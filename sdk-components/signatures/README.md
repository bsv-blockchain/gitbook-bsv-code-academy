# Signature

## Overview

The `Signature` class in the BSV TypeScript SDK provides comprehensive functionality for creating, managing, and verifying ECDSA (Elliptic Curve Digital Signature Algorithm) signatures used in Bitcoin transactions. It handles DER encoding/decoding, sighash type management, and signature verification operations that are fundamental to Bitcoin's security model.

## Purpose

- Create and verify ECDSA signatures for Bitcoin transactions
- Handle DER (Distinguished Encoding Rules) encoding and decoding of signatures
- Manage sighash types and flags for transaction signing
- Support various signature formats (DER, compact, raw R and S values)
- Provide cryptographic verification of signatures against public keys
- Enable custom signature operations for advanced Bitcoin Script use cases

## Basic Usage

```typescript
import { Signature, PrivateKey, PublicKey, Hash } from '@bsv/sdk';

// Sign a message hash with a private key
const privKey = PrivateKey.fromRandom();
const messageHash = Hash.sha256(Array.from(Buffer.from('Hello BSV')));

// Create signature
const signature = privKey.sign(messageHash);

// Verify signature
const pubKey = privKey.toPublicKey();
const isValid = signature.verify(messageHash, pubKey);
console.log('Signature valid:', isValid); // true

// Get DER encoding of signature
const derEncoded = signature.toDER();
console.log('DER:', Buffer.from(derEncoded).toString('hex'));
// Parse DER encoded signature
const parsedSig = Signature.fromDER(derEncoded);
console.log(
  'Parsed signature equals original:',
  Buffer.from(parsedSig.toDER()).equals(Buffer.from(signature.toDER()))
); // true
```

## Key Features

### 1. Creating Signatures with Sighash Types

Sighash types control which parts of a transaction are signed, enabling various transaction patterns:

```typescript
import { Signature, PrivateKey, Hash, SighashType } from '@bsv/sdk';

const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');
const txHash = Hash.sha256sha256(Buffer.from('transaction_data'));

// Sign with SIGHASH_ALL (default - signs all inputs and outputs)
const sigAll = Signature.sign(txHash, privKey, true); // ephemeralKey = true
sigAll.scope = SighashType.SIGHASH_ALL;

// Sign with SIGHASH_NONE (signs inputs but no outputs)
const sigNone = Signature.sign(txHash, privKey);
sigNone.scope = SighashType.SIGHASH_NONE;

// Sign with SIGHASH_SINGLE (signs one input and corresponding output)
const sigSingle = Signature.sign(txHash, privKey);
sigSingle.scope = SighashType.SIGHASH_SINGLE;

// Sign with SIGHASH_ANYONECANPAY flag (allows others to add inputs)
const sigAnyoneCanPay = Signature.sign(txHash, privKey);
sigAnyoneCanPay.scope = SighashType.SIGHASH_ALL | SighashType.SIGHASH_ANYONECANPAY;

// Get signature with sighash byte appended (for Bitcoin Script)
const sigForScript = sigAll.toChecksigFormat();
console.log('Signature for checksig:', sigForScript.toString('hex'));
```

### 2. DER Encoding and Decoding

DER encoding is the standard format for signatures in Bitcoin transactions:

```typescript
import { Signature, PrivateKey, Hash } from '@bsv/sdk';

const privKey = PrivateKey.fromRandom();
const hash = Hash.sha256(Buffer.from('message'));

// Create signature
const signature = Signature.sign(hash, privKey);

// Convert to DER format (ASN.1 encoding)
const derBytes = signature.toDER();
console.log('DER length:', derBytes.length); // Typically 70-72 bytes

// Parse from DER
const fromDER = Signature.fromDER(derBytes);
console.log('R:', fromDER.r.toString(16));
console.log('S:', fromDER.s.toString(16));

// Convert to compact format (64 bytes)
const compactFormat = signature.toCompact();
console.log('Compact length:', compactFormat.length); // Always 64 bytes

// Parse from compact format
const fromCompact = Signature.fromCompact(compactFormat);
```

### 3. Low-S Signature Normalization

Bitcoin requires signatures to use low-S values (S <= N/2) to prevent transaction malleability:

```typescript
import { Signature, PrivateKey, Hash } from '@bsv/sdk';

const privKey = PrivateKey.fromRandom();
const hash = Hash.sha256(Buffer.from('data'));

// Create signature (SDK automatically uses low-S)
const signature = Signature.sign(hash, privKey);

// Check if S value is low
const curveN = BigInt('0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141');
const isLowS = signature.s <= (curveN / 2n);
console.log('Is low-S:', isLowS); // true

// Manually normalize to low-S if needed
if (signature.s > (curveN / 2n)) {
  signature.s = curveN - signature.s;
}

// Verify the normalized signature still works
const pubKey = privKey.toPublicKey();
const isValid = signature.verify(hash, pubKey);
console.log('Valid after normalization:', isValid); // true
```

### 4. Signature Recovery and Verification

Signatures can be verified against public keys, and public keys can be recovered from signatures:

```typescript
import { Signature, PrivateKey, PublicKey, Hash } from '@bsv/sdk';

const privKey = PrivateKey.fromRandom();
const pubKey = privKey.toPublicKey();
const message = Buffer.from('Important message');
const hash = Hash.sha256(message);

// Create signature with recovery information
const signature = Signature.sign(hash, privKey, true);

// Verify signature
const isValid = signature.verify(hash, pubKey);
console.log('Signature is valid:', isValid); // true

// Verify with wrong public key
const wrongPubKey = PrivateKey.fromRandom().toPublicKey();
const isInvalid = signature.verify(hash, wrongPubKey);
console.log('Wrong key validation:', isInvalid); // false

// Verify with wrong hash
const wrongHash = Hash.sha256(Buffer.from('Wrong message'));
const isWrongHash = signature.verify(wrongHash, pubKey);
console.log('Wrong hash validation:', isWrongHash); // false
```

## API Reference

### Constructor

```typescript
constructor(r: bigint, s: bigint, recoveryParam?: number)
```

Creates a new Signature instance with R and S values.

**Parameters:**
- `r: bigint` - The R component of the ECDSA signature
- `s: bigint` - The S component of the ECDSA signature
- `recoveryParam?: number` - Optional recovery parameter (0-3) for public key recovery

### Static Methods

#### `Signature.sign(hash: number[], privKey: PrivateKey, ephemeralKey?: boolean): Signature`

Creates a signature for a given hash using a private key.

**Parameters:**
- `hash: number[]` - The hash to sign (typically 32 bytes)
- `privKey: PrivateKey` - The private key to sign with
- `ephemeralKey?: boolean` - Whether to use deterministic (RFC 6979) or random k value

**Returns:** `Signature` - The created signature

**Example:**
```typescript
const signature = Signature.sign(hashArray, privateKey, true);
```

#### `Signature.fromDER(der: number[] | Buffer): Signature`

Parses a DER-encoded signature.

**Parameters:**
- `der: number[] | Buffer` - The DER-encoded signature bytes

**Returns:** `Signature` - The parsed signature

#### `Signature.fromCompact(compact: number[] | Buffer): Signature`

Parses a compact 64-byte signature format.

**Parameters:**
- `compact: number[] | Buffer` - The compact signature (32 bytes R + 32 bytes S)

**Returns:** `Signature` - The parsed signature

### Instance Methods

#### `verify(hash: number[], pubKey: PublicKey): boolean`

Verifies the signature against a hash and public key.

**Parameters:**
- `hash: number[]` - The hash that was signed
- `pubKey: PublicKey` - The public key to verify against

**Returns:** `boolean` - True if signature is valid

#### `toDER(): number[]`

Converts the signature to DER encoding.

**Returns:** `number[]` - DER-encoded signature bytes

#### `toCompact(): number[]`

Converts the signature to compact 64-byte format.

**Returns:** `number[]` - Compact signature (32 bytes R + 32 bytes S)

#### `toChecksigFormat(): number[]`

Converts signature to format used in Bitcoin Script CHECKSIG operations (DER + sighash byte).

**Returns:** `number[]` - DER signature with sighash type appended

### Instance Properties

```typescript
signature.r        // bigint - The R component
signature.s        // bigint - The S component
signature.scope    // number - Sighash type (SIGHASH_ALL, SIGHASH_NONE, etc.)
signature.recoveryParam  // number | undefined - Recovery parameter for pubkey recovery
```

## Common Patterns

### Pattern 1: Transaction Signature Creation

Creating signatures for Bitcoin transaction inputs:

```typescript
import {
  Transaction,
  PrivateKey,
  PublicKey,
  Signature,
  Hash,
  SighashType
} from '@bsv/sdk';

// Transaction signing workflow
const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');
const pubKey = privKey.toPublicKey();

// Create transaction
const tx = new Transaction();
// ... add inputs and outputs ...

// Get sighash preimage for input 0
const inputIndex = 0;
const sighashType = SighashType.SIGHASH_ALL | SighashType.SIGHASH_FORKID;

// Calculate transaction hash to sign
const preimage = tx.getPreimage(inputIndex, sighashType);
const txHash = Hash.sha256sha256(preimage);

// Create signature
const signature = Signature.sign(txHash, privKey, true);
signature.scope = sighashType;

// Convert to format for unlocking script
const sigBytes = signature.toChecksigFormat();

// Create unlocking script (scriptSig)
import { Script, OP } from '@bsv/sdk';
const unlockingScript = new Script()
  .writeBin(sigBytes)
  .writeBin(pubKey.encode(true)); // compressed public key

console.log('Unlocking script created:', unlockingScript.toHex());
```

### Pattern 2: Message Signing and Verification

Signing arbitrary messages for proof of ownership:

```typescript
import { PrivateKey, PublicKey, Signature, Hash } from '@bsv/sdk';

// Message signing
class MessageSigner {
  static sign(message: string, privKey: PrivateKey): string {
    // Create Bitcoin Signed Message format
    const prefix = 'Bitcoin Signed Message:\n';
    const messageBuffer = Buffer.concat([
      Buffer.from(prefix, 'utf8'),
      Buffer.from([message.length]),
      Buffer.from(message, 'utf8')
    ]);

    // Hash the message
    const hash = Hash.sha256sha256(Array.from(messageBuffer));

    // Sign with recovery parameter for address recovery
    const signature = Signature.sign(hash, privKey, true);

    // Return base64 encoded signature
    const sigBytes = signature.toCompact();
    return Buffer.from(sigBytes).toString('base64');
  }

  static verify(
    message: string,
    signatureBase64: string,
    pubKey: PublicKey
  ): boolean {
    // Reconstruct message hash
    const prefix = 'Bitcoin Signed Message:\n';
    const messageBuffer = Buffer.concat([
      Buffer.from(prefix, 'utf8'),
      Buffer.from([message.length]),
      Buffer.from(message, 'utf8')
    ]);

    const hash = Hash.sha256sha256(Array.from(messageBuffer));

    // Parse signature
    const sigBytes = Buffer.from(signatureBase64, 'base64');
    const signature = Signature.fromCompact(Array.from(sigBytes));

    // Verify
    return signature.verify(hash, pubKey);
  }
}

// Usage
const privKey = PrivateKey.fromRandom();
const pubKey = privKey.toPublicKey();
const message = 'I own this Bitcoin address';

const signature = MessageSigner.sign(message, privKey);
console.log('Signature:', signature);

const isValid = MessageSigner.verify(message, signature, pubKey);
console.log('Valid:', isValid); // true
```

### Pattern 3: Multi-Signature Verification

Verifying multiple signatures for multi-sig transactions:

```typescript
import { Signature, PublicKey, Hash } from '@bsv/sdk';

class MultiSigVerifier {
  /**
   * Verify M-of-N multisig signatures
   */
  static verifyMultiSig(
    hash: number[],
    signatures: Signature[],
    publicKeys: PublicKey[],
    requiredSigs: number
  ): boolean {
    if (signatures.length < requiredSigs) {
      return false;
    }

    let validCount = 0;
    let sigIndex = 0;
    let pubKeyIndex = 0;

    // Check signatures against public keys in order
    while (sigIndex < signatures.length && pubKeyIndex < publicKeys.length) {
      const sig = signatures[sigIndex];
      const pubKey = publicKeys[pubKeyIndex];

      if (sig.verify(hash, pubKey)) {
        validCount++;
        sigIndex++;
      }

      pubKeyIndex++;

      // Early exit if we have enough valid signatures
      if (validCount >= requiredSigs) {
        return true;
      }

      // Early exit if not enough pubkeys remain
      if (publicKeys.length - pubKeyIndex < requiredSigs - validCount) {
        return false;
      }
    }

    return validCount >= requiredSigs;
  }
}

// Usage: 2-of-3 multisig
const hash = Hash.sha256(Buffer.from('transaction data'));

// Create 3 key pairs
const keys = [1, 2, 3].map(() => ({
  priv: PrivateKey.fromRandom(),
  pub: PrivateKey.fromRandom().toPublicKey()
}));

// Sign with 2 of the 3 keys
const signatures = [
  Signature.sign(Array.from(hash), keys[0].priv),
  Signature.sign(Array.from(hash), keys[2].priv)
];

const publicKeys = keys.map(k => k.pub);

const isValid = MultiSigVerifier.verifyMultiSig(
  Array.from(hash),
  signatures,
  publicKeys,
  2 // Require 2 signatures
);

console.log('2-of-3 multisig valid:', isValid);
```

## Security Considerations

1. **Always Use Low-S Signatures**: The SDK automatically normalizes signatures to use low-S values to prevent transaction malleability. Never modify this behavior.

2. **Deterministic vs Random K Values**: Using `ephemeralKey: true` enables RFC 6979 deterministic signatures, which is more secure than random k values that could leak private keys if reused.

3. **Hash Before Signing**: Always hash your data before signing. Never sign raw data directly.

4. **Sighash Type Validation**: Ensure the correct sighash type is used for your use case. SIGHASH_ALL is the most secure default.

5. **Signature Malleability**: DER encoding prevents signature malleability. Always use DER format for transaction signatures.

6. **Private Key Protection**: Never expose private keys. Keep them in secure storage and use them only for signature generation.

## Performance Considerations

1. **Signature Verification**: ECDSA verification is computationally expensive. Cache verification results when possible.

2. **Batch Verification**: When verifying multiple signatures, consider implementing batch verification algorithms for better performance.

3. **DER Encoding Overhead**: DER encoding adds 6-8 bytes overhead compared to compact format. Use compact format for storage when sighash type is not needed.

4. **Deterministic Signatures**: RFC 6979 deterministic signatures are slightly slower than random k values but provide better security.

## Related Components

- [PrivateKey](../private-key/README.md) - Generate private keys for signing
- [PublicKey](../public-key/README.md) - Verify signatures and derive addresses
- [Transaction](../transaction/README.md) - Use signatures in transaction inputs
- [Script](../script/README.md) - Include signatures in Bitcoin Script
- [Hash](../hash/README.md) - Hash data before signing

## Code Examples

See complete working examples in:
- [Transaction Signing](../../code-features/transaction-signing/README.md)
- [Message Signing](../../code-features/message-signing/README.md)
- [Multi-Signature](../../code-features/multi-signature/README.md)
- [Custom Scripts](../../code-features/custom-scripts/README.md)

## Best Practices

1. **Always use deterministic signatures** (RFC 6979) by setting `ephemeralKey: true` in `Signature.sign()`
2. **Verify sighash types match** your transaction signing requirements before broadcasting
3. **Use low-S signatures** to prevent transaction malleability (automatic in SDK)
4. **Hash your data** with SHA-256 or SHA-256d before signing
5. **Include recovery parameters** when you need public key recovery from signatures
6. **Validate signatures** before accepting them in your application logic
7. **Use appropriate sighash flags** for your specific transaction pattern (ANYONECANPAY for crowdfunding, etc.)
8. **Store signatures in DER format** for Bitcoin transactions
9. **Never reuse signatures** across different messages or transactions
10. **Implement proper error handling** for signature verification failures

## Troubleshooting

### Issue: Signature verification fails

**Solution**: Ensure you're using the same hash that was signed and the correct public key.

```typescript
// Correct approach
const hash = Hash.sha256(data);
const signature = Signature.sign(hash, privKey);
const isValid = signature.verify(hash, privKey.toPublicKey()); // Same hash!
```

### Issue: Transaction signature invalid

**Solution**: Verify you're using the correct sighash type and preimage.

```typescript
// Ensure sighash type matches
signature.scope = SighashType.SIGHASH_ALL | SighashType.SIGHASH_FORKID;
const sigBytes = signature.toChecksigFormat(); // Includes sighash byte
```

### Issue: DER encoding error

**Solution**: Ensure R and S values are valid and properly normalized.

```typescript
// SDK handles this automatically, but if working with raw values:
const curveN = BigInt('0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141');
if (signature.s > curveN / 2n) {
  signature.s = curveN - signature.s; // Normalize to low-S
}
```

### Issue: Cannot recover public key from signature

**Solution**: Ensure the signature was created with a recovery parameter.

```typescript
// Create signature with recovery information
const signature = Signature.sign(hash, privKey, true);
console.log('Recovery param:', signature.recoveryParam); // Should be 0-3
```

## Further Reading

- [Bitcoin ECDSA Signatures](https://github.com/bitcoin-sv/BRCs/blob/master/key-derivation/0003.md) - BRC-3: Digital Signatures
- [DER Encoding](https://wiki.bitcoinsv.io/index.php/ECDSA) - BSV Wiki on ECDSA
- [Sighash Types](https://wiki.bitcoinsv.io/index.php/SIGHASH_flags) - Understanding signature hash types
- [RFC 6979](https://tools.ietf.org/html/rfc6979) - Deterministic ECDSA signatures
- [Transaction Malleability](https://wiki.bitcoinsv.io/index.php/Transaction_Malleability) - Why low-S matters
- [BSV SDK Documentation](https://bsv-blockchain.github.io/ts-sdk) - Official SDK docs

## Status

✅ Complete
