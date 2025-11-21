# Beginner Learning Path

Welcome to the Beginner Learning Path for BSV blockchain development! This path is designed for developers who are new to BSV or blockchain development in general.

## What You'll Learn

By completing this learning path, you will:
- Understand BSV blockchain fundamentals
- Set up a complete development environment
- Create and manage wallets
- Build your first transactions
- Interact with the BSV network

## Prerequisites

- Basic programming knowledge (JavaScript/TypeScript)
- Familiarity with command line interfaces
- Understanding of basic cryptography concepts (helpful but not required)

## Learning Modules

### 1. [Getting Started](getting-started/README.md)
**Duration**: 30 minutes

Introduction to BSV blockchain and the BSV Code Academy structure.

**Topics**:
- What is BSV blockchain?
- Key differences from other blockchains
- Use cases and applications
- Academy navigation guide

### 2. [Development Environment](development-environment/README.md)
**Duration**: 45 minutes

Set up your complete development environment for BSV development.

**Topics**:
- Installing Node.js and npm
- Installing the BSV TypeScript SDK
- Setting up your IDE
- Installing helpful tools
- Creating your first project

**Code Features Used**:
- Project initialization templates

### 3. [Development Paradigms](development-paradigms/README.md)
**Duration**: 30 minutes

Understand the two fundamental approaches to BSV application development.

**Topics**:
- Backend/service development (custodial)
- Frontend integration development (non-custodial)
- When to use each paradigm
- Security considerations

### 4. [BSV Fundamentals](bsv-fundamentals/README.md)
**Duration**: 1 hour

Learn the core concepts of BSV blockchain.

**Topics**:
- Transactions and UTXOs
- Public/private key cryptography
- Addresses and scripts
- Satoshis and denominations
- Block structure and confirmations
- SPV and blockchain verification

**SDK Components Referenced**:
- [Transaction](../../sdk-components/transaction/README.md)
- [Private Keys](../../sdk-components/private-keys/README.md)
- [Script](../../sdk-components/script/README.md)

### 5. [Your First Wallet](first-wallet/README.md)
**Duration**: 45 minutes

Create and manage a BSV wallet with the TypeScript SDK.

**Topics**:
- Generating private keys
- Creating wallet addresses
- HD wallet structure
- Backing up and restoring wallets
- Security best practices

**Code Features Used**:
- [Generate Private Key](../../code-features/generate-private-key/README.md)
- [Create Wallet](../../code-features/create-wallet/README.md)

**SDK Components Referenced**:
- [Private Keys](../../sdk-components/private-keys/README.md)
- [HD Wallets](../../sdk-components/hd-wallets/README.md)

### 6. [Your First Transaction](first-transaction/README.md)
**Duration**: 1 hour

Build, sign, and broadcast your first BSV transaction.

**Topics**:
- Understanding transaction structure
- Creating transaction inputs and outputs
- Calculating fees
- Signing transactions
- Broadcasting to the network
- Monitoring confirmations

**Code Features Used**:
- [Transaction Creation](../../code-features/transaction-creation/README.md)
- [Sign Transaction](../../code-features/sign-transaction/README.md)
- [Broadcast with ARC](../../code-features/broadcast-arc/README.md)

**SDK Components Referenced**:
- [Transaction](../../sdk-components/transaction/README.md)
- [P2PKH](../../sdk-components/p2pkh/README.md)
- [ARC](../../sdk-components/arc/README.md)

### 7. [Wallet Client Integration](wallet-client-integration/README.md)
**Duration**: 45 minutes

Integrate MetaNet Desktop Wallet into your frontend applications.

**Topics**:
- WalletClient basics
- Connecting to user wallets
- Requesting transactions
- Message signing
- Error handling

**Code Features Used**:
- [Create Wallet](../../code-features/create-wallet/README.md)

**SDK Components Referenced**:
- [HD Wallets](../../sdk-components/hd-wallets/README.md)
- [BRC-42](../../sdk-components/brc-42/README.md)

## Hands-On Projects

### Project 1: Simple Payment App
Build a basic application that can:
- Generate a wallet
- Display balance
- Send payments to other addresses
- View transaction history

### Project 2: Testnet Faucet User
Practice on testnet:
- Get testnet coins from a faucet
- Create multiple addresses
- Send transactions between them
- Track confirmations

## Learning Path Completion

After completing this path, you should be able to:
- ✅ Set up a BSV development environment
- ✅ Generate and manage wallets
- ✅ Create basic P2PKH transactions
- ✅ Broadcast transactions to the network
- ✅ Understand BSV blockchain fundamentals

## Next Steps

Ready for more? Continue to the [Intermediate Learning Path](../intermediate/README.md) to learn about:
- Advanced transaction types
- Smart contracts and custom scripts
- BRC standards implementation
- SPV and blockchain verification
- Overlay networks

## Getting Help

- Check the [SDK Components](../../sdk-components/README.md) for detailed API reference
- Browse [Code Features](../../code-features/README.md) for more examples
- Visit [BSV Skills Center](https://hub.bsvblockchain.org/bsv-skills-center) for additional resources

## Estimated Time

**Total Path Duration**: ~5 hours

This includes reading, hands-on coding, and completing the projects. Take your time and practice each concept before moving forward.

---

Ready to begin? Start with [Getting Started](getting-started/README.md)!
