# SDK Components

This directory contains detailed documentation for each component of the BSV TypeScript SDK. These components are the building blocks for BSV blockchain applications.

## Purpose

SDK Components serve as modular reference documentation that can be:
- Referenced from learning path modules
- Used as quick-reference guides
- Linked from code examples
- Combined to build complete solutions

## Organization

Components are organized by functionality and complexity level. Each component includes:
- Overview and purpose
- API reference
- Usage examples
- Related components
- Common patterns

## Core Components

### Transaction Management
- [Transaction](transaction/README.md) - Creating and managing BSV transactions
- [Transaction Input](transaction-input/README.md) - Working with transaction inputs
- [Transaction Output](transaction-output/README.md) - Working with transaction outputs
- [UTXO Management](utxo-management/README.md) - Managing unspent transaction outputs

### Cryptography & Keys
- [Private Keys](private-keys/README.md) - Key generation and management
- [Public Keys](public-keys/README.md) - Public key derivation
- [Signatures](signatures/README.md) - Digital signatures and verification
- [HD Wallets](hd-wallets/README.md) - Hierarchical Deterministic wallets (BIP32)

### Scripts
- [Script](script/README.md) - Bitcoin Script fundamentals
- [Script Templates](script-templates/README.md) - Common script patterns
- [P2PKH](p2pkh/README.md) - Pay-to-Public-Key-Hash
- [Custom Scripts](custom-scripts/README.md) - Building custom locking scripts

### Blockchain Verification
- [SPV](spv/README.md) - Simplified Payment Verification
- [Merkle Proofs](merkle-proofs/README.md) - Merkle tree verification
- [BEEF Format](beef/README.md) - Background Evaluation Extended Format
- [Chain Tracker](chain-tracker/README.md) - Blockchain state tracking

### Broadcasting & Network
- [ARC](arc/README.md) - Transaction broadcasting with ARC
- [Broadcast](broadcast/README.md) - Transaction broadcast mechanisms
- [Network](network/README.md) - P2P network communication

### Standards (BRC)
- [BRC-42](brc-42/README.md) - Key derivation protocol
- [BRC-43](brc-43/README.md) - Wallet security levels
- [BRC-29](brc-29/README.md) - Payment protocol
- [BRC-56](brc-56/README.md) - Certificate management

### Data Management
- [Encryption](encryption/README.md) - Data encryption utilities
- [Message Signing](message-signing/README.md) - Message authentication
- [OP_RETURN](op-return/README.md) - Data storage on-chain

## How to Use

Each component directory contains:
1. `README.md` - Component overview and API
2. `examples.md` - Code examples and patterns
3. `api-reference.md` - Detailed API documentation

Components can be referenced in learning paths like this:
```markdown
For more details on transactions, see [Transaction Component](../../sdk-components/transaction/README.md)
```

## Component Status

- ✅ Core components documented
- 🚧 Additional components in progress
- 📋 Planned components

## Contributing

When adding new components:
1. Create a dedicated folder with descriptive name
2. Include README.md with overview
3. Add examples.md with practical code
4. Document all public APIs
5. Link related components
