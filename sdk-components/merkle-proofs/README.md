# Merkle Proofs

## Overview

The Merkle Proofs component implements Simplified Payment Verification (SPV) using merkle trees, enabling lightweight clients to verify transaction inclusion in blocks without downloading the entire blockchain. The `MerklePath` class provides functionality for creating, parsing, and verifying merkle proofs according to BSV's TSC (Transaction Signature Component) format.

Merkle proofs are essential for SPV validation, allowing wallets and applications to cryptographically prove that a transaction exists in a specific block by providing only the merkle branch path and block header, rather than all transactions in the block.

## Purpose

Merkle Proofs in the BSV SDK solve several critical problems:

- **SPV Verification**: Verify transaction inclusion without full blockchain data
- **Bandwidth Efficiency**: Reduce data transmission by providing only merkle paths
- **Scalability**: Enable lightweight clients to operate without full node infrastructure
- **Proof of Inclusion**: Cryptographically prove transaction existence in blocks
- **BEEF Integration**: Support transaction envelope format (BRC-62) with merkle proofs
- **Chain Verification**: Validate merkle roots against block headers

This component is fundamental for building scalable BSV applications that don't require full node infrastructure while maintaining cryptographic security guarantees.

## Basic Usage

### Parse Merkle Path from Hex

```typescript
import { MerklePath } from '@bsv/sdk';

// Parse a merkle path from hexadecimal format
const merklePathHex = 'fed7c509000a02fddd01...';
const merklePath = MerklePath.fromHex(merklePathHex);

// Access merkle path properties
console.log('Block height:', merklePath.blockHeight);
console.log('Path length:', merklePath.path.length);

// Get the merkle root calculated from this path
const merkleRoot = merklePath.computeRoot();
console.log('Merkle root:', Buffer.from(merkleRoot).toString('hex'));
```

### Attach Merkle Proof to Transaction

```typescript
import { Transaction, MerklePath } from '@bsv/sdk';

// Parse transaction and merkle proof
const tx = Transaction.fromHex('0100000001...');
const merklePath = MerklePath.fromHex('fed7c509000a02fddd01...');

// Attach merkle proof to transaction for SPV
tx.merklePath = merklePath;

// Now the transaction includes SPV proof
console.log('Transaction has merkle proof:', tx.merklePath !== undefined);

// Use this transaction as a source transaction
// The merkle path proves it exists on-chain
```

### Verify Merkle Proof Against Block Header

```typescript
import { MerklePath, ChainTracker } from '@bsv/sdk';

// Custom chain tracker for merkle root verification
class CustomChainTracker implements ChainTracker {
  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    // Fetch block header from a block explorer or node
    const response = await fetch(
      `https://api.whatsonchain.com/v1/bsv/main/block/height/${height}`
    );
    const block = await response.json();

    // Verify merkle root matches
    return block.merkleroot === root;
  }
}

// Verify merkle proof
const merklePath = MerklePath.fromHex('fed7c509000a02fddd01...');
const chainTracker = new CustomChainTracker();

// Calculate merkle root from path
const calculatedRoot = Buffer.from(merklePath.computeRoot()).toString('hex');

// Verify against blockchain
const isValid = await chainTracker.isValidRootForHeight(
  calculatedRoot,
  merklePath.blockHeight
);

console.log('Merkle proof valid:', isValid);
```

### Create Merkle Path from Transaction Data

```typescript
import { MerklePath } from '@bsv/sdk';

// Create a merkle path manually (typically received from a mining node)
const merklePath = new MerklePath();

// Set block height where transaction was mined
merklePath.blockHeight = 825000;

// Build merkle path with hash pairs
// Each element is a step up the merkle tree
merklePath.path = [
  [
    { offset: 0, hash: Buffer.from('abc123...', 'hex') },
    { offset: 1, hash: Buffer.from('def456...', 'hex') }
  ],
  [
    { offset: 0, hash: Buffer.from('789ghi...', 'hex') }
  ]
];

// Serialize for transmission
const merklePathHex = merklePath.toHex();
console.log('Merkle path hex:', merklePathHex);
```

## Key Features

### 1. TSC Merkle Path Format

The SDK implements the TSC (Transaction Signature Component) format for merkle paths, providing a standardized binary representation:

```typescript
import { MerklePath, Transaction } from '@bsv/sdk';

// Parse TSC format merkle path
const tscMerklePathHex = 'fed7c509000a02fddd01...';
const merklePath = MerklePath.fromHex(tscMerklePathHex);

// TSC format structure:
// - Block height (VarInt)
// - Tree height (1 byte)
// - Path elements as offset/hash pairs
console.log('Block height:', merklePath.blockHeight);
console.log('Tree height:', merklePath.path.length);

// Iterate through merkle path levels
merklePath.path.forEach((level, index) => {
  console.log(`Level ${index}:`, level.length, 'hashes');
  level.forEach((node) => {
    console.log(`  Offset ${node.offset}: ${Buffer.from(node.hash).toString('hex')}`);
  });
});

// Serialize back to TSC format
const serialized = merklePath.toHex();
console.log('TSC hex:', serialized);

// Binary format for compact transmission
const binary = merklePath.toBinary();
console.log('Binary size:', binary.length, 'bytes');
```

### 2. SPV Transaction Validation

Merkle proofs enable SPV validation by proving transaction inclusion:

```typescript
import { Transaction, MerklePath, ChainTracker } from '@bsv/sdk';

// SPV validation function
async function validateSPVTransaction(
  tx: Transaction,
  merklePath: MerklePath,
  chainTracker: ChainTracker
): Promise<boolean> {
  // 1. Calculate transaction ID
  const txid = tx.id('hex');
  console.log('Validating TXID:', txid);

  // 2. Compute merkle root from path
  const computedRoot = Buffer.from(merklePath.computeRoot(txid)).toString('hex');
  console.log('Computed merkle root:', computedRoot);

  // 3. Verify merkle root against blockchain
  const isValidRoot = await chainTracker.isValidRootForHeight(
    computedRoot,
    merklePath.blockHeight
  );

  if (!isValidRoot) {
    console.error('Invalid merkle root for block height', merklePath.blockHeight);
    return false;
  }

  console.log('✓ Transaction verified in block', merklePath.blockHeight);
  return true;
}

// WhatsOnChain chain tracker implementation
class WhatsOnChainTracker implements ChainTracker {
  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    try {
      const response = await fetch(
        `https://api.whatsonchain.com/v1/bsv/main/block/${height}/header`
      );
      const header = await response.json();
      return header.merkleroot === root;
    } catch (error) {
      console.error('Failed to fetch block header:', error);
      return false;
    }
  }
}

// Usage
const tx = Transaction.fromHex('...');
const proof = MerklePath.fromHex('...');
const tracker = new WhatsOnChainTracker();

const isValid = await validateSPVTransaction(tx, proof, tracker);
```

### 3. Merkle Path Computation

Calculate merkle roots from transaction hashes:

```typescript
import { MerklePath, Transaction } from '@bsv/sdk';

// Function to compute merkle root from a transaction and its path
function computeMerkleRoot(tx: Transaction, merklePath: MerklePath): Buffer {
  // Get transaction ID as the leaf hash
  const txid = tx.id('hex');
  console.log('Transaction ID:', txid);

  // Compute merkle root by traversing the path
  const merkleRoot = merklePath.computeRoot(txid);

  return merkleRoot;
}

// Example: Verify a transaction's merkle path
const tx = Transaction.fromHex('0100000001...');
const merklePath = MerklePath.fromHex('fed7c509000a02fddd01...');

const root = computeMerkleRoot(tx, merklePath);
console.log('Merkle root:', Buffer.from(root).toString('hex'));

// Combine transaction with its proof
tx.merklePath = merklePath;

// Now tx can be used as a source transaction with SPV proof
console.log('Transaction block height:', tx.merklePath.blockHeight);
```

### 4. Source Transaction with SPV Proofs

Use merkle proofs with source transactions for SPV-enabled transaction chains:

```typescript
import { Transaction, MerklePath, P2PKH, PrivateKey } from '@bsv/sdk';

// Create a new transaction spending from an SPV-verified source
async function createSPVTransaction(
  sourceTransactionHex: string,
  merkleProofHex: string,
  sourceOutputIndex: number,
  recipientAddress: string,
  privateKey: PrivateKey
): Promise<Transaction> {
  // Parse source transaction and attach merkle proof
  const sourceTransaction = Transaction.fromHex(sourceTransactionHex);
  const merklePath = MerklePath.fromHex(merkleProofHex);
  sourceTransaction.merklePath = merklePath;

  // Create new transaction
  const tx = new Transaction();

  // Add input with SPV-verified source
  tx.addInput({
    sourceTransaction,  // Includes merkle proof
    sourceOutputIndex,
    unlockingScriptTemplate: new P2PKH().unlock(privateKey)
  });

  // Add output
  tx.addOutput({
    lockingScript: new P2PKH().lock(recipientAddress),
    satoshis: 2500
  });

  // Add change output
  tx.addOutput({
    lockingScript: new P2PKH().lock(privateKey.toPublicKey().toAddress()),
    change: true
  });

  // Calculate fee and sign
  await tx.fee();
  await tx.sign();

  console.log('Created SPV transaction:', tx.id('hex'));
  console.log('Source block height:', sourceTransaction.merklePath.blockHeight);

  return tx;
}

// Usage
const privKey = PrivateKey.fromWif('L5EY1SbTvvPNSdCYQe1EJHfXCBBT4PmnF6CDbzCm9iifZptUvDGB');
const spvTx = await createSPVTransaction(
  '0100000001...',                  // Source transaction hex
  'fed7c509000a02fddd01...',        // Merkle proof hex
  0,                                 // Output index
  '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa', // Recipient
  privKey
);
```

## API Reference

### MerklePath Class

```typescript
class MerklePath {
  blockHeight: number;
  path: Array<Array<{ offset: number; hash: Uint8Array }>>;

  // Static constructors
  static fromHex(hex: string): MerklePath;
  static fromBinary(binary: number[]): MerklePath;
  static fromReader(reader: Reader): MerklePath;

  // Serialization methods
  toHex(): string;
  toBinary(): number[];
  toWriter(writer: Writer): void;

  // Computation methods
  computeRoot(txid?: string): Uint8Array;

  // Verification methods
  verify(txid: string, chainTracker: ChainTracker): Promise<boolean>;
}
```

### MerklePath Properties

- **blockHeight**: Block height where transaction was included
- **path**: Array of merkle tree levels, each containing offset/hash pairs

### MerklePath Methods

#### `static fromHex(hex: string): MerklePath`
Parse merkle path from hexadecimal string.

#### `static fromBinary(binary: number[]): MerklePath`
Parse merkle path from binary array.

#### `toHex(): string`
Serialize merkle path to hexadecimal string.

#### `toBinary(): number[]`
Serialize merkle path to binary array.

#### `computeRoot(txid?: string): Uint8Array`
Compute merkle root from the path and optional transaction ID.

#### `verify(txid: string, chainTracker: ChainTracker): Promise<boolean>`
Verify the merkle proof against blockchain using ChainTracker.

### ChainTracker Interface

```typescript
interface ChainTracker {
  isValidRootForHeight(root: string, height: number): Promise<boolean>;
}
```

Implementations must verify that a given merkle root is valid for a specific block height by checking against blockchain data.

## Common Patterns

### 1. WhatsOnChain SPV Verification

```typescript
import { MerklePath, Transaction, ChainTracker } from '@bsv/sdk';

/**
 * WhatsOnChain ChainTracker implementation
 * Verifies merkle roots against WhatsOnChain API
 */
class WhatsOnChainTracker implements ChainTracker {
  private baseUrl: string;
  private network: 'main' | 'test';

  constructor(network: 'main' | 'test' = 'main') {
    this.network = network;
    this.baseUrl = `https://api.whatsonchain.com/v1/bsv/${network}`;
  }

  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    try {
      // Fetch block header for the given height
      const response = await fetch(`${this.baseUrl}/block/height/${height}`);

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      const block = await response.json();

      // Compare merkle roots
      return block.merkleroot === root;
    } catch (error) {
      console.error(`Failed to verify merkle root at height ${height}:`, error);
      return false;
    }
  }

  /**
   * Additional utility: Get block header by height
   */
  async getBlockHeader(height: number): Promise<any> {
    const response = await fetch(`${this.baseUrl}/block/${height}/header`);
    if (!response.ok) {
      throw new Error(`Failed to fetch block header: ${response.statusText}`);
    }
    return response.json();
  }
}

/**
 * Complete SPV verification workflow
 */
async function verifySPVTransaction(
  txHex: string,
  merklePathHex: string
): Promise<{ valid: boolean; blockHeight: number; merkleRoot: string }> {
  // Parse transaction and merkle proof
  const tx = Transaction.fromHex(txHex);
  const merklePath = MerklePath.fromHex(merklePathHex);

  // Attach proof to transaction
  tx.merklePath = merklePath;

  // Initialize chain tracker
  const chainTracker = new WhatsOnChainTracker('main');

  // Compute merkle root
  const txid = tx.id('hex');
  const merkleRoot = Buffer.from(merklePath.computeRoot(txid)).toString('hex');

  // Verify against blockchain
  const valid = await chainTracker.isValidRootForHeight(
    merkleRoot,
    merklePath.blockHeight
  );

  return {
    valid,
    blockHeight: merklePath.blockHeight,
    merkleRoot
  };
}

// Usage
const result = await verifySPVTransaction(
  '0100000001...',              // Transaction hex
  'fed7c509000a02fddd01...'     // Merkle path hex
);

console.log('SPV Verification Result:', result);
// Output: { valid: true, blockHeight: 825000, merkleRoot: 'abc123...' }
```

### 2. Transaction Chain with SPV Proofs

```typescript
import { Transaction, MerklePath, P2PKH, PrivateKey, ARC } from '@bsv/sdk';

/**
 * Manages a chain of SPV-verified transactions
 */
class SPVTransactionChain {
  private transactions: Map<string, Transaction>;
  private chainTracker: ChainTracker;

  constructor(chainTracker: ChainTracker) {
    this.transactions = new Map();
    this.chainTracker = chainTracker;
  }

  /**
   * Add a transaction with its merkle proof
   */
  async addTransaction(
    txHex: string,
    merklePathHex: string
  ): Promise<boolean> {
    const tx = Transaction.fromHex(txHex);
    const merklePath = MerklePath.fromHex(merklePathHex);

    // Attach merkle proof
    tx.merklePath = merklePath;

    // Verify SPV proof
    const txid = tx.id('hex');
    const merkleRoot = Buffer.from(merklePath.computeRoot(txid)).toString('hex');
    const isValid = await this.chainTracker.isValidRootForHeight(
      merkleRoot,
      merklePath.blockHeight
    );

    if (!isValid) {
      throw new Error(`Invalid merkle proof for transaction ${txid}`);
    }

    // Store verified transaction
    this.transactions.set(txid, tx);
    console.log(`✓ Added verified transaction ${txid} from block ${merklePath.blockHeight}`);

    return true;
  }

  /**
   * Get a verified transaction by ID
   */
  getTransaction(txid: string): Transaction | undefined {
    return this.transactions.get(txid);
  }

  /**
   * Create a new transaction spending from verified sources
   */
  async createSpendingTransaction(
    sourceTxid: string,
    sourceOutputIndex: number,
    recipientAddress: string,
    amount: number,
    privateKey: PrivateKey
  ): Promise<Transaction> {
    // Get source transaction (already SPV verified)
    const sourceTransaction = this.transactions.get(sourceTxid);
    if (!sourceTransaction) {
      throw new Error(`Source transaction ${sourceTxid} not found`);
    }

    // Create new transaction
    const tx = new Transaction();

    // Add input from verified source
    tx.addInput({
      sourceTransaction,  // Already has merkle proof attached
      sourceOutputIndex,
      unlockingScriptTemplate: new P2PKH().unlock(privateKey)
    });

    // Add payment output
    tx.addOutput({
      lockingScript: new P2PKH().lock(recipientAddress),
      satoshis: amount
    });

    // Add change output
    tx.addOutput({
      lockingScript: new P2PKH().lock(privateKey.toPublicKey().toAddress()),
      change: true
    });

    // Calculate fee and sign
    await tx.fee();
    await tx.sign();

    return tx;
  }

  /**
   * Get all verified transactions
   */
  getAllTransactions(): Transaction[] {
    return Array.from(this.transactions.values());
  }

  /**
   * Get transactions by block height
   */
  getTransactionsByHeight(height: number): Transaction[] {
    return this.getAllTransactions().filter(
      tx => tx.merklePath?.blockHeight === height
    );
  }
}

// Usage example
const chainTracker = new WhatsOnChainTracker('main');
const spvChain = new SPVTransactionChain(chainTracker);

// Add verified transactions
await spvChain.addTransaction(
  '0100000001...',           // TX1 hex
  'fed7c509000a02...'        // TX1 merkle proof
);

await spvChain.addTransaction(
  '0100000002...',           // TX2 hex
  'abc12345000b03...'        // TX2 merkle proof
);

// Create new transaction spending verified UTXO
const privKey = PrivateKey.fromWif('L5EY...');
const newTx = await spvChain.createSpendingTransaction(
  'abc123...',                              // Source TXID
  0,                                        // Output index
  '1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa',  // Recipient
  5000,                                     // Amount in satoshis
  privKey
);

// Broadcast new transaction
const arc = new ARC('https://api.taal.com/arc', {
  apiKey: 'mainnet_xxx'
});
await newTx.broadcast(arc);
```

### 3. Merkle Proof Caching and Storage

```typescript
import { MerklePath, Transaction } from '@bsv/sdk';

/**
 * Merkle proof cache for performance optimization
 */
class MerkleProofCache {
  private cache: Map<string, { merklePath: MerklePath; timestamp: number }>;
  private maxAge: number; // Cache TTL in milliseconds

  constructor(maxAge: number = 24 * 60 * 60 * 1000) { // Default: 24 hours
    this.cache = new Map();
    this.maxAge = maxAge;
  }

  /**
   * Store merkle proof in cache
   */
  set(txid: string, merklePath: MerklePath): void {
    this.cache.set(txid, {
      merklePath,
      timestamp: Date.now()
    });
    console.log(`Cached merkle proof for ${txid}`);
  }

  /**
   * Retrieve merkle proof from cache
   */
  get(txid: string): MerklePath | undefined {
    const cached = this.cache.get(txid);

    if (!cached) {
      return undefined;
    }

    // Check if cache entry is still valid
    const age = Date.now() - cached.timestamp;
    if (age > this.maxAge) {
      console.log(`Cache expired for ${txid}`);
      this.cache.delete(txid);
      return undefined;
    }

    return cached.merklePath;
  }

  /**
   * Check if proof exists in cache
   */
  has(txid: string): boolean {
    return this.get(txid) !== undefined;
  }

  /**
   * Clear expired cache entries
   */
  clearExpired(): number {
    const now = Date.now();
    let cleared = 0;

    for (const [txid, cached] of this.cache.entries()) {
      if (now - cached.timestamp > this.maxAge) {
        this.cache.delete(txid);
        cleared++;
      }
    }

    console.log(`Cleared ${cleared} expired cache entries`);
    return cleared;
  }

  /**
   * Clear all cache entries
   */
  clear(): void {
    this.cache.clear();
    console.log('Cache cleared');
  }

  /**
   * Get cache statistics
   */
  getStats(): { size: number; entries: number } {
    let totalSize = 0;

    for (const cached of this.cache.values()) {
      totalSize += cached.merklePath.toBinary().length;
    }

    return {
      size: totalSize,
      entries: this.cache.size
    };
  }

  /**
   * Persist cache to storage
   */
  toJSON(): any {
    const entries: any[] = [];

    for (const [txid, cached] of this.cache.entries()) {
      entries.push({
        txid,
        merklePathHex: cached.merklePath.toHex(),
        timestamp: cached.timestamp
      });
    }

    return { entries };
  }

  /**
   * Load cache from storage
   */
  static fromJSON(json: any): MerkleProofCache {
    const cache = new MerkleProofCache();

    for (const entry of json.entries) {
      cache.cache.set(entry.txid, {
        merklePath: MerklePath.fromHex(entry.merklePathHex),
        timestamp: entry.timestamp
      });
    }

    return cache;
  }
}

/**
 * Merkle proof manager with network fetching
 */
class MerkleProofManager {
  private cache: MerkleProofCache;
  private baseUrl: string;

  constructor(baseUrl: string = 'https://api.whatsonchain.com/v1/bsv/main') {
    this.cache = new MerkleProofCache();
    this.baseUrl = baseUrl;
  }

  /**
   * Get merkle proof for a transaction (cache-first)
   */
  async getMerkleProof(txid: string): Promise<MerklePath> {
    // Check cache first
    const cached = this.cache.get(txid);
    if (cached) {
      console.log(`Using cached merkle proof for ${txid}`);
      return cached;
    }

    // Fetch from network
    console.log(`Fetching merkle proof for ${txid}...`);
    const response = await fetch(`${this.baseUrl}/tx/${txid}/proof`);

    if (!response.ok) {
      throw new Error(`Failed to fetch merkle proof: ${response.statusText}`);
    }

    const proofData = await response.json();
    const merklePath = MerklePath.fromHex(proofData.proof);

    // Store in cache
    this.cache.set(txid, merklePath);

    return merklePath;
  }

  /**
   * Attach merkle proof to transaction
   */
  async attachProof(tx: Transaction): Promise<Transaction> {
    const txid = tx.id('hex');
    const merklePath = await this.getMerkleProof(txid);
    tx.merklePath = merklePath;
    return tx;
  }

  /**
   * Save cache to file/storage
   */
  saveCache(): string {
    return JSON.stringify(this.cache.toJSON());
  }

  /**
   * Load cache from file/storage
   */
  loadCache(json: string): void {
    this.cache = MerkleProofCache.fromJSON(JSON.parse(json));
  }
}

// Usage
const proofManager = new MerkleProofManager();

// Get merkle proof (uses cache if available)
const merklePath = await proofManager.getMerkleProof('abc123...');

// Attach proof to transaction
const tx = Transaction.fromHex('0100000001...');
await proofManager.attachProof(tx);

// Save cache for later use
const cacheData = proofManager.saveCache();
localStorage.setItem('merkleProofCache', cacheData);

// Load cache
const savedCache = localStorage.getItem('merkleProofCache');
if (savedCache) {
  proofManager.loadCache(savedCache);
}
```

## Security Considerations

### 1. Always Verify Merkle Roots

**BAD - Trusting merkle proofs without verification:**
```typescript
// DANGEROUS: Accepting merkle proof without verification
const tx = Transaction.fromHex(txHex);
const merklePath = MerklePath.fromHex(untrustedProofHex);
tx.merklePath = merklePath;  // No verification!
// This transaction could have a fake merkle proof
```

**GOOD - Always verify against blockchain:**
```typescript
// SECURE: Verify merkle root against blockchain
const tx = Transaction.fromHex(txHex);
const merklePath = MerklePath.fromHex(untrustedProofHex);

// Compute merkle root
const txid = tx.id('hex');
const merkleRoot = Buffer.from(merklePath.computeRoot(txid)).toString('hex');

// Verify against trusted chain tracker
const chainTracker = new WhatsOnChainTracker();
const isValid = await chainTracker.isValidRootForHeight(
  merkleRoot,
  merklePath.blockHeight
);

if (!isValid) {
  throw new Error('Invalid merkle proof - possible forgery attempt');
}

tx.merklePath = merklePath;  // Now safe to use
```

### 2. Validate ChainTracker Sources

**BAD - Using untrusted chain tracker:**
```typescript
// DANGEROUS: Custom tracker with no validation
class UntrustedTracker implements ChainTracker {
  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    return true;  // Always returns true - completely insecure!
  }
}
```

**GOOD - Use trusted data sources:**
```typescript
// SECURE: Multiple trusted sources with consensus
class MultiSourceChainTracker implements ChainTracker {
  private sources: string[];

  constructor() {
    this.sources = [
      'https://api.whatsonchain.com/v1/bsv/main',
      'https://api.blockchair.com/bitcoin-sv',
      'https://bsvexplorer.io/api'
    ];
  }

  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    const results = await Promise.all(
      this.sources.map(async (source) => {
        try {
          const response = await fetch(`${source}/block/height/${height}`);
          const block = await response.json();
          return block.merkleroot === root;
        } catch {
          return false;
        }
      })
    );

    // Require majority consensus
    const validCount = results.filter(r => r).length;
    return validCount > this.sources.length / 2;
  }
}
```

### 3. Protect Against Merkle Path Manipulation

```typescript
// Validate merkle path structure before using
function validateMerklePath(merklePath: MerklePath): boolean {
  // Check block height is reasonable
  if (merklePath.blockHeight < 0 || merklePath.blockHeight > 10_000_000) {
    console.error('Invalid block height:', merklePath.blockHeight);
    return false;
  }

  // Check path depth is reasonable (BSV blocks typically < 2^20 transactions)
  if (merklePath.path.length > 20) {
    console.error('Suspicious path depth:', merklePath.path.length);
    return false;
  }

  // Validate each level has proper structure
  for (let i = 0; i < merklePath.path.length; i++) {
    const level = merklePath.path[i];

    if (!Array.isArray(level) || level.length === 0) {
      console.error('Invalid path level:', i);
      return false;
    }

    for (const node of level) {
      if (typeof node.offset !== 'number' || !node.hash) {
        console.error('Invalid node structure at level', i);
        return false;
      }

      if (node.hash.length !== 32) {
        console.error('Invalid hash length at level', i);
        return false;
      }
    }
  }

  return true;
}
```

### 4. Handle Merkle Proof Errors Gracefully

```typescript
async function safelyVerifyMerkleProof(
  txHex: string,
  merklePathHex: string,
  chainTracker: ChainTracker
): Promise<{ valid: boolean; error?: string }> {
  try {
    // Parse with validation
    const tx = Transaction.fromHex(txHex);
    const merklePath = MerklePath.fromHex(merklePathHex);

    // Validate structure
    if (!validateMerklePath(merklePath)) {
      return { valid: false, error: 'Invalid merkle path structure' };
    }

    // Compute merkle root
    const txid = tx.id('hex');
    const merkleRoot = Buffer.from(merklePath.computeRoot(txid)).toString('hex');

    // Verify with timeout
    const timeoutPromise = new Promise<boolean>((_, reject) =>
      setTimeout(() => reject(new Error('Verification timeout')), 30000)
    );

    const verifyPromise = chainTracker.isValidRootForHeight(
      merkleRoot,
      merklePath.blockHeight
    );

    const isValid = await Promise.race([verifyPromise, timeoutPromise]);

    return { valid: isValid };
  } catch (error) {
    return {
      valid: false,
      error: error instanceof Error ? error.message : 'Unknown error'
    };
  }
}
```

## Performance Considerations

### 1. Cache Merkle Proofs

Merkle proofs don't change once a transaction is confirmed, so aggressive caching is beneficial:

```typescript
class OptimizedMerkleProofManager {
  private memoryCache: Map<string, MerklePath>;
  private persistentStorage: Storage;

  constructor(storage: Storage = localStorage) {
    this.memoryCache = new Map();
    this.persistentStorage = storage;
    this.loadFromStorage();
  }

  async getMerkleProof(txid: string): Promise<MerklePath> {
    // 1. Check memory cache (fastest)
    if (this.memoryCache.has(txid)) {
      return this.memoryCache.get(txid)!;
    }

    // 2. Check persistent storage
    const stored = this.persistentStorage.getItem(`merkle_${txid}`);
    if (stored) {
      const merklePath = MerklePath.fromHex(stored);
      this.memoryCache.set(txid, merklePath);
      return merklePath;
    }

    // 3. Fetch from network
    const merklePath = await this.fetchFromNetwork(txid);

    // Store in both caches
    this.memoryCache.set(txid, merklePath);
    this.persistentStorage.setItem(`merkle_${txid}`, merklePath.toHex());

    return merklePath;
  }

  private async fetchFromNetwork(txid: string): Promise<MerklePath> {
    const response = await fetch(
      `https://api.whatsonchain.com/v1/bsv/main/tx/${txid}/proof`
    );
    const data = await response.json();
    return MerklePath.fromHex(data.proof);
  }

  private loadFromStorage(): void {
    // Load all cached proofs into memory on startup
    for (let i = 0; i < this.persistentStorage.length; i++) {
      const key = this.persistentStorage.key(i);
      if (key?.startsWith('merkle_')) {
        const txid = key.substring(7);
        const hex = this.persistentStorage.getItem(key);
        if (hex) {
          this.memoryCache.set(txid, MerklePath.fromHex(hex));
        }
      }
    }
  }
}
```

### 2. Batch Merkle Proof Fetching

```typescript
class BatchMerkleProofFetcher {
  private batchSize: number = 20;
  private pendingRequests: Map<string, Promise<MerklePath>>;

  constructor() {
    this.pendingRequests = new Map();
  }

  async getMerkleProofs(txids: string[]): Promise<Map<string, MerklePath>> {
    const results = new Map<string, MerklePath>();

    // Process in batches
    for (let i = 0; i < txids.length; i += this.batchSize) {
      const batch = txids.slice(i, i + this.batchSize);

      // Fetch batch in parallel
      const batchResults = await Promise.all(
        batch.map(txid => this.fetchMerkleProof(txid))
      );

      // Store results
      batch.forEach((txid, index) => {
        results.set(txid, batchResults[index]);
      });
    }

    return results;
  }

  private async fetchMerkleProof(txid: string): Promise<MerklePath> {
    // Deduplicate concurrent requests for same txid
    if (this.pendingRequests.has(txid)) {
      return this.pendingRequests.get(txid)!;
    }

    const promise = this.doFetch(txid);
    this.pendingRequests.set(txid, promise);

    try {
      const result = await promise;
      return result;
    } finally {
      this.pendingRequests.delete(txid);
    }
  }

  private async doFetch(txid: string): Promise<MerklePath> {
    const response = await fetch(
      `https://api.whatsonchain.com/v1/bsv/main/tx/${txid}/proof`
    );
    const data = await response.json();
    return MerklePath.fromHex(data.proof);
  }
}
```

### 3. Optimize Merkle Root Computation

```typescript
// Pre-compute and cache merkle roots
class MerkleRootCache {
  private cache: Map<string, Uint8Array>;

  constructor() {
    this.cache = new Map();
  }

  computeRoot(merklePath: MerklePath, txid: string): Uint8Array {
    const cacheKey = `${txid}_${merklePath.blockHeight}`;

    // Return cached result if available
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey)!;
    }

    // Compute and cache
    const root = merklePath.computeRoot(txid);
    this.cache.set(cacheKey, root);

    return root;
  }

  clear(): void {
    this.cache.clear();
  }
}
```

## Related Components

- **[Transaction](../transaction/README.md)** - Attach merkle proofs to transactions for SPV
- **[SPV](../spv/README.md)** - SPV verification using merkle proofs and chain tracking
- **[BEEF](../beef/README.md)** - Transaction envelopes with merkle proofs (BRC-62)
- **[ARC](../arc/README.md)** - Broadcasting transactions with merkle proof callbacks
- **[Script](../script/README.md)** - Transaction scripts verified through merkle proofs

## Best Practices

### 1. Always Verify Merkle Proofs

Never trust merkle proofs without verification against the blockchain:

```typescript
// GOOD: Always verify before trusting
async function trustMerkleProof(
  tx: Transaction,
  merklePath: MerklePath
): Promise<boolean> {
  const chainTracker = new WhatsOnChainTracker();
  const txid = tx.id('hex');
  const merkleRoot = Buffer.from(merklePath.computeRoot(txid)).toString('hex');

  return await chainTracker.isValidRootForHeight(
    merkleRoot,
    merklePath.blockHeight
  );
}
```

### 2. Use Merkle Proofs for Source Transactions

Always attach merkle proofs when using previous transactions as inputs:

```typescript
// GOOD: Attach merkle proof for SPV verification
const sourceTransaction = Transaction.fromHex(sourceTxHex);
const merklePath = await getMerkleProof(sourceTransaction.id('hex'));
sourceTransaction.merklePath = merklePath;

// Now can be used as SPV-verified source
tx.addInput({
  sourceTransaction,  // Includes merkle proof
  sourceOutputIndex: 0,
  unlockingScriptTemplate: new P2PKH().unlock(privKey)
});
```

### 3. Implement Robust ChainTrackers

Use multiple data sources for merkle root verification:

```typescript
// GOOD: Multi-source verification with fallback
class RobustChainTracker implements ChainTracker {
  private primarySource: string;
  private fallbackSources: string[];

  constructor() {
    this.primarySource = 'https://api.whatsonchain.com/v1/bsv/main';
    this.fallbackSources = [
      'https://api.blockchair.com/bitcoin-sv',
      'https://bsvexplorer.io/api'
    ];
  }

  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    // Try primary source first
    try {
      return await this.checkSource(this.primarySource, root, height);
    } catch (error) {
      console.warn('Primary source failed, trying fallbacks');
    }

    // Try fallback sources
    for (const source of this.fallbackSources) {
      try {
        return await this.checkSource(source, root, height);
      } catch {
        continue;
      }
    }

    throw new Error('All chain tracker sources failed');
  }

  private async checkSource(
    source: string,
    root: string,
    height: number
  ): Promise<boolean> {
    const response = await fetch(`${source}/block/height/${height}`);
    const block = await response.json();
    return block.merkleroot === root;
  }
}
```

### 4. Cache Merkle Proofs Aggressively

Merkle proofs are immutable once confirmed, so cache them indefinitely:

```typescript
// GOOD: Persistent caching with no expiration
class PersistentMerkleCache {
  async getMerkleProof(txid: string): Promise<MerklePath> {
    // Check IndexedDB/localStorage first
    const cached = await this.getFromStorage(txid);
    if (cached) {
      return cached;
    }

    // Fetch from network
    const merklePath = await this.fetchFromNetwork(txid);

    // Store permanently (confirmed merkle proofs never change)
    await this.saveToStorage(txid, merklePath);

    return merklePath;
  }

  private async getFromStorage(txid: string): Promise<MerklePath | null> {
    const hex = localStorage.getItem(`merkle_${txid}`);
    return hex ? MerklePath.fromHex(hex) : null;
  }

  private async saveToStorage(txid: string, merklePath: MerklePath): Promise<void> {
    localStorage.setItem(`merkle_${txid}`, merklePath.toHex());
  }

  private async fetchFromNetwork(txid: string): Promise<MerklePath> {
    const response = await fetch(
      `https://api.whatsonchain.com/v1/bsv/main/tx/${txid}/proof`
    );
    const data = await response.json();
    return MerklePath.fromHex(data.proof);
  }
}
```

### 5. Validate Merkle Path Structure

Always validate the structure of merkle paths before using them:

```typescript
// GOOD: Validate before trusting
function validateAndUseMerklePath(merklePathHex: string): MerklePath {
  const merklePath = MerklePath.fromHex(merklePathHex);

  // Validate block height
  if (merklePath.blockHeight < 0 || merklePath.blockHeight > 10_000_000) {
    throw new Error('Invalid block height');
  }

  // Validate path depth
  if (merklePath.path.length > 20) {
    throw new Error('Suspicious path depth');
  }

  // Validate each level
  for (const level of merklePath.path) {
    if (!Array.isArray(level) || level.length === 0) {
      throw new Error('Invalid path structure');
    }

    for (const node of level) {
      if (node.hash.length !== 32) {
        throw new Error('Invalid hash length');
      }
    }
  }

  return merklePath;
}
```

## Troubleshooting

### Invalid Merkle Root

**Problem:** Computed merkle root doesn't match blockchain.

**Solution:**
```typescript
// Check if transaction ID is correct
const txid = tx.id('hex');
console.log('TXID:', txid);

// Verify merkle path was parsed correctly
console.log('Block height:', merklePath.blockHeight);
console.log('Path depth:', merklePath.path.length);

// Compute root manually
const merkleRoot = Buffer.from(merklePath.computeRoot(txid)).toString('hex');
console.log('Computed root:', merkleRoot);

// Fetch expected root from blockchain
const response = await fetch(
  `https://api.whatsonchain.com/v1/bsv/main/block/height/${merklePath.blockHeight}`
);
const block = await response.json();
console.log('Expected root:', block.merkleroot);

// Compare
if (merkleRoot !== block.merkleroot) {
  console.error('Merkle root mismatch - merkle path may be invalid or for different transaction');
}
```

### ChainTracker Failures

**Problem:** ChainTracker always returns false or throws errors.

**Solution:**
```typescript
// Test chain tracker separately
const chainTracker = new WhatsOnChainTracker();

// Test with known block
const testResult = await chainTracker.isValidRootForHeight(
  '000000000000000000000000000000000000000000000000000000000000000a',
  825000
);

console.log('Chain tracker test result:', testResult);

// Check network connectivity
try {
  const response = await fetch('https://api.whatsonchain.com/v1/bsv/main/chain/info');
  const chainInfo = await response.json();
  console.log('Chain info:', chainInfo);
} catch (error) {
  console.error('Network error:', error);
}

// Implement retry logic
async function retryChainTracker(
  root: string,
  height: number,
  maxRetries: number = 3
): Promise<boolean> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await chainTracker.isValidRootForHeight(root, height);
    } catch (error) {
      console.warn(`Retry ${i + 1}/${maxRetries}:`, error);
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
  }
  throw new Error('ChainTracker failed after retries');
}
```

### Merkle Path Parsing Errors

**Problem:** Cannot parse merkle path from hex string.

**Solution:**
```typescript
try {
  const merklePath = MerklePath.fromHex(merklePathHex);
} catch (error) {
  console.error('Failed to parse merkle path:', error);

  // Validate hex string
  if (!/^[0-9a-fA-F]+$/.test(merklePathHex)) {
    console.error('Invalid hex characters in merkle path');
  }

  // Check length
  if (merklePathHex.length % 2 !== 0) {
    console.error('Merkle path hex has odd length');
  }

  // Try parsing binary
  try {
    const binary = Array.from(Buffer.from(merklePathHex, 'hex'));
    const merklePath = MerklePath.fromBinary(binary);
    console.log('Successfully parsed from binary');
  } catch (binaryError) {
    console.error('Binary parsing also failed:', binaryError);
  }
}
```

### Performance Issues with Large Chains

**Problem:** Slow performance when verifying many merkle proofs.

**Solution:**
```typescript
// Batch verification with parallel processing
async function batchVerifyMerkleProofs(
  transactions: Array<{ tx: Transaction; merklePath: MerklePath }>
): Promise<Map<string, boolean>> {
  const results = new Map<string, boolean>();
  const chainTracker = new WhatsOnChainTracker();

  // Process in parallel batches of 10
  const batchSize = 10;

  for (let i = 0; i < transactions.length; i += batchSize) {
    const batch = transactions.slice(i, i + batchSize);

    const batchResults = await Promise.all(
      batch.map(async ({ tx, merklePath }) => {
        const txid = tx.id('hex');
        const root = Buffer.from(merklePath.computeRoot(txid)).toString('hex');
        const isValid = await chainTracker.isValidRootForHeight(
          root,
          merklePath.blockHeight
        );
        return { txid, isValid };
      })
    );

    batchResults.forEach(({ txid, isValid }) => {
      results.set(txid, isValid);
    });

    console.log(`Verified ${Math.min(i + batchSize, transactions.length)}/${transactions.length}`);
  }

  return results;
}
```

## Further Reading

- **[BRC-9: Simplified Payment Verification](https://github.com/bitcoin-sv/BRCs/blob/master/peer-to-peer/0009.md)** - SPV implementation specification
- **[BRC-67: SPV Validation Rules](https://github.com/bitcoin-sv/BRCs/blob/master/payments/0067.md)** - SPV validation requirements
- **[BRC-62: BEEF Format](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0062.md)** - Transaction envelopes with merkle proofs
- **[BRC-8: Transaction Envelopes](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0008.md)** - Background exchange of SPV proofs
- **[BSV SDK Documentation](https://docs.bsvblockchain.org/)** - Official SDK documentation
- **[Bitcoin Merkle Trees](https://wiki.bitcoinsv.io/index.php/Merkle_tree)** - Understanding merkle tree structure

## Status

✅ **Complete** - Comprehensive documentation with TSC format, SPV verification, and merkle path management patterns.
