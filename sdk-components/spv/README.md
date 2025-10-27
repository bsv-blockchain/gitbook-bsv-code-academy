# SPV (Simplified Payment Verification)

## Overview

SPV (Simplified Payment Verification) enables lightweight blockchain clients to verify transactions without downloading the entire blockchain. The BSV SDK implements BRC-9 and BRC-67 standards, providing robust SPV verification through the `ChainTracker` interface, `MerklePath` class, and header validation mechanisms.

SPV allows applications to verify that a transaction is included in a block by checking the merkle proof against the block header's merkle root, requiring only block headers rather than full block data.

## Purpose

SPV verification solves critical challenges for BSV applications:

- **Lightweight Verification**: Verify transactions without full blockchain download
- **Trustless Operation**: Verify transaction inclusion cryptographically without trusting third parties
- **Scalability**: Support massive transaction throughput with minimal resource requirements
- **Header Chain Validation**: Validate proof-of-work and block header chains
- **Merkle Root Verification**: Confirm transactions are included in valid blocks

This enables mobile wallets, IoT devices, and web applications to securely interact with BSV blockchain.

## Basic Usage

### Verify Transaction with Merkle Proof

```typescript
import { MerklePath, Transaction, ChainTracker } from '@bsv/sdk';

// Parse merkle proof from hex
const merklePath = MerklePath.fromHex('fed7c509000a02fddd01...');

// Get the transaction
const tx = Transaction.fromHex('...');

// Attach merkle proof to transaction
tx.merklePath = merklePath;

// Verify the transaction is in the merkle tree
const txid = tx.id('hex');
const computedRoot = merklePath.computeRoot(txid);

console.log('Computed merkle root:', computedRoot);
console.log('Block height:', merklePath.blockHeight);
```

### Implement ChainTracker

```typescript
import { ChainTracker } from '@bsv/sdk';

class WhatsOnChainTracker implements ChainTracker {
  private network: string;

  constructor(network: string = 'main') {
    this.network = network;
  }

  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    try {
      const response = await fetch(
        `https://api.whatsonchain.com/v1/bsv/${this.network}/block/height/${height}`
      );

      const data = await response.json();

      // Compare merkle roots
      return data[0].merkleroot === root;
    } catch (error) {
      console.error('Error validating root:', error);
      return false;
    }
  }
}

// Usage
const tracker = new WhatsOnChainTracker('main');
const isValid = await tracker.isValidRootForHeight(
  'a1b2c3d4...',
  800000
);

console.log('Root valid:', isValid);
```

### Verify SPV Proof

```typescript
import { MerklePath, Transaction, ChainTracker } from '@bsv/sdk';

async function verifySPVProof(
  tx: Transaction,
  merklePath: MerklePath,
  chainTracker: ChainTracker
): Promise<boolean> {
  // Compute merkle root from transaction and path
  const txid = tx.id('hex');
  const computedRoot = merklePath.computeRoot(txid);

  // Verify root matches block header at specified height
  const blockHeight = merklePath.blockHeight;

  if (blockHeight === undefined) {
    throw new Error('Merkle path missing block height');
  }

  const isValid = await chainTracker.isValidRootForHeight(
    computedRoot,
    blockHeight
  );

  return isValid;
}

// Usage
const tx = Transaction.fromHex('...');
const proof = MerklePath.fromHex('...');
const tracker = new WhatsOnChainTracker();

const verified = await verifySPVProof(tx, proof, tracker);
console.log('SPV proof verified:', verified);
```

### Validate Block Headers

```typescript
import { BlockHeader } from '@bsv/sdk';

class BlockHeaderValidator {
  // Validate proof-of-work
  static validateProofOfWork(header: BlockHeader): boolean {
    const hash = header.hash;
    const target = header.bits;

    // Check hash meets difficulty target
    return BigInt('0x' + hash) < this.getTargetFromBits(target);
  }

  // Validate header chain
  static validateHeaderChain(headers: BlockHeader[]): boolean {
    for (let i = 1; i < headers.length; i++) {
      const prev = headers[i - 1];
      const curr = headers[i];

      // Check previous hash matches
      if (curr.prevHash !== prev.hash) {
        return false;
      }

      // Check proof-of-work
      if (!this.validateProofOfWork(curr)) {
        return false;
      }

      // Check timestamp ordering
      if (curr.time <= prev.time) {
        return false;
      }
    }

    return true;
  }

  private static getTargetFromBits(bits: number): bigint {
    // Convert compact bits format to target
    const exponent = bits >> 24;
    const mantissa = bits & 0xffffff;
    return BigInt(mantissa) * (BigInt(256) ** BigInt(exponent - 3));
  }
}
```

## Key Features

### 1. ChainTracker Interface

Implement custom chain trackers for merkle root verification:

```typescript
import { ChainTracker } from '@bsv/sdk';

// ChainTracker interface
interface ChainTracker {
  isValidRootForHeight(root: string, height: number): Promise<boolean>;
}

// In-memory chain tracker (for testing)
class InMemoryChainTracker implements ChainTracker {
  private knownRoots: Map<number, string> = new Map();

  addBlockRoot(height: number, root: string): void {
    this.knownRoots.set(height, root);
  }

  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    const knownRoot = this.knownRoots.get(height);
    return knownRoot === root;
  }
}

// API-based chain tracker
class APIChainTracker implements ChainTracker {
  private apiUrl: string;
  private cache: Map<number, string> = new Map();

  constructor(apiUrl: string) {
    this.apiUrl = apiUrl;
  }

  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    // Check cache first
    if (this.cache.has(height)) {
      return this.cache.get(height) === root;
    }

    // Fetch from API
    try {
      const response = await fetch(`${this.apiUrl}/block/${height}/header`);
      const data = await response.json();

      // Cache the result
      this.cache.set(height, data.merkleroot);

      return data.merkleroot === root;
    } catch (error) {
      return false;
    }
  }

  clearCache(): void {
    this.cache.clear();
  }
}

// Node-based chain tracker (connects to BSV node)
class NodeChainTracker implements ChainTracker {
  private rpcUrl: string;
  private rpcUser: string;
  private rpcPassword: string;

  constructor(rpcUrl: string, rpcUser: string, rpcPassword: string) {
    this.rpcUrl = rpcUrl;
    this.rpcUser = rpcUser;
    this.rpcPassword = rpcPassword;
  }

  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    const blockHash = await this.getBlockHashAtHeight(height);
    const header = await this.getBlockHeader(blockHash);

    return header.merkleroot === root;
  }

  private async rpcCall(method: string, params: any[]): Promise<any> {
    const response = await fetch(this.rpcUrl, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Basic ' + Buffer.from(
          `${this.rpcUser}:${this.rpcPassword}`
        ).toString('base64')
      },
      body: JSON.stringify({
        jsonrpc: '2.0',
        id: 'spv',
        method,
        params
      })
    });

    const data = await response.json();
    return data.result;
  }

  private async getBlockHashAtHeight(height: number): Promise<string> {
    return await this.rpcCall('getblockhash', [height]);
  }

  private async getBlockHeader(blockHash: string): Promise<any> {
    return await this.rpcCall('getblockheader', [blockHash]);
  }
}
```

### 2. Merkle Path Verification

Verify transactions using merkle paths per BRC-9:

```typescript
import { MerklePath, Transaction } from '@bsv/sdk';

class MerklePathVerifier {
  // Verify merkle path for transaction
  static verify(
    tx: Transaction,
    merklePath: MerklePath,
    expectedRoot: string
  ): boolean {
    const txid = tx.id('hex');
    const computedRoot = merklePath.computeRoot(txid);

    return computedRoot === expectedRoot;
  }

  // Verify merkle path with chain tracker
  static async verifyWithChainTracker(
    tx: Transaction,
    merklePath: MerklePath,
    chainTracker: ChainTracker
  ): Promise<boolean> {
    const txid = tx.id('hex');
    const computedRoot = merklePath.computeRoot(txid);
    const blockHeight = merklePath.blockHeight;

    if (blockHeight === undefined) {
      throw new Error('Missing block height in merkle path');
    }

    return await chainTracker.isValidRootForHeight(computedRoot, blockHeight);
  }

  // Verify multiple transactions
  static async verifyBatch(
    transactions: Array<{
      tx: Transaction;
      merklePath: MerklePath;
    }>,
    chainTracker: ChainTracker
  ): Promise<boolean[]> {
    const verifications = transactions.map(({ tx, merklePath }) =>
      this.verifyWithChainTracker(tx, merklePath, chainTracker)
    );

    return await Promise.all(verifications);
  }

  // Extract merkle path information
  static getMerklePathInfo(merklePath: MerklePath): {
    blockHeight?: number;
    path: Array<{ offset: number; hash?: string }>;
    txIndex?: number;
  } {
    return {
      blockHeight: merklePath.blockHeight,
      path: merklePath.path,
      txIndex: merklePath.txIndex
    };
  }
}

// Usage
const tx = Transaction.fromHex('...');
const merklePath = MerklePath.fromHex('...');
const chainTracker = new WhatsOnChainTracker();

// Simple verification
const isValid = MerklePathVerifier.verify(
  tx,
  merklePath,
  'expectedRootHash'
);

// Verification with chain tracker
const verified = await MerklePathVerifier.verifyWithChainTracker(
  tx,
  merklePath,
  chainTracker
);

console.log('Transaction verified:', verified);
```

### 3. Transaction SPV Envelope (BRC-8)

Create and verify SPV envelopes for transactions:

```typescript
import { Transaction, MerklePath, ChainTracker } from '@bsv/sdk';

class SPVEnvelope {
  transaction: Transaction;
  merklePath?: MerklePath;
  rawTx: string;

  constructor(tx: Transaction) {
    this.transaction = tx;
    this.merklePath = tx.merklePath;
    this.rawTx = tx.toHex();
  }

  // Create envelope from transaction with proof
  static fromTransaction(
    tx: Transaction,
    merklePath?: MerklePath
  ): SPVEnvelope {
    const envelope = new SPVEnvelope(tx);

    if (merklePath) {
      envelope.merklePath = merklePath;
      tx.merklePath = merklePath;
    }

    return envelope;
  }

  // Serialize envelope
  toJSON(): {
    rawTx: string;
    proof?: string;
    blockHeight?: number;
  } {
    return {
      rawTx: this.rawTx,
      proof: this.merklePath?.toHex(),
      blockHeight: this.merklePath?.blockHeight
    };
  }

  // Deserialize envelope
  static fromJSON(json: {
    rawTx: string;
    proof?: string;
    blockHeight?: number;
  }): SPVEnvelope {
    const tx = Transaction.fromHex(json.rawTx);
    const merklePath = json.proof ? MerklePath.fromHex(json.proof) : undefined;

    return SPVEnvelope.fromTransaction(tx, merklePath);
  }

  // Verify envelope
  async verify(chainTracker: ChainTracker): Promise<boolean> {
    if (!this.merklePath) {
      throw new Error('No merkle proof attached to envelope');
    }

    const txid = this.transaction.id('hex');
    const computedRoot = this.merklePath.computeRoot(txid);
    const blockHeight = this.merklePath.blockHeight;

    if (blockHeight === undefined) {
      throw new Error('Missing block height');
    }

    return await chainTracker.isValidRootForHeight(computedRoot, blockHeight);
  }

  // Check if envelope has valid proof
  hasValidProof(): boolean {
    return this.merklePath !== undefined &&
           this.merklePath.blockHeight !== undefined;
  }
}

// Usage
const tx = Transaction.fromHex('...');
const merklePath = MerklePath.fromHex('...');

// Create envelope
const envelope = SPVEnvelope.fromTransaction(tx, merklePath);

// Serialize
const json = envelope.toJSON();
console.log('Envelope:', JSON.stringify(json));

// Deserialize
const restored = SPVEnvelope.fromJSON(json);

// Verify
const chainTracker = new WhatsOnChainTracker();
const isValid = await restored.verify(chainTracker);
console.log('Envelope valid:', isValid);
```

### 4. Header Chain Validation

Validate block header chains and proof-of-work:

```typescript
import { BlockHeader } from '@bsv/sdk';

class HeaderChainValidator {
  private headers: Map<string, BlockHeader> = new Map();
  private heightToHash: Map<number, string> = new Map();

  // Add header to chain
  addHeader(header: BlockHeader): boolean {
    // Validate proof-of-work
    if (!this.validateProofOfWork(header)) {
      return false;
    }

    // If not genesis, validate previous header exists
    if (header.prevHash !== '0'.repeat(64)) {
      const prevHeader = this.headers.get(header.prevHash);

      if (!prevHeader) {
        throw new Error('Previous header not found');
      }

      // Validate timestamp
      if (header.time <= prevHeader.time) {
        return false;
      }
    }

    // Store header
    const hash = header.hash;
    this.headers.set(hash, header);

    // Calculate height (simplified - in production track properly)
    const prevHeight = this.getHeightForHash(header.prevHash) || -1;
    this.heightToHash.set(prevHeight + 1, hash);

    return true;
  }

  // Validate proof-of-work
  private validateProofOfWork(header: BlockHeader): boolean {
    const hash = header.hash;
    const target = this.getTargetFromBits(header.bits);

    // Hash must be less than target
    const hashNum = BigInt('0x' + hash);

    return hashNum < target;
  }

  // Convert compact bits to target
  private getTargetFromBits(bits: number): bigint {
    const exponent = bits >> 24;
    const mantissa = bits & 0xffffff;

    return BigInt(mantissa) * (BigInt(256) ** BigInt(exponent - 3));
  }

  // Get header by hash
  getHeader(hash: string): BlockHeader | undefined {
    return this.headers.get(hash);
  }

  // Get header by height
  getHeaderAtHeight(height: number): BlockHeader | undefined {
    const hash = this.heightToHash.get(height);
    return hash ? this.headers.get(hash) : undefined;
  }

  // Get height for hash
  private getHeightForHash(hash: string): number | undefined {
    for (const [height, h] of this.heightToHash.entries()) {
      if (h === hash) return height;
    }
    return undefined;
  }

  // Validate chain between two heights
  validateChainSegment(startHeight: number, endHeight: number): boolean {
    for (let height = startHeight + 1; height <= endHeight; height++) {
      const header = this.getHeaderAtHeight(height);
      const prevHeader = this.getHeaderAtHeight(height - 1);

      if (!header || !prevHeader) {
        return false;
      }

      // Validate link
      if (header.prevHash !== prevHeader.hash) {
        return false;
      }

      // Validate PoW
      if (!this.validateProofOfWork(header)) {
        return false;
      }

      // Validate timestamp
      if (header.time <= prevHeader.time) {
        return false;
      }
    }

    return true;
  }

  // Get cumulative work
  getCumulativeWork(height: number): bigint {
    let work = BigInt(0);

    for (let h = 0; h <= height; h++) {
      const header = this.getHeaderAtHeight(h);

      if (header) {
        const target = this.getTargetFromBits(header.bits);
        work += BigInt(2) ** BigInt(256) / target;
      }
    }

    return work;
  }
}

// Usage
const validator = new HeaderChainValidator();

// Add headers sequentially
const header1 = BlockHeader.fromHex('...');
const header2 = BlockHeader.fromHex('...');
const header3 = BlockHeader.fromHex('...');

validator.addHeader(header1);
validator.addHeader(header2);
validator.addHeader(header3);

// Validate chain segment
const isValid = validator.validateChainSegment(0, 2);
console.log('Chain valid:', isValid);

// Get cumulative work
const work = validator.getCumulativeWork(2);
console.log('Cumulative work:', work.toString());
```

## API Reference

### ChainTracker Interface

```typescript
interface ChainTracker {
  isValidRootForHeight(root: string, height: number): Promise<boolean>;
}
```

**Methods:**
- `isValidRootForHeight(root, height)`: Verify merkle root matches block at height

### MerklePath Class

```typescript
class MerklePath {
  blockHeight?: number;
  path: Array<{ offset: number; hash?: string }>;
  txIndex?: number;

  static fromHex(hex: string): MerklePath;
  toHex(): string;
  computeRoot(txid: string): string;
}
```

**Methods:**
- `fromHex(hex)`: Parse merkle path from hex string
- `toHex()`: Serialize merkle path to hex
- `computeRoot(txid)`: Calculate merkle root from txid and path

### BlockHeader Class

```typescript
class BlockHeader {
  version: number;
  prevHash: string;
  merkleRoot: string;
  time: number;
  bits: number;
  nonce: number;
  hash: string;

  static fromHex(hex: string): BlockHeader;
  toHex(): string;
}
```

## Common Patterns

### Pattern 1: SPV Client Implementation

```typescript
import { Transaction, MerklePath, ChainTracker } from '@bsv/sdk';

class SPVClient {
  private chainTracker: ChainTracker;
  private verifiedTransactions: Set<string> = new Set();

  constructor(chainTracker: ChainTracker) {
    this.chainTracker = chainTracker;
  }

  // Verify and track transaction
  async verifyTransaction(
    tx: Transaction,
    merklePath: MerklePath
  ): Promise<boolean> {
    const txid = tx.id('hex');

    // Check if already verified
    if (this.verifiedTransactions.has(txid)) {
      return true;
    }

    // Compute merkle root
    const computedRoot = merklePath.computeRoot(txid);
    const blockHeight = merklePath.blockHeight;

    if (blockHeight === undefined) {
      throw new Error('Missing block height');
    }

    // Verify with chain tracker
    const isValid = await this.chainTracker.isValidRootForHeight(
      computedRoot,
      blockHeight
    );

    if (isValid) {
      this.verifiedTransactions.add(txid);
    }

    return isValid;
  }

  // Check if transaction is verified
  isVerified(txid: string): boolean {
    return this.verifiedTransactions.has(txid);
  }

  // Get verification count
  getVerifiedCount(): number {
    return this.verifiedTransactions.size;
  }
}

// Usage
const spvClient = new SPVClient(new WhatsOnChainTracker());

const tx = Transaction.fromHex('...');
const proof = MerklePath.fromHex('...');

const verified = await spvClient.verifyTransaction(tx, proof);
console.log('Transaction verified:', verified);
console.log('Total verified:', spvClient.getVerifiedCount());
```

### Pattern 2: SPV Proof Service

```typescript
class SPVProofService {
  private apiUrl: string;

  constructor(apiUrl: string) {
    this.apiUrl = apiUrl;
  }

  // Fetch merkle proof for transaction
  async getMerkleProof(txid: string): Promise<MerklePath> {
    const response = await fetch(`${this.apiUrl}/tx/${txid}/proof`);
    const data = await response.json();

    return MerklePath.fromHex(data.proof);
  }

  // Fetch and verify transaction
  async fetchAndVerify(
    txid: string,
    chainTracker: ChainTracker
  ): Promise<{
    transaction: Transaction;
    verified: boolean;
  }> {
    // Fetch transaction
    const txResponse = await fetch(`${this.apiUrl}/tx/${txid}/hex`);
    const txHex = await txResponse.text();
    const tx = Transaction.fromHex(txHex);

    // Fetch proof
    const merklePath = await this.getMerkleProof(txid);

    // Verify
    const computedRoot = merklePath.computeRoot(txid);
    const blockHeight = merklePath.blockHeight!;

    const verified = await chainTracker.isValidRootForHeight(
      computedRoot,
      blockHeight
    );

    return { transaction: tx, verified };
  }
}

// Usage
const proofService = new SPVProofService('https://api.whatsonchain.com/v1/bsv/main');
const chainTracker = new WhatsOnChainTracker();

const result = await proofService.fetchAndVerify(
  'abc123...',
  chainTracker
);

console.log('Verified:', result.verified);
```

### Pattern 3: Multi-Source Chain Tracker

```typescript
class MultiSourceChainTracker implements ChainTracker {
  private trackers: ChainTracker[];
  private requiredConfirmations: number;

  constructor(trackers: ChainTracker[], requiredConfirmations: number = 2) {
    this.trackers = trackers;
    this.requiredConfirmations = requiredConfirmations;
  }

  async isValidRootForHeight(root: string, height: number): Promise<boolean> {
    // Query all trackers
    const results = await Promise.allSettled(
      this.trackers.map(tracker =>
        tracker.isValidRootForHeight(root, height)
      )
    );

    // Count confirmations
    let confirmations = 0;

    for (const result of results) {
      if (result.status === 'fulfilled' && result.value === true) {
        confirmations++;
      }
    }

    return confirmations >= this.requiredConfirmations;
  }
}

// Usage
const multiTracker = new MultiSourceChainTracker([
  new WhatsOnChainTracker(),
  new APIChainTracker('https://api.example.com'),
  new NodeChainTracker('http://localhost:8332', 'user', 'pass')
], 2); // Require 2 out of 3 confirmations

const isValid = await multiTracker.isValidRootForHeight(
  'merkleroot...',
  800000
);
```

## Security Considerations

- **Always verify merkle roots** against trusted chain tracker
- **Validate proof-of-work** for header chains
- **Check block heights** are within expected range
- **Use multiple data sources** for critical verifications
- **Implement caching** with expiration for chain tracker results

## Performance Considerations

- **Cache merkle root lookups** to reduce API calls
- **Batch verification** of multiple transactions
- **Pre-fetch headers** for expected block ranges
- **Use local chain tracker** when full node available

## Related Components

- **[Merkle Proofs](../merkle-proofs/README.md)**: Detailed merkle proof operations
- **[Transactions](../transactions/README.md)**: Transaction structure and SPV attachment
- **[BEEF](../beef/README.md)**: BEEF format for transaction packages with proofs

## Best Practices

1. **Always use ChainTracker** for production verification
2. **Cache verification results** to improve performance
3. **Validate block heights** are reasonable
4. **Use multiple sources** for critical applications
5. **Implement fallback trackers** for reliability

## Troubleshooting

### Issue: Merkle Root Mismatch

**Solution:** Ensure transaction ID is computed correctly:
```typescript
const txid = tx.id('hex'); // Use 'hex' format
const root = merklePath.computeRoot(txid);
```

### Issue: Missing Block Height

**Solution:** Check merkle path includes block height:
```typescript
if (merklePath.blockHeight === undefined) {
  throw new Error('Merkle path must include block height');
}
```

## Further Reading

- **[BRC-9](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0009.md)**: SPV Implementation
- **[BRC-67](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0067.md)**: SPV Validation Rules
- **[BRC-8](https://github.com/bitcoin-sv/BRCs/blob/master/transactions/0008.md)**: Transaction Envelopes

## Status

✅ Complete - Comprehensive SPV documentation with ChainTracker implementation and header validation.
