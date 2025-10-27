# HD Wallets (Hierarchical Deterministic Wallets)

## Overview

The HD Wallets component implements the BSV Key Derivation Scheme (BKDS) as defined in BRC-42 and BRC-43. It provides privacy-enhanced key derivation for generating unique keys per transaction, counterparty, and protocol while maintaining a single master key for recovery.

The `KeyDeriver` class is the core implementation that enables deterministic key generation using protocol IDs, key IDs, and counterparty information, ensuring that keys are never reused and maintaining privacy across different contexts.

## Purpose

HD Wallets in the BSV SDK solve several critical problems:

- **Privacy**: Generate unique keys for each transaction context without address reuse
- **Deterministic Recovery**: Recover all keys from a single master key using BIP-32 derivation
- **Protocol Isolation**: Separate keys by security level and protocol to prevent cross-contamination
- **Counterparty-Specific Keys**: Derive unique keys for each counterparty relationship
- **Symmetric Key Support**: Generate shared encryption keys for secure communications

This component is essential for building privacy-respecting BSV applications that comply with modern key management standards.

## Basic Usage

### Initialize Key Deriver

```typescript
import { KeyDeriver, PrivateKey } from '@bsv/sdk';

// Initialize with a master private key
const rootKey = PrivateKey.fromRandom();
const keyDeriver = new KeyDeriver(rootKey);

// Or use an existing master key
const masterKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');
const deriver = new KeyDeriver(masterKey);
```

### Derive Public Keys

```typescript
import { KeyDeriver, PrivateKey, PublicKey } from '@bsv/sdk';

const keyDeriver = new KeyDeriver(PrivateKey.fromRandom());

// Define protocol ID [securityLevel, protocolName]
const protocolID = [2, '3241645161d8'];
const keyID = 'invoice-42';
const counterparty = PublicKey.fromString('027a...'); // Or 'self' or 'anyone'

// Derive public key for this context
const derivedPubKey = keyDeriver.derivePublicKey(
  protocolID,
  keyID,
  counterparty,
  false // forSelf: false means for counterparty, true means for yourself
);

console.log('Derived public key:', derivedPubKey.toString());
```

### Derive Private Keys

```typescript
import { KeyDeriver, PrivateKey } from '@bsv/sdk';

const keyDeriver = new KeyDeriver(PrivateKey.fromRandom());

const protocolID = [2, 'payment-protocol'];
const keyID = 'tx-12345';
const counterparty = 'anyone'; // Special value for public contexts

// Derive private key (only for keys you control)
const derivedPrivKey = keyDeriver.derivePrivateKey(
  protocolID,
  keyID,
  counterparty
);

// Use the derived key for signing
const signature = derivedPrivKey.sign(messageHash);
```

### Derive Symmetric Keys

```typescript
import { KeyDeriver, PrivateKey, PublicKey } from '@bsv/sdk';

const keyDeriver = new KeyDeriver(PrivateKey.fromRandom());
const counterpartyPubKey = PublicKey.fromString('027a...');

const protocolID = [2, 'secure-messaging'];
const keyID = 'conversation-789';

// Derive symmetric key for encryption
const symmetricKey = keyDeriver.deriveSymmetricKey(
  protocolID,
  keyID,
  counterpartyPubKey
);

// Use for AES-256-GCM encryption (BRC-2)
console.log('Symmetric key:', Buffer.from(symmetricKey).toString('hex'));
```

## Key Features

### 1. BRC-42 Compliant Key Derivation

The SDK implements the complete BRC-42 specification for hierarchical deterministic key derivation:

```typescript
import { KeyDeriver, PrivateKey, PublicKey } from '@bsv/sdk';

const rootKey = PrivateKey.fromRandom();
const keyDeriver = new KeyDeriver(rootKey);

// Protocol ID structure: [securityLevel, protocolName]
// Security levels from BRC-43:
// - 0: No security (public data)
// - 1: Low security (semi-public)
// - 2: Standard security (most applications)
// - 3: High security (financial)
// - 4: Critical security (identity/custody)

const protocols = [
  [0, 'public-data'],      // No security
  [1, 'social-media'],     // Low security
  [2, 'e-commerce'],       // Standard security
  [3, 'banking'],          // High security
  [4, 'identity-system']   // Critical security
];

// Derive keys for different security contexts
protocols.forEach(([level, name]) => {
  const pubKey = keyDeriver.derivePublicKey(
    [level, name],
    'key-1',
    'anyone',
    false
  );
  console.log(`Security Level ${level} (${name}): ${pubKey.toString()}`);
});
```

### 2. Counterparty-Specific Derivation

Generate unique keys for each relationship or counterparty:

```typescript
import { KeyDeriver, PrivateKey, PublicKey } from '@bsv/sdk';

const keyDeriver = new KeyDeriver(PrivateKey.fromRandom());

// Different counterparty types
const specificCounterparty = PublicKey.fromString('027a...');
const selfCounterparty = 'self';
const anyoneCounterparty = 'anyone';

const protocolID = [2, 'payment-system'];
const invoiceID = 'invoice-001';

// Key for a specific counterparty (unique per relationship)
const key1 = keyDeriver.derivePublicKey(
  protocolID,
  invoiceID,
  specificCounterparty,
  false
);

// Key for yourself (self-payment or change)
const key2 = keyDeriver.derivePublicKey(
  protocolID,
  invoiceID,
  selfCounterparty,
  true
);

// Key for anyone (public context)
const key3 = keyDeriver.derivePublicKey(
  protocolID,
  invoiceID,
  anyoneCounterparty,
  false
);

console.log('Each key is unique:',
  key1.toString() !== key2.toString() &&
  key2.toString() !== key3.toString()
);
```

### 3. Protocol Isolation

Separate keys by protocol to prevent cross-contamination:

```typescript
import { KeyDeriver, PrivateKey } from '@bsv/sdk';

const keyDeriver = new KeyDeriver(PrivateKey.fromRandom());

// Different protocols maintain separate key spaces
const paymentProtocol = [2, 'payment-v1'];
const messagingProtocol = [2, 'messaging-v1'];
const storageProtocol = [1, 'storage-v1'];

const keyID = 'user-123';
const counterparty = 'anyone';

// Each protocol generates different keys even with same keyID
const paymentKey = keyDeriver.derivePublicKey(
  paymentProtocol,
  keyID,
  counterparty,
  false
);

const messagingKey = keyDeriver.derivePublicKey(
  messagingProtocol,
  keyID,
  counterparty,
  false
);

const storageKey = keyDeriver.derivePublicKey(
  storageProtocol,
  keyID,
  counterparty,
  false
);

console.log('Protocol isolation ensures different keys:',
  paymentKey.toString() !== messagingKey.toString() &&
  messagingKey.toString() !== storageKey.toString()
);
```

### 4. Master Key Recovery

Recover all derived keys from a single master key:

```typescript
import { KeyDeriver, PrivateKey } from '@bsv/sdk';

// Original wallet setup
const masterKey = PrivateKey.fromRandom();
const originalDeriver = new KeyDeriver(masterKey);

// Derive some keys
const protocolID = [2, 'wallet-v1'];
const keys = ['addr-1', 'addr-2', 'addr-3'].map(keyID =>
  originalDeriver.derivePublicKey(protocolID, keyID, 'anyone', false)
);

// Later, recover wallet from master key backup
const masterKeyWIF = masterKey.toWif();
// ... time passes, device lost, recovery needed ...

// Restore from WIF
const recoveredMaster = PrivateKey.fromWif(masterKeyWIF);
const recoveredDeriver = new KeyDeriver(recoveredMaster);

// Derive the same keys
const recoveredKeys = ['addr-1', 'addr-2', 'addr-3'].map(keyID =>
  recoveredDeriver.derivePublicKey(protocolID, keyID, 'anyone', false)
);

// Verify all keys match
const allMatch = keys.every((key, i) =>
  key.toString() === recoveredKeys[i].toString()
);

console.log('All keys recovered successfully:', allMatch);
```

## API Reference

### KeyDeriver Class

The main class for hierarchical deterministic key derivation.

#### Constructor

```typescript
constructor(rootKey: PrivateKey, identityKey?: string)
```

**Parameters:**
- `rootKey`: The master private key for all derivations
- `identityKey` (optional): Custom identity key for specialized derivation paths

#### derivePublicKey()

```typescript
derivePublicKey(
  protocolID: [number, string],
  keyID: string,
  counterparty: PublicKey | 'self' | 'anyone',
  forSelf: boolean
): PublicKey
```

Derives a public key for a specific protocol, key ID, and counterparty context.

**Parameters:**
- `protocolID`: Tuple of `[securityLevel, protocolName]`
  - `securityLevel`: 0-4 per BRC-43
  - `protocolName`: String identifier for the protocol
- `keyID`: Unique identifier within the protocol context
- `counterparty`: Public key of counterparty, or 'self'/'anyone'
- `forSelf`: Whether the key is for yourself (true) or counterparty (false)

**Returns:** Derived `PublicKey`

**Example:**
```typescript
const pubKey = keyDeriver.derivePublicKey(
  [2, 'invoice-protocol'],
  'invoice-42',
  counterpartyPublicKey,
  false
);
```

#### derivePrivateKey()

```typescript
derivePrivateKey(
  protocolID: [number, string],
  keyID: string,
  counterparty: PublicKey | 'self' | 'anyone'
): PrivateKey
```

Derives a private key for a specific context. Only derive private keys for keys you control.

**Parameters:**
- `protocolID`: Tuple of `[securityLevel, protocolName]`
- `keyID`: Unique identifier within the protocol context
- `counterparty`: Public key of counterparty, or 'self'/'anyone'

**Returns:** Derived `PrivateKey`

**Example:**
```typescript
const privKey = keyDeriver.derivePrivateKey(
  [3, 'payment-protocol'],
  'tx-12345',
  'anyone'
);
```

#### deriveSymmetricKey()

```typescript
deriveSymmetricKey(
  protocolID: [number, string],
  keyID: string,
  counterparty: PublicKey
): number[]
```

Derives a symmetric encryption key using ECDH key agreement.

**Parameters:**
- `protocolID`: Tuple of `[securityLevel, protocolName]`
- `keyID`: Unique identifier within the protocol context
- `counterparty`: Public key of the other party

**Returns:** Array of numbers representing the symmetric key bytes

**Example:**
```typescript
const symmetricKey = keyDeriver.deriveSymmetricKey(
  [2, 'secure-messaging'],
  'conversation-789',
  counterpartyPublicKey
);
```

#### revealCounterpartySecret()

```typescript
revealCounterpartySecret(
  counterparty: PublicKey
): number[]
```

Reveals the shared secret with a counterparty (use carefully for debugging or specific protocols).

**Parameters:**
- `counterparty`: Public key of the counterparty

**Returns:** Array of numbers representing the shared secret

#### revealSpecificSecret()

```typescript
revealSpecificSecret(
  protocolID: [number, string],
  keyID: string,
  counterparty: PublicKey | 'self' | 'anyone'
): number[]
```

Reveals the specific secret for a given derivation path.

**Parameters:**
- `protocolID`: Tuple of `[securityLevel, protocolName]`
- `keyID`: Unique identifier within the protocol context
- `counterparty`: Public key of counterparty, or 'self'/'anyone'

**Returns:** Array of numbers representing the specific secret

## Common Patterns

### Pattern 1: Payment Address Generation

Generate unique payment addresses per invoice without address reuse:

```typescript
import { KeyDeriver, PrivateKey, P2PKH, Transaction } from '@bsv/sdk';

class PaymentAddressManager {
  private keyDeriver: KeyDeriver;
  private protocolID: [number, string] = [2, 'payment-system-v1'];

  constructor(masterKey: PrivateKey) {
    this.keyDeriver = new KeyDeriver(masterKey);
  }

  // Generate unique address for each invoice
  generateInvoiceAddress(invoiceID: string): string {
    const publicKey = this.keyDeriver.derivePublicKey(
      this.protocolID,
      `invoice-${invoiceID}`,
      'anyone', // Public payment address
      false
    );

    return publicKey.toAddress();
  }

  // Get private key to spend from invoice address
  getInvoicePrivateKey(invoiceID: string): PrivateKey {
    return this.keyDeriver.derivePrivateKey(
      this.protocolID,
      `invoice-${invoiceID}`,
      'anyone'
    );
  }

  // Create transaction spending from invoice
  async spendInvoice(
    invoiceID: string,
    sourceTransaction: Transaction,
    sourceOutputIndex: number,
    recipientAddress: string,
    amount: number
  ): Promise<Transaction> {
    const privKey = this.getInvoicePrivateKey(invoiceID);

    const tx = new Transaction();
    tx.addInput({
      sourceTransaction,
      sourceOutputIndex,
      unlockingScriptTemplate: new P2PKH().unlock(privKey)
    });

    tx.addOutput({
      lockingScript: new P2PKH().lock(recipientAddress),
      satoshis: amount
    });

    await tx.fee();
    await tx.sign();

    return tx;
  }
}

// Usage
const manager = new PaymentAddressManager(PrivateKey.fromRandom());

// Each invoice gets unique address
const addr1 = manager.generateInvoiceAddress('INV-001');
const addr2 = manager.generateInvoiceAddress('INV-002');
const addr3 = manager.generateInvoiceAddress('INV-003');

console.log('Unique addresses generated:', addr1, addr2, addr3);
```

### Pattern 2: Secure Messaging with Symmetric Keys

Implement end-to-end encrypted messaging using derived symmetric keys:

```typescript
import { KeyDeriver, PrivateKey, PublicKey } from '@bsv/sdk';
import { encrypt, decrypt } from '@bsv/sdk/messages';

class SecureMessaging {
  private keyDeriver: KeyDeriver;
  private protocolID: [number, string] = [2, 'secure-messaging-v1'];

  constructor(masterKey: PrivateKey) {
    this.keyDeriver = new KeyDeriver(masterKey);
  }

  // Encrypt message for specific counterparty and conversation
  encryptMessage(
    conversationID: string,
    counterpartyPubKey: PublicKey,
    message: string
  ): number[] {
    // Derive symmetric key for this conversation
    const symmetricKey = this.keyDeriver.deriveSymmetricKey(
      this.protocolID,
      `conversation-${conversationID}`,
      counterpartyPubKey
    );

    // Use BRC-78 encryption with derived sender key
    const senderPrivKey = this.keyDeriver.derivePrivateKey(
      this.protocolID,
      `conversation-${conversationID}`,
      counterpartyPubKey
    );

    const messageBytes = Array.from(Buffer.from(message, 'utf8'));

    return encrypt(messageBytes, senderPrivKey, counterpartyPubKey);
  }

  // Decrypt message from specific counterparty and conversation
  decryptMessage(
    conversationID: string,
    counterpartyPubKey: PublicKey,
    encryptedMessage: number[]
  ): string {
    // Derive private key for this conversation
    const recipientPrivKey = this.keyDeriver.derivePrivateKey(
      this.protocolID,
      `conversation-${conversationID}`,
      counterpartyPubKey
    );

    const decryptedBytes = decrypt(encryptedMessage, recipientPrivKey);

    return Buffer.from(decryptedBytes).toString('utf8');
  }
}

// Usage
const aliceKey = PrivateKey.fromRandom();
const bobKey = PrivateKey.fromRandom();

const aliceMessaging = new SecureMessaging(aliceKey);
const bobMessaging = new SecureMessaging(bobKey);

const conversationID = 'conv-12345';

// Alice encrypts message to Bob
const encrypted = aliceMessaging.encryptMessage(
  conversationID,
  bobKey.toPublicKey(),
  'Hello Bob, this is private!'
);

// Bob decrypts message from Alice
const decrypted = bobMessaging.decryptMessage(
  conversationID,
  aliceKey.toPublicKey(),
  encrypted
);

console.log('Decrypted message:', decrypted);
```

### Pattern 3: Multi-Protocol Wallet with Change Addresses

Manage multiple protocols with proper change address handling:

```typescript
import { KeyDeriver, PrivateKey, PublicKey, Transaction, P2PKH } from '@bsv/sdk';

class MultiProtocolWallet {
  private keyDeriver: KeyDeriver;
  private protocols: Map<string, [number, string]> = new Map();
  private addressCounters: Map<string, number> = new Map();

  constructor(masterKey: PrivateKey) {
    this.keyDeriver = new KeyDeriver(masterKey);

    // Register protocols
    this.protocols.set('payments', [2, 'payment-protocol-v1']);
    this.protocols.set('tokens', [2, 'token-protocol-v1']);
    this.protocols.set('storage', [1, 'storage-protocol-v1']);
  }

  // Get next unused address for protocol
  getNextAddress(protocolName: string): string {
    const protocolID = this.protocols.get(protocolName);
    if (!protocolID) throw new Error(`Unknown protocol: ${protocolName}`);

    const counter = this.addressCounters.get(protocolName) || 0;
    this.addressCounters.set(protocolName, counter + 1);

    const publicKey = this.keyDeriver.derivePublicKey(
      protocolID,
      `address-${counter}`,
      'anyone',
      false
    );

    return publicKey.toAddress();
  }

  // Get change address (never reuse addresses!)
  getChangeAddress(protocolName: string): string {
    const protocolID = this.protocols.get(protocolName);
    if (!protocolID) throw new Error(`Unknown protocol: ${protocolName}`);

    const counter = this.addressCounters.get(protocolName) || 0;
    this.addressCounters.set(protocolName, counter + 1);

    const publicKey = this.keyDeriver.derivePublicKey(
      protocolID,
      `change-${counter}`,
      'self', // Change goes to self
      true
    );

    return publicKey.toAddress();
  }

  // Get private key for address
  getPrivateKey(
    protocolName: string,
    addressIndex: number,
    isChange: boolean = false
  ): PrivateKey {
    const protocolID = this.protocols.get(protocolName);
    if (!protocolID) throw new Error(`Unknown protocol: ${protocolName}`);

    const prefix = isChange ? 'change' : 'address';
    const counterparty = isChange ? 'self' : 'anyone';

    return this.keyDeriver.derivePrivateKey(
      protocolID,
      `${prefix}-${addressIndex}`,
      counterparty
    );
  }

  // Create transaction with protocol-specific change
  async createTransaction(
    protocolName: string,
    inputs: Array<{
      sourceTransaction: Transaction;
      sourceOutputIndex: number;
      addressIndex: number;
      isChange: boolean;
    }>,
    recipientAddress: string,
    amount: number
  ): Promise<Transaction> {
    const tx = new Transaction();

    // Add inputs with derived keys
    for (const input of inputs) {
      const privKey = this.getPrivateKey(
        protocolName,
        input.addressIndex,
        input.isChange
      );

      tx.addInput({
        sourceTransaction: input.sourceTransaction,
        sourceOutputIndex: input.sourceOutputIndex,
        unlockingScriptTemplate: new P2PKH().unlock(privKey)
      });
    }

    // Add payment output
    tx.addOutput({
      lockingScript: new P2PKH().lock(recipientAddress),
      satoshis: amount
    });

    // Add change output with new address
    const changeAddress = this.getChangeAddress(protocolName);
    tx.addOutput({
      lockingScript: new P2PKH().lock(changeAddress),
      change: true
    });

    await tx.fee();
    await tx.sign();

    return tx;
  }
}

// Usage
const wallet = new MultiProtocolWallet(PrivateKey.fromRandom());

// Get addresses for different protocols
const paymentAddr1 = wallet.getNextAddress('payments');
const paymentAddr2 = wallet.getNextAddress('payments');
const tokenAddr = wallet.getNextAddress('tokens');
const storageAddr = wallet.getNextAddress('storage');

console.log('Protocol-isolated addresses:', {
  payment: [paymentAddr1, paymentAddr2],
  token: tokenAddr,
  storage: storageAddr
});

// Each protocol maintains separate address spaces
```

## Security Considerations

### Master Key Protection

The master key is the single point of failure for HD wallets:

```typescript
import { KeyDeriver, PrivateKey } from '@bsv/sdk';

// CRITICAL: Protect master key with encryption at rest
class SecureKeyStorage {
  // Never store master key in plain text
  encryptAndStoreMasterKey(masterKey: PrivateKey, password: string): void {
    // Use strong encryption (AES-256-GCM per BRC-2)
    // Store encrypted key in secure location
    // Consider hardware security modules (HSM) for production
  }

  // Decrypt only when needed, keep in memory briefly
  decryptMasterKey(password: string): PrivateKey {
    // Decrypt from secure storage
    // Return for immediate use
    // Clear from memory after use
    return PrivateKey.fromWif('...');
  }
}

// GOOD: Use master key temporarily
function performOperation(): void {
  const storage = new SecureKeyStorage();
  const masterKey = storage.decryptMasterKey('user-password');

  try {
    const keyDeriver = new KeyDeriver(masterKey);
    // ... perform operations ...
  } finally {
    // Clear sensitive data
    // (In production, use secure memory clearing)
  }
}

// BAD: Don't keep master key in memory
let globalMasterKey: PrivateKey; // NEVER DO THIS
```

### Security Level Selection

Choose appropriate security levels per BRC-43:

```typescript
import { KeyDeriver, PrivateKey } from '@bsv/sdk';

const keyDeriver = new KeyDeriver(PrivateKey.fromRandom());

// Security level guidelines:

// Level 0: Public data, no privacy needed
const publicKey0 = keyDeriver.derivePublicKey(
  [0, 'public-announcements'],
  'announcement-1',
  'anyone',
  false
);

// Level 1: Low security, semi-public data
const publicKey1 = keyDeriver.derivePublicKey(
  [1, 'social-media'],
  'post-123',
  'anyone',
  false
);

// Level 2: Standard security, most applications (DEFAULT)
const publicKey2 = keyDeriver.derivePublicKey(
  [2, 'e-commerce'],
  'order-456',
  'anyone',
  false
);

// Level 3: High security, financial transactions
const publicKey3 = keyDeriver.derivePublicKey(
  [3, 'banking-app'],
  'transfer-789',
  'anyone',
  false
);

// Level 4: Critical security, identity and custody
const publicKey4 = keyDeriver.derivePublicKey(
  [4, 'identity-verification'],
  'identity-001',
  'anyone',
  false
);

// RULE: Higher security levels for more sensitive operations
```

### Counterparty Validation

Always validate counterparty public keys:

```typescript
import { KeyDeriver, PrivateKey, PublicKey } from '@bsv/sdk';

class SecureKeyDerivation {
  private keyDeriver: KeyDeriver;

  constructor(masterKey: PrivateKey) {
    this.keyDeriver = new KeyDeriver(masterKey);
  }

  deriveForCounterparty(
    protocolID: [number, string],
    keyID: string,
    counterpartyPubKeyStr: string
  ): PublicKey {
    // CRITICAL: Validate counterparty key format
    try {
      const counterpartyPubKey = PublicKey.fromString(counterpartyPubKeyStr);

      // Verify key is valid point on curve
      if (!this.isValidPublicKey(counterpartyPubKey)) {
        throw new Error('Invalid public key point');
      }

      return this.keyDeriver.derivePublicKey(
        protocolID,
        keyID,
        counterpartyPubKey,
        false
      );
    } catch (error) {
      throw new Error(`Invalid counterparty public key: ${error.message}`);
    }
  }

  private isValidPublicKey(pubKey: PublicKey): boolean {
    // Verify key is on secp256k1 curve
    // Check for valid compression
    // Ensure not point at infinity
    return pubKey.verify(); // SDK method
  }
}
```

### Key ID Uniqueness

Ensure key IDs are truly unique:

```typescript
import { KeyDeriver, PrivateKey } from '@bsv/sdk';
import { createHash } from 'crypto';

class UniqueKeyIDGenerator {
  private keyDeriver: KeyDeriver;
  private usedKeyIDs: Set<string> = new Set();

  constructor(masterKey: PrivateKey) {
    this.keyDeriver = new KeyDeriver(masterKey);
  }

  // Generate cryptographically unique key ID
  generateKeyID(
    baseID: string,
    timestamp: number = Date.now(),
    nonce: string = Math.random().toString(36)
  ): string {
    // Combine multiple entropy sources
    const components = `${baseID}-${timestamp}-${nonce}`;
    const hash = createHash('sha256')
      .update(components)
      .digest('hex')
      .substring(0, 32);

    // Ensure uniqueness
    if (this.usedKeyIDs.has(hash)) {
      // Collision detected (extremely rare), retry with new nonce
      return this.generateKeyID(baseID, timestamp, nonce + '1');
    }

    this.usedKeyIDs.add(hash);
    return hash;
  }

  deriveUniqueKey(protocolID: [number, string], baseID: string): PublicKey {
    const uniqueKeyID = this.generateKeyID(baseID);

    return this.keyDeriver.derivePublicKey(
      protocolID,
      uniqueKeyID,
      'anyone',
      false
    );
  }
}

// Usage
const generator = new UniqueKeyIDGenerator(PrivateKey.fromRandom());

// Generate truly unique keys even with same base ID
const key1 = generator.deriveUniqueKey([2, 'protocol'], 'user-123');
const key2 = generator.deriveUniqueKey([2, 'protocol'], 'user-123');

console.log('Keys are unique:', key1.toString() !== key2.toString());
```

## Performance Considerations

### Key Derivation Caching

Cache derived keys to avoid repeated computation:

```typescript
import { KeyDeriver, PrivateKey, PublicKey } from '@bsv/sdk';

class CachedKeyDeriver {
  private keyDeriver: KeyDeriver;
  private publicKeyCache: Map<string, PublicKey> = new Map();
  private privateKeyCache: Map<string, PrivateKey> = new Map();
  private cacheHits: number = 0;
  private cacheMisses: number = 0;

  constructor(masterKey: PrivateKey, maxCacheSize: number = 1000) {
    this.keyDeriver = new KeyDeriver(masterKey);
  }

  derivePublicKey(
    protocolID: [number, string],
    keyID: string,
    counterparty: PublicKey | 'self' | 'anyone',
    forSelf: boolean
  ): PublicKey {
    // Create cache key
    const cacheKey = this.createCacheKey(
      protocolID,
      keyID,
      counterparty,
      forSelf
    );

    // Check cache
    if (this.publicKeyCache.has(cacheKey)) {
      this.cacheHits++;
      return this.publicKeyCache.get(cacheKey)!;
    }

    // Cache miss - derive key
    this.cacheMisses++;
    const derivedKey = this.keyDeriver.derivePublicKey(
      protocolID,
      keyID,
      counterparty,
      forSelf
    );

    // Store in cache
    this.publicKeyCache.set(cacheKey, derivedKey);

    return derivedKey;
  }

  derivePrivateKey(
    protocolID: [number, string],
    keyID: string,
    counterparty: PublicKey | 'self' | 'anyone'
  ): PrivateKey {
    const cacheKey = this.createCacheKey(protocolID, keyID, counterparty, true);

    if (this.privateKeyCache.has(cacheKey)) {
      this.cacheHits++;
      return this.privateKeyCache.get(cacheKey)!;
    }

    this.cacheMisses++;
    const derivedKey = this.keyDeriver.derivePrivateKey(
      protocolID,
      keyID,
      counterparty
    );

    // SECURITY: Be cautious caching private keys
    // Consider memory clearing and cache size limits
    this.privateKeyCache.set(cacheKey, derivedKey);

    return derivedKey;
  }

  private createCacheKey(
    protocolID: [number, string],
    keyID: string,
    counterparty: PublicKey | 'self' | 'anyone',
    forSelf: boolean
  ): string {
    const counterpartyStr = typeof counterparty === 'string'
      ? counterparty
      : counterparty.toString();

    return `${protocolID[0]}-${protocolID[1]}-${keyID}-${counterpartyStr}-${forSelf}`;
  }

  getCacheStats(): { hits: number; misses: number; hitRate: number } {
    const total = this.cacheHits + this.cacheMisses;
    const hitRate = total > 0 ? this.cacheHits / total : 0;

    return {
      hits: this.cacheHits,
      misses: this.cacheMisses,
      hitRate
    };
  }

  clearCache(): void {
    this.publicKeyCache.clear();
    this.privateKeyCache.clear();
    this.cacheHits = 0;
    this.cacheMisses = 0;
  }
}

// Usage
const cachedDeriver = new CachedKeyDeriver(PrivateKey.fromRandom());

// First derivation - cache miss
const key1 = cachedDeriver.derivePublicKey([2, 'protocol'], 'key-1', 'anyone', false);

// Second derivation - cache hit (much faster)
const key2 = cachedDeriver.derivePublicKey([2, 'protocol'], 'key-1', 'anyone', false);

console.log('Cache stats:', cachedDeriver.getCacheStats());
// { hits: 1, misses: 1, hitRate: 0.5 }
```

### Batch Key Derivation

Derive multiple keys efficiently:

```typescript
import { KeyDeriver, PrivateKey, PublicKey } from '@bsv/sdk';

class BatchKeyDeriver {
  private keyDeriver: KeyDeriver;

  constructor(masterKey: PrivateKey) {
    this.keyDeriver = new KeyDeriver(masterKey);
  }

  // Derive multiple public keys in batch
  derivePublicKeyBatch(
    protocolID: [number, string],
    keyIDs: string[],
    counterparty: PublicKey | 'self' | 'anyone',
    forSelf: boolean
  ): PublicKey[] {
    return keyIDs.map(keyID =>
      this.keyDeriver.derivePublicKey(protocolID, keyID, counterparty, forSelf)
    );
  }

  // Derive range of sequential keys
  deriveKeyRange(
    protocolID: [number, string],
    baseKeyID: string,
    count: number,
    counterparty: PublicKey | 'self' | 'anyone',
    forSelf: boolean
  ): PublicKey[] {
    const keys: PublicKey[] = [];

    for (let i = 0; i < count; i++) {
      const keyID = `${baseKeyID}-${i}`;
      keys.push(
        this.keyDeriver.derivePublicKey(protocolID, keyID, counterparty, forSelf)
      );
    }

    return keys;
  }

  // Generate address pool
  generateAddressPool(
    protocolID: [number, string],
    poolSize: number
  ): string[] {
    const keys = this.deriveKeyRange(
      protocolID,
      'pool-address',
      poolSize,
      'anyone',
      false
    );

    return keys.map(key => key.toAddress());
  }
}

// Usage
const batchDeriver = new BatchKeyDeriver(PrivateKey.fromRandom());

// Generate 20 addresses at once
const addressPool = batchDeriver.generateAddressPool(
  [2, 'payment-protocol'],
  20
);

console.log(`Generated ${addressPool.length} addresses`);
```

## Related Components

- **[Primitives](../primitives/README.md)**: Core `PrivateKey` and `PublicKey` classes
- **[Transactions](../transactions/README.md)**: Use derived keys in transactions
- **[P2PKH](../p2pkh/README.md)**: P2PKH template for derived addresses
- **[Messages](../messages/README.md)**: Encrypted messaging with symmetric keys
- **[Signatures](../signatures/README.md)**: Sign with derived private keys

## Best Practices

### 1. Master Key Management

```typescript
// GOOD: Store master key encrypted
class MasterKeyManager {
  private encryptedMaster: string;

  constructor(masterKey: PrivateKey, password: string) {
    // Encrypt master key with strong password
    this.encryptedMaster = this.encrypt(masterKey.toWif(), password);
  }

  getDeriver(password: string): KeyDeriver {
    const wif = this.decrypt(this.encryptedMaster, password);
    return new KeyDeriver(PrivateKey.fromWif(wif));
  }

  private encrypt(data: string, password: string): string {
    // Use BRC-2 encryption
    return '...encrypted...';
  }

  private decrypt(encrypted: string, password: string): string {
    return '...decrypted...';
  }
}

// BAD: Store master key in plain text
const masterKey = PrivateKey.fromRandom();
localStorage.setItem('masterKey', masterKey.toWif()); // NEVER DO THIS
```

### 2. Protocol Versioning

```typescript
// GOOD: Version your protocols
class VersionedProtocols {
  static PAYMENT_V1: [number, string] = [2, 'payment-v1'];
  static PAYMENT_V2: [number, string] = [2, 'payment-v2'];

  static MESSAGING_V1: [number, string] = [2, 'messaging-v1'];

  // When upgrading, create new protocol ID
  static getCurrentPaymentProtocol(): [number, string] {
    return this.PAYMENT_V2; // Use latest version
  }
}

// BAD: Change protocol name without versioning
const PROTOCOL: [number, string] = [2, 'payment']; // What happens when you change it?
```

### 3. Key ID Documentation

```typescript
// GOOD: Document key ID structure
class DocumentedKeyIDs {
  /**
   * Generate invoice key ID
   * Format: invoice-{invoiceNumber}-{timestamp}
   * Example: invoice-INV-001-1699564800000
   */
  static invoice(invoiceNumber: string, timestamp: number): string {
    return `invoice-${invoiceNumber}-${timestamp}`;
  }

  /**
   * Generate conversation key ID
   * Format: conversation-{conversationId}
   * Example: conversation-8f3a9c12-4b7d-4e2a-a9f1-3c8d7e6f5a4b
   */
  static conversation(conversationId: string): string {
    return `conversation-${conversationId}`;
  }
}

// BAD: Unclear key ID structure
const keyID = `${Math.random()}`; // What does this represent?
```

### 4. Counterparty Context

```typescript
// GOOD: Use appropriate counterparty context
class CounterpartyContexts {
  // Public addresses anyone can pay to
  static publicAddress(
    deriver: KeyDeriver,
    protocolID: [number, string],
    keyID: string
  ): string {
    return deriver.derivePublicKey(
      protocolID,
      keyID,
      'anyone', // Correct: public context
      false
    ).toAddress();
  }

  // Change addresses for yourself
  static changeAddress(
    deriver: KeyDeriver,
    protocolID: [number, string],
    keyID: string
  ): string {
    return deriver.derivePublicKey(
      protocolID,
      keyID,
      'self', // Correct: change goes to self
      true    // Correct: for yourself
    ).toAddress();
  }

  // Counterparty-specific address
  static counterpartyAddress(
    deriver: KeyDeriver,
    protocolID: [number, string],
    keyID: string,
    counterparty: PublicKey
  ): string {
    return deriver.derivePublicKey(
      protocolID,
      keyID,
      counterparty, // Correct: specific counterparty
      false         // Correct: for them, not you
    ).toAddress();
  }
}
```

### 5. Backup and Recovery

```typescript
// GOOD: Implement proper backup strategy
class WalletBackup {
  // Backup master key with mnemonic (BIP-39 compatible)
  static backupMasterKey(masterKey: PrivateKey): {
    wif: string;
    mnemonic: string;
  } {
    const wif = masterKey.toWif();
    const mnemonic = this.generateMnemonic(masterKey);

    return { wif, mnemonic };
  }

  // Recovery from backup
  static recoverFromBackup(wif: string): KeyDeriver {
    const masterKey = PrivateKey.fromWif(wif);
    return new KeyDeriver(masterKey);
  }

  // Test recovery before trusting backup
  static testRecovery(
    originalDeriver: KeyDeriver,
    recoveredDeriver: KeyDeriver,
    testProtocol: [number, string]
  ): boolean {
    const keyID = 'recovery-test-1';

    const original = originalDeriver.derivePublicKey(
      testProtocol,
      keyID,
      'anyone',
      false
    );

    const recovered = recoveredDeriver.derivePublicKey(
      testProtocol,
      keyID,
      'anyone',
      false
    );

    return original.toString() === recovered.toString();
  }

  private static generateMnemonic(masterKey: PrivateKey): string {
    // Generate BIP-39 mnemonic for user-friendly backup
    return 'abandon abandon abandon...'; // Implement properly
  }
}

// Usage
const originalKey = PrivateKey.fromRandom();
const originalDeriver = new KeyDeriver(originalKey);

// Backup
const backup = WalletBackup.backupMasterKey(originalKey);
console.log('Store securely:', backup.mnemonic);

// Recovery
const recoveredDeriver = WalletBackup.recoverFromBackup(backup.wif);

// Test
const success = WalletBackup.testRecovery(
  originalDeriver,
  recoveredDeriver,
  [2, 'payment-v1']
);

console.log('Recovery successful:', success);
```

## Troubleshooting

### Issue: Derived Keys Don't Match After Recovery

**Problem:** Keys derived after recovery don't match original keys.

**Solution:**
```typescript
// Checklist for recovery issues:
class RecoveryTroubleshooting {
  static diagnose(
    originalDeriver: KeyDeriver,
    recoveredDeriver: KeyDeriver
  ): void {
    const testProtocol: [number, string] = [2, 'test-protocol'];
    const testKeyID = 'test-key-1';

    // 1. Verify master keys match
    console.log('Master keys match:',
      this.verifyMasterKeys(originalDeriver, recoveredDeriver)
    );

    // 2. Test simple derivation
    const orig = originalDeriver.derivePublicKey(
      testProtocol,
      testKeyID,
      'anyone',
      false
    );
    const recv = recoveredDeriver.derivePublicKey(
      testProtocol,
      testKeyID,
      'anyone',
      false
    );

    console.log('Simple derivation matches:',
      orig.toString() === recv.toString()
    );

    // 3. Check protocol ID format
    console.log('Protocol ID format:', testProtocol);
    console.log('Security level:', testProtocol[0]);
    console.log('Protocol name:', testProtocol[1]);

    // 4. Verify counterparty value
    console.log('Counterparty:', 'anyone');
    console.log('ForSelf:', false);

    // 5. Check key ID exact match
    console.log('Key ID:', testKeyID);
  }

  private static verifyMasterKeys(d1: KeyDeriver, d2: KeyDeriver): boolean {
    // Derive same key with both derivers
    const testProtocol: [number, string] = [0, 'verify'];
    const k1 = d1.derivePublicKey(testProtocol, 'master-test', 'anyone', false);
    const k2 = d2.derivePublicKey(testProtocol, 'master-test', 'anyone', false);
    return k1.toString() === k2.toString();
  }
}
```

### Issue: Performance Degradation with Many Derivations

**Problem:** Application slows down when deriving many keys.

**Solution:**
```typescript
// Use caching and batch operations
class PerformanceOptimized {
  private cache: CachedKeyDeriver;
  private batcher: BatchKeyDeriver;

  constructor(masterKey: PrivateKey) {
    this.cache = new CachedKeyDeriver(masterKey, 5000); // Large cache
    this.batcher = new BatchKeyDeriver(masterKey);
  }

  // Derive many keys efficiently
  deriveMany(
    protocolID: [number, string],
    count: number
  ): string[] {
    // Use batch derivation
    const keys = this.batcher.deriveKeyRange(
      protocolID,
      'batch',
      count,
      'anyone',
      false
    );

    return keys.map(k => k.toAddress());
  }

  // Reuse common keys
  getCommonKey(keyID: string): PublicKey {
    // Cache automatically handles reuse
    return this.cache.derivePublicKey(
      [2, 'common-protocol'],
      keyID,
      'anyone',
      false
    );
  }
}
```

### Issue: Incorrect Counterparty Key Type

**Problem:** Error when passing counterparty key to derivation.

**Solution:**
```typescript
// Validate and convert counterparty keys
class CounterpartyKeyHandler {
  static parseCounterparty(
    input: string | PublicKey
  ): PublicKey | 'self' | 'anyone' {
    // Handle special values
    if (input === 'self' || input === 'anyone') {
      return input;
    }

    // Handle string public key
    if (typeof input === 'string') {
      try {
        return PublicKey.fromString(input);
      } catch (error) {
        throw new Error(`Invalid public key string: ${input}`);
      }
    }

    // Already a PublicKey object
    if (input instanceof PublicKey) {
      return input;
    }

    throw new Error(`Invalid counterparty type: ${typeof input}`);
  }

  static deriveWithValidation(
    deriver: KeyDeriver,
    protocolID: [number, string],
    keyID: string,
    counterpartyInput: string | PublicKey,
    forSelf: boolean
  ): PublicKey {
    const counterparty = this.parseCounterparty(counterpartyInput);

    return deriver.derivePublicKey(
      protocolID,
      keyID,
      counterparty,
      forSelf
    );
  }
}

// Usage
const result = CounterpartyKeyHandler.deriveWithValidation(
  keyDeriver,
  [2, 'protocol'],
  'key-1',
  '027a1b2c3d...', // String will be converted
  false
);
```

### Issue: Protocol Name Collision

**Problem:** Different protocols accidentally use the same name.

**Solution:**
```typescript
// Use namespaced protocol registry
class ProtocolRegistry {
  private static protocols: Map<string, [number, string]> = new Map();

  static register(
    namespace: string,
    name: string,
    securityLevel: number
  ): [number, string] {
    const fullName = `${namespace}:${name}`;
    const protocolID: [number, string] = [securityLevel, fullName];

    if (this.protocols.has(fullName)) {
      throw new Error(`Protocol ${fullName} already registered`);
    }

    this.protocols.set(fullName, protocolID);
    return protocolID;
  }

  static get(namespace: string, name: string): [number, string] {
    const fullName = `${namespace}:${name}`;
    const protocol = this.protocols.get(fullName);

    if (!protocol) {
      throw new Error(`Protocol ${fullName} not registered`);
    }

    return protocol;
  }
}

// Usage
ProtocolRegistry.register('myapp', 'payments', 2);
ProtocolRegistry.register('myapp', 'messaging', 2);

const paymentProtocol = ProtocolRegistry.get('myapp', 'payments');
// Returns: [2, 'myapp:payments']
```

## Further Reading

### BRC Standards
- **[BRC-42](https://github.com/bitcoin-sv/BRCs/blob/master/key-derivation/0042.md)**: BSV Key Derivation Scheme (BKDS)
- **[BRC-43](https://github.com/bitcoin-sv/BRCs/blob/master/key-derivation/0043.md)**: Security Levels and Protocol IDs
- **[BRC-44](https://github.com/bitcoin-sv/BRCs/blob/master/key-derivation/0044.md)**: Protocol Security Requirements

### Related Documentation
- **[SDK Wallet Documentation](https://bsv-blockchain.github.io/ts-sdk/classes/KeyDeriver.html)**: Official KeyDeriver API
- **[BIP-32 Specification](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)**: Hierarchical Deterministic Wallets (original)
- **[BIP-39 Specification](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)**: Mnemonic code for generating deterministic keys

### Tutorials and Examples
- **[BSV SDK Examples](https://github.com/bsv-blockchain/ts-sdk/tree/master/examples)**: Complete working examples
- **[Key Derivation Guide](https://docs.bsvblockchain.org/guides/key-derivation)**: Step-by-step tutorial
- **[Privacy Best Practices](https://docs.bsvblockchain.org/guides/privacy)**: Using HD wallets for privacy

## Status

✅ Complete - Comprehensive documentation with BRC-42/43 implementation details, security considerations, and production-ready examples.
